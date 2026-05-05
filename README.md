# VisitorLog PWA — Setup Guide

## Files Included
- `index.html` — The complete app (single file)
- `sw.js` — Service worker (offline support)
- `manifest.json` — PWA manifest (install to home screen)
- `icon-192.png` / `icon-512.png` — App icons (create these yourself or use any PNG)

---

## Hosting (Pick One)

### Option A — Netlify Drop (Zero cost, 30 seconds)
1. Go to https://app.netlify.com/drop
2. Drag the entire `visitor-log/` folder onto the page
3. You get a live HTTPS URL instantly (required for PWA + IndexedDB)

### Option B — GitHub Pages (Free)
1. Create a GitHub repo, push these files
2. Settings → Pages → Deploy from branch `main`

### Option C — Any web server / NAS
Serve the files over HTTPS. A self-signed cert works for local network use.

---

## Database — Supabase Setup (Free, No Server Install)

Supabase is a hosted Postgres database with a REST API — no server to manage,
data lives in the cloud, and you can export CSV at any time.

### 1. Create a free project
- Go to https://supabase.com → New Project (free tier is generous)
- Note your **Project URL** and **anon public key** (Settings → API)

### 2. Create the table
Go to SQL Editor and run:

```sql
CREATE TABLE visits (
  id          uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  name        text NOT NULL,
  soundex     text,
  metaphone   text,
  badge_id    text,
  host        text,
  reason      text,
  notes       text,
  location    text,
  signature   text,          -- base64 PNG of handwritten signature
  time_in     timestamptz DEFAULT now(),
  time_out    timestamptz,
  created_at  timestamptz DEFAULT now()
);

-- Allow public read/write (for reception desk without login)
ALTER TABLE visits ENABLE ROW LEVEL SECURITY;
CREATE POLICY "allow_all" ON visits FOR ALL TO anon USING (true) WITH CHECK (true);
```

### 3. Connect the app
Open the app → Settings tab → enter your URL and anon key → Save → Test Connection.

### Exporting Data (CSV)
- **From the app**: Log tab → CSV button (exports local IndexedDB data)
- **From Supabase**: Dashboard → Table Editor → visits → Export as CSV
- **Via HTTP**: `curl "https://YOUR_PROJECT.supabase.co/rest/v1/visits?select=*" -H "apikey: YOUR_KEY" -H "Authorization: Bearer YOUR_KEY" > visits.csv`
- You can schedule the curl command via cron for automatic daily exports.

---

## How Offline Works

The app uses a **two-layer** approach:

1. **IndexedDB** (browser built-in database) — stores every record locally, immediately.
   Works 100% without internet. All searching, viewing, and CSV export work offline.

2. **Sync Queue** — when a record is saved while offline, it's added to a queue.
   When internet returns, the queue auto-syncs to Supabase in the background.

The sync indicator in the top-right shows:
- 🟢 Green = connected to Supabase
- 🔴 Red = offline or not configured
- 🟡 Yellow blinking = syncing

---

## Phonetic Search

Visitor names are stored with two phonetic codes alongside the real name:

| Code | What it does | Example |
|------|-------------|---------|
| **Soundex** | Encodes consonant sounds (ignores vowels, doubles) | PAPADOPOULOS → P130 |
| **Metaphone** | More accurate phonetic encoding | Nikolaos ≈ Nicolaos |

Searching "Nikolas" will find "Nikolaos", "Nicolaos", "Nikos" etc.
Searching "Papado" will find "Papadopoulos", "Papadopoulou" etc.

Previously seen visitors also appear as autocomplete suggestions ranked by visit count.

---

## Signature

The visitor signs on a **canvas element** using:
- Finger on touchscreen / phone / tablet
- Mouse on desktop
- Stylus on Surface / iPad

The signature is saved as a **base64 PNG** directly in the database.
Click the thumbnail in the log to view full size.

For a **legal-grade** setup, add a disclaimer above the pad:
*"By signing, I confirm I have read and agree to the visitor policy."*

---

## Multi-Device Use

Since everything syncs to Supabase, multiple devices (front desk + security)
can use the same Supabase project simultaneously. Each device maintains its own
local IndexedDB cache.

---

## Scheduled CSV Export via HTTP (Automation)

```bash
# Add to crontab: daily at 23:55
55 23 * * * curl -s "https://YOUR_PROJECT.supabase.co/rest/v1/visits?select=*&order=time_in.desc" \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Authorization: Bearer YOUR_ANON_KEY" \
  -o "/backups/visits_$(date +\%Y-\%m-\%d).csv"
```

---

## Icons

You need two PNG icons for the PWA manifest:
- `icon-192.png` (192×192 px)
- `icon-512.png` (512×512 px)

Create them from any image editor, or use a favicon generator like
https://realfavicongenerator.net — just name the outputs correctly.
