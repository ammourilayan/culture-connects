-- Culture Connects · Teachers' Lounge setup
-- Paste this whole file into Supabase → SQL Editor → New query, then press Run.
-- Safe to run more than once.

-- 1) Tables -------------------------------------------------------------

create table if not exists public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  name text not null check (char_length(name) between 1 and 60),
  created_at timestamptz not null default now()
);

create table if not exists public.lessons (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  author uuid not null default auth.uid() references auth.users(id) on delete cascade,
  country text not null check (char_length(country) <= 60),
  grade text check (char_length(grade) <= 20),
  title text not null check (char_length(title) between 1 and 120),
  summary text check (char_length(summary) <= 1200),
  link text check (link is null or link = '' or link ~* '^https?://'),
  asset_path text check (char_length(asset_path) <= 300),
  asset_name text check (char_length(asset_name) <= 120)
);

create table if not exists public.posts (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  author uuid not null default auth.uid() references auth.users(id) on delete cascade,
  channel text not null default 'general' check (char_length(channel) <= 60),
  text text not null check (char_length(text) between 1 and 1500),
  parent uuid references public.posts(id) on delete cascade
);

create table if not exists public.admins (
  user_id uuid primary key references auth.users(id) on delete cascade
);

create index if not exists lessons_created_idx on public.lessons (created_at desc);
create index if not exists posts_created_idx on public.posts (created_at desc);
create index if not exists posts_parent_idx on public.posts (parent);

-- 2) Who is a moderator? -----------------------------------------------

create or replace function public.is_admin() returns boolean
language sql stable security definer set search_path = public as $$
  select exists (select 1 from public.admins where user_id = auth.uid());
$$;
grant execute on function public.is_admin() to anon, authenticated;

-- 3) Security rules: anyone can read, signed-in teachers can post,
--    people can delete their own posts, moderators can delete anything.

alter table public.profiles enable row level security;
alter table public.lessons  enable row level security;
alter table public.posts    enable row level security;
alter table public.admins   enable row level security;

drop policy if exists "profiles read"   on public.profiles;
drop policy if exists "profiles insert" on public.profiles;
drop policy if exists "profiles update" on public.profiles;
create policy "profiles read"   on public.profiles for select using (true);
create policy "profiles insert" on public.profiles for insert to authenticated with check (id = auth.uid());
create policy "profiles update" on public.profiles for update to authenticated using (id = auth.uid()) with check (id = auth.uid());

drop policy if exists "lessons read"   on public.lessons;
drop policy if exists "lessons insert" on public.lessons;
drop policy if exists "lessons delete" on public.lessons;
create policy "lessons read"   on public.lessons for select using (true);
create policy "lessons insert" on public.lessons for insert to authenticated with check (author = auth.uid());
create policy "lessons delete" on public.lessons for delete to authenticated using (author = auth.uid() or public.is_admin());

drop policy if exists "posts read"   on public.posts;
drop policy if exists "posts insert" on public.posts;
drop policy if exists "posts delete" on public.posts;
create policy "posts read"   on public.posts for select using (true);
create policy "posts insert" on public.posts for insert to authenticated with check (author = auth.uid());
create policy "posts delete" on public.posts for delete to authenticated using (author = auth.uid() or public.is_admin());

drop policy if exists "admins read own" on public.admins;
create policy "admins read own" on public.admins for select to authenticated using (user_id = auth.uid());

-- 4) File uploads (PDFs, images, text up to 20 MB) ----------------------

insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('lesson-files', 'lesson-files', true, 20971520,
        array['application/pdf','image/png','image/jpeg','image/gif','image/webp','text/plain','text/markdown','text/csv'])
on conflict (id) do update set public = true, file_size_limit = excluded.file_size_limit, allowed_mime_types = excluded.allowed_mime_types;

drop policy if exists "lesson files upload" on storage.objects;
drop policy if exists "lesson files delete" on storage.objects;
create policy "lesson files upload" on storage.objects for insert to authenticated
  with check (bucket_id = 'lesson-files' and (storage.foldername(name))[1] = auth.uid()::text);
create policy "lesson files delete" on storage.objects for delete to authenticated
  using (bucket_id = 'lesson-files' and ((storage.foldername(name))[1] = auth.uid()::text or public.is_admin()));

-- 5) Live updates, so new lessons and posts appear without refreshing ----

do $$ begin
  begin alter publication supabase_realtime add table public.lessons; exception when duplicate_object then null; end;
  begin alter publication supabase_realtime add table public.posts;   exception when duplicate_object then null; end;
end $$;

-- 6) Make yourself a moderator -----------------------------------------
-- After you've signed in on your site once, replace the email below with yours,
-- then run just these two lines:
--
-- insert into public.admins (user_id)
-- select id from auth.users where email = 'YOUR-EMAIL@example.com' on conflict do nothing;
