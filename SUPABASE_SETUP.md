# Настройка Supabase для калькулятора стеллажей

Калькулятор хранит общие настройки цен/размеров и сохранённые расчёты в базе данных Supabase (Postgres), чтобы они были одинаковыми для всех, кто открывает страницу — а не только в вашем браузере, как раньше.

## 1. Создать проект Supabase

1. Зайдите на https://supabase.com и войдите (можно через GitHub).
2. Нажмите **New Project**.
3. Укажите название (например `rack-calculator`), задайте пароль базы данных (сохраните его — понадобится для тонкой настройки, но не для самого калькулятора), выберите ближайший регион.
4. Нажмите **Create new project** и подождите 1–2 минуты, пока проект поднимется.

## 2. Выполнить SQL-схему

1. В боковом меню откройте **SQL Editor** → **New query**.
2. Вставьте и выполните (Run) следующий скрипт целиком:

```sql
create extension if not exists pgcrypto;

create table rack_settings (
  id int primary key default 1,
  data jsonb not null,
  updated_at timestamptz not null default now(),
  constraint single_row check (id = 1)
);

create table rack_calculations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  data jsonb not null,
  created_at timestamptz not null default now()
);

alter table rack_settings enable row level security;
alter table rack_calculations enable row level security;

-- Настройки цен: читать могут все, менять — только вошедший админ
create policy "settings_public_read" on rack_settings
  for select using (true);
create policy "settings_admin_insert" on rack_settings
  for insert with check (auth.role() = 'authenticated');
create policy "settings_admin_update" on rack_settings
  for update using (auth.role() = 'authenticated');

-- Сохранённые расчёты: сохранять может любой, удалять/менять — только админ
create policy "calc_public_read" on rack_calculations
  for select using (true);
create policy "calc_public_insert" on rack_calculations
  for insert with check (true);
create policy "calc_admin_update" on rack_calculations
  for update using (auth.role() = 'authenticated');
create policy "calc_admin_delete" on rack_calculations
  for delete using (auth.role() = 'authenticated');
```

## 3. Создать логин администратора (для редактирования цен)

1. В боковом меню откройте **Authentication** → **Users** → **Add user** → **Create new user**.
2. Укажите email и пароль, которым будете входить в калькулятор для редактирования цен.
3. Включите **Auto Confirm User**, чтобы не пришлось подтверждать почту.
4. Сохраните — это единственный логин/пароль, дайте его только тем, кто должен менять цены.

## 4. Забрать ключи проекта

1. Откройте **Project Settings** (шестерёнка) → **API**.
2. Скопируйте:
   - **Project URL** (например `https://xxxxxxxx.supabase.co`)
   - **anon public** ключ (длинная строка в разделе Project API keys)
3. Пришлите мне эти два значения — я вставлю их в код (`SUPABASE_URL` и `SUPABASE_ANON_KEY` в начале `<script>` в `index.html`) и включу синхронизацию.

Эти ключи безопасно использовать в клиентском коде — защита данных обеспечивается не секретностью ключа, а политиками доступа (RLS), которые мы включили в шаге 2: настройки цен и удаление расчётов доступны только вошедшему администратору, а просмотр и сохранение своих расчётов — всем.

## 5. Деплой на Vercel (после того как код будет готов)

1. Зайдите на https://vercel.com, войдите через GitHub.
2. **Add New** → **Project** → выберите репозиторий `rack-calculator`.
3. Framework Preset можно оставить **Other** — сборка не нужна, это статический файл.
4. Нажмите **Deploy**. Через минуту получите постоянную ссылку вида `rack-calculator.vercel.app`.
5. При каждом пуше в `main` Vercel будет обновлять сайт автоматически.
