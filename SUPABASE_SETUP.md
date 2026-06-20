# Move Free Logistics — Dashboard & Live Tracking Setup (Supabase)

This adds a real **operator login + dashboard** and **live tracking** without changing your hosting. Your site stays on GitHub Pages; it just talks to a free Supabase database in the browser.

You only do this **once**, and it takes about 5–10 minutes.

---

## Step 1 — Create a free Supabase project
1. Go to **https://supabase.com** → sign up (free) → **New project**.
2. Give it a name (e.g. `move-free-logistics`), set a strong **database password** (save it somewhere), pick a region near your clients, and create it.
3. Wait ~1 minute for it to finish provisioning.

## Step 2 — Create the database table + rules
1. In your project, open **SQL Editor** (left sidebar) → **New query**.
2. Paste **all** of the SQL below and click **Run**.

```sql
-- Shipments table -----------------------------------------------------
create table if not exists public.shipments (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  client_name text,
  client_phone text,
  client_address text,
  piano_type text,
  service text,
  origin text,
  destination text,
  status text not null default 'Preparing for Shipping',
  estimated_delivery date,
  current_location text,
  events jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- Row Level Security --------------------------------------------------
alter table public.shipments enable row level security;

-- Only a logged-in operator can read/write the full table:
drop policy if exists "operator full access" on public.shipments;
create policy "operator full access" on public.shipments
  for all to authenticated using (true) with check (true);

-- Public tracking: clients can look up ONE shipment by its code,
-- and only see shipping fields (never the client's name/phone/address).
create or replace function public.track_shipment(p_code text)
returns table (
  code text, status text, piano_type text, service text,
  origin text, destination text, estimated_delivery date,
  current_location text, events jsonb, updated_at timestamptz
)
language sql
security definer
set search_path = public
as $$
  select code, status, piano_type, service, origin, destination,
         estimated_delivery, current_location, events, updated_at
  from public.shipments
  where upper(code) = upper(trim(p_code))
  limit 1;
$$;

grant execute on function public.track_shipment(text) to anon, authenticated;
```

## Step 3 — Create the operator's login
1. Left sidebar → **Authentication** → **Users** → **Add user** → **Create new user**.
2. Enter the operator's **email** and a **password**, and tick **Auto Confirm User**.
3. (Optional) Authentication → **Providers** → Email → turn **OFF** "Allow new users to sign up", so only you can add operator accounts.

## Step 4 — Connect your site
1. Project Settings (gear icon) → **API**. Copy:
   - **Project URL**
   - **anon public** key
2. In your GitHub repo, edit **`config.js`** (pencil icon → paste the two values → Commit):
   ```js
   window.SUPA_URL  = "https://YOURPROJECT.supabase.co";
   window.SUPA_ANON = "eyJhbGciOi...your anon key...";
   ```
3. Wait ~1 minute for GitHub Pages to rebuild.

## Done ✅
- **Operator dashboard:** `https://movefreelogistics.online/admin.html` — log in with the email/password from Step 3. Register clients (a tracking code is generated automatically), and update each shipment's status (Preparing for Shipping → Shipped → At Customs → Delivered).
- **Clients track at:** `https://movefreelogistics.online/track.html` using the generated code.

---

### Notes
- The **anon key is public by design** — the database rules (Step 2) are what keep data safe. Clients can only fetch one shipment by code and never see personal contact details; only a logged-in operator can list or edit anything.
- To add more operator accounts later, repeat **Step 3**.
- Before configuring Supabase, the tracking page keeps working from `tracking.json` (the sample data), so the site is never broken.
