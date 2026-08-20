# Medication Returns Tracker — Setup Guide

Multi-site version for HCTM and HABTAR: anonymous public drop-boxes, numbered
per hospital, sorted later by pharmacy staff. Built to be **free to run
indefinitely**, and tuned for **sub-15-second entries** so staff actually
keep doing it.

Stack: static HTML/JS (`index.html`) + Supabase (free-tier Postgres). No
build step, no server, no password login.

---

## How the design resolves "harvest everything" vs. "bloody simple"

Two things do the work:

1. **A pre-built drug list** (you provide, see step 4) turns entry from
   *typing* into *search-and-tap* — faster AND keeps drug names consistent
   across thousands of rows, which matters a lot once you're trying to
   analyze this for a paper. Selecting a drug also auto-fills unit and
   cost, so nobody types a price per item.
2. **Optional fields stay optional and collapsed** (label data, reason,
   batch/expiry) — they cost nothing on the ~90% of items where they don't
   apply, and add real value on the ones where they do.

What staff actually do per item: search-tap a drug (or tap "unidentified"),
tap a condition chip, tap a disposition chip, type a quantity. Four taps
and one number. Everything else is opt-in. Or, skip typing almost
entirely — see Voice dictation below.

## Voice dictation (🎤 button on the entry form)

Tap the mic, say the whole item in one breath — e.g. *"amlodipine five
milligram, twenty tablets, sealed, restock"* — and the app fills in
drug, quantity, condition, and disposition for you to check and confirm.

**How it works, honestly:** this uses the browser's free built-in
speech-to-text, not an AI service — turning that into "structured data"
would need a paid API call per item, which breaks "free to use," so
instead the transcript is parsed with plain keyword/number matching plus
a fuzzy match against your drug list. This means:

- Numbers, and the fixed vocabulary words (sealed/opened/expired/damaged,
  restock/destroy/redistribute/pending) are recognized reliably.
- Drug names work well when the transcription comes out reasonably close
  to correct, and may miss if speech-to-text badly mangles the word —
  medical/drug names are exactly what generic speech engines struggle
  with most. When it can't find a confident match, it leaves the drug
  field for you to fill in manually rather than guessing wrong.
- **Nothing is ever auto-submitted.** The app always shows what it heard
  and what it parsed, and staff still tap Save themselves — this is a
  deliberate data-quality guardrail, not an oversight.
- **The full raw transcript is always saved as text**, in the Notes
  field (prefixed `[Voice]`), regardless of how well it parsed — so even
  if the structured fields miss something, what was actually said isn't
  lost. The "more detail" section auto-opens after dictating so staff can
  see it landed there.
- Browser support: works on Chrome (Android and desktop). I'm not fully
  certain of current support on iPhone Safari — test on your actual
  staff devices before relying on it. If the browser doesn't support it
  at all, the mic button hides itself automatically and the fields work
  normally.

Photo and video capture were considered and explicitly left out per your
call — easy to add later if that changes.

## 1. Create the free database (5 minutes)

1. https://supabase.com → sign up → new project (Singapore region is
   closest to Malaysia).
2. **SQL Editor → New query** → paste all of `schema.sql` → Run.
   This creates: `sites` (HCTM/HABTAR, seeded), `bins` (1-10 per site,
   seeded — delete rows for bins that don't physically exist, or add
   more), `staff_assignments`, `drugs`, and `medication_returns` with
   write access locked to insert-only from the browser.
3. **Project Settings → API** → copy the **Project URL** and **anon
   public** key.

## 2. Configure the app

Edit these two lines near the top of `index.html`:

```js
const SUPABASE_URL = "REPLACE_WITH_YOUR_SUPABASE_URL";
const SUPABASE_ANON_KEY = "REPLACE_WITH_YOUR_SUPABASE_ANON_KEY";
```

Paste in your values, save.

## 3. Host it for free

- **Netlify Drop**: https://app.netlify.com/drop — drag `index.html` in,
  get a live URL instantly.
- **Vercel**: `npx vercel` in this folder, or drag-and-drop via dashboard.
- **GitHub Pages**: push to a repo, enable Pages.

One URL serves both hospitals — site is picked at sign-in, not baked into
the deployment.

## 4. Fill in the master drug list (do this before go-live)

Open `drugs_template.csv`, fill in your commonly-returned drugs (name,
strength/form, unit, unit cost in RM, category — doesn't need to be
exhaustive, "type it manually" always remains possible via the search
box even for drugs not in the list). Then in Supabase: **Table Editor →
drugs → Insert → Import data from CSV** → upload your filled-in file.

No coding needed for this step or to update it later — just re-import
whenever the list needs adding to.

## 5. Fill in staff assignments

Same idea: fill in `staff_template.csv` with each staff member's name and
which site+bin number(s) they're responsible for (one row per
staff+bin — a staff member covering 3 bins gets 3 rows). Import via
**Table Editor → staff_assignments → Import data from CSV**.

This is what makes the sign-in step fast: staff type their name, the app
shows only their assigned bin(s) to pick from.

**Note on "login":** this is name-based attribution, not a password
system — nothing stops someone from typing a different name. If that
becomes a real problem (spam, deliberate misattribution), Supabase Auth
can be added later; skipped here because you asked for lightweight.

## 6. Check/edit physical bins

`schema.sql` seeds bins 1-10 for both HCTM and HABTAR. If your actual count
differs, edit directly in **Table Editor → bins** — delete rows for bins
that don't exist, or add rows (with a bin_number) for ones that do.

## 7. Using the app

- First open: staff type their name, pick their bin (or it auto-selects
  if they only cover one), tap "Start logging." This is remembered on
  that device until they tap "Switch" in the header — so a phone
  dedicated to one bin only asks once.
- **Log an Item**: repeat as many times as there are items in that bin.
- **Dashboard**: cost/volume by drug, by site, by bin (for your
  HCTM-vs-HABTAR and bin-vs-bin comparison), by condition, by disposition,
  reason breakdown (expect mostly "Unknown," by design), and the
  label-derived leftover-rate/days-since-dispensing stats when that data
  exists. CSV export for stats software.

## 8. Data governance note

Even though no patient is ever identified, check with your institution
whether analysis-for-publication still needs ethics sign-off separately
from the operational QI framing — practice varies, and that's not a call
I can make for you.

## 9. Known limitations

- Name-based attribution isn't real access control (see note in step 5).
- Unidentified/loose items are logged without a drug name by design (per
  your instruction) — so total item counts stay honest, but drug-level
  breakdowns will always exclude whatever fraction of the bin can't be
  identified. Track this via "Unidentified/mixed" in the drug table.
- Label-derived stats (leftover rate, days since dispensing) will only
  ever cover the fraction of items where a label survived — you told me
  that's inconsistent, so treat those numbers as suggestive on a subset,
  not a full picture.
- No edit/delete from the app UI (append-only by design); corrections go
  through Supabase Studio's table editor.
- Free Supabase projects pause after 7 days of inactivity (auto-resumes,
  ~30s wake delay) — a non-issue at daily-use volume.
