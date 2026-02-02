# naturebap
NEXT_PUBLIC_SUPABASE_URL=TU_URL_SUPABASE
NEXT_PUBLIC_SUPABASE_ANON_KEY=TU_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY=TU_SERVICE_ROLE_KEY   # solo para server actions/api
ADMIN_EMAIL=tuemail@tudominio.com
WHATSAPP_DEFAULT=593999999999
-- 1) tablas
create table if not exists public.products (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text unique not null,
  price numeric(10,2) not null,
  category text not null,
  description text,
  benefits text[] default '{}',
  usage text,
  ingredients text,
  warnings text,
  tags text[] default '{}', -- ["nuevo","mas_vendido"]
  is_active boolean default true,
  stock int,
  is_sold_out boolean default false,
  images text[] default '{}', -- urls
  created_at timestamp with time zone default now(),
  updated_at timestamp with time zone default now()
);

create table if not exists public.site_settings (
  id int primary key default 1,
  hero_title text default 'Bienestar premium',
  hero_subtitle text default 'Suplementos y productos seleccionados con enfoque en hábitos.',
  hero_cta_text text default 'Ver catálogo',
  hero_cta_link text default '/catalogo',
  banner_text text default 'Envíos a todo el país',
  whatsapp_number text default '593999999999',
  contact_email text default 'tuemail@tudominio.com',
  instagram text,
  tiktok text,
  phone text,
  address text,
  cover_image text, -- url portada
  updated_at timestamp with time zone default now()
);

insert into public.site_settings (id) values (1)
on conflict (id) do nothing;

create table if not exists public.pages (
  slug text primary key, -- "privacidad", "terminos", etc.
  title text not null,
  content text not null,
  updated_at timestamp with time zone default now()
);

insert into public.pages (slug, title, content) values
('privacidad', 'Política de Privacidad', 'Contenido editable...'),
('terminos', 'Términos y Condiciones', 'Contenido editable...'),
('cookies', 'Política de Cookies', 'Contenido editable...'),
('envios', 'Envíos y Entregas', 'Contenido editable...'),
('devoluciones', 'Cambios y Devoluciones', 'Contenido editable...'),
('faq', 'Preguntas Frecuentes', 'Contenido editable...')
on conflict (slug) do nothing;

create table if not exists public.orders (
  id uuid primary key default gen_random_uuid(),
  customer_name text not null,
  city text not null,
  phone text not null,
  address text,
  notes text,
  items jsonb not null, -- [{productId, name, qty, price}]
  total numeric(10,2) not null,
  created_at timestamp with time zone default now()
);

-- 2) RLS
alter table public.products enable row level security;
alter table public.site_settings enable row level security;
alter table public.pages enable row level security;
alter table public.orders enable row level security;

-- 3) políticas: público puede leer, solo admin puede escribir
-- products
create policy "public read products" on public.products
for select using (is_active = true);

create policy "admin write products" on public.products
for all using (auth.jwt() ->> 'email' = current_setting('app.admin_email', true))
with check (auth.jwt() ->> 'email' = current_setting('app.admin_email', true));

-- site_settings (público read)
create policy "public read settings" on public.site_settings
for select using (true);

create policy "admin write settings" on public.site_settings
for all using (auth.jwt() ->> 'email' = current_setting('app.admin_email', true))
with check (auth.jwt() ->> 'email' = current_setting('app.admin_email', true));

-- pages (público read)
create policy "public read pages" on public.pages
for select using (true);

create policy "admin write pages" on public.pages
for all using (auth.jwt() ->> 'email' = current_setting('app.admin_email', true))
with check (auth.jwt() ->> 'email' = current_setting('app.admin_email', true));

-- orders: público puede insertar (crear pedido), admin lee
create policy "public insert orders" on public.orders
for insert with check (true);

create policy "admin read orders" on public.orders
for select using (auth.jwt() ->> 'email' = current_setting('app.admin_email', true));

-- 4) set variable para admin email (se setea desde API con RPC simple)
create or replace function public.set_admin_email(admin_email text)
returns void language plpgsql security definer as $$
begin
  perform set_config('app.admin_email', admin_email, true);
end $$;

