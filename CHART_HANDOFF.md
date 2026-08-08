# ==========================================
# FILE: CHART_HANDOFF.md
# ==========================================

# chart — Project Handoff Summary

This document exists so that any AI assistant (or human) picking this
project up later can understand what has been built, why certain
decisions were made, and what's still outstanding — without needing to
rediscover them from scratch.

The user who owns this project is **not a programmer** and has no prior
GitHub experience. All instructions given to them should assume a
complete-beginner level, including basic GitHub steps.

**If the user shares a zip of this repo (via the "Chart Export" GitHub
Action) or an individual file, cross-reference it against this document
first** — the protocol format, known limitations, and design decisions
below explain things that would otherwise look like anomalies.

**If a change made during a conversation would make this document
inaccurate or out of date, flag that to the user at the time** — a
brief note is enough. Don't rewrite it automatically; the user prefers
to batch documentation updates and will ask for the refreshed file when
ready.

---

## 1. What this project is, in plain terms

A TradingView-style charting tool whose real purpose is to act as a
**two-way text communication bridge** between the user and an AI
assistant (Claude), since Claude has no native ability to see or
interact with a live web app:

1. Claude writes instructions in a simple text format (e.g. "put a
   marker here," "draw a line there").
2. The user pastes that text into a box on the chart. The app parses it
   and draws it visually — the user never draws anything by hand from
   Claude's instructions.
3. When the user interacts with the chart directly (currently: clicking
   a candle to mark it), the app automatically converts that action
   into the same text format and shows it in a separate box.
4. The user copies that text and pastes it back to Claude, so both
   sides are always looking at the same thing, described in a format
   Claude can actually read.

The eventual goal (not yet started): the user will explain trading
concepts in plain English, Claude will show its understanding visually
through this tool, and once they agree the understanding is correct,
that logic gets turned into permanent code.

---

## 2. Repository structure

| File | Purpose |
|---|---|
| `index.html` | The entire app - a single self-contained HTML file (no build tools, no server). Must keep this exact filename; GitHub Pages requires it. |
| `chart_watchlist.json` | Quote summary (ticker, company name, sector, last price, % change) for every stock in the watchlist panel. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (e.g. `hpq_daily.json`, `cohr_weekly.json`). One set of 7 files per watchlist stock. |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the entire repo into one downloadable file, for handing the project to an AI assistant. |

**Naming convention for this repo specifically:** the user has asked
for a bias toward naming new things with "chart" in this repo (e.g.
`chart_watchlist.json`, `chart_export.yml`). Exception: `index.html`
must keep its exact name for GitHub Pages to serve it.

**Hosting:** GitHub Pages, deployed from the `main` branch root. The
repo is currently **public** — a deliberate cost-saving choice (GitHub
Pages requires a paid Pro plan for a private repo, $4/month) since
nothing in this repo is sensitive. Revisit only if that changes.

---

## 3. The text protocol

Two commands supported so far, one per line:

```
MARKER date=YYYY-MM-DD pos=above|below shape=arrowUp|arrowDown|circle color=green|red|blue text="label"
LINE price=NUMBER color=green|red|blue label="text"
```

- Parsed with a simple `key=value` attribute parser; quoted values
  (`text="..."`) support spaces.
- Unknown commands or missing required attributes are reported back in
  the status line rather than silently failing.
- **This protocol is intentionally minimal and expected to grow** as
  new concepts get explained and need new visual vocabulary (zones,
  trend lines, shaded regions, etc. are natural next additions).

---

## 4. Visual design

Mimics the user's **actual personal TradingView setup** (confirmed via
a screenshot), not a generic "TradingView-style" guess:

- Light theme (white background), not dark - an early version used a
  dark theme before the user's screenshot corrected this.
- Candles: hollow white fill with a dark border for up bars, solid red
  for down bars (explicit request - ignore that some real TradingView
  setups show green up-candles).
- Top-left legend overlay showing the current ticker, timeframe, %
  change, and a list of active indicators with color swatches.
- Four EMA lines overlaid on the price chart (10/20/50/100 periods),
  each individually toggleable via a checkbox next to its legend entry
  - checkbox state persists across timeframe switches.
- A turnover sub-panel pinned to the bottom ~25% of the same chart
  (via Lightweight Charts' price-scale margin trick - a real second
  chart instance was not needed).

### Known simplifications vs. the user's real setup (not yet resolved)
- The real setup shows an indicator called **"RichRoad MA v3"** and
  what looks like a 5th, thicker moving-average line. Its actual
  formula is unknown - the 4 standard EMAs were used as a placeholder
  instead. This is a strong candidate for the first "explain a concept
  to Claude" session once that workflow starts.
- The real setup's volume/turnover panel shows both `AT (20)` and
  `AT (3)` labels. Only the 20-period turnover EMA is currently
  plotted; the 3-period one has not been added yet.

---

## 5. Data

- **Library:** TradingView's own open-source "Lightweight Charts"
  (loaded via CDN), chosen specifically so the chart's visual behavior
  matches TradingView's real product, not a hand-built approximation.
- **Source:** every number in every `{ticker}_{timeframe}.json` file
  came from the actual `2026-08-07_RichRoadStockTimeframes.db` database
  built in the sibling `RichRoadStockScreenerUS` project - real OHLCV,
  real EMA-10/20/50/100, real turnover figures, not invented data.
- **How it got into this repo: manually, as a one-time export**, not
  automatically. A Python script queried the database and wrote out
  the JSON files, which were then hand-uploaded to this repo.
- **This is flagged as a known limitation, explicitly deferred by the
  user ("let's address this issue later"):** the chart currently shows
  a frozen snapshot from one specific day, not live/current data, and
  there is no automated pipeline keeping it updated. Two real options
  were discussed for later: (a) extend the other project's daily
  workflow to also export these small JSON files and commit them
  automatically, keeping the chart always current; or (b) leave it as
  occasional manual exports. Nothing has been decided yet.
- **Why not just read the `.db` file directly in the browser:** it's
  ~266MB, both too large to fit GitHub's normal file-size limits for a
  git commit and impractical to download/query client-side for a
  lightweight static page.

### Current watchlist (11 stocks, one per sector, chosen for turnover/variety)
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX`

Each has all 7 timeframes exported: `daily` (last 500 bars), `weekly`,
`monthly`, `quarterly` (full history for these three), `hourly` (last
600 bars), `m15`, `m5` (last 600 bars each).

---

## 6. Behavior notes worth knowing

- **Switching timeframe clears all current drawings.** A date-based
  marker on the daily chart doesn't have an obvious meaning on a
  5-minute chart, so - matching how most real trading platforms behave
  - changing resolution starts fresh rather than carrying annotations
  over incorrectly.
- **Clicking the watchlist switches the active ticker** and reloads
  whatever timeframe is currently selected for the new stock.
- **The input/output text boxes are intentionally small** (~90px tall,
  fixed size rather than stretching to fill the sidebar) - the user
  clarified they don't need to read the contents, the boxes are purely
  a copy/paste transport mechanism between the user and Claude.
- Both boxes were confirmed to need further size refinement later
  ("smaller still") - explicitly deferred, not yet actioned.

---

## 7. Known limitations / things intentionally left for later

- **Data is a static, manually-exported snapshot** - see Section 5.
- **Only 11 of the ~155 stocks from a typical scan day are in the
  watchlist** - a deliberate starting scope, not a limit of the design.
- **"RichRoad MA v3" and the `AT (3)` turnover line are not yet
  replicated** - see Section 4.
- **Text/output box sizing** was flagged for further shrinking but not
  yet revisited.
- **No zones, trend lines, or shaded regions yet** in the text
  protocol - only `MARKER` and `LINE` exist so far.
