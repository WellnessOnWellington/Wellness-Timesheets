# Running Well — working notes

Single-file app: everything is `index.html` (~10,700 lines), plus `sw.js` and
`manifest.json`. No build step, no framework, no npm. Deployed via GitHub Pages
from `main` — **pushing to `main` deploys to live staff immediately.**

Sole record of staff hours since 11 Aug 2026. Feeds MYOB payroll for real
wages. Clinic is in **Melbourne, Australia** (`Australia/Melbourne`, UTC+10/+11)
— the `en-AU` locale throughout is the reliable signal, not the clinic's name.

---

## ⚠ After ANY change to the Supabase schema

Adding, renaming or dropping a table or column affects three things, and only
one of them notices on its own. **Do all three in the same sitting.**

### 1. Re-capture the schema — manual, nothing checks this
Run `Runningwell-Vault/capture_schema.sql` in the Supabase SQL Editor. Replace
`Runningwell-Vault/schema/schema.sql` with the output, commit, push.

### 2. The backup checks itself — but only against step 1
`Runningwell-Vault/backup.py` parses `schema.sql` and **fails with an email** if
it declares a table that `TABLES` does not cover. So once step 1 is done, the
backup tells you if it needs updating.

New *columns* need nothing — the backup does `select=*`.
New *tables* must be added to `TABLES`.

### 3. Run the journal coverage query — the only live check
In `Runningwell-Vault/journal_tables.sql`, the block headed **COVERAGE CHECK**.
Expect **zero rows**. Anything listed has no `trg_journal`, meaning deletions
from it are unrecoverable — silently. New tables also need adding to the array
in that file, then re-run it (safe to re-run; it drops and recreates).

### Why this checklist exists
`schema.sql` is a file in a repo, not something living in Supabase. The
automatic check compares two files in GitHub — **it cannot tell that Supabase
has gained a table nobody wrote down.** In that case the file and the script
agree with each other and are simply both stale, and everything looks green.

Only the journal coverage query reads the live database (`pg_class`).

*Not theoretical: `timesheet_audit` (868 rows) and `public_holidays` existed for
months with neither backup nor journal coverage before a review caught it.*

---

## Data protection currently in place

- **`data_journal`** — append-only triggers on all 15 tables capturing every row
  version. Unreachable with the published anon key (`42501` on read, `401` on
  delete). Makes any wipe recoverable. Setup: `journal_tables.sql`.
- **Daily off-site backup** — private repo `WellnessOnWellington/Runningwell-Vault`,
  12:00 UTC via GitHub Actions. Refuses to write if a table that had rows returns
  zero, or if total rows halve. Failure email verified working.
- **Local snapshot** — `backups/backup_supabase.py`, run by hand any time.

**Pushing to `.github/workflows/` needs the `workflow` token scope**, which the
current PAT lacks. Workflow edits must go through the GitHub web UI; everything
else pushes normally.

---

## Things that will bite you

- **The Supabase anon key is published** in this file and grants full read and
  delete on every table. All PIN checks are client-side and bypassable. See the
  Database Lockdown plan. Remediation is planned, not built.
- **Two `ON DELETE CASCADE` chains**: `employees → entries` and
  `entries → notes`. Deleting one employee row removes **every timesheet they
  ever had**. The app soft-deletes (`active:false`) so it never fires normally —
  a direct API call would. Restore parents before children.
- **`entries.date` is `text`; `roster_shifts.date` is `date`.** Same name,
  different types. Never join or compare them.
- **Two fortnight grids, one day apart.** The roster is Monday-anchored
  (`cycle_start_date`); pay periods are Tuesday-anchored (`PAY_PERIOD_ANCHOR`).
  `fn_start` in `roster_fortnights` is **not** comparable to `fn_start` in
  `timesheet_finalizations`. See the comment above `getFortnightStart()`.
- **The MYOB export dedup key includes `shift_id`.** It will pay out duplicate
  leave rows if any reappear — `rbCleanupLinkedEntryOnShiftDelete` is what
  prevents that. Re-check for duplicates after any bulk roster rebuild.
- **No `pay_period_anchor` column exists.** The value is the `PAY_PERIOD_ANCHOR`
  constant; the Settings editor for it was removed because it always failed.

---

## Conventions

- Match the surrounding code: it is dense, comment-heavy where something is
  non-obvious, and uses no semicolon-free or modern-syntax flourishes.
- All DB access goes through the single `db` object (~line 3229). **Keep it that
  way** — that one choke point is what makes the planned write-proxy days of
  work rather than months.
- Escape user text with `escHtml()` before it reaches `innerHTML`. The Staff
  Portal module does this consistently and is the model.
