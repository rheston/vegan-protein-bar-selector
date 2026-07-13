# Backend setup (soyboysrus.com)

The site is a single static `index.html` (works on GitHub Pages as-is). Two features need
free external services. Until you fill in the keys in the `CONFIG` block near the top of the
`<script>` in `index.html`, the site still works:

- **Add-a-Bar form** → submitted bars appear in that visitor's rankings for their session only.
- **Feedback form** → shows "not configured yet".

Fill in `CONFIG` to make submissions global and feedback email you.

```js
const CONFIG = {
    SUPABASE_URL: '',        // Supabase → Project Settings → API → Project URL
    SUPABASE_ANON_KEY: '',   // Supabase → Project Settings → API → anon public key
    WEB3FORMS_KEY: ''        // web3forms.com access key
};
```

---

## 1. Global bar submissions — Supabase (free)

1. Create an account at https://supabase.com and make a new project.
2. Open the **SQL Editor** and run:

   ```sql
   create table public.bars (
     id uuid primary key default gen_random_uuid(),
     created_at timestamptz default now(),
     name text not null,
     region text not null default 'US',
     unit_price numeric not null,
     calories numeric not null,
     protein numeric not null,
     sugar numeric not null,
     net_carbs numeric not null,
     sweetener text,
     url text
   );

   alter table public.bars enable row level security;

   -- anyone can read submitted bars
   create policy "public read" on public.bars
     for select using (true);

   -- anyone can submit a bar
   create policy "public insert" on public.bars
     for insert with check (true);
   ```

3. Go to **Project Settings → API** and copy the **Project URL** and the **anon public** key
   into `CONFIG.SUPABASE_URL` / `CONFIG.SUPABASE_ANON_KEY`.
   (The anon key is designed to be public/embedded in the page — that's fine with the RLS policies above.)

### Moderation (recommended)
Public insert means anyone can add a bar, so spam is possible. Two easy guards:

- **Review before showing:** add an `approved boolean default false` column, change the read
  policy to `for select using (approved = true)`, and flip `approved` to true yourself in the
  Supabase table editor. (You'd remove the "appears immediately" behavior for other visitors.)
- Bar **names/URLs are HTML-escaped** in the app already, and only `http(s)` links are allowed,
  so submissions can't inject scripts.

---

## 2. Feedback form — Web3Forms (free, no dashboard)

1. Go to https://web3forms.com, enter **roxanne.heston@gmail.com**, and it emails you an
   **access key**.
2. Paste it into `CONFIG.WEB3FORMS_KEY`.

Submissions then arrive in your inbox. (Swap for Formspree/Netlify Forms later if you prefer.)

---

## Deploying to GitHub Pages
- Repo → **Settings → Pages** → deploy from `main` / root.
- Point `soyboysrus.com` at it: add a `CNAME` file containing `soyboysrus.com` and set the DNS
  records GitHub shows you under Pages → Custom domain.
