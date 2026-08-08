# ==========================================
# FILE: PROJECT_HANDOFF.md
# ==========================================

# RichRoadStockScreenerUS — Project Handoff Summary

**Latest scan: — stocks found on —.** *(updated automatically by
`cb_report.py` after every run - this line will be empty until the
first run after this line was added.)*

This document exists so that any AI assistant (or human) picking this
project up later can understand what has been built, why certain
decisions were made, and what problems were already solved — without
needing to rediscover them from scratch.

The user who owns this project is **not a programmer** and has no prior
GitHub experience. All instructions given to them should assume a
complete-beginner level, including basic GitHub steps (where to click,
what a commit is, etc.).

**This project is one half of a bigger effort.** The sibling repo,
`chart`, is a TradingView-style visual/communication tool built on top
of the data this project produces (see `CHART_HANDOFF.md` there for
details). Notably, the chart project currently calculates several
indicators (EMA 200, volume EMAs, RSI, and various multi-timeframe EMA
overlays) client-side in JavaScript because this pipeline doesn't yet
provide them - the stated long-term direction is to move that
calculation here (most likely into `fetch_stock_timeframes.py`,
alongside where `ema_10/20/50/100` and the turnover EMAs already get
computed) so there's one consistent source of truth. Not started yet.

**Explicitly planned next-session work here** (see `chart`'s
`CHART_HANDOFF.md` Section 0 for the full ordered plan): (1) redo this
pipeline's scan/data-fetch scripts so they produce everything the
chart project currently calculates client-side (EMA 200, RSI(14),
Volume EMA 3/20, ideally the multi-timeframe/relative EMA overlays
too), and (2) work on expanding/improving the chart's watchlist -
likely connecting it to the full daily scan output rather than the
current fixed 11 hand-picked stocks. Both are prerequisites the user
wants solid before moving on to the actual end goal of teaching Claude
their trading-decision logic through the chart tool.

**If the user shares an output file (a scan CSV, a `.db` database, or a
`_PasteableValues.txt` list) in this conversation, cross-reference it
against this document first** -- the column meanings, known quirks
(e.g. blank `monthly_rsi` values, the `history_days`/`full_history`
flags, per-timeframe turnover), and file-naming conventions are all
explained below, and will make sense of things that would otherwise
look like anomalies.

**If a change made during a conversation would make this document (or
`README.md` / `QUICKSTART.md`) inaccurate or out of date, flag that to
the user at the time** -- a brief note is enough (e.g. "heads up, this
also means the timeframes table in `PROJECT_HANDOFF.md` is now out of
date"). Don't rewrite the documentation automatically after every
change -- the user prefers to batch documentation updates and will
explicitly ask for the updated file(s) when ready, typically at the end
of a session.

---

## 1. What this project does, in plain terms

Every US trading day, shortly after market close, an automated process:

1. Scans **every stock on NASDAQ, NYSE, and AMEX** (excluding ETFs and
   OTC-traded stocks) for end-of-day data.
2. Filters that list down to only the stocks matching a specific set of
   technical criteria (see Section 3).
3. Saves the matching stocks to a downloadable spreadsheet (CSV).
4. Exports that same list as a plain, comma-separated text list — handy
   for pasting straight into a watchlist tool (TradingView, StockCharts,
   etc.) without opening a spreadsheet at all.
5. Takes that day's matching stocks and pulls **detailed multi-timeframe
   price history and technical indicators** for each one (daily, weekly,
   monthly, quarterly, hourly, 15-minute, 5-minute), saving it all into a
   single downloadable database file.
6. Looks up each stock's **sector, industry, market cap, and full
   earnings-date history** (past and upcoming), adding this into the
   same database as two more tables.
7. Updates the **CB Report** — a running, day-by-day log of how many
   stocks were found each day, broken down by sector, with an Excel
   table and chart. This log is committed back into the repository
   itself so it persists and accumulates over time (see Section 7).
8. Bundles that day's spreadsheet, database, ticker list, CB Report,
   all documentation, and all the code itself into **one single
   downloadable zip file**, named with that day's date (see Section 8).

This all runs automatically via **GitHub Actions** (no server or local
computer needs to be running), and can also be triggered manually at
any time.

---

## 2. Repository structure

| File | Purpose |
|---|---|
| `scan.py` | The main daily stock screener. Builds the stock universe, applies the search criteria, outputs a CSV. |
| `fetch_stock_timeframes.py` | Runs after `scan.py` in the same workflow. Takes that day's scan results and fetches detailed multi-timeframe data + indicators into a SQLite database. |
| `fetch_sector_industry.py` | Runs after `fetch_stock_timeframes.py`. Looks up each stock's sector/industry/market cap and full earnings-date history, adding two more tables to that day's database. |
| `pasteable_values.py` | Runs after `fetch_sector_industry.py`. Reads that day's scan CSV and outputs a plain, comma-separated ticker list as a `.txt` file (also printed directly in the run's log). |
| `cb_report.py` | Runs last. Reads that day's sector counts from the database, appends them to a persistent history log, and regenerates an Excel report (table + stacked bar chart) from the full history. See Section 7. |
| `cb_report_history.csv` | The persistent log `cb_report.py` reads from and appends to - one row per sector per day. Committed to the repo automatically; this is what makes the "running total" possible across days. |
| `CB_Report.xlsx` | The regenerated Excel report itself - also committed to the repo automatically on every run, so it's always viewable at its current state without needing to download an artifact. |
| `*_log.txt` files | One per script (`scan_log.txt`, `fetch_stock_timeframes_log.txt`, `fetch_sector_industry_log.txt`, `pasteable_values_log.txt`, `cb_report_log.txt`) - a full copy of that script's console output for the run, captured via `tee` in the workflow. Not committed to the repo, just included in the day's zip. |
| `.github/workflows/stock_scan.yml` | The GitHub Actions workflow that runs all the scripts above, on a schedule and on-demand, commits the CB Report files back to the repo, then bundles everything into one dated zip (see Section 8). |
| `requirements.txt` | Python dependencies: `yfinance`, `pandas`, `requests`, `numpy`, `openpyxl`, `lxml`. |
| `PROJECT_HANDOFF.md` | This document - the main technical reference, kept up to date. Hand this to any AI assistant helping with the project. |
| `README.md` | Short repo front-page summary, points to this document and `QUICKSTART.md`. |
| `QUICKSTART.md` | Plain-language, click-by-click guide for the user personally (running the workflow, downloading results, troubleshooting, editing criteria) - not written for an AI audience. |
| `PROJECT_STATE.md` | **Deprecated / recommended for deletion.** Leftover from an earlier version of the project; superseded entirely by this document. If it still exists in the repo, its content should not be trusted. |

Earlier versions of this project used `yahoo_fin` and `requests_html` —
these have been **fully removed** and are no longer needed anywhere.

---

## 3. `scan.py` — the daily screener

### Stock universe
- Includes: **NASDAQ, NYSE, AMEX**
- Excludes: **ETFs**, **OTC-traded stocks**
- Universe is pulled from **NASDAQ's public stock-screener API**
  (`https://api.nasdaq.com/api/screener/stocks`), NOT the older
  `ftp.nasdaqtrader.com` listing files. See Section 9 for why.

### Search criteria (all editable in the `CRITERIA` dict at the top of the file)
Originally translated from a StockCharts.com scan definition:
- Close price >= $10
- Today's % change >= 5%
- 30-day EMA of volume > 20,000 shares
- Weekly dollar volume (close x weekly volume) > $5,000,000
- Weekly RSI(14) >= 40
- Monthly RSI(14) >= 40 -- **exempted** for stocks with under 200 days of
  trading history (see Section 9)
- 50-day EMA >= 200-day EMA (uptrend filter) -- **automatically scales
  down proportionally** for stocks with under 200 days of history (e.g.
  a 60-day-old stock gets roughly a 15-day vs. 60-day EMA comparison
  instead), so newer listings aren't unfairly excluded

### Other behavior
- Only ever uses the **last FULLY COMPLETED trading day** -- if run while
  the market is still open, it automatically uses the prior completed
  session instead of a partial/live one.
- Every network call (universe download, price data) automatically
  retries on failure (5 attempts, 10 seconds apart) before giving up.
- **Individually failed tickers get their own separate retry passes.**
  If a specific ticker fails to fetch within an otherwise-successful
  batch (e.g. a single ticker timing out while the other 99 in that
  batch succeed), it's tracked separately and retried in smaller,
  gentler batches (20 at a time) for up to 3 extra rounds, waiting 10
  seconds between rounds, stopping early once everything is recovered.
  This is distinct from -- and does NOT apply to -- tickers that
  legitimately don't meet the minimum trading history requirement, since
  retrying those would never help (see `min_days_required` above).
- Output filename format: `YYYY-MM-DD_RichRoadUSStockScan_1d.csv`, where
  the date is the actual last completed trading day the data reflects
  (not necessarily the day the script ran).
- Output columns include: ticker, exchange, date, OHLCV, today's
  turnover + 3-day/20-day turnover EMAs, % change, volume EMA(30),
  weekly dollar volume, weekly/monthly RSI, EMA-50/200, uptrend flag,
  plus `history_days` and `full_history` (True/False) so the user can
  see which results relied on the scaled-down "newer stock" logic.
- Has a `test_mode` toggle (currently `False`) that, when `True`, limits
  the scan to a small sample of tickers for fast testing.

---

## 4. `fetch_stock_timeframes.py` — detailed multi-timeframe data

Runs as a second step in the same workflow, right after `scan.py`, and
reads the CSV that `scan.py` just produced to get its ticker list.

### Timeframes fetched (each becomes its own table in the database)
| Table | Interval | Lookback | Note |
|---|---|---|---|
| `daily` | 1 day | As far back as Yahoo has data | Uses `period="max"` |
| `weekly` | 1 week | 5 years | |
| `monthly` | 1 month | 10 years | |
| `quarterly` | 3 months | 15 years | |
| `hourly` | 1 hour | ~725 days | Just under Yahoo's 730-day max, for safety margin (see Section 9) |
| `m15` | 15 min | ~58 days | Just under Yahoo's 60-day max, for safety margin (see Section 9) |
| `m5` | 5 min | ~58 days | Just under Yahoo's 60-day max, for safety margin (see Section 9) |

Lookback for every timeframe except `daily` is calculated as an exact
start/end date range in Python (rather than passed as a Yahoo Finance
`period=` shorthand like `"730d"`) — see Section 9 for why.

### Columns on every table
`ticker, datetime, open, high, low, close, volume, rsi, ema_10, ema_20,
ema_50, ema_100, ema_10_20_cross, ema_10_20_cross_direction, turnover,
turnover_ema_3, turnover_ema_20`

- **RSI and EMA-10/20/50/100** are calculated fresh on each timeframe's
  own data (e.g. hourly RSI uses hourly bars).
- **`ema_10_20_cross`** is an **event flag** -- `True` ONLY on the exact
  bar where the 10 EMA crosses the 20 EMA (not a running "is currently
  above" state). `ema_10_20_cross_direction` is `"up"` or `"down"` on
  that bar.
- **Turnover is calculated per-timeframe**, directly from that
  timeframe's own bars (e.g. the hourly table's turnover reflects hourly
  volume, not daily volume). `turnover` is that bar's own price x volume
  value; `turnover_ema_3`/`turnover_ema_20` are EMAs of that series, in
  that timeframe's own units. (Earlier version of this script always
  used daily-based turnover copied across every table -- changed on
  request.)
- **Earnings dates are NOT in these tables** -- they're handled
  separately, by `fetch_sector_industry.py` (see Section 5), which adds
  its own `earnings_dates` table to the same database.

### Retry protection (three layers)
1. **Per-batch retry** -- an entire batch download failing outright
   retries up to 5 times (same mechanism as `scan.py`).
2. **Per-timeframe retry** -- a ticker failing silently within an
   otherwise-successful batch (e.g. Yahoo's misleading "possibly
   delisted" error -- see Section 9) is tracked separately per timeframe
   and gets up to 3 extra retry rounds in smaller batches of 10, 10
   seconds apart, stopping early once everything recovers.
3. **Final cross-timeframe pass** -- after ALL 7 timeframes have
   finished (including their own retry rounds), anything still failing
   anywhere gets one last attempt, after a 30-second pause -- by this
   point whatever caused the original failure has had several extra
   minutes (the time it took to process the other timeframes) to clear
   up on its own. Recovered rows are appended into the already-saved
   table rather than needing to redo that whole timeframe.

### Why SQLite instead of a spreadsheet
With up to ~500 stocks/day and 6 timeframes each, some timeframes (1-hour
and 5-minute in particular) would produce **over 1 million rows combined
across all stocks** -- which exceeds Excel/Google Sheets' hard limit of
1,048,576 rows per sheet. A SQLite database has no such row limit, stays
fast, and is still just a single downloadable file. The trade-off: it
needs a free viewer app (e.g. "DB Browser for SQLite") to browse
casually, though Claude can read `.db` files directly if uploaded to a
chat.

### Daily snapshot design (not accumulating history)
Each day's `.db` file is a **fresh, standalone snapshot** -- it is NOT
appended to previous days' data. This was a deliberate simplicity
trade-off (Option A of two discussed): a single continuously-growing
database was considered but rejected for now due to added complexity
(avoiding duplicate rows) and GitHub's ~100MB file size limits if it
were ever committed to the repo instead of just uploaded as a
run artifact.

Output filename format: `YYYY-MM-DD_RichRoadStockTimeframes.db`

---

## 5. `fetch_sector_industry.py` — sector, industry, and earnings dates

Runs as a third step in the same workflow, right after
`fetch_stock_timeframes.py`. Reads that day's scan CSV for the ticker
list AND that day's `.db` file (writes its results into that same
database, as two new tables), rather than producing a separate output
file.

This data is fundamentally different from the price/volume data the
other scripts pull: Yahoo only allows fetching it **one ticker at a
time** (unlike price data, which can be batch-fetched for 50-100
tickers in a single request), and this specific type of request is far
more sensitive to Yahoo's rate limiting than the batch price downloads
used elsewhere in this project - see Section 9 for the "Too Many
Requests" issue this caused and how it was fixed.

### Pacing / rate-limit protection
- A **30-second cooldown before this step even starts**, since the
  previous step (`fetch_stock_timeframes.py`) just hit Yahoo hundreds
  of times fetching price data, and this gives that rate-limit window
  time to clear first.
- A **2-second pause between every ticker** (not just on failure).
- Retries use **exponential backoff** - 15s, then 30s, then 60s - since
  a flat retry delay wasn't long enough to clear Yahoo's "Too Many
  Requests" rejection.
- Because of all this, this step alone realistically takes several
  minutes at minimum (roughly 2+ seconds x number of tickers, before
  counting any retries) - this is an intentional trade-off for
  reliability, not something to "optimize away."

### `company_info` table
One row per ticker: `ticker, company_name, sector, industry,
market_cap, days_since_last_earnings, days_until_next_earnings`

- `days_since_last_earnings` / `days_until_next_earnings` are derived
  entirely from the `earnings_dates` table below (no extra API calls) -
  whichever is `None` simply means there's no past/future earnings date
  on record for that stock.
- A stock reporting earnings **today** shows `days_since_last_earnings
  = 0`. A same-day upcoming report showing `days_until_next_earnings =
  0` would be unusual given this runs after market close - in practice
  these two numbers are never both 0 for the same event (a given
  earnings date only ever counts toward ONE of the two columns, past OR
  future, never both).

### `earnings_dates` table
One row **per earnings event, per ticker** (not summarized) - both past
and any upcoming scheduled date Yahoo has on record:
`ticker, earnings_date, eps_estimate, reported_eps, surprise_pct`

Past events have real `reported_eps`/`surprise_pct` values; any
upcoming date will have those blank, since they haven't happened yet.
Fetches up to 100 events per ticker (`limit=100`) - in practice this is
simply "as much history as exists," since no stock actually has
anywhere near 100 quarters of easily available earnings data. Requires
the `lxml` package (see Section 9 for the issue this caused when it was
initially missing from `requirements.txt`).

**Some tickers will have NO rows here at all, and that's expected, not
a bug.** This shows up in the log as a message like `AGMB: No earnings
dates found, symbol may be delisted`.

**Important: the "may be delisted" wording is unreliable and should NOT
be taken at face value.** It's a generic, hardcoded fallback message
inside the `yfinance` library itself, printed automatically whenever
this one specific earnings-calendar lookup comes back empty - it is
NOT an actual check of delisting status. This was confirmed directly:
`AGMB` (AgomAb Therapeutics NV) triggered this exact message despite
being a completely normal, actively-trading NASDAQ stock that IPO'd in
February 2026 and already had a real Q4 2025 earnings report on record
elsewhere on Yahoo's own site. The likely real explanation for that
specific case is that it trades via ADS (American Depositary Shares) as
a foreign-domiciled company, which seems to fall into a coverage gap
for this particular Yahoo endpoint - but the broader point is: treat
"may be delisted" purely as "no data available from this lookup," never
as a genuine signal about the stock's listing status. This message is
printed directly by Yahoo's own library code, not something our retry
system catches (there's nothing to retry; it's an empty result, not an
error). Affected tickers simply won't appear in `earnings_dates`, and
their `days_since_last_earnings` / `days_until_next_earnings` columns in
`company_info` will be blank.

---

## 6. `pasteable_values.py` — plain-text ticker list

Runs as a fourth step in the same workflow, right after
`fetch_sector_industry.py`. Reads the same day's scan CSV (independently
of the other scripts -- it only needs the CSV, not the database) and
outputs the day's tickers as one comma-separated line,
e.g. `AEVA, ALNT, ALOY, ...` -- meant for pasting directly into a
watchlist tool (TradingView, StockCharts, etc.).

- Saved to a downloadable `.txt` file: `YYYY-MM-DD_PasteableValues.txt`
- Also printed directly in the GitHub Actions run log, so it can be
  copied straight from there without downloading anything
- The separator (`", "` by default) is configurable at the top of the
  file if a different format is ever needed (e.g. one ticker per line)

---

## 7. `cb_report.py` — the CB Report (running total by sector)

Runs as the fifth and final data step, after `pasteable_values.py`.
Unlike every other script in this project, **this one maintains state
across days** - the whole point is a running history, which the
"fresh snapshot each day" design used everywhere else can't provide.

### How it works
1. Reads that day's `company_info` table from the database, counts how
   many stocks are in each sector (nulls bucketed as `"Unknown"`).
2. Appends those counts as new rows to a **persistent CSV log**,
   `cb_report_history.csv` - one row per sector per day. If the script
   somehow runs twice for the same date, that date's rows are replaced,
   not duplicated.
3. Rebuilds `CB_Report.xlsx` from the ENTIRE history every run (not just
   appending to the spreadsheet) - a data table (one column per sector,
   one row per day, plus `Total` and `Threshold` columns) and a native,
   embedded stacked bar chart (one bar per day, segmented by sector),
   with a dashed red line drawn across the chart at the "low count"
   threshold.
4. **`cb_report_history.csv`, `CB_Report.xlsx`, and `PROJECT_HANDOFF.md`
   are committed directly back into the repository** by the workflow
   (the "Commit the updated CB Report and handoff doc status line" step
   in `stock_scan.yml`) - this is the only way the history can actually
   persist, since GitHub Action artifacts don't stick around long-term.
   This means the workflow needs `permissions: contents: write` (set at
   the job level), and a normal commit shows up in the repo's history
   every single run.
5. **This commit step pulls in any changes made directly on GitHub
   since the run started, then pushes on top of them** (`git pull
   --rebase` before `git push`) - handling the case where the repo moved
   forward while the workflow was running (e.g. you editing a file on
   GitHub mid-run). If it still can't push after that, the step is
   allowed to fail WITHOUT stopping the rest of the workflow
   (`continue-on-error: true`) - your actual day's outputs
   (CSV/db/report/logs) always still get uploaded regardless, since
   losing them over an unrelated git conflict would be far worse than
   occasionally missing one day's CB Report auto-commit. See Section 9
   for the incident that led to this fix.

### The "under 20" threshold
Configurable via `CONFIG["low_count_threshold"]` (currently `20`).
Shown two ways: a red highlight on any day's `Total` cell below the
threshold (conditional formatting in the spreadsheet), and a dashed red
reference line drawn across the chart at that value.

### Compatibility
Standard `.xlsx` format (via `openpyxl`) - opens natively in Excel,
Google Sheets, or LibreOffice Calc with no conversion needed.

### Honest caveat
Sector history only starts accumulating from whenever this script was
first run - there's no way to retroactively backfill sector breakdowns
for scan days that happened before `fetch_sector_industry.py` existed.

---

## 8. The combined output zip

The workflow's final step bundles everything from that day's run into a
**single downloadable zip artifact**, rather than separate downloads per
file.

- **Named:** `YYYY-MM-DD_stock-scan-outputs.zip`, where the date is
  pulled directly from that day's scan CSV filename (NOT simply
  "today's date") - guaranteeing the zip's date always exactly matches
  the spreadsheet's date, even if the workflow happens to run before
  market close or over a weekend/holiday edge case.
- **Contents:** that day's CSV, `.db`, `.txt`, and `.xlsx` (the CB
  Report) outputs, plus `PROJECT_HANDOFF.md`, `README.md`,
  `QUICKSTART.md`, `WORKING_WITH_ME.md`, and all the project's code
  (`scan.py`, `fetch_stock_timeframes.py`, `fetch_sector_industry.py`,
  `pasteable_values.py`, `cb_report.py`, `requirements.txt`, and the
  workflow file itself).
- **Also included: a plain-text log file per script** -
  `scan_log.txt`, `fetch_stock_timeframes_log.txt`,
  `fetch_sector_industry_log.txt`, `pasteable_values_log.txt`,
  `cb_report_log.txt`. Each is a full copy of that script's console
  output for the run - the same thing visible live in the Actions tab,
  but saved and downloadable so you don't need to dig through a
  specific run to check what happened. Captured via `tee` in the
  workflow, with `set -o pipefail` added so a genuine script failure
  still shows up as a red X and stops the workflow as normal - capturing
  the log doesn't hide or swallow real errors.
- **Why bundle the code and docs too, not just the outputs:** so a
  single file handed to an AI assistant (or kept as an archive) is fully
  self-contained - no separate context needed.

---

## 9. Key issues encountered and how they were resolved

- **`ftp.nasdaqtrader.com` connection timeouts from GitHub's servers** --
  the original approach (downloading `nasdaqlisted.txt` /
  `otherlisted.txt`) consistently failed with `ConnectTimeout` errors,
  even with retries and browser-like headers. This looked like NASDAQ
  blocking/rate-limiting traffic from cloud data center IPs. **Fixed**
  by switching entirely to NASDAQ's public stock-screener API
  (`api.nasdaq.com`), a different service that works reliably.

- **`NHPAF` / `NHPBF` ticker mismatch** -- the scan returned these two
  tickers, but research confirmed the real, correctly-listed securities
  are `NHPAP` / `NHPBP` (National Healthcare Properties preferred
  stock). The root cause was **not** traced to anything in this
  project's own code (no processing step alters that character) -- it
  appears to be a labeling quirk in NASDAQ's own screener data for
  preferred shares specifically. **This is a known, unresolved minor
  data-quality issue** affecting a very small number of preferred-stock
  tickers; a possible future fix is cross-validating tickers against
  Yahoo Finance before trusting them.

- **Comparison against a StockCharts.com scan for the same day**
  surfaced a few expected differences: ETFs appear in StockCharts'
  results but not this scan's (the original StockCharts rule had ETF
  exclusion commented out/disabled, but the user explicitly chose to
  exclude ETFs here). Some borderline stocks differ due to Yahoo Finance
  vs. StockCharts using slightly different underlying data/RSI
  calculations.

- **Row-limit problem for the detailed multi-timeframe data** -- see
  Section 4 above. Solved by moving from a spreadsheet to SQLite.

- **Yahoo Finance's misleading "possibly delisted" errors on hourly
  data** -- `fetch_stock_timeframes.py` was seeing errors like `"1h data
  not available... requested range must be within the last 730 days"`
  for perfectly normal, currently-listed stocks (e.g. `ANL`, `RAPP`,
  `LB`). Investigation showed the actual requested date range in some of
  these errors spanned over 1,000 days, despite only `period="730d"`
  being requested -- a bug in yfinance's own internal period-to-date-
  range conversion for intraday intervals, not a real problem with those
  stocks. **Fixed** by calculating exact start/end dates directly in
  Python instead of relying on yfinance's `period=` shorthand, plus a
  small safety buffer (725 days instead of 730, 58 instead of 60) to
  stay clear of the exact boundary. Daily data was left using
  `period="max"` since this bug is specific to intraday intervals.

- **GitHub Actions schedules only run in UTC**, and the US shifts clocks
  for Daylight Saving Time twice a year. A single fixed UTC cron time
  can't track this shift. After discussion, the user chose a **single
  fixed time (21:30 UTC)** rather than a self-correcting dual-schedule
  solution -- this means the scan runs **30 minutes after market close in
  winter (Standard Time)** and **1.5 hours after close in summer
  (Daylight Saving Time)** -- always after close, just with a varying gap
  depending on time of year. This was a deliberate, informed trade-off
  for simplicity.

- **Silent-looking "stuck" runs** -- Python buffers `print()` output when
  not running in a real terminal, so GitHub Actions logs could appear to
  show no progress for many minutes even though the script was working
  correctly. **Fixed** by running the scripts with `python -u` (unbuffered
  output) so log lines appear live.

- **200-day minimum history requirement was too strict** -- originally
  any stock with under 200 days of trading history was skipped entirely
  (excluding legitimate recent IPOs from the scan). **Changed** to a
  60-day minimum, with trend/RSI calculations automatically scaling down
  proportionally for stocks with less than the full 200 days, and the
  monthly RSI check specifically exempted for those stocks (since a
  meaningful monthly RSI genuinely cannot be calculated with under ~200
  days of data, regardless of scaling).

- **Individual tickers occasionally timing out mid-batch** (e.g. one
  ticker like `EPR` failing while 99 others in the same batch succeed
  fine) were previously just silently skipped forever, even though this
  is usually a brief, one-off network hiccup rather than a real problem
  with that stock. **Fixed** by tracking these separately from
  legitimate exclusions (insufficient history) and giving them up to 3
  additional retry rounds in smaller batches before finally giving up.
  Applied to `scan.py` first, then the same pattern (plus a further final
  cross-timeframe retry pass) was added to `fetch_stock_timeframes.py`.

- **Yahoo "Too Many Requests" rate-limit errors in
  `fetch_sector_industry.py`** -- every single ticker failed on all
  retry attempts, starting from the very first ticker. Unlike the
  timeout-style failures above, this is Yahoo actively rejecting
  requests, not a network hiccup - and it happened because this script's
  per-ticker profile lookups are far more rate-limit-sensitive than the
  batch price downloads used elsewhere, compounded by running
  immediately after `fetch_stock_timeframes.py` had already made
  hundreds of requests to Yahoo in the same job. **Fixed** with three
  changes: a 30-second cooldown before this step starts at all, a longer
  2-second pause between every ticker (up from 0.5s), and retries that
  back off exponentially (15s, 30s, 60s) instead of a flat 5 seconds
  that clearly wasn't enough to clear the rate limit.

- **Missing `lxml` package broke earnings-date fetching entirely** --
  `fetch_sector_industry.py`'s `get_earnings_dates()` call needs `lxml`
  internally (for parsing data from Yahoo), but it was never added to
  `requirements.txt`. This failed identically on every single attempt
  for every ticker - unlike the rate-limit issue above, retries could
  never fix this, since it's a missing package, not a transient
  problem. **Fixed** by adding `lxml` to `requirements.txt`. Worth
  remembering as a category: if an error repeats identically across
  every retry with no variation, that's usually a sign of a permanent
  environment/dependency problem rather than something retry logic can
  help with - cancel the run and fix the root cause rather than letting
  it burn through every retry attempt for every ticker.

- **A rejected `git push` in the CB Report commit step silently lost an
  entire day's outputs.** The commit itself succeeded locally, but the
  push was rejected because the repo on GitHub had moved forward since
  the run started (the user had edited/committed a file directly on
  GitHub while the workflow was mid-run). Because this step failed, the
  workflow stopped before ever reaching "Upload all outputs" -
  meaning that day's CSV, database, report, and logs never got uploaded
  at all, even though the scan itself had run successfully. **Fixed**
  two ways: the commit step now runs `git pull --rebase` before pushing,
  so it automatically incorporates changes made on GitHub mid-run
  instead of just failing; and the step is now allowed to fail without
  stopping the rest of the workflow (`continue-on-error: true`), so even
  in a genuine unresolvable conflict, the actual outputs still always
  get uploaded. The broader lesson: a step that commits back to the repo
  should never be allowed to block the delivery of the day's real
  results.

---

## 10. Known limitations / things intentionally left for later

- **Does not account for US market holidays** -- the schedule fires on
  every Monday-Friday regardless of whether the market was actually
  open (e.g. Thanksgiving, Christmas). This is harmless (it just
  re-reports the same last completed trading day) but does mean
  occasional unnecessary runs. A US market holiday calendar could be
  added later if desired.
- **The `NHPAF`/`NHPBF` ticker labeling issue** (Section 9) has not been
  root-caused with certainty and has no automated fix in place yet.
- **Each day's detailed database (`fetch_stock_timeframes.py`'s output)
  is a standalone snapshot**, not an accumulating history -- see
  Section 4 for the reasoning and the alternative that was considered
  but not built. (This is different from the CB Report's history log,
  Section 7, which DOES accumulate over time by design.)
- **CB Report sector history only starts from whenever `cb_report.py`
  was first run** -- there's no way to retroactively backfill sector
  breakdowns for earlier scan days.
- **The CB Report's Excel chart (stacked bar + threshold line) has not
  been visually verified by opening it in Excel/LibreOffice** - it was
  built using standard, documented `openpyxl` chart features, but
  hasn't yet been confirmed to render exactly as intended. Check the
  first real report generated and report back anything that needs
  adjusting.
