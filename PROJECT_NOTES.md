# PVID Gauge Explorer — Project Notes & Design Decisions

*A companion to the code: the "why" behind the choices, the things to remember,
and how to do the recurring tasks. Written for future-you and the next
maintainer.*

Current version: **Beta Version 2c**

---

## 1. What this is

An interactive web tool for monitoring irrigation flow across the
Nambé–Pojoaque–Tesuque (NPT) basin — the Pojoaque Valley Irrigation District
(PVID) acequia network — against the backdrop of the Aamodt 1978 water-rights
decree. It shows, per gauge:

- **Flow (cfs)** — instantaneous discharge at each acequia headgate.
- **Total (AF)** — cumulative acre-feet delivered since the window start.
- **TotalAF/acre** — cumulative acre-feet per irrigable acre (a fair
  cross-acequia comparison), available in Adjust mode over a Season or All window.

Its most important capability is **plan-versus-actual**: overlaying each cycle's
*scheduled* diversions (pastel bands) against the *measured* gauge readings, so
anyone can see whether acequias are following the schedule.

---

## 2. The architecture in one picture

Two halves that never run at the same time or place, communicating only through
CSV files in the repo:

- **The builder** — Python (`fetch_ose_gauges.py`) run on a schedule by a
  GitHub Action, *on GitHub's servers*. Its only job is to fetch data and write
  CSV files back into the repository. This is build-time, not visit-time.
- **The website** — `index.html` (the explorer): HTML/CSS/JavaScript that runs
  *in the visitor's browser*. It fetches CSV files and does all computation
  locally.

GitHub Pages is a **static file host** — it serves files exactly as they sit in
the repo; it does **not** run Python when a visitor loads the page. This
"static site + separate scheduled job that regenerates the files" pattern is
common (sometimes called Jamstack).

**How a visitor's data loads:** the browser downloads `index.html` first, then
fetches data *on demand*. The current-year wide CSV, `schedules.json`, the
season file, and the current-year acres file load on arrival. A prior year's
file is **not** downloaded until the user selects that year — that click causes
a fresh fetch from GitHub. (This is why prior-year switching needs the page
*served*, and why it can't work from a double-clicked `file://` page.) All data
is public — anyone with the URL can read any CSV, which is appropriate for
public water-measurement data.

---

## 3. The files

**Builder / data-prep (Python, run by you or the Action — never by visitors):**

- `fetch_ose_gauges.py` — scrapes the 21 OSE gauges and fetches the USGS dam
  gauge; writes per-gauge logs in `data/by_gauge/` and the combined wide table.
- `backfill_from_csv.py` — imports hand-downloaded OSE history. No `--year`
  merges into the current-year data; `--year YYYY` builds a standalone prior-year
  file. Both also pull the dam from USGS.
- `schedule_from_pdf.py` — converts a PVID cycle PDF into a schedule CSV and
  updates the schedules manifest.
- `launchFromMyPC.py` — local launcher/server for testing (serves the page so
  fetches work, unlike opening the file directly).
- `fetch-gauges.yml` — the GitHub Action (schedule lives in its `cron:` line).

**The website:**

- `npt-flow-explorer.html` (deployed as `index.html`) — the whole explorer, one
  self-contained file.

**Data (in `data/`), the only thing the two halves share:**

- `discharge_cfs_wide.csv` — live current-year readings, one column per gauge.
- `discharge_cfs_wide_YYYY.csv` — completed prior years.
- `by_gauge/*.csv` — per-gauge logs (used to rebuild the live wide table).
- `schedules.json` — manifest of available cycles (drives the SCHEDULES buttons).
- `YYYYcycleN.csv` — one irrigation cycle's schedule (`acequia,cfs,start,end`).
- `season_start_and_end_dates.csv` — per-year irrigation-season start/end.
- `YYYYirrigableAcres.csv` — per-year irrigable-acre denominators for TotalAF/acre.

---

## 4. Key design decisions (and why)

**Static site + scheduled builder.** Keeps hosting free and simple, and cleanly
separates "prepare the data" from "show the data." The explorer needs no server
of its own.

**The explorer is data-driven from the CSV header.** It builds its gauge list
from whatever columns exist in the wide file. Consequence: adding a gauge (like
the dam) required almost no explorer change — the new column just appears. Gauge
order in the file controls legend order ("upstream on top").

**Wide CSV format.** One `timestamp` column plus one column per gauge, on a
union time-grid (empty cells where a gauge has no reading at that timestamp).
Timestamps are local Mountain wall-clock time, `YYYY-MM-DD HH:MM`, matching how
all sources report — so everything lines up on one axis.

**"Human verifies at the boundary."** Wherever a machine could confidently
produce a *wrong* answer, the design surfaces it for human review instead of
guessing: the schedule converter flags names it can't map and dates that look
wrong; the explorer validates cycle files and shows a red warning; live-fetch
values are confirmed by you on your own machine before committing. In an
enforcement tool, a confidently-wrong number is the dangerous failure mode.

**Adjust mode — derived series, computed in the browser; the CSV is never
modified.**
- `NambePuebloNet = HighLine + UpperConsol − Nueva − Llano − Comunidad`
- `ConsolNet = UpperConsol − Nueva − Llano − Comunidad` *(revised from an
  earlier Upper − Lower definition as understanding of the interconnections
  improved)*
- `IndiosAdj = 1.0724 × (Indios − 0.135)`, floored at 0 (a gauge zero-offset
  correction)
- Negative net values are **legitimate** — ungauged channels interconnect some
  upstream acequias, and it takes about an hour for water to flow from Upper to
  Lower Consolidated, so a net can genuinely go negative. This is why schedule
  editing needs human judgment and why the y-axis-min control exists (to clip
  the confusing-but-real negatives when showing others).

**Schedules as an underlay.** Each cycle's scheduled diversions are drawn as
pastel bands *behind* the actual readings. In cfs the band is a step function;
in AF it's the cumulative ramp. In Adjust mode the bands do the **same net
arithmetic** as the curves (signed constituents), so ConsolNet's band is Upper's
schedule minus the three ditches'. Emphasis (clicking gauge names) filters the
bands to reduce clutter. Cycle files are validated on load (bad dates, overlaps).

**TotalAF/acre view.** The denominators live in per-year `YYYYirrigableAcres.csv` 
files. The measure is cumulative AF ÷ irrigable acres. The button's hover popup
shows the file's provenance comment lines *and* the acres used, so a wrong
denominator is visible, not buried.

**Season default window.** A season file, season_start_and_end_dates.csv, clips
the default axis to the irrigation season for each year (now Apr 1–Oct 31), 
because the full record — including USGS dam data from January 1 — opens "ugly."
"All" still reveals the entire record. The season start is expected to move
earlier in the future as the climate warms; it's a one-line edit per year.

**The USGS dam gauge.** Added as an upstream reference curve (no Adjust
arithmetic, no schedule). It comes from a *different* source than the OSE gauges
— the USGS instantaneous-values web service, clean JSON/RDB — and slots in as
the first wide-table column.

**Discovery mechanisms.** Prior years are found by *probing* (`2011`..last year).
Cycles, seasons, and acres use small **manifest / per-year files** instead,
because they're keyed by more than a year and probing would be clumsy.

---

## 5. Things to remember / known caveats

- **GitHub's scheduler is slow and unreliable** — real intervals stretch to
  hours, and it can skip runs. Fine for historical/analysis use; **not** fine
  for real-time management, where 20–30 minute freshness is wanted. Moving the
  fetcher to an always-on host (a small VPS, or cPanel cron if the eventual host
  supports Python) is a *where-it-runs* change only — the script, CSVs, and
  explorer are unchanged.
- **The real cadence ceiling is OSE's servers**, which are already unstable.
  Confirm an acceptable request frequency with OSE before increasing it — a
  cadence they've blessed won't get throttled or blocked.
- **USGS is retiring `waterservices.usgs.gov` in early 2027**, moving to
  `api.waterdata.usgs.gov`. The switch is a one-line change at the `USGS_IV_BASE`
  constant in `fetch_ose_gauges.py` (and the parser only if the format changes).
- **The irrigable-acre numbers are placeholders** pending real figures from
  Rob H or OSE. The tool faithfully divides by whatever's in the file. Could
  be TBI acres in the future.
- **`Consolidated - PVID`** and the Highline/Consolidated split are handled by
  human judgment, not code — the converter flags them for you to resolve.
- **`file://` limitations:** opened by double-clicking, the page can't fetch its
  data (auto-load, prior years, schedules, seasons, acres all need the page
  *served*, via `launchFromMyPC.py` locally or GitHub Pages / a real host).

---

## 6. Recurring tasks (quick reference)

- **Add a cycle schedule:** `pip install pdfplumber`, then
  `python schedule_from_pdf.py <cycle>.pdf`. Review the CSV (fix any flagged
  names/dates), copy it and `schedules.json` into `data/`, commit.
- **Add a prior year:** download that year's 21 OSE gauge CSVs into a folder,
  `python backfill_from_csv.py --year YYYY --dir <folder>` (the dam comes from
  USGS automatically), check the output, copy the year file into `data/`, commit.
- **Refresh the current year's data by hand:** `python backfill_from_csv.py`
  (no `--year`).
- **Update irrigable acres:** edit `data/YYYYirrigableAcres.csv` (canonical
  names; comment lines at the top show up in the popup). No code change.
- **Change a season's dates:** edit one line in
  `data/season_start_and_end_dates.csv`. No code change.
- **Speed up live updates later:** run the same `fetch_ose_gauges.py` on an
  always-on host via cron at the OSE-approved interval; point the explorer at
  the same `data/` folder. Nothing else changes.
- **End of year chores?** Greg should remember to ask Claude for advice when
  the December 31 -- January 1 transition occurs.  Maybe Claude already 
  planned for this, archiving current year "wide data file" into standard
  prior-year format.

---

## 7. Working style that served this project

- The CSVs are the source of truth; the explorer never modifies them — every
  transformation (nets, per-acre, schedule arithmetic) happens in the browser at
  render time, so raw data is always recoverable.
- Small, reversible changes, each verified before moving on.
- Honesty about limits: when a value couldn't be confirmed (e.g. a live fetch
  the sandbox couldn't reach), it was flagged for human check rather than
  asserted.
- Keep a stable known-good version (e.g. `...Rev1-final.html`) while a new
  revision is in progress.
