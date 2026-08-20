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

What staff actually do per item: search-tap a **brand name** (or tap
"unidentified"), tap a condition chip, tap a disposition chip, type a
quantity. Four taps and one number. Everything else is opt-in. Or, skip
typing almost entirely — see Voice dictation below.

## Brand name is the primary search field

Staff usually recognize the brand printed on the packaging before the
generic name, so **Brand name** is the field with the search list,
barcode scan, and "unidentified" toggle attached to it. Search or scan a
brand and the generic **Drug** name auto-fills underneath it (from your
drugs master list) — editable if it needs correcting, and still there to
type into directly for drugs where only the generic name is known (e.g.
nothing branded printed on the packaging).

## Voice dictation (🎤 next to each field)

Every voice-able field — **brand name**, **quantity**, **condition**,
**disposition** — has its own small 🎤 button right beside it. Tap one:
the app speaks that one question out loud ("What's the brand name?"),
listens, and fills just that field. No mode picker, no multi-step
sequence to get lost in — tap whichever field you want to dictate, in
whatever order you like. Saying either the brand or the generic name
works — matching checks both.

Tapping a field's mic again while it's listening cancels it. Tapping
directly into a field (typing) always stops its mic and takes over —
a manual edit is never overridden by voice.

**Live transcript, so you can see it's actually working:** once a mic
is listening, a line appears — *"Hearing: ..."* — updating in real time
as it picks up sound, before you've even finished the sentence. If that
line never appears or never changes while you talk, the mic isn't
capturing audio at all (check the browser's mic permission for the
site) — this is meant to make that failure visible immediately instead
of silently doing nothing. If the mic ever fails to start, the status
line says exactly why instead of quietly doing nothing.

(Also fixed: a bug where the app could silently ignore everything it
heard — mic genuinely listening, but every result thrown away — because
it was checking the browser's own "is it still speaking" flag, which can
get stuck `true` forever on some Chrome builds. That check is now done
with the app's own timer-backed flag instead, so it can't get stuck.)

**Voice quality:** the spoken questions use the most natural-sounding
voice your browser has available (Chrome's network-based voices, where
present) instead of the default robotic OS voice — automatic, no
setting needed.

**Honest note:** the spoken question is a fixed script (always the same
question for that field), and your answer is parsed with plain
keyword/number matching plus a fuzzy match against your drug list for
the drug-name field — not an AI that understands free-form replies. That
was a deliberate choice to keep the app free to run — a model that
actually understands open-ended speech needs a paid API call per use,
and the key for that couldn't be safely embedded in a static page anyone
can view-source anyway.

This means:

- Numbers, and the fixed vocabulary words (sealed/opened/expired/damaged,
  restock/destroy/redistribute/pending) are recognized reliably.
- Drug names work well when the transcription comes out reasonably close
  to correct, and may miss if speech-to-text badly mangles the word —
  medical/drug names are exactly what generic speech engines struggle
  with most. When it can't find a confident match, it leaves the drug
  field for you to fill in manually rather than guessing wrong.
- **Nothing is ever auto-submitted.** Staff still tap Save themselves —
  a deliberate data-quality guardrail, not an oversight.
- **The full raw transcript of every field dictated is always saved as
  text**, in the Notes field (prefixed `[Voice:fieldname]`), regardless
  of how well it parsed — so even if a structured field misses, what was
  actually said isn't lost.
- Browser support: works on Chrome (Android and desktop). I'm not fully
  certain of current support on iPhone Safari — test on your actual
  staff devices before relying on it. If the browser doesn't support it
  at all, the mic buttons hide themselves automatically and the fields
  work normally.

**Free-text dictation for Notes:** the Notes field (in "more detail") has
its own separate `🎤 Dictate` button next to it. Unlike the guided flow
above, this just transcribes whatever you say, continuously, straight
into the box — no parsing, no auto-advance, for anything open-ended that
doesn't fit the structured fields. Tap it to start, tap `🔴 Stop` when
done; it keeps listening (restarting itself automatically if the browser
pauses on silence) until you stop it. Starting either mic automatically
stops the other, so they never fight over the microphone.

Photo and video capture were considered and explicitly left out per your
call — easy to add later if that changes.

## Barcode scanning (📷 button, next to the drug field)

Tap it, point the camera at the barcode on the packaging. Uses a free
camera-only library (no external lookup service, no cost) that reads the
code and checks it against the `barcode` column on your drugs master
list. If it matches, the drug/unit/cost auto-fill exactly like picking it
from the search list. If it doesn't match — because that drug's barcode
isn't in your list yet, or it's a barcode you've never seen — it shows
you the raw scanned number and lets you search/type manually instead.
Scanning never blocks entry; it's a shortcut when it works, not a
requirement.

To make more barcodes recognized over time: add the barcode value to the
`barcode` column in `drugs_template.csv` (or directly in Supabase's Table
Editor) whenever you notice a drug you scan often isn't matching yet.

**Honest limitation:** this only recognizes barcodes you've told it
about — there's no free public database of Malaysian pharmaceutical
barcodes to draw from, so recognition is entirely built from what you
add. It gets more useful over time as your list grows, not on day one.

## Units — dropdown, remembers what you last used per drug

The unit field is now a dropdown of standard pharmacy units (tablet,
capsule, strip, bottle, vial, sachet, etc.), with an "Other" option for
anything not listed. Selecting a drug (by search, scan, or voice)
defaults the unit to the master list's default for that drug — but once
staff have corrected it for a specific drug (e.g. this one usually comes
back as loose strips, not full bottles), the app remembers that
correction on that device and defaults to it next time, while still
letting you change it. Nothing is enforced; it's just a smarter default.

## Date collected — defaults to today

On the sign-in screen, alongside name/site/bin, there's now a date field
that defaults to today automatically. It only needs changing if you're
logging a bin that was actually collected on an earlier day (e.g.
catching up on a backlog) — otherwise ignore it, it's already right.

## Manufacturer/MAL registration number

Lives in the collapsible "more detail" section since almost nobody has
it memorized — fill it in only if you're actually going to use it for
something; leaving it blank on every entry costs nothing. (Brand name is
covered above — it moved up to be the primary field.)

Both can be pre-filled per-drug via the `drugs` master list — see
`drugs_template.csv`, now with `brand_name` and `mal_registration_no`
columns (leave them blank for drugs where you don't have this).

## Photo (documentation only — not analyzed)

Optional camera capture in the "more detail" section. Tap it, take a
photo, it uploads to Supabase's free storage tier and attaches to that
row as a thumbnail you can review later — purely a human backup, e.g.
for double-checking an odd entry. **Nothing reads text out of the photo
automatically** — that was a deliberate choice: real text-extraction
either costs money per photo (an AI vision API call) or relies on free
OCR that's honestly unreliable on real pharmacy packaging (small print,
curved bottles, glare). If a photo fails to upload for any reason (poor
connection, etc.), the rest of the item still saves — you'll just see a
warning that the photo specifically didn't make it.

## 1. Create the free database (5 minutes)

1. https://supabase.com → sign up → new project (Singapore region is
   closest to Malaysia).
2. **SQL Editor → New query** → paste all of `schema.sql` → Run.
   This creates: `sites` (HCTM/HABTAR, seeded), `bins` (1-10 per site,
   seeded — delete rows for bins that don't physically exist, or add
   more), `drugs` (with a `barcode` column for scanning), and
   `medication_returns` with write access locked to insert-only from the
   browser. (If you already ran an earlier version of this schema, it's
   safe to run again — it only adds what's missing, like the barcode
   column, without touching your existing data.)
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
strength/form, unit, unit cost in RM, category, and optionally a
barcode if you know it — leave that column blank for drugs where you
don't have it, it's entirely optional and just enables barcode-scan
lookup for that drug). Doesn't need to be exhaustive — "type it
manually" always remains possible via the search box even for drugs not
in the list. Then in Supabase: **Table Editor → drugs → Insert → Import
data from CSV** → upload your filled-in file.

No coding needed for this step or to update it later — just re-import
whenever the list needs adding to.

## 5. Staff sign-in — no setup needed

There's no staff pre-registration step anymore (removed per your request
to cut friction further). Anyone opens the app, types their name, picks
their site, picks their bin, and starts logging — no admin step
required before someone new can use it. Name suggestions in that field
grow automatically from whoever has already logged something before.

`staff_assignments` and `staff_template.csv` still exist but are no
longer used by the app — harmless leftovers from an earlier version, safe
to ignore (or delete the table if you want to tidy up).

**Note on "login":** this is free-text name attribution, not a password
system — nothing stops someone from typing a different name, or anyone's
name, for any bin. That's the deliberate trade-off you chose over setup
friction. If it ever becomes a real problem (spam, misattribution),
Supabase Auth can be added later.

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

- Name-based attribution isn't real access control (see step 5).
- Barcode recognition only covers whatever you've manually added to the
  `barcode` column — there's no free public database behind it.
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
