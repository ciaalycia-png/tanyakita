# tanyakita
Tanya apa saja yang ingin kamu ketahui
-- Jalankan seluruh file ini di Supabase: SQL Editor > New query > Run

-- ===== Profil pengguna (nama tampilan) =====
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  display_name text not null,
  created_at timestamptz not null default now()
);

create function public.handle_new_user() returns trigger
language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, display_name)
  values (
    new.id,
    coalesce(nullif(trim(new.raw_user_meta_data->>'display_name'), ''), split_part(new.email, '@', 1))
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();

-- ===== Pertanyaan & jawaban =====
create table public.questions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.profiles(id) on delete cascade,
  title text not null check (char_length(title) between 10 and 150),
  body text not null check (char_length(body) >= 20),
  image_path text,
  image_alt text,
  created_at timestamptz not null default now()
);

create table public.answers (
  id uuid primary key default gen_random_uuid(),
  question_id uuid not null references public.questions(id) on delete cascade,
  user_id uuid not null references public.profiles(id) on delete cascade,
  body text not null check (char_length(body) >= 5),
  created_at timestamptz not null default now()
);

create index on public.questions (created_at desc);
create index on public.answers (question_id, created_at);

-- ===== Row Level Security =====
alter table public.profiles  enable row level security;
alter table public.questions enable row level security;
alter table public.answers   enable row level security;

create policy "profiles dapat dibaca semua" on public.profiles for select using (true);
create policy "ubah profil sendiri" on public.profiles for update using (auth.uid() = id);

create policy "pertanyaan dapat dibaca semua" on public.questions for select using (true);
create policy "buat pertanyaan sendiri" on public.questions for insert to authenticated with check (auth.uid() = user_id);
create policy "hapus pertanyaan sendiri" on public.questions for delete to authenticated using (auth.uid() = user_id);

create policy "jawaban dapat dibaca semua" on public.answers for select using (true);
create policy "buat jawaban sendiri" on public.answers for insert to authenticated with check (auth.uid() = user_id);
create policy "hapus jawaban sendiri" on public.answers for delete to authenticated using (auth.uid() = user_id);

-- ===== Storage untuk foto pertanyaan =====
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('question-images', 'question-images', true, 2097152,
        array['image/jpeg', 'image/png', 'image/webp', 'image/gif'])
on conflict (id) do nothing;

create policy "foto dapat dilihat semua" on storage.objects
  for select using (bucket_id = 'question-images');

create policy "unggah foto ke folder sendiri" on storage.objects
  for insert to authenticated
  with check (bucket_id = 'question-images' and (storage.foldername(name))[1] = auth.uid()::text);

create policy "hapus foto sendiri" on storage.objects
  for delete to authenticated
  using (bucket_id = 'question-images' and (storage.foldername(name))[1] = auth.uid()::text);
