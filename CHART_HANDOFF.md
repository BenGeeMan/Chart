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

Section 0, item 1 from the last handoff (debug the four drawing tools)
is now **done** — see Section 6. What's left, in order:

1. **Write an "annotation guide" prompt** — a clean, well-written
   reference covering the full text protocol (`MARKER`/`LINE`/
   `TRENDLINE`/`RAY`/`HRAY`, the `RSI_` variants, and whatever gets
   added below), aimed at teaching good annotation practice, not just
   listing syntax.
2. **Bake that guide directly into the "Copy image for Claude"
   capture** (see Section 5) — so a future Claude session with no
   memory of this one can read the protocol straight out of the image
   itself. This is explicitly for easing future-session onboarding.
3. **Connect the chart to a real database** instead of the current
   static, manually-exported snapshot — the long-deferred item
   described in Section 4.
4. **Redo the scan/pipeline** (in the sibling `RichRoadStockScreenerUS`
   repo) so it produces everything this chart project currently
   calculates client-side — EMA 200, RSI(14), Volume EMA 3/20, and
   ideally the multi-timeframe/relative EMA overlays too. See Section 4
   for the full current client-side-vs-pipeline breakdown.
5. **Expand/improve the watchlist** — likely connecting it to the full
   daily scan output rather than the fixed 11 hand-picked stocks.
6. **Add a "Technical Reference" section to both this document AND
   `PROJECT_HANDOFF.md`** — the user asked whether current docs contain
   enough for an AI to rebuild either project from scratch, possibly on
   a different platform. Honest answer given: the *concept*, *history*,
   and *gotchas* are well covered, but hard implementation detail isn't
   — exact data schema/field names, exact color values, exact layout
   proportions currently only exist in the actual code, not written
   into the docs as a standalone reference. Needs fixing in both repos'
   docs, not just this one.
7. **Still not built:** Rectangle, Arrow, Text, Price Range, Date
   Range, Date and Price Range, Path tool, Highlighter, Eraser. See
   Section 6 for the full confirmed TradingView tool-name list.
8. **Text protocol doesn't cover style yet** — a pasted `TRENDLINE`/
   `LINE`/etc. command always creates a default-styled drawing (solid,
   2px, full opacity). The color/opacity/thickness/line-type controls
   added this session (Section 5/6) are mouse-only for now — extending
   the protocol with optional `opacity=`/`width=`/`style=` attributes
   is a natural next step if Claude needs to specify style precisely.
9. **Only after 1-8 are solid**, move on to the actual end goal of this
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

The eventual goal (see Section 0, item 9): the user explains trading
concepts/decision logic in plain English, Claude shows its
understanding visually through this tool, and once agreed correct,
that logic gets turned into permanent, automatable code.

---

## 2. Repository structure

| File | Purpose |
|---|---|
| `index.html` | The entire app - a single self-contained HTML file. Must keep this exact filename; GitHub Pages requires it. Contains a visible build-number watermark (currently 47), manually incremented on every handoff, purely so the user can confirm a hard refresh actually picked up new code after repeated caching confusion during development. |
| `chart_watchlist.json` | Quote summary for the 11 watchlist stocks. |
| `chart_earnings.json` | Earnings dates (past ~3 years + upcoming) per watchlist stock. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (77 files: 11 stocks × 7 timeframes). |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the repo for handoff to an AI assistant. |

**Naming convention:** bias toward "chart" in new filenames. Exception:
`index.html`.

**Hosting:** GitHub Pages, public repo (free-tier limitation — Pages
needs a paid plan for private repos).

---

## 3. The text protocol (as of build 47)

```
MARKER date=YYYY-MM-DD pos=above|below shape=arrowUp|arrowDown|circle color=green|red|blue text="label"
LINE price=NUMBER color=green|red|blue label="text"
TRENDLINE time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
RAY time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
HRAY time=YYYY-MM-DD price=NUMBER color=green|red|blue
```

Every command except `MARKER` also has an **`RSI_` prefixed twin**
(`RSI_LINE`, `RSI_TRENDLINE`, `RSI_RAY`, `RSI_HRAY`) that targets the
RSI panel instead of the main chart — same syntax, same attributes,
just landing on a different pane. See Section 5/6 for why RSI needed
its own full copy of the drawing tools rather than sharing the main
chart's.

- Parsed with a simple `key=value` attribute parser; quoted values
  support spaces.
- **Bidirectional and consistent by design:** every command Claude can
  send, the user can also produce by drawing directly on the chart
  (via the toolbar, Section 6) or pasting the same syntax back - both
  paths call the same underlying `addUserDrawing()`/`addHline()`
  functions, so there's no divergent behavior between "Claude drew
  this" and "the user drew this." Items added by the user directly are
  tagged `[added by user on chart]` in the Chart State output.
- **`time1`/`time2` (or `time`) must fall on different bars for
  `TRENDLINE`/`RAY`/their `RSI_` twins.** Two points on the same bar
  crash the underlying charting library outright (see Section 6's
  "same-bar crash" bugs) — the parser rejects same-bar commands with an
  error rather than letting them through.
- **Style (color/opacity/thickness/line-type) isn't in the protocol
  yet** — see Section 0, item 8. Pasted commands always get the
  default style for their type; style can currently only be changed
  afterward via the ⚙ Style panel (mouse only).
- **Still needs the tools from Section 0, item 7** before the protocol
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

**This is explicitly item 4 in Section 0's next-session plan** -
moving this into `fetch_stock_timeframes.py` in the sibling repo is
the stated direction, not yet started.

**Multi-timeframe time-format handling:** daily-and-above stores `time`
as a date string; intraday stores Unix timestamps. Overlaying one
timeframe's data onto a chart of a different timeframe needs
conversion - see `alignTimeToCurrentChart()` and `timeToNumeric()` /
`extrapolatePrice()` in `index.html` (the latter two are also what the
Ray drawing tool uses to extend a line while preserving its slope - see
Section 7 for the log-scale nuance this involves).

**Current watchlist (11 stocks):**
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX` — one
per sector, chosen for turnover/variety. Section 0 item 5 plans to
expand this.

---

## 5. Visual design (current state, build 47)

**Layout:** Watchlist (left) | main area | — no right sidebar anymore
(removed a while back; see Section 7). Header row: ticker/name,
timeframe buttons (5m/15m/1H/D/W/M/Q, left-aligned), then far-right:
seven icon buttons — 🗑 Clear all, 🗒 Copy chart state text, 📄 Save
chart state text to file, 📥 Paste from Claude (opens a temporary
modal), 📋 Copy image, ⬇ Save image, ✦ Copy image for Claude (bakes the
full Chart State text into the image itself, so one paste gives Claude
both the picture and the context in a single action). A thin status
bar sits directly below the header.

**Main chart:** candlesticks, **semi-log (logarithmic) price scale**,
wicks/borders always black — this log scale is the source of two
separate, now-fixed drawing-tool bugs, see Section 7. Header stats line
(ticker · timeframe · change/%/range/volume) is hover-reactive -
updates to whichever bar the crosshair is over (defaulting to the
latest bar), computing change and % vs. that bar's own previous close
(not the visible range's start), and a "Range" figure that includes any
gap from the prior close. All text in this line is plain black
(color-coding was removed per user request); the word "Range" itself
was also removed, leaving just a signed number in that slot.

**Volume panel:** bottom ~25% via scaleMargins trick, raw share volume
(red/teal), Volume EMA 20 dashed line, "Volume" alone omitted from the
Chart State text (visually obvious from the image) but Volume EMA 20
still listed.

**RSI panel:** separate chart instance (architecture reasons in
Section 7), **linear 0-100 scale (not logarithmic)**, with its own
independent timeframe selector (5m/15m/1H/D/W/M/Q) - defaults to
matching the main chart but can be set differently; pan/zoom sync
between the two charts only happens when both show the same timeframe.
**Now has its own full drawing toolbar** (Section 6) - a separate
`rsiBarsData` array tracks RSI's own current bars (which can be a
different timeframe than the main chart's `barsData`) so drawing tools
there snap to the right bars.

**Drawing toolbars:** main chart has one (left edge, vertical, 5
buttons), RSI panel has its own compact copy (same 5 buttons, scaled
down to fit the shorter pane) - Cursor, Trend Line, Horizontal Line,
Ray, Horizontal Ray. **All fully working** as of build 47 - draw,
select (click a line), drag to reshape from either end, drag a
midpoint handle to move the whole shape, delete (button or
Delete/Backspace key), and a ⚙ Style panel (color picker + 12 preset
swatches, opacity slider, thickness 1-4px, line type solid/dashed/
dotted). See Section 6 for the full history of what had to be fixed to
get here, and Section 7 for the architecture that makes both panes
share the same code.

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

## 6. Drawing tools — status and history

All five confirmed TradingView tools from the original toolbar audit:

| Tool | Status |
|---|---|
| Cursor (select/pan) | **Built** |
| Trend Line | **Built, fully working (build 47)** |
| Horizontal Line | **Built, fully working (build 47)** |
| Ray | **Built, fully working (build 47)** |
| Horizontal Ray | **Built, fully working (build 47)** |
| Rectangle | Not built |
| Price Range | Not built |
| Text | Not built |
| Arrow Marker | Not built |
| Date and Price Range | Not built |
| Path tool | Not built |
| Date Range | Not built |
| Arrow | Not built (distinct from Arrow Marker above) |
| Highlighter | Not built |
| Eraser | Not built |

**What "fully working" covers, per tool:** draw via 2 clicks (Trend
Line/Ray) or 1 click (Horizontal Line/Ray) with a live dashed preview
that follows the mouse before the shape commits; click an existing
shape to select it (drag handles appear at each editable point, plus a
square handle at the midpoint for moving the whole shape as a unit);
Delete button or Delete/Backspace key removes the selected shape;
⚙ Style panel for color (picker + 12 presets)/opacity/thickness/line
type. Both the main chart and the RSI panel have this full set,
independently.

**The debugging history that got here** (useful context if similar
symptoms reappear):

1. **No live preview while drawing, no way to select/edit after the
   fact** — the original state going into this work. Fixed by adding a
   dashed preview series during the 2-click draw, and a full selection
   system (handles, move, delete, style) for after.
2. **`Uncaught RangeError: Maximum call stack size exceeded`,
   repeatedly, when finishing a Trend Line/Ray** — a real bug in the
   underlying Lightweight Charts library, triggered by two data points
   on the *same bar* (which happens constantly, since the mouse always
   passes back over the anchor's own bar right after the first click).
   Fixed with same-bar guards at every point a line's two times could
   coincide: the live preview, the finishing click, drag handles, and
   the pasted-command parser.
3. **Same crash, still happening** after fix #2 — turned out to be a
   *second*, unrelated trigger for the identical library bug: calling
   `setData()` synchronously on every single mouse-move event during
   the live preview. Fixed by (a) excluding every drawing series from
   the price scale's auto-fit calculation via
   `autoscaleInfoProvider: () => null`, and (b) throttling preview
   updates to once per animation frame via `requestAnimationFrame`
   instead of updating synchronously inside the event handler.
4. **A "strange dot" appearing off in empty space near a Ray, and its
   midpoint not actually sitting on the line** — two related but
   distinct bugs, both stemming from the chart's **logarithmic price
   scale**:
   - The move-handle/Style-Delete toolbar were positioned using the
     midpoint between the anchor and Ray's *extrapolated far edge*,
     which can land at a price way outside the visible chart for a
     short, steep ray. Fixed by positioning that UI from the two real
     click points instead (`getAnchorPoints()`), never the
     extrapolated edge.
   - Separately, `extrapolatePrice()` was doing **linear** math on raw
     price values, but a straight line on a **logarithmic** axis is
     only straight in `log(price)` space. This meant the real second
     click that defined a Ray's slope was never genuinely on the
     rendered line — confirmed numerically (4.83px off) before fixing,
     and 0px off after switching the extrapolation to log space.
   **If a pane's price-scale mode is ever changed (log ↔ linear), the
   `isLog` flag passed to that pane's `createDrawingPane({...})` call
   must be updated to match, or these same two categories of bug come
   back** — see the large comment above `createDrawingPane` in
   `index.html`.
5. **Dragging a shape via its move-handle visibly changed its angle** —
   same log-scale root cause as #4, this time in the move math: it was
   shifting both points by a constant *raw price* amount, but a
   constant raw-price offset changes the `log(price)` difference (and
   so the visual angle) depending on where on the log scale you are.
   Fixed by moving with a constant price *ratio* instead, which
   preserves the log-difference — confirmed numerically before
   shipping (log-diff 0.182 stayed 0.182 after the fix; the old
   additive method drifted to 0.065).
6. **Selection handles could appear anywhere on the page** — clicking a
   scroll-off-screen shape's handles rendered them wherever their raw,
   unclamped pixel coordinates landed: in the watchlist (negative x),
   the header/timeframe row (negative y), the price-axis gutter (x past
   the plot), or a neighboring pane (y past the pane height). The
   selection overlay was never actually clipped to the pane. Fixed by
   checking every handle/button's position against the pane's real
   pixel bounds before rendering it at all, rather than trusting the
   raw coordinate math to land inside the chart.

**Ray/Horizontal Ray extension nuance (still true):** neither is truly
infinite - both extend only as far as the currently *loaded* data (500
bars for daily, full history for weekly/monthly/quarterly, etc.),
computed once at draw time. This is a deliberate, pragmatic tradeoff
(avoiding the complexity of dynamically re-extending lines on every
pan/zoom event) — see Section 0, item 7/8 and Section 8.

---

## 7. Architecture notes worth knowing (hard-won, don't repeat these)

**Drawing tools are one shared, pane-agnostic implementation
(`createDrawingPane()`), not copy-pasted per pane.** Adding RSI's
drawing toolbar didn't duplicate the ~500 lines of draw/select/move/
delete/style logic - it's a factory function taking a small config
(getters for the pane's current chart/series/container/bars array, its
own overlay `<div>` id and toolbar selector, and whether its price
scale is logarithmic), instantiated once for the main chart
(`mainPane`) and once for RSI (`rsiPane`). **To add drawing tools to a
future indicator pane:** give it its own toolbar + overlay `<div>` in
the HTML (copy `#rsi-draw-toolbar`/`#rsi-drawing-overlay` as a
template), then one `createDrawingPane({...})` call — no changes
needed to the factory itself. Every chart-instance reference inside the
factory goes through a getter rather than being captured once, which
matters for RSI specifically since its chart is destroyed and recreated
every time RSI is toggled off/on (`rsiChart`/`rsiSeries` become `null`
then get fresh instances) - `rsiPane.attach()` gets called again on the
new instance each time, and `rsiPane.reset()` clears its own state
before the old one is torn down.

**The main chart's logarithmic price scale is a recurring source of
subtle bugs for anything involving slope/extrapolation.** See Section
6, bugs #4 and #5, for two real examples and their fixes. The general
lesson: any code computing a point "along a line" on this chart needs
to interpolate in `log(price)` space, not raw price, or the result will
be mathematically correct in the wrong space and visibly wrong on
screen. RSI's panel is a plain linear 0-100 scale, so this doesn't
apply there — which is exactly why `isLog` is a per-pane flag rather
than a global assumption.

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

**When debugging "creates successfully but doesn't render" (or "should
work but doesn't"), build a real headless-JS simulation rather than
guessing repeatedly.** Used successfully several times now: the
original `barsData` temporal-dead-zone crash that broke the entire page
(found via a hand-rolled Node `vm` context mimicking `document`/
`window`), and — more recently — the same-bar crash and both log-scale
drawing-tool bugs in Section 6, each confirmed numerically with a small
Node script *before* shipping the fix, rather than trusting the theory
alone. Worth reaching for this early rather than after several rounds
of speculative fixes.

**Earnings badges are plain HTML, not Lightweight Charts markers, and
use a rounded square, not a pentagon.** The pentagon shape (CSS
`clip-path`) looked right on screen but `html2canvas` doesn't support
`clip-path`, so captured screenshots showed a plain square instead -
inconsistent with the live page. Simplified the on-screen badge to
match what the capture can actually reproduce, rather than the other
way around, so both are consistent everywhere.

**Column master checkboxes remember individual state** on toggle
off/on, rather than resetting to "everything on."

**For "must sit at an exact, predictable pixel position" requirements,
prefer plain CSS over fighting a charting library's own coordinate
system.** Getting the earnings badges to sit at a consistent height
took several failed rounds using Lightweight Charts' own marker/
price-scale system (synthetic anchor values, fixed-range scales, a
dedicated price scale) - none behaved predictably. Switching to plain
HTML `<div>` elements (X position from the chart's own
`timeToCoordinate()`, Y position an ordinary fixed CSS value) solved it
immediately and reliably. The drawing tools' selection handles and ⚙
Style panel use the same approach for the same reason - just with the
added step (Section 6, bug #6) of clamping to the pane's actual pixel
bounds before rendering, since "exact pixel position" also needs to
account for that position sometimes falling outside the visible pane
entirely.

**GitHub Pages URLs are case-sensitive.** A repo named `Chart`
(capital C) serves at `.../Chart/`, not `.../chart/` - visiting the
lowercase version 404s even though it looks like the same address.
Caused real confusion once already; if a live page ever 404s
unexpectedly, check the exact repo name's capitalization before
assuming a deployment or code problem.

**Browser caching caused repeated false "it's still broken" reports.**
The build-number watermark (Section 2) exists specifically to catch
this. **Whenever the user reports something as not working that should
already be fixed, ask them to confirm the watermark number and do a
hard refresh (Ctrl+Shift+R / Cmd+Shift+R) before investigating
further** - this has repeatedly turned out to be chasing a stale cached
version rather than a real bug.

**The right sidebar was removed entirely** (replaced by header icon
buttons + a temporary paste modal + a status bar). Every function that
depended on the old sidebar elements was individually re-wired to its
new home - `input-box` and `output-box` still exist in the DOM as
hidden elements (not deleted) since other code still reads/writes them
programmatically.

---

## 8. Known limitations / deferred items

See Section 0 for the actively-planned next steps. Additionally:

- **"RichRoad MA v3"** (an indicator visible in the user's real
  TradingView setup) is still not replicated - formula unknown, a
  strong candidate for the eventual concept-teaching phase (Section 0,
  item 9).
- **Rays are not truly infinite** - see Section 6.
- **Only 11 of ~155 typical scan-day stocks are in the watchlist** -
  Section 0 item 5.
- **The text protocol doesn't carry style (color/opacity/thickness/
  line-type)** - Section 0 item 8, Section 3.
- **Rectangle, Arrow, Text, Price Range, Date Range, Date and Price
  Range, Path tool, Highlighter, Eraser** - none built yet, Section 6.
