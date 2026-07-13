# Backend setup (soyboysrus.com)

The site is a single static `index.html` (works on GitHub Pages). Two free external services
power the dynamic parts. Everything degrades gracefully until you add the keys in the `CONFIG`
block near the top of the `<script>` in `index.html`:

```js
const CONFIG = {
    SUPABASE_URL: '',        // Supabase → Project Settings → API → Project URL
    SUPABASE_ANON_KEY: '',   // Supabase → Project Settings → API → anon public key
    WEB3FORMS_KEY: ''        // web3forms.com access key (emails you feedback)
};
```

- **Without `SUPABASE_*`:** the site shows the ~94 bars baked into `index.html` as a fallback,
  and submissions only appear for that visitor's session.
- **With `SUPABASE_*`:** **Supabase is the single source of truth for all bars.** The site loads
  every bar from the `bars` table, and submissions are saved there. (The baked-in list stays only
  as an offline fallback if Supabase is ever unreachable — you edit bars in Supabase, not the HTML.)

---

## 1. Supabase — the bar database (free)

### a. Create the project + table
1. Sign up at https://supabase.com and create a new project (pick any region; save the DB password).
2. Left sidebar → **SQL Editor** → **New query**, paste this, and **Run**:

   ```sql
   create table public.bars (
     id uuid primary key default gen_random_uuid(),
     created_at timestamptz default now(),
     name text not null,
     region text not null default 'US',      -- 'US' or 'UK'
     unit_price numeric,
     calories numeric,
     protein numeric,
     sugar numeric,
     net_carbs numeric,
     sweetener text,
     url text,
     image text,
     rating_stars numeric,
     rating_count integer,
     source text not null default 'community' -- 'core' = seeded, 'community' = user-submitted
   );

   alter table public.bars enable row level security;
   create policy "public read"   on public.bars for select using (true);
   create policy "public insert" on public.bars for insert with check (true);
   ```

### b. Load all existing bars
3. Still in the **SQL Editor**, open a new query, paste the **entire contents of `bars_seed.sql`**
   (in this repo — all 94 bars with their stats, images, ratings, sweeteners), and **Run**.
   You can re-run it anytime to reset the core bars; it won't touch community submissions.

### c. Connect the site
4. **Project Settings → API** → copy **Project URL** → `CONFIG.SUPABASE_URL`, and the
   **anon public** key → `CONFIG.SUPABASE_ANON_KEY`. (The anon key is meant to be public in the
   page; the RLS policies above are what keep it safe.)

### Moderation (optional)
Public insert = anyone can submit a bar. If you want to review before they show publicly:
add `approved boolean default false`, change the read policy to
`for select using (approved = true)`, and flip rows to approved in the Table Editor.
(Names/URLs are already HTML-escaped and limited to `http(s)`, so submissions can't inject scripts.)

---

## 2. Navigating Supabase (edit bars like a spreadsheet)

- **Table Editor** (left sidebar → *Table Editor* → **bars**): a spreadsheet grid of every bar.
  - **Edit a value:** double-click a cell, change it, press Enter. It's live on the next page load.
  - **Delete a bar:** check its row → **Delete** (top).
  - **Add a bar:** **Insert → Insert row** (or just use the site's "Add a Bar" form).
  - **Tell core vs submitted apart:** the **`source`** column (`core` = seeded by you,
    `community` = submitted through the site). Filter by it with the **Filter** button.
- **SQL Editor**: run queries/bulk edits (e.g. re-run `bars_seed.sql`, or
  `update public.bars set unit_price = 3.10 where name = 'No Cow';`).
- **Project Settings → API**: where your URL + anon key live.
- **Table Editor → Filter / Sort**: same idea as spreadsheet filters.

---

## 3. Web3Forms — feedback to your inbox (free, no dashboard)

1. Go to https://web3forms.com, enter **roxanne.heston@gmail.com**, click **Create Access Key**.
2. Check your inbox for the **access key** and paste it into `CONFIG.WEB3FORMS_KEY`.

Feedback submissions then arrive in your Gmail.

---

## 4. Hosting on GitHub Pages + soyboysrus.com

1. Push the repo, then repo → **Settings → Pages** → Source: **Deploy from a branch** →
   **main** / **/(root)** → Save.
2. **Custom domain:** the repo's `CNAME` file already contains `soyboysrus.com`. In
   Settings → Pages the domain should appear; once DNS is set (below) it verifies and you can
   tick **Enforce HTTPS**.
3. **DNS at your registrar** (where soyboysrus.com is registered) — this is what fixes the
   "Domain does not resolve to the GitHub Pages server" error. Add:

   **Four A records** for the apex (`@` / soyboysrus.com):
   ```
   185.199.108.153
   185.199.109.153
   185.199.110.153
   185.199.111.153
   ```
   **One CNAME record:** host `www` → value `rheston.github.io`

   Remove any old parking/forwarding A or CNAME records for `@`/`www` that point elsewhere.
4. Wait for propagation (usually minutes, up to a few hours). Verify with:
   `dig soyboysrus.com +short` → should list the four 185.199.108–111.153 IPs.
   Then GitHub auto-issues the HTTPS certificate.
