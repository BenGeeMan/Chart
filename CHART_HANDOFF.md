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

**This project is one half of a bigger effort.** The sibling repo,
`RichRoadStockScreenerUS`, runs the actual daily stock-scanning
pipeline and owns `PROJECT_HANDOFF.md` for its own details. This repo
(`chart`) is the visual/communication layer built on top of that data.
Read both documents when working across the two - decisions in one
sometimes explain behavior in the other.

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
3. The chart also has a "Chart State" box that always reflects whatever
   is currently drawn, which the user copies and pastes back to Claude
   so both sides are looking at the same thing in a format Claude can
   read. (Note: this used to also capture the user clicking directly on
   candles - that click-to-mark feature was explicitly removed at the
   user's request; see Section 6.)

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
| `chart_earnings.json` | Earnings dates (past ~3 years + upcoming) per watchlist stock, keyed by ticker. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (e.g. `hpq_daily.json`, `cohr_weekly.json`). One set of 7 files per watchlist stock (77 files total for 11 stocks). |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the entire repo into one downloadable file, for handing the project to an AI assistant. |

**Naming convention for this repo specifically:** bias toward naming
new things with "chart" (e.g. `chart_watchlist.json`,
`chart_export.yml`). Exception: `index.html` must keep its exact name
for GitHub Pages to serve it.

**Hosting:** GitHub Pages, deployed from the `main` branch root. The
repo is currently **public** — a deliberate cost-saving choice (GitHub
Pages requires a paid Pro plan for a private repo, $4/month) since
nothing in this repo is sensitive.

**A visible build-number watermark** (large, faint, centered on the
main chart) exists purely so the user can confirm a hard refresh
actually picked up new code, after repeated browser-caching confusion
during development. It gets manually incremented by one on every
`index.html` handoff. If picking this project back up after a gap,
just keep incrementing from whatever number is currently in the file.

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
a screenshot), not a generic "TradingView-style" guess. Layout, top to
bottom:

- **Watchlist** (left column) - all 11 stocks, click to switch ticker.
- **Header** - ticker/company name, then timeframe buttons
  (5m/15m/1H/D/W/M/Q) positioned immediately after it on the left side
  (not right-aligned).
- **Main price chart** - candlesticks, semi-log price scale. Hollow
  white body for up bars, solid red for down; **wicks and borders are
  always black regardless of direction** (explicit request).
- **Volume panel** (bottom ~25% of the same canvas, via Lightweight
  Charts' scaleMargins trick) - raw share volume bars (not dollar
  turnover - see Section 5), colored red/teal by direction, with a
  Volume EMA 20 dashed overlay line.
- **RSI panel** - a genuinely **separate chart instance** below the
  main one, not sharing its canvas. This matters architecturally - see
  the callout in Section 6.

### Indicator legend
Collapsible ("Indicators ▾" button, top-left) rather than always
expanded. When open, a card-style panel shows **five columns**, each
with its own master checkbox (turning a column off remembers each
row's individual state and restores exactly that, not "all on", when
turned back on):

1. **Price EMAs** - EMA 10 (black), 20 (blue), 50 (green), 200 (red),
   and the 10/20 EMA crossover marker (orange dot, sits precisely on
   the EMA 10 line's value via a dedicated invisible tracking series -
   independent visibility from the EMA 10/20 lines themselves).
2. **Timeframe EMAs (10)** - seven checkboxes (5m/15m/1H/Daily/
   Weekly/Monthly/Quarterly), each independently toggleable, each
   overlaying that timeframe's own EMA(10) onto whatever chart is
   currently showing - e.g. show the Weekly EMA 10 on top of Daily
   candles. See Section 5 for how time-format conversion works here.
3. **Relative EMAs (10)** - "1 TF above" / "2 TFs above". Same idea,
   but the target timeframe is recalculated relative to whatever the
   main chart currently shows (viewing Daily, "1 above" = Weekly;
   switch to Weekly, "1 above" becomes Monthly automatically).
4. **Volume & Earnings** - Earnings badges, Volume bars, Volume EMA 20.
5. **RSI** - just the one RSI(14) checkbox today, but deliberately its
   own column since the user expects to add more RSI-related
   indicators later.

### Earnings badges
Small orange triangles/pentagon-with-"E" badges are **plain HTML
elements positioned on top of the chart**, not Lightweight Charts'
native marker system. Their X position comes from the chart's own
`timeToCoordinate()`, but Y is an ordinary fixed CSS `bottom` value -
see the debugging story in Section 6 for why this ended up being the
right approach. Only shown on daily-and-above timeframes (a single
earnings date doesn't map cleanly onto intraday bars).

### Info boxes
- **Bottom-left, white box:** `LT` / `AT (3)` / `AT (20)` - updates
  live to whatever bar the crosshair is hovering, defaults to the
  latest bar when the cursor isn't on the chart.
- **Top-right, "Current daily":** always shows the latest *daily* bar's
  figures regardless of which timeframe is currently selected -
  independent fetch, not tied to the main chart's own data.
- Both are figures for **volume**, not dollar turnover (renamed - see
  Section 5). Both boxes' positions are calculated in JS relative to
  chart boundaries (price-axis width, panel edges) rather than fixed
  pixel guesses, so they stay correctly placed across different
  tickers/timeframes.

### "No Data" messaging
A faint, centered "No Data" watermark (matching the build-number
style) appears on the main chart or RSI panel whenever a data file
fails to load, or loads successfully but is empty - so a genuinely
missing file is never silently indistinguishable from a rendering bug.

### Known simplifications vs. the user's real setup (not yet resolved)
- The real setup shows an indicator called **"RichRoad MA v3"** and
  what looks like a 5th, thicker moving-average line. Its actual
  formula is still unknown - a strong candidate for the first "explain
  a concept to Claude" session once that workflow starts.

---

## 5. Data

- **Library:** TradingView's own open-source "Lightweight Charts"
  (loaded via CDN), chosen specifically so the chart's visual behavior
  matches TradingView's real product.
- **Source:** every number in every `{ticker}_{timeframe}.json` file
  came from the actual `RichRoadStockTimeframes.db` database built in
  the sibling `RichRoadStockScreenerUS` project - real OHLCV data, not
  invented.
- **How it got into this repo: manually, as a one-time export**, not
  automatically. A Python script queried the database and wrote out
  the JSON files, which were then hand-uploaded. **This is still a
  frozen snapshot from one specific day** - no automated pipeline keeps
  it updated. Explicitly deferred by the user more than once; nothing
  decided yet on whether/how to automate it.

### What's pre-computed in the data files vs. calculated live in the browser
The original export included `ema_10/20/50/100`, `turnover`,
`turnover_ema_3/20` per bar (all computed server-side by the sibling
project's pipeline). Everything added **since** that original export
is calculated **client-side, in JavaScript, on the fly**, because the
data files themselves were never regenerated to include it:

- **EMA 200** - not in the original export (only 10/20/50/100 were).
- **Volume EMA 3 / Volume EMA 20** - the export only had *turnover*
  EMAs, not raw *volume* EMAs, after the panel was changed from
  dollar-turnover to share-volume.
- **RSI(14)** - never in the export at all.
- **Every Timeframe EMA / Relative EMA overlay** - each one re-fetches
  that timeframe's raw OHLCV file and computes EMA(10) from scratch in
  the browser, every time it's toggled on.

**The user has explicitly stated the long-term direction: all of this
should eventually move into the actual data pipeline** (most likely
`fetch_stock_timeframes.py` in the sibling repo, alongside where
`ema_10/20/50/100` and the turnover EMAs already get computed
server-side) rather than being recalculated in-browser every time. This
hasn't been started - flagging it here is the first step. When it
happens, the client-side `calculateEMA()` / `calculateRSI()` JS
functions in `index.html` become unnecessary for anything the pipeline
now provides directly in the JSON files, and can be simplified or
removed accordingly.

### Multi-timeframe time-format handling
Daily-and-above timeframes store `time` as a date string
(`"2026-08-07"`); intraday timeframes store it as a Unix timestamp.
Overlaying one timeframe's EMA onto a chart of a *different* timeframe
required a conversion step (`alignTimeToCurrentChart()` in
`index.html`): date strings convert to midnight-UTC timestamps when
overlaying onto an intraday chart; timestamps collapse to one date
string per calendar day (keeping the last value) when overlaying onto
a daily-or-higher chart.

### Current watchlist (11 stocks, one per sector, chosen for turnover/variety)
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX`

Each has all 7 timeframes exported: `daily` (last 500 bars), `weekly`,
`monthly`, `quarterly` (full history for these three), `hourly` (last
600 bars), `m15`, `m5` (last 600 bars each).

---

## 6. Architecture notes worth knowing (hard-won, don't repeat these)

**RSI needed a genuinely separate chart instance, not a shared-canvas
trick.** The original approach tried to give RSI its own "panel" via
Lightweight Charts' `scaleMargins` (the same trick used for the volume
panel, which shares the main chart's single canvas). This does NOT
work for a third stacked panel: a price scale's axis *labels* are
drawn across the chart's **entire canvas height regardless of
scaleMargins** - margins only affect where *data* gets plotted, not
where axis labels render. The symptom was price-axis numbers (e.g.
`8.05`) appearing to repeat all the way down through where RSI should
have been. The fix was giving RSI its own actual `LightweightCharts.
createChart()` instance in a separate DOM element below the main
chart, with the two chart instances' time scales synced via
`subscribeVisibleLogicalRangeChange` (only when both show the same
timeframe - see below).

**RSI's toggle-off/toggle-on bug was a missing `min-height: 0` on a
CSS Grid item, not a JavaScript problem at all.** After extensive
diagnostic logging confirmed the RSI chart object, its canvas, and its
data were all being created correctly with correct dimensions, the
real cause turned out to be that `#chart-pane` (a grid item inside a
`100vh`-constrained layout) was missing `min-height: 0`. Without it, a
grid item's own content can force it to grow taller than its assigned
row rather than compressing to fit - pushing the RSI panel down past
the visible viewport, with `overflow: hidden` on `<body>` meaning there
was never any way to scroll to it. **All three grid columns
(`#chart-pane`, `#watchlist`, `#sidebar`) need both `min-height: 0` and
`min-width: 0`** - the same class of bug recurred horizontally (the
right sidebar disappearing entirely) when the browser window moved to
a different monitor with different available width. If a similarly
"impossible" layout bug shows up again, check this class of issue
early rather than assuming it's a data/timing/JS problem.

**When RSI shows a different timeframe than the main chart, pan/zoom
syncing between them is deliberately disabled** (`rsiTF !== currentTF`
guards on both sync subscriptions) - keeping bar-index-based ranges
locked together only makes sense when both charts are on the same
resolution.

**Earnings badges are plain HTML, not Lightweight Charts markers.**
Several rounds of trying to use the library's native marker/price-line
system for a "fixed height, always the same distance from the volume
panel's bottom" badge produced inconsistent, hard-to-predict results.
Switching to ordinary positioned `<div>` elements (X from
`timeToCoordinate()`, Y from a plain CSS value) gave direct, reliable
control. If a similar "needs to be pinned to an exact, predictable
pixel position regardless of data" requirement comes up again, prefer
this HTML-overlay approach over fighting the charting library's own
positioning system.

**Column master checkboxes remember individual state.** Turning a
whole indicator column off saves each row's checked state first, then
restores exactly that (not "everything on") when switched back on.

---

## 7. Known limitations / things intentionally left for later

- **Data is a static, manually-exported snapshot**, and every indicator
  added since that export is calculated client-side rather than in the
  pipeline - see Section 5. Moving this into the pipeline is the
  explicitly stated next architectural step, not yet started.
- **Only 11 of the ~155 stocks from a typical scan day are in the
  watchlist** - a deliberate starting scope, not a limit of the design.
- **"RichRoad MA v3" is not yet replicated** - see Section 4.
- **No zones, trend lines, or shaded regions yet** in the text
  protocol - only `MARKER` and `LINE` exist so far.
- **Click-to-mark was removed** (see Section 1) - there is currently no
  way for the user's direct chart interaction to get captured as text;
  only Claude-authored `MARKER`/`LINE` commands populate the Chart
  State box now.
