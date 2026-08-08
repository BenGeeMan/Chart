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
Read both documents when working across the two.

**If the user shares a screenshot or a zip of this repo (via the
"Chart Export" GitHub Action), cross-reference it against this
document first** — the protocol format, known limitations, and design
decisions below explain things that would otherwise look like
anomalies.

**If a change made during a conversation would make this document
inaccurate, flag that to the user at the time** — a brief note is
enough. Don't rewrite it automatically; the user prefers to batch
documentation updates and will ask for the refreshed file when ready.

---

## 0. NEXT SESSION PRIORITIES (read this first)

The user was tired and asked to stop here, with this explicit plan for
next time, in this order:

1. **Finish the remaining annotation tools.** Trend Line, Horizontal
   Line, Ray, and Horizontal Ray were built in this session (build 36)
   but **the user reports they "aren't working as imagined"** - no
   further detail was given before stopping. Debugging this is the
   very first thing to do next session; don't assume the existing
   implementation is sound. Still to build after that's fixed:
   Rectangle, Arrow, Text, Price Range, Date Range, Date and Price
   Range, Path tool, Highlighter. See Section 6 for the full confirmed
   TradingView tool-name list this was based on.
2. **Write an "annotation guide" prompt** - a clean, well-written
   reference covering the full text protocol (`MARKER`/`LINE`/
   `TRENDLINE`/`RAY`/`HRAY` plus whatever gets added in step 1), aimed
   at teaching good annotation practice, not just listing syntax.
3. **Bake that guide directly into the "Copy image for Claude"
   capture** (see Section 5) - so a future Claude session with no
   memory of this one can read the protocol straight out of the image
   itself. This is explicitly for easing future-session onboarding.
4. **Connect the chart to a real database** instead of the current
   static, manually-exported snapshot - the long-deferred item
   described in Section 4.
5. **Redo the scan/pipeline** (in the sibling `RichRoadStockScreenerUS`
   repo) so it produces everything this chart project currently
   calculates client-side - EMA 200, RSI(14), Volume EMA 3/20, and
   ideally the multi-timeframe/relative EMA overlays too. See Section 4
   for the full current client-side-vs-pipeline breakdown.
6. **Expand/improve the watchlist** - likely connecting it to the full
   daily scan output rather than the fixed 11 hand-picked stocks.
7. **Only after 1-6 are solid**, move on to the actual end goal of this
   whole project: teaching Claude the user's trading-decision logic
   through this visual tool, so it can eventually become real
   automated logic. This has not been started.

---

## 1. What this project is, in plain terms

A TradingView-style charting tool whose real purpose is to act as a
**two-way text communication bridge** between the user and an AI
assistant (Claude), since Claude has no native ability to see or
interact with a live web app:

1. Claude writes instructions in a simple text format (e.g. "put a
   marker here," "draw a trend line there").
2. The user pastes that text into a box on the chart (via the header's
   📥 button, which opens a temporary paste modal). The app parses it
   and draws it visually.
3. The chart's Chart State (accessed via the header's 🗒/📄 buttons, or
   baked into the ✦ "Copy image for Claude" screenshot) always
   reflects the current ticker, timeframe, every indicator's on/off
   state, and everything drawn — including things the *user* draws
   directly on the chart via the drawing toolbar (Section 6) — so both
   sides stay in sync in a format Claude can read.

The eventual goal (see Section 0, item 7): the user explains trading
concepts/decision logic in plain English, Claude shows its
understanding visually through this tool, and once agreed correct,
that logic gets turned into permanent, automatable code.

---

## 2. Repository structure

| File | Purpose |
|---|---|
| `index.html` | The entire app - a single self-contained HTML file. Must keep this exact filename; GitHub Pages requires it. Contains a visible build-number watermark (currently 36), manually incremented on every handoff, purely so the user can confirm a hard refresh actually picked up new code after repeated caching confusion during development. |
| `chart_watchlist.json` | Quote summary for the 11 watchlist stocks. |
| `chart_earnings.json` | Earnings dates (past ~3 years + upcoming) per watchlist stock. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (77 files: 11 stocks × 7 timeframes). |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the repo for handoff to an AI assistant. |

**Naming convention:** bias toward "chart" in new filenames. Exception:
`index.html`.

**Hosting:** GitHub Pages, public repo (free-tier limitation — Pages
needs a paid plan for private repos).

---

## 3. The text protocol (as of build 36)

```
MARKER date=YYYY-MM-DD pos=above|below shape=arrowUp|arrowDown|circle color=green|red|blue text="label"
LINE price=NUMBER color=green|red|blue label="text"
TRENDLINE time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
RAY time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
HRAY time=YYYY-MM-DD price=NUMBER color=green|red|blue
```

- Parsed with a simple `key=value` attribute parser; quoted values
  support spaces.
- **Bidirectional and consistent by design:** every command Claude can
  send, the user can also produce by drawing directly on the chart
  (via the toolbar, Section 6) or pasting the same syntax back - both
  paths call the same underlying `addUserDrawing()`/rendering
  functions, so there's no divergent behavior between "Claude drew
  this" and "the user drew this." Items added by the user directly are
  tagged `[added by user on chart]` in the Chart State output.
- `TRENDLINE`/`RAY`/`HRAY` were added in build 36 (see Section 6 for
  how Ray/Horizontal Ray extension actually works - important nuance).
- **Still needs the tools from Section 0, item 1** before the protocol
  can grow to cover Rectangle/Arrow/Text/measurement tools.

---

## 4. Data — what's pre-computed vs. calculated client-side

Unchanged from the last full audit: the original data export included
`ema_10/20/50/100`, `turnover`, `turnover_ema_3/20` per bar. Everything
else is calculated live in the browser:

- **EMA 200** - not in the original export.
- **Volume EMA 3 / Volume EMA 20** - export only had *turnover* EMAs,
  not raw *volume* EMAs (the panel changed from dollar-turnover to
  share-volume after export).
- **RSI(14)** - never in the export.
- **Every Timeframe EMA / Relative EMA overlay** - each re-fetches that
  timeframe's raw file and computes EMA(10) fresh every time it's
  toggled on.

**This is explicitly item 4-5 in Section 0's next-session plan** -
moving this into `fetch_stock_timeframes.py` in the sibling repo is
the stated direction, not yet started.

**Multi-timeframe time-format handling:** daily-and-above stores `time`
as a date string; intraday stores Unix timestamps. Overlaying one
timeframe's data onto a chart of a different timeframe needs
conversion - see `alignTimeToCurrentChart()` and `timeToNumeric()` /
`extrapolatePrice()` in `index.html` (the latter two are also what the
Ray drawing tool uses to extend a line while preserving its slope).

**Current watchlist (11 stocks):**
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX` — one
per sector, chosen for turnover/variety. Section 0 item 6 plans to
expand this.

---

## 5. Visual design (current state, build 36)

**Layout:** Watchlist (left) | main area | — no right sidebar anymore
(removed this session; see Section 7). Header row: ticker/name,
timeframe buttons (5m/15m/1H/D/W/M/Q, left-aligned), then far-right:
seven icon buttons — 🗑 Clear all, 🗒 Copy chart state text, 📄 Save
chart state text to file, 📥 Paste from Claude (opens a temporary
modal), 📋 Copy image, ⬇ Save image, ✦ Copy image for Claude (bakes the
full Chart State text into the image itself, so one paste gives Claude
both the picture and the context in a single action). A thin status
bar sits directly below the header.

**Main chart:** candlesticks, semi-log price scale, wicks/borders
always black. **Header stats line** (ticker · timeframe · change/%/
range/volume) is now **hover-reactive** - it updates to whichever bar
the crosshair is over (defaulting to the latest bar), computing change
and % vs. that bar's own previous close (not the visible range's
start), and a "Range" figure that includes any gap from the prior
close. All text in this line is plain black (color-coding was removed
per user request); the word "Range" itself was also removed, leaving
just a signed number in that slot.

**Volume panel:** bottom ~25% via scaleMargins trick, raw share volume
(red/teal), Volume EMA 20 dashed line, "Volume" alone omitted from the
Chart State text (visually obvious from the image) but Volume EMA 20
still listed.

**RSI panel:** separate chart instance (architecture reasons in
Section 7), with its **own independent timeframe selector**
(5m/15m/1H/D/W/M/Q) - defaults to matching the main chart but can be
set differently; pan/zoom sync between the two charts only happens
when both show the same timeframe.

**Drawing toolbar** (new, build 36): vertical, left edge of the chart,
5 buttons - Cursor, Trend Line, Horizontal Line, Ray, Horizontal Ray.
**User reports this isn't working as expected** - see Section 0.

**Indicator legend:** collapsible ("Indicators ▾"), closes on
outside-click. Six columns, each with its own master checkbox
(remembers individual row states when toggled off/on):
1. **Standard EMAs** - EMA 10/20/50/200.
2. **M10s** - 7 fixed-timeframe EMA(10) overlays.
3. **Relative M10s** - Current TF / 1 TF above / 2 TFs above (offset
   relative to whatever the main chart currently shows).
4. **Volume** - Volume bars, Volume EMA 20.
5. **Events** - Earnings, 10/20 EMA cross (both moved here from other
   columns per user request).
6. **Indicators** - currently just RSI(14), deliberately its own
   column since more RSI-related indicators are expected later.

**Standing instruction from the user:** whenever a new indicator is
added, (a) add its swatch color to the `COLOR_NAMES` lookup (used to
name colors in the Chart State text), and (b) explicitly ask the user
whether it should be included in that text output - never assume
either way.

**Earnings badges:** small rounded-square orange badges (changed from
an earlier pentagon shape - see Section 7), plain HTML overlays
positioned via the chart's `timeToCoordinate()` for X and a fixed CSS
value for Y. Daily-and-above timeframes only.

**Screenshot capture:** uses `html2canvas` (not the charting library's
own limited screenshot function) so HTML overlays - earnings badges,
info boxes - are correctly included. Scoped to just the chart area,
never the watchlist or header. The "for Claude" variant additionally
renders the full Chart State text as real, readable text appended
below the image.

---

## 6. Confirmed TradingView tool identification

The user went through their actual TradingView toolbar with Claude and
confirmed (via repeated correction) the following tool names, which is
what Section 0 item 1's remaining build list is based on:

| Tool | Status |
|---|---|
| Horizontal Ray | **Built, build 36** |
| Trend Line | **Built, build 36** |
| Ray | **Built, build 36** |
| Rectangle | Not built |
| Price Range | Not built |
| Text | Not built |
| Arrow Marker | Not built |
| Date and Price Range | Not built |
| Path tool | Not built |
| Date Range | Not built |
| Horizontal Line | **Built, build 36** |
| Arrow | Not built (distinct from Arrow Marker above - both confirmed as separate tools) |
| Highlighter | Not built |
| Eraser | Not built |

**Important nuance on how "Ray" and "Horizontal Ray" actually render:**
neither is truly infinite - both extend only as far as the currently
*loaded* data (500 bars for daily, full history for weekly/monthly/
quarterly, etc.), computed once at draw time via
`extrapolatePrice()`/`timeToNumeric()` in `index.html`. This is a
deliberate, pragmatic tradeoff (avoiding the complexity of dynamically
re-extending lines on every pan/zoom event) and is very likely related
to why the user found these "not working as imagined" - worth
reconsidering this approach next session rather than assuming it's
just a bug to patch.

---

## 7. Architecture notes worth knowing (hard-won, don't repeat these)

**RSI needed a genuinely separate chart instance, not a shared-canvas
trick.** A single canvas's price-axis labels render across its FULL
height regardless of `scaleMargins` - margins only affect where data
plots, not where axis labels draw. Symptom was price numbers appearing
to repeat all the way through where RSI should have been. Fixed with
a real second `LightweightCharts.createChart()` instance, synced via
`subscribeVisibleLogicalRangeChange` (only when both charts show the
same timeframe).

**A missing `min-height: 0` / `min-width: 0` on CSS Grid items caused
two separate "impossible" bugs** - RSI refusing to reopen after being
toggled off (vertical), and the entire right sidebar disappearing on a
different monitor (horizontal, before the sidebar was removed
entirely). Without these properties, a grid item's own content can
force it larger than its assigned space rather than compressing to
fit, and `overflow: hidden` on `<body>` meant there was no way to
scroll to the overflow. All three grid columns needed both properties.
**If a similarly "impossible" layout bug appears again, check this
class of issue early.**

**When debugging "creates successfully but doesn't render," build a
real headless-JS simulation rather than guessing repeatedly.** The
`barsData` temporal-dead-zone crash that broke the entire page (not
just RSI) was found this way - a hand-rolled Node `vm` context
mimicking `document`/`window` caught the exact ReferenceError that
several rounds of speculative fixes had missed. Reusable technique for
any "this should work but doesn't" situation in future sessions.

**Earnings badges are plain HTML, not Lightweight Charts markers, and
use a rounded square, not a pentagon.** The pentagon shape (CSS
`clip-path`) looked right on screen but `html2canvas` doesn't support
`clip-path`, so captured screenshots showed a plain square instead -
inconsistent with the live page. Simplified the on-screen badge to
match what the capture can actually reproduce, rather than the other
way around, so both are consistent everywhere.

**Column master checkboxes remember individual state** on toggle
off/on, rather than resetting to "everything on."

**The right sidebar was removed entirely this session** (replaced by
header icon buttons + a temporary paste modal + a status bar). Every
function that depended on the old sidebar elements was individually
re-wired to its new home - `input-box` and `output-box` still exist in
the DOM as hidden elements (not deleted) since other code still
reads/writes them programmatically.

---

## 8. Known limitations / deferred items

See Section 0 for the actively-planned next steps. Additionally:

- **"RichRoad MA v3"** (an indicator visible in the user's real
  TradingView setup) is still not replicated - formula unknown, a
  strong candidate for the eventual concept-teaching phase (Section 0,
  item 7).
- **Rays are not truly infinite** - see Section 6's nuance box.
- **Only 11 of ~155 typical scan-day stocks are in the watchlist** -
  Section 0 item 6.
