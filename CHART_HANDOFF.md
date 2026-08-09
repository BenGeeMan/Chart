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
enough. Don't rewrite it automatically; the user prefers to note
documentation-worthy changes as they come up during a session, then do
one full refresh at the end, rather than updating the doc after every
small change.

---

## 0. NEXT SESSION PRIORITIES (read this first)

Everything from the previous handoff's Section 0 items 1-2 (annotation
guide, baking it into image capture) is **still not started**. Item 3
(real database) is **still not started but now better-scoped** — see
the new Section 4 subsection. Items 4-5 (pipeline/watchlist expansion)
are sibling-repo work, still not started. What's new or changed:

1. **User has now raised "rebuild this project on another platform"
   twice** — once as a hypothetical (previous session, prompted the
   old item 6 below) and once directly this session ("it may be
   necessary in the future to rebuild the project on another
   platform"). Treat this as a real possibility, not just a
   documentation exercise. **Honest current state: the gap identified
   last time has gotten WIDER, not narrower** — this session added
   several new subsystems (RSI settings/layout persistence, per-tool
   style/timeframe defaults, the future-placeholder-bars data model)
   that live entirely in `index.html`'s code and browser localStorage,
   with no standalone spec anywhere else. A rebuild-from-docs attempt
   today would need to reverse-engineer all of it from source. If a
   platform migration looks likely, the highest-leverage next step is
   writing an actual **Technical Reference doc** (exact data schemas,
   localStorage key formats, the whole timeframe-visibility model,
   color/layout constants) BEFORE more features get added on top,
   not after.
2. **Persistent storage for annotations** (so drawings aren't lost
   switching tickers) — discussed but deliberately deferred. User
   floated a database on their own Google Drive; also compared against
   using this GitHub repo itself as storage, and Firebase/Supabase.
   Leaning toward GitHub-repo-as-storage given how comfortable the user
   already is with this platform, but no decision made yet.
3. **Any future indicator pane should get full feature parity with
   RSI** — its own settings panel (style + on/off + value per
   configurable line), its own independent multi-timeframe selector,
   and the drawing tools attached to it. This is now a **standing
   requirement**, not a one-off. The drawing-tools half of this is
   already trivial (Section 7's `createDrawingPane` factory). The
   settings-panel and layout (resize/reposition) halves are NOT yet
   factored into a reusable pattern — RSI's version
   (`RSI_SETTINGS_FACTORY_DEFAULT`, `applyRsiSettings`,
   `renderRsiSettingsPanel`, `rsiLayout`/`applyRsiLayout`) is a good
   template to copy, but a second indicator would currently mean
   copy-pasting and renaming all of it, not reusing a shared factory.
   Worth generalizing once a second indicator is actually needed.
4. **Extend 5-minute data's historical depth.** 15-minute was extended
   this session (31 days → 138 days, matching Hourly) but 5-minute
   deliberately wasn't (see Section 4) — old drawings still won't
   resolve on 5m if their date predates its ~9-14 day window. Revisit
   if this becomes a live complaint rather than the earlier 15m one.
5. **Text protocol still doesn't carry style** — a pasted `TRENDLINE`/
   `LINE`/etc. still has no `opacity=`/`width=`/`style=` attribute.
   **New nuance this session:** a pasted (Claude-authored) shape now
   DOES inherit that tool's saved default opacity/thickness/line-type
   (via the same mechanism new mouse-drawn shapes use) — only color
   still comes from the pasted command's own `color=` attribute. This
   was a deliberate choice (consistency between hand-drawn and pasted
   shapes of the same tool) but means Claude can no longer assume a
   pasted TRENDLINE is always solid/2px — it now depends on whatever
   the user last saved as that tool's default.
6. **Still not built:** Rectangle, Arrow, Text, Price Range, Date
   Range, Date and Price Range, Path tool, Highlighter, Eraser. See
   Section 6 for the full confirmed TradingView tool-name list.
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
| `index.html` | The entire app - a single self-contained HTML file. Must keep this exact filename; GitHub Pages requires it. Contains a visible build-number watermark (currently **68**), manually incremented on every handoff, purely so the user can confirm a hard refresh actually picked up new code after repeated caching confusion during development. |
| `chart_watchlist.json` | Quote summary for the 11 watchlist stocks. |
| `chart_earnings.json` | Earnings dates (past ~3 years + upcoming) per watchlist stock. Reconciled this session — see Section 4. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (77 files: 11 stocks × 7 timeframes). All 77 now extend past "today" with future placeholder bars — see Section 4. 15m files are noticeably bigger than before (~650-680KB vs the others' ~250-350KB) due to the historical-depth extension. |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the repo for handoff to an AI assistant. |

**Naming convention:** bias toward "chart" in new filenames. Exception:
`index.html`.

**Hosting:** GitHub Pages, public repo (free-tier limitation — Pages
needs a paid plan for private repos).

---

## 3. The text protocol (as of build 68)

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
just landing on a different pane.

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
- **Style (color/opacity/thickness/line-type) still isn't in the
  protocol** — see Section 0, item 5 for an important nuance added
  this session: a pasted shape's non-color style now comes from the
  tool's saved default, not always a hardcoded factory value.
- **A new drawing's default relative-timeframe visibility changed this
  session** — see Section 6.
- **Still needs the tools from Section 0, item 6** before the protocol
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

This is still item 3 in Section 0's next-session plan (renumbered from
the previous handoff) - moving this into `fetch_stock_timeframes.py` in
the sibling repo, not yet started.

**Multi-timeframe time-format handling:** daily-and-above stores `time`
as a date string; intraday stores Unix timestamps. Overlaying one
timeframe's data onto a chart of a different timeframe needs
conversion - see `alignTimeToCurrentChart()` and `timeToNumeric()` /
`extrapolatePrice()` in `index.html`.

**Current watchlist (11 stocks):**
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX` — one
per sector, chosen for turnover/variety.

### New this session: future placeholder bars + historical depth extension

**The problem that started this:** a drawing made on Daily around
2026-04-09 wasn't showing on Hourly or 15m when it should have. Root
cause (confirmed by directly testing `resolveDrawingTime()` against the
real data files, not just theorizing): **not a logic bug** - the
timeframe-matching logic was working correctly, but the intraday data
files simply didn't go back far enough in calendar time for that date
to exist in them at all. All three intraday timeframes had the SAME
bar *count* (600 each) but wildly different calendar *coverage*, since
bar density per day differs enormously:
- Hourly: 600 bars = 119 calendar days
- 15m: 600 bars = only 31 calendar days
- 5m: 600 bars = only 9 calendar days

**Fix #1 - 15m historical extension:** 15m extended backward from 31 to
138 days (matching Hourly's range), by **synthesizing intraday bars
grounded in the real daily OHLC** for each added day - not random
noise. For each day, the daily bar's own open/high/low/close bounds a
random-walk path (open → close, touching the day's real high and low
at some point, small bar-to-bar noise layered on top), volume/turnover
split across the day's bars with light randomness, EMA columns held at
that day's daily EMA value (a simplification - real intraday EMA would
evolve within the day, this doesn't, but it's a minor visual
imperfection for what's fundamentally placeholder/demo data). Verified
before shipping: strictly chronological, valid OHLC relationships
throughout (no bar where high < max(open,close) etc.), and a sample
day's synthesized open/close exactly matches that day's real daily
bar. **5m was deliberately NOT extended** - matching Hourly's range for
5m would need ~9,000+ bars per ticker (vs 15m's ~2,600), a large jump
in file size for a smaller marginal benefit. See Section 0 item 4.

**Fix #2 - future placeholder bars:** the user separately asked for
"about half a screen of blank future" on every timeframe, so
annotations aren't confined to the past, with known earnings dates
marked on that future space. All 77 files now extend past their last
real bar with **flat placeholder bars**: OHLC all equal to the last
real close (a flat line, not fake price movement), volume/turnover
zero, EMAs held at their last real value. Counts per timeframe were
sized to reach comfortably past the furthest researched earnings date
(2026-11-11): daily +75 trading days, weekly +16 weeks, monthly +6
months, quarterly +3 quarters, hourly +30 trading days, 15m +12 trading
days, 5m +6 trading days. Cadence (weekday-only for daily/intraday,
Monday-anchored weekly, 1st-of-month monthly, Feb/May/Aug/Nov-anchored
quarterly) matches each file's own existing pattern exactly - verified
by inspecting the last 5 real bars' day-of-week/date pattern for each
timeframe before writing the generator.

**`chart_earnings.json` was reconciled against fresh web research**
for all 11 tickers' next earnings dates: fixed one stale `is_future`
flag (IREN had a 2024 date incorrectly still flagged `true`), upgraded
three tickers from estimated to company-confirmed dates (HPQ Aug 26 →
Sep 1 confirmed; NTRA Nov 5 → Nov 11 confirmed; IREN Aug 27 → Sep 16
confirmed), and added TTWO's next date (Nov 5) which was completely
missing. Every future earnings date was verified to land on an exact
bar in that ticker's daily file (not just "close enough") so the
existing `findEarningsBarTimes()` marker logic anchors to the right
day instead of silently snapping to today's bar.

**Two follow-on bugs this surfaced and fixed, worth remembering for any
future data-extension work:**
1. **`fitContent()` fits ALL loaded bars**, including the new flat
   future ones - left as-is, this would zoom out to squash real
   candles down to make room for a long flat line by default. Replaced
   with a calculated `setVisibleLogicalRange` showing real bars and
   future bars in roughly equal proportion (both the main chart and
   RSI's own `fitContent()` call needed this fix).
2. **Several places assumed `array.length - 1` meant "today's bar"**
   (headline price/change stats, the daily-stats box, the
   crosshair-leave fallback) - no longer true once arrays extend into
   the future. Added `lastRealBarIndex(bars)` (scans backward for the
   first bar with non-zero volume) and fixed every call site that had
   this assumption. **If more future-dated data is ever added to this
   app, grep for `.length - 1` first** - it's the fastest way to find
   every place carrying this same assumption.

---

## 5. Visual design (current state, build 68)

**Layout:** Watchlist (left) | main area | RSI pane (bottom by default,
**now repositionable to top** - see below). Header row: ticker/name,
timeframe buttons (5m/15m/1H/D/W/M/Q), **then the drawing toolbar
(moved here this session - previously a floating vertical bar on the
main chart's left edge)**, separated by a thin vertical divider, then
far-right: seven icon buttons — 🗑 Clear all, 🗒 Copy chart state text,
📄 Save chart state text to file, 📥 Paste from Claude, 📋 Copy image,
⬇ Save image, ✦ Copy image for Claude. A thin status bar sits directly
below the header.

**Main chart:** candlesticks, semi-log (logarithmic) price scale,
wicks/borders always black. Header stats line is hover-reactive,
defaulting to the latest bar - **now correctly means the latest REAL
bar, not the last array index** (see Section 4's `lastRealBarIndex`
fix).

**Volume panel:** bottom ~25% via scaleMargins trick, unchanged.

**RSI panel:** separate chart instance, linear 0-100 scale, own
independent timeframe selector. Several new capabilities this session:
- **⚙ RSI Settings panel** (gear icon in the RSI header row, panel
  itself renders centered in the main chart area - it started
  anchored to the RSI pane itself, then the top-right corner of the
  main chart, before landing on centered, since the RSI pane is too
  short to fit the whole panel and the corner anchor wasn't visible
  enough). Covers: **Pane** position (Top/Bottom dropdown), **RSI
  Line** style (color/thickness/line-type), and **Upper/Middle/Lower**
  levels in that order, each with its own on/off + threshold value AND
  its own independent color/thickness/line-type (not a shared style
  across all three, per explicit user request). Defaults: Upper 60 on,
  Lower 40 on, Middle 50 off. Changes apply live immediately; **Set as
  Default** persists the current settings to localStorage
  (`rsiSettings_v1`); **Restore Default** reloads whatever was last
  saved there (or the factory defaults if nothing ever was) -
  discarding any live-but-unsaved tweaks, INCLUDING resetting the
  pane's height back to 150px (position is left alone).
- **Mouse-resizable** - a drag handle sits on whichever edge borders
  the main chart (flips automatically with position), clamped 80-500px
  tall.
- **Repositionable to the top of the page** (default: bottom) via the
  settings panel's Position dropdown.
- Both height and position **auto-persist immediately** on change
  (`rsiLayout_v1` in localStorage) - no separate save step, unlike the
  style settings above, since there's only ever one "current" layout.

**Drawing toolbar:** now a single shared toolbar living in the header
(not two separate copies per pane anymore in terms of UI - it was
always one shared `sharedToolState` underneath, see Section 7, but the
main chart previously had its own floating vertical copy; RSI's was
already compact). Same 5 buttons: Cursor, Trend Line, Horizontal Line,
Ray, Horizontal Ray.

**Floating panels (Style, Timeframe, RSI Settings) are all
draggable** - each has a small labeled title bar at the top (`Style`,
`Timeframe`, `RSI Settings`) that can be mouse-dragged anywhere on
screen, via one shared `makeDraggable(panelEl, handleEl)` helper. Added
because panels could sometimes render partially off-screen depending
on where the triggering drawing/button was.

**Timeframe visibility model — significantly reworked this session**
(see Section 6 for the full history of iterations). Current final
state:
- Two independent columns, each with its own On/Off master switch that
  is a pure bypass toggle (does NOT read or write the individual
  checkboxes under it) - Absolute ("Show on" 5m/15m/1H/D/W/M/Q) starts
  **off** by default now; Relative starts **on**.
- The Relative column's higher/lower boxes are **cumulative ranges**
  ("2 higher" means home + 1 + 2 higher, not just exactly 2 higher),
  displayed **radio-button style** (picking a number unticks the
  others on that side, but still means "up to that number"
  underneath).
- A range always includes its own home timeframe - picking "2 higher"
  shows on home + 1 + 2 higher, not "2 higher, excluding home."
  "Current" alone means "only home, nothing else."
- **New default for new drawings: "2 higher" AND "2 lower" both on**
  (previously just "2 higher"), absolute column off. A drawing made on
  Daily now shows on 15m through Monthly by default.

**Per-tool style/timeframe defaults ("Set as Default" / "Restore
Default"), same pattern as RSI settings above** - each of the 4 line
tools (Trend Line, Horizontal Line, Ray, Horizontal Ray) independently
remembers its own saved style (color/opacity/thickness/line-type) and
its own saved timeframe-visibility defaults, persisted to localStorage
(`chartToolDefaults_v1`) and applied to every NEW drawing of that tool
going forward - separate per tool, separate between style and
timeframe (four independent save slots per tool). See Section 0 item 5
for how this interacts with pasted (Claude-authored) commands.

**Indicator legend, earnings badges, screenshot capture:** unchanged
from last handoff.

---

## 6. Drawing tools — status and history

All five confirmed TradingView tools from the original toolbar audit:

| Tool | Status |
|---|---|
| Cursor (select/pan) | **Built** |
| Trend Line | **Built, fully working** |
| Horizontal Line | **Built, fully working** |
| Ray | **Built, fully working** |
| Horizontal Ray | **Built, fully working** |
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
Line/Ray) or 1 click (Horizontal Line/Ray) with a live dashed preview;
click to select (drag handles at each editable point, plus a midpoint
handle to move the whole shape); Delete button or key; ⚙ Style panel;
🕐 Timeframe panel (visibility controls, see below). Both the main
chart and the RSI panel have this full set via the same shared
`createDrawingPane()` factory (Section 7).

### The timeframe-visibility model's design history (multiple iterations this session)

This went through several rounds before landing on the state described
in Section 5 - worth knowing if the design gets revisited again:

1. Started as two columns (absolute per-TF checkboxes, relative
   higher/lower/current) each defaulting all-checked, with a master
   "All" checkbox that tried to remember prior state on toggle.
2. **Bug:** the "All" master both READ and WROTE the individual
   checkboxes, so unchecking one box auto-unchecked "All," and
   re-checking "All" reset everything to all-true rather than
   restoring what was there before - not genuinely "remembering."
   **Fixed** by making the master a pure bypass switch
   (`timeframesColumnEnabled`/`relativeTFColumnEnabled`) that never
   touches the individual boxes at all - flipping it off just means
   the column is ignored, flipping it back on means whatever was
   checked underneath still applies, since it was never changed.
3. **Redefined "higher N"/"lower N" as cumulative ranges** ("3 higher"
   = up to 3 higher, not just exactly 3) rather than independent single
   offsets - this is what real trading platforms mean by this kind of
   control, and avoids a confusing "nothing shows because you only
   checked +3 without +1/+2" state.
4. **Switched the checkbox display to radio-button style per side**
   (picking N unticks the others on that side) after the user found
   multiple simultaneously-ticked boxes confusing to read, even though
   the underlying cumulative meaning didn't change - `matchesRelativeTF`
   already worked off "highest checked value per side," so this was a
   pure UI change, no logic change needed.
5. **Made "current" (home) always implied by ANY picked range**, not
   mutually exclusive from it - a "2 higher" pick now also shows on
   home itself, not just the two timeframes above it. Only bare
   "current" with nothing else picked means "only home."
6. **Root-caused an apparent visibility bug that turned out to be a
   data-coverage limitation, not a logic bug** - see Section 4's "the
   problem that started this" writeup. Confirmed via direct console
   inspection of a real drawing's raw stored data
   (`mainPane.userDrawings[0]`), not just re-reading the UI - the panel
   accurately reflected the underlying state the whole time; the issue
   was that the intraday files didn't go back far enough for the
   drawing's own date to exist in them.
7. **Changed the default for new drawings** from "2 higher only" to "2
   higher AND 2 lower," absolute column off by default - directly
   requested once the historical-coverage gap above made the
   asymmetry (showing on Weekly/Monthly but not Hourly/15m) obvious in
   practice.

**Ray/Horizontal Ray extension nuance (still true):** neither is truly
infinite - both extend only as far as the currently *loaded* data,
computed once at draw time.

---

## 7. Architecture notes worth knowing (hard-won, don't repeat these)

**Drawing tools are one shared, pane-agnostic implementation
(`createDrawingPane()`), not copy-pasted per pane.** Unchanged from
last handoff - see previous version of this doc (or the code comment
above `createDrawingPane` in `index.html`) for the full explanation.
Still the right template for adding drawing tools to any future
indicator pane (Section 0, item 3).

**The temporal-dead-zone crash class recurred this session with a
DIFFERENT variable, and broke the entire page again the same way.**
`LINE_STYLE_MAP` (a `const`) was defined down in the drawing-tools
section of the script, but `createRsiChart()` - called synchronously
near the top of the script on every page load - now calls
`applyRsiSettings()`, which references `LINE_STYLE_MAP` before its
declaration line had executed. Symptom was identical to the earlier
`barsData` incident: completely blank page, empty watchlist, nothing
loads, one red `ReferenceError: Cannot access 'X' before initialization`
in the console. **Fixed** by moving `LINE_STYLE_MAP`'s declaration up
next to the other early declarations that already carry a comment
explaining this exact failure class. **General rule reinforced: before
adding any new top-level `const` that a function called synchronously
early in the script will reference, check it's declared before that
call point - or declare the helper as a `function` instead of a `const
() => {}` arrow, since function declarations are fully hoisted and
immune to this entire bug class.** This is exactly the pattern used
for `lastRealBarIndex` (Section 4) - deliberately a `function`
declaration so it's safely callable from code earlier in the file
regardless of its own textual position.

**Extending an app's data to include future-dated placeholder entries
breaks any code that assumed "last array item = the current/latest
real one."** See Section 4's writeup - this is a general lesson beyond
just this specific feature: any time historical data gets extended
with placeholder/future/synthetic entries at either end, search for
every place indexing via `.length - 1` (or equivalent "last item"
logic) before shipping, not just the one call site that prompted the
change.

**Resizable + repositionable panes: a pane's height goes through a
single `applyLayout()`-style function that (a) sets the container's
CSS height, (b) repositions any drag handle to whichever edge borders
the neighboring pane, and (c) calls the chart library's own
`applyOptions({width, height})` to force a redraw** - the container
resizing via CSS alone does NOT make the chart library redraw itself,
same class of gotcha as the `fitContent()` issue in Section 4. Position
toggling (top/bottom) is a single CSS `order` flip on the flex
container, no DOM reordering needed. See `rsiLayout`/`applyRsiLayout`/
`resizeChartsForRsiLayout` in `index.html` for the concrete pattern -
worth reusing directly for a second indicator pane.

**Two-step "live preview vs. saved default" is now a repeated pattern
across three different settings surfaces** (drawing-tool Style panel,
drawing-tool Timeframe panel, RSI Settings panel) - every field change
applies immediately so you see the effect, but only becomes "the
default for next time" when "Set as Default" is explicitly clicked;
"Restore Default" reloads whatever's currently saved (or the factory
default if nothing's been saved), discarding unsaved live tweaks. Each
of the three currently has its own separate implementation of this
pattern rather than one shared helper - a reasonable target for
consolidation if a fourth settings surface is ever added.

**Draggable floating panels share one helper, `makeDraggable(panelEl,
handleEl)`** - converts whatever positioning got the panel there
(a centering transform, a corner anchor) into explicit top-left pixels
on the first mousedown, using the panel's actual on-screen
`getBoundingClientRect()`, then just tracks mouse delta from there.
Works regardless of the panel's original CSS positioning scheme.

**Synthesizing plausible intraday data from real daily OHLC:** the
technique used to extend 15m's historical range (Section 4) -
constrained random walk from the day's open to its close, forced to
touch the day's actual high and low at some point, small bar-to-bar
noise layered on top, volume/turnover split across the day's bars with
light randomness, EMA columns held flat at that day's daily EMA value.
Reusable directly if 5m ever needs the same treatment (Section 0, item
4), or if a future indicator needs synthetic historical intraday data
for any other reason.

**The main chart's logarithmic price scale is a recurring source of
subtle bugs for anything involving slope/extrapolation** - see the
previous version of this document for the two concrete examples (a
Ray's move-handle landing off the line, and dragging changing a
shape's angle). Still true, no new incidents this session.

**RSI needed a genuinely separate chart instance, not a shared-canvas
trick.** Unchanged from last handoff.

**A missing `min-height: 0` / `min-width: 0` on CSS Grid items caused
two separate "impossible" layout bugs** previously. Unchanged, no new
incidents this session, but keep this class of issue in mind if a
similarly "impossible" layout bug appears again.

**When debugging "creates successfully but doesn't render" (or "should
work but doesn't"), build a real headless-JS simulation rather than
guessing repeatedly.** Reinforced heavily again this session - used to
confirm the TDZ crash above (reproduced the exact error in isolation
before fixing), to verify the timeframe cumulative-range/radio-button
logic at each redesign step, to verify the resize-drag direction math
for both RSI pane positions, and to verify the future-bars data
generator's output (chronological order, valid OHLC relationships,
exact match against real daily bars) before ever shipping it. Also used
direct browser console inspection (`mainPane.userDrawings[0]`,
expanded fully) to settle a case where the reported symptom didn't
match what pure code-reading predicted - turned out to be a data-
coverage issue, not a logic bug (Section 4/6). **When a user's
report contradicts what the code should do, get the actual runtime
state (console inspection or a targeted simulation) before assuming
either "user is misreading the UI" or "there's a hidden bug" - both
have turned out to be true at different points in this project.**

**Earnings badges are plain HTML, not Lightweight Charts markers, and
use a rounded square, not a pentagon.** Unchanged from last handoff.

**For "must sit at an exact, predictable pixel position" requirements,
prefer plain CSS over fighting a charting library's own coordinate
system.** Unchanged from last handoff.

**GitHub Pages URLs are case-sensitive.** Unchanged from last handoff.

**Browser caching caused repeated false "it's still broken" reports.**
Still true, still the first thing to check - confirmed useful again
this session (several rounds of "check the build-number watermark and
hard-refresh" before investigating further, more than once correctly
resolved what looked like a bug).

**The right sidebar was removed entirely.** Unchanged from last
handoff.

---

## 8. Known limitations / deferred items

See Section 0 for the actively-planned next steps. Additionally:

- **"RichRoad MA v3"** (an indicator visible in the user's real
  TradingView setup) is still not replicated.
- **Rays are not truly infinite** - see Section 6.
- **Only 11 of ~155 typical scan-day stocks are in the watchlist.**
- **The text protocol doesn't carry style** - Section 0 item 5.
- **5-minute data's historical depth is unchanged (~9-14 days)** -
  Section 0 item 4, Section 4.
- **No persistent storage** - annotations live only in the browser's
  current session/localStorage, lost on switching ticker or clearing
  browser data. Discussed at length, deliberately deferred - Section 0
  item 2.
- **Rectangle, Arrow, Text, Price Range, Date Range, Date and Price
  Range, Path tool, Highlighter, Eraser** - none built yet, Section 6.
- **No standalone Technical Reference doc exists yet** despite this
  being flagged as needed twice now - Section 0 item 1. The gap
  between "what's in this document" and "what would be needed to
  rebuild the app from scratch on another platform" is real and has
  grown this session, not shrunk.
