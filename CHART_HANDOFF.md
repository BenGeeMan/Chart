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
small change. **The user has now explicitly confirmed this end-of-
session documentation pass is something they want done every session,
not just this once** — treat it as standing practice, not a one-off.

---

## 0. NEXT SESSION PRIORITIES (read this first)

**Framing for this update:** the previous session added features
(future placeholder bars, RSI settings, per-tool defaults). This
session was almost entirely the opposite — hardening the relationship
between the main chart and RSI (sync, gap display, data reservation)
through a long chain of real, reproducible bugs, most of which only
surfaced once RSI was set to a *different* timeframe than the main
chart and zoomed/scrolled/switched in combination. Build number jumped
from 68 to **82**. Section 7 has the full list of hard-won lessons —
read it before touching pan/zoom sync, future-bar handling, or
anything involving `getVisibleRange()`/`setVisibleRange()` again.

Carried over, still true:

1. **"Rebuild this project on another platform" risk — gap has grown
   again, not shrunk.** This session added a genuinely substantial
   piece of undocumented architecture: the whole cross-pane time sync
   system (`sourceVisibleTime`, `logicalIndexToTFTime`,
   `timeToLogicalIndex`, `convertTimeBetweenTF`, `suppressSync`, the
   settle-then-reassert pattern) plus the gap-extension feature
   (`buildLastValueExtension`, `rsiExtensionSeries`, its own settings
   section). All of it lives only in `index.html` and this document's
   Section 7 — there is still no standalone Technical Reference doc.
   Same recommendation as last time, now with more urgency: write that
   doc (data schemas, localStorage formats, the sync model, the
   timeframe-visibility model, color/layout constants) before more
   features get layered on.
2. **Persistent storage for annotations** — still not started, still
   deliberately deferred. No decision made yet on GitHub-repo-as-
   storage vs. Firebase/Supabase vs. the user's own Google Drive.
3. **Future indicator panes need full parity with RSI — and RSI's own
   template just got bigger.** Beyond the settings panel, own
   timeframe selector, and drawing tools already noted last time, a
   second indicator pane would now also need: whitespace points on its
   future placeholder region (Section 4), its own
   `buildLastValueExtension`-style gap-bridging line and matching
   settings section (Section 5), and to participate correctly in the
   sync system (Section 7) — none of the sync/whitespace machinery is
   factored out into a reusable per-pane abstraction yet; it's written
   specifically in terms of `chart`/`rsiChart`. Worth factoring out
   *before* a second indicator is added, not after — several of this
   session's bugs came from exactly that kind of implicit,
   RSI-specific assumption.
4. **Extend 5-minute data's historical depth** — still not done, still
   deliberately deferred (Section 4).
5. **Text protocol still doesn't carry style** — unchanged from last
   session.
6. **Still not built:** Rectangle, Arrow, Text, Price Range, Date
   Range, Date and Price Range, Path tool, Highlighter, Eraser
   (Section 6).
7. **Only after 1-6 are solid**, move on to the actual end goal:
   teaching Claude the user's trading-decision logic through this
   tool. Not started.

New from this session:

8. **One known, accepted gap in the sync/gap-extension work:** if RSI
   is independently pinned to a coarser timeframe (e.g. Quarterly) and
   the *main* chart's own timeframe is then changed (e.g. Daily →
   Hourly), the gap-extension line's target date doesn't immediately
   recompute — it only updates the next time RSI's own timeframe is
   touched, or the main chart happens to pass through
   `rsiTF === tf`. In practice this barely matters, since "today" is
   nearly identical between Daily and Hourly, but it's a real,
   deliberately-accepted gap, not an oversight to silently fix later
   without checking in first.
9. **The sync system was tested extensively but not exhaustively.**
   During testing, an extreme, unrealistic overshoot (asking
   `setVisibleLogicalRange` for a range ~10x the loaded data) produced
   a visibly wrong, narrow result from the charting library itself —
   not something `sourceVisibleTime`'s per-edge logic was designed to
   catch, since real user scrolling doesn't reach that range (the
   library's own `minBarSpacing` zoom-out cap kicks in well before
   it). Noted rather than chased down; revisit only if a real user
   report ever traces back to it.

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
| `index.html` | The entire app - a single self-contained HTML file. Must keep this exact filename; GitHub Pages requires it. Contains a visible build-number watermark (currently **82**), manually incremented on every handoff, purely so the user can confirm a hard refresh actually picked up new code after repeated caching confusion during development. |
| `chart_watchlist.json` | Quote summary for the 11 watchlist stocks. |
| `chart_earnings.json` | Earnings dates (past ~3 years + upcoming) per watchlist stock. |
| `{ticker}_{timeframe}.json` | Price + indicator data per stock per timeframe (77 files: 11 stocks × 7 timeframes). All 77 extend past "today" with future placeholder bars (flat OHLC in the raw JSON) - see Section 4 for how `index.html` actually renders that trailing region now, which changed significantly this session. |
| `.github/workflows/chart_export.yml` | Manual-trigger workflow that zips the repo for handoff to an AI assistant. |

**Naming convention:** bias toward "chart" in new filenames. Exception:
`index.html`.

**Hosting:** GitHub Pages, public repo (free-tier limitation — Pages
needs a paid plan for private repos).

---

## 3. The text protocol (as of build 82, unchanged this session)

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
  protocol** — a pasted shape's non-color style comes from the tool's
  saved default, not always a hardcoded factory value; only color
  comes from the pasted command's own `color=` attribute.
- **Still needs the tools from Section 0, item 6** before the protocol
  can grow to cover Rectangle/Arrow/Text/measurement tools.

---

## 4. Data — what's pre-computed vs. calculated client-side

The original data export included `ema_10/20/50/100`, `turnover`,
`turnover_ema_3/20` per bar. Everything else is calculated live in the
browser:

- **EMA 200** - not in the original export.
- **Volume EMA 3 / Volume EMA 20** - export only had *turnover* EMAs,
  not raw *volume* EMAs (the panel changed from dollar-turnover to
  share-volume after export).
- **RSI(14)** - never in the export.
- **Every Timeframe EMA / Relative EMA overlay** - each re-fetches that
  timeframe's raw file and computes EMA(10) fresh every time it's
  toggled on.

Still item 3 in Section 0's next-session plan - moving this into
`fetch_stock_timeframes.py` in the sibling repo, not yet started.

**Multi-timeframe time-format handling:** daily-and-above stores `time`
as a date string; intraday stores Unix timestamps. Overlaying one
timeframe's data onto a chart of a different timeframe needs
conversion - see `alignTimeToCurrentChart()` and `timeToNumeric()` /
`extrapolatePrice()` in `index.html`. This session added several more
conversion helpers alongside these for the sync system - see Section 7.

**Current watchlist (11 stocks):**
`HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS, CVSA, RMAX` — one
per sector, chosen for turnover/variety.

### Future placeholder bars: raw data vs. how `index.html` actually renders them (revised this session)

The raw JSON files still work exactly as documented previously: every
`{ticker}_{timeframe}.json` file extends past its last real bar with
flat placeholder rows (OHLC all equal to the last real close, volume/
turnover zero, EMA columns held at their last real value). Counts per
timeframe (daily +75 trading days, weekly +16 weeks, monthly +6
months, quarterly +3 quarters, hourly +30 trading days, 15m +12
trading days, 5m +6 trading days) and 15m's historical-depth extension
(31 → 138 days, matching Hourly) are unchanged - see previous versions
of this doc for the full synthesis methodology if that ever needs
revisiting (still reusable directly for 5m per Section 0 item 4).

**What changed this session is how `index.html` treats that trailing
region for each series it plots - it no longer plots the raw flat
values for anything:**

- **Candlesticks:** used to plot the raw flat OHLC rows directly. This
  turned out to be a real, visible bug (see Section 7) - a run of
  ~75+ identical flat candles with black borders visually merges into
  a solid horizontal line across the future space. Fixed by converting
  every future row to a Lightweight Charts **whitespace point**
  (`{ time }`, no `open`/`high`/`low`/`close`) before calling
  `series.setData()` - this still reserves that time on the chart's
  axis (so scrolling/drawing into it still works), but renders
  nothing. Volume histogram bars are left as-is (volume is already
  zero there, so they're already invisible) but did need
  `priceLineVisible: false` added (Lightweight Charts' own default
  "last value" reference line was extending across the reserved
  future space otherwise).
- **Price EMAs (10/20/50/100/200), the volume/turnover EMA, and RSI:**
  used to plot the raw flat/held values across the whole future
  region too (a genuinely flat but real-looking continuation of the
  indicator). Now truncated to stop at `lastRealBarIndex()` - only
  earnings markers are meant to occupy the future space, per explicit
  user direction. **RSI specifically still needs whitespace points for
  its OWN future region even though its real line stops early** - see
  Section 7 for why omitting that (truncating outright instead of
  whitespace-padding) caused a real, hard-to-diagnose sync bug.
- **The gap-extension line (new this session):** when an indicator's
  own timeframe is coarser than the main chart's, its last real bar
  can be meaningfully behind the main chart's "today" (e.g. Quarterly
  RSI won't have a bar for the current quarter until it closes). A
  second series, `rsiExtensionSeries`, carries the indicator's last
  real value forward flat from its own last real bar to the main
  chart's current real boundary, styled independently (its own color/
  width/line-style, configurable via RSI Settings → Extension Line -
  see Section 5) so it reads as "not real data" rather than actual
  computed values. The bridging logic itself,
  `buildLastValueExtension(lastRealPoint, targetTime)`, is written
  generically (indicator-agnostic) specifically so a future indicator
  pane can reuse it via its own second series - see Section 0 item 3.

**Two follow-on bugs from the ORIGINAL future-bars work, still true,
worth remembering for any future data-extension work:**
1. **`fitContent()` fits ALL loaded bars**, including the flat future
   ones - a calculated `setVisibleLogicalRange` showing real bars and
   future bars in roughly equal proportion is used instead (both the
   main chart and RSI's own initial-view logic need this).
2. **Several places assumed `array.length - 1` meant "today's bar"** -
   `lastRealBarIndex(bars)` (scans backward for the first bar with
   non-zero volume) is the fix, used throughout. **If more future-dated
   data is ever added to this app, grep for `.length - 1` first.**

---

## 5. Visual design (current state, build 82)

**Layout:** Watchlist (left) | main area | RSI pane (bottom by
default, repositionable to top). Header row: ticker/name, timeframe
buttons (5m/15m/1H/D/W/M/Q), the drawing toolbar, a thin vertical
divider, then far-right icon buttons — 🗑 Clear all, 🗒 Copy chart state
text, 📄 Save chart state text to file, 📥 Paste from Claude, 📋 Copy
image, ⬇ Save image, ✦ Copy image for Claude. A thin status bar sits
directly below the header.

**Main chart:** candlesticks, semi-log (logarithmic) price scale,
wicks/borders always black. Candlestick series has `priceLineVisible:
false` (added this session - see Section 4). Header stats line is
hover-reactive, defaulting to the latest **real** bar (via
`lastRealBarIndex`), not the last array index.

**Volume panel:** bottom ~25% via scaleMargins trick. Histogram series
also has `priceLineVisible: false` now (Section 4).

**RSI panel:** separate chart instance, linear 0-100 scale, own
independent timeframe selector.
- **⚙ RSI Settings panel** (gear icon in the RSI header row, renders
  centered in the main chart area). Sections, in order: **Pane**
  position (Top/Bottom), **RSI Line** style, **Extension Line** style
  (new this session - see below), **Upper/Middle/Lower** levels, each
  with its own on/off + threshold value AND its own independent color/
  thickness/line-type. Defaults: Upper 60 on, Lower 40 on, Middle 50
  off. Changes apply live; **Set as Default** persists to localStorage
  (`rsiSettings_v1`); **Restore Default** reloads whatever was last
  saved (or factory defaults), discarding live-but-unsaved tweaks,
  including resetting pane height to 150px (position is left alone).
- **Extension Line section (new this session):** same
  color/thickness/line-style control pattern as RSI Line and the
  threshold levels (all four share one `makeStyleRows()` helper).
  Independent color (factory default `#9a9ea6`, a muted gray - NOT
  derived from the real RSI line's own color), thickness (factory
  default 4px - deliberately thicker than the real line's 1px default,
  since a short bridging segment at 1px was hard to see, especially
  with sparse dot spacing), and line style. The "Line" dropdown across
  all four style sections now offers **four** options, not three:
  Solid, Dashed, **Dotted (tight)**, **Dotted (wide)** - see Section 7
  for why this needed a new `LINE_STYLE_MAP` key (`tightDotted`)
  rather than repointing the existing `dotted` key.
- **Mouse-resizable** (80-500px), **repositionable to top**, both
  auto-persist immediately (`rsiLayout_v1`).

**Drawing toolbar:** one shared toolbar in the header, same 5 buttons
(Cursor, Trend Line, Horizontal Line, Ray, Horizontal Ray) driving both
the main chart and RSI pane via one shared `sharedToolState`.

**Floating panels (Style, Timeframe, RSI Settings) are all
draggable** via one shared `makeDraggable(panelEl, handleEl)` helper.

**Timeframe visibility model:** unchanged from last session - two
independent columns (Absolute per-TF checkboxes, off by default;
Relative higher/lower, cumulative ranges, radio-button style, on by
default), new drawings default to "2 higher AND 2 lower."

**Per-tool style/timeframe defaults:** unchanged from last session -
each of the 4 line tools independently remembers its own saved style
and timeframe-visibility defaults (`chartToolDefaults_v1`).

**Indicator legend, earnings badges, screenshot capture:** unchanged.

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
🕐 Timeframe panel (visibility controls). Both the main chart and the
RSI panel have this full set via the same shared `createDrawingPane()`
factory (Section 7).

### Midpoint handle bugs fixed this session (a chain of three, same underlying symptom)

The move-handle for a selected Trend Line/Ray (the small square at the
segment's midpoint, used to drag the whole shape) would visibly float
off the actual rendered line under several distinct triggers. All
three were confirmed as genuine timing/staleness issues, not logic
bugs, by directly inspecting live chart state rather than guessing:

1. **Right after a timeframe switch**, if the shape was still selected
   from having just been drawn - the chart's internal coordinate-
   conversion state could still be settling from the just-applied
   `setVisibleLogicalRange()` call at the moment the handle position
   was read. Fixed by deferring one extra `renderSelectionOverlay()`
   pass via `requestAnimationFrame` after a timeframe switch.
2. **During mouse-wheel zoom/pan**, the same class of staleness, but
   firing on every scroll step via `subscribeVisibleLogicalRangeChange`.
   Generalized into a shared, rAF-coalesced `scheduleOverlaySync()`
   helper (coalesced so a continuous scroll doesn't queue a growing
   backlog of redundant frames) used in both places instead of two
   separate fixes.
3. These two fixes together were sufficient — no further midpoint-
   handle drift found after (2) shipped.

**Root cause pattern worth remembering:** the handle overlay is plain
absolutely-positioned HTML, computed once synchronously via
`timeToCoordinate()`/`priceToCoordinate()` at render time, while the
charting library's own canvas repaint can settle on a *later* frame
after certain state changes (data swaps, range changes). Any code that
reads coordinate-conversion functions synchronously right after
mutating the chart needs to consider whether a deferred re-read is
needed - see Section 7 for the much deeper version of this same
pattern found in the sync system.

### The timeframe-visibility model's design history (from the previous session, unchanged)

See the previous version of this document (or search git history) for
the full multi-iteration writeup if the design gets revisited -
compressed here since it's settled, stable, and wasn't touched this
session: bypass-switch masters (not read/write), cumulative "higher N"
ranges, radio-button display, "current" implied by any picked range,
and the root-cause finding that an apparent visibility bug was
actually a data-coverage limitation (intraday files not extending far
enough back), not a logic bug.

**Ray/Horizontal Ray extension nuance (still true):** neither is truly
infinite - both extend only as far as the currently *loaded* data,
computed once at draw time.

---

## 7. Architecture notes worth knowing (hard-won, don't repeat these)

**Drawing tools are one shared, pane-agnostic implementation
(`createDrawingPane()`), not copy-pasted per pane.** Unchanged - still
the right template for adding drawing tools to any future indicator
pane (Section 0, item 3).

### This session's real subject: cross-pane pan/zoom sync between the main chart and RSI

This took a long chain of genuinely distinct bugs to get right, each
confirmed live (never fixed on theory alone) via direct chart-state
inspection, monkey-patched API instrumentation, or pixel-level
measurement. Read this whole block before touching sync again - fixing
one of these without understanding the others tends to reintroduce a
previously-fixed one.

**1. `setVisibleRange()` (time-based) silently CLAMPS when the target
date is beyond the chart's own loaded data - it has nothing to anchor
a time value to out there.** Confirmed directly: asked a chart to show
a date a year past its own last bar, and it just clamped back to its
own last bar instead of erroring or showing blank overscroll.
**`setVisibleLogicalRange()` (index-based) has no such problem** - it
happily accepts indices outside `[0, bars.length-1]` and shows blank
margin. Every place in the sync system that needs to position a chart
at a calendar time now converts that time to a logical index first
(`timeToLogicalIndex(time, bars)`) and calls `setVisibleLogicalRange`,
never `setVisibleRange`, for anything that might be out of bounds.

**2. `getVisibleRange()`'s clamping behavior is ASYMMETRIC between the
two edges, and that asymmetry matters for sync.** The "from" (past)
edge has no reserved space before the first real bar - once scrolled
past it, `getVisibleRange()` correctly clamps to the true first-bar
date, and trusting that clamped value for sync is *correct* (it
matches what's actually displayed). The "to" (future) edge DOES have
reserved space (the whitespace-padded future placeholder bars) - once
scrolled past even that, `getVisibleRange()` still returns a value,
but a clamped one that silently hides how much blank overscroll margin
is actually being shown. Trusting that clamped "to" for sync makes the
other pane's pixel width diverge from this one's, even though their
reported dates stay close - confirmed directly via pixel measurement
(candles stopped ~200px before the container's right edge while RSI's
line ran almost all the way to it, both consistent with what
`getVisibleRange()` was reporting). Fixed with `sourceVisibleTime()`,
which checks each edge INDEPENDENTLY against the bounds of the pane's
own bars array and only trusts `getVisibleRange()` for whichever
edge(s) are still within it - falling back to `logicalIndexToTFTime`'s
own extrapolation (average bar-to-bar spacing) for whichever edge(s)
aren't.

**3. A single boolean re-entrancy guard, reset synchronously right
after the nested call it protects, is NOT enough to prevent a
sync bounce-back loop.** The assumption "the other chart's own
range-change event fires synchronously within this call" is usually
true, but confirmed FALSE right after a big data swap (e.g. a
timeframe switch) - the chart library fires a *cascade* of several
intermediate range-change events over the next few frames as it
internally reconciles, not just one, and some of those fire later than
the guard's synchronous reset. Direct console instrumentation (logging
every sync event with its guard state and a timestamp) caught a real
~10-event cascade with overlapping fire/bounce pairs slipping through
gaps between one guard resetting and the next being set. Fixed two
ways together: (a) deferred the guard's reset by a frame
(`requestAnimationFrame`) instead of resetting synchronously, and (b)
added a separate `suppressSync` flag that silences BOTH sync
directions entirely during a timeframe-switch transition, then
explicitly re-asserts the correct final state on both panes once
things settle, rather than trusting the cascade of individual events
to have propagated correctly.

**4. "Settling" after a data swap isn't reliably done after 2 animation
frames - the charting library can apply corrections OUTSIDE
`requestAnimationFrame` timing entirely.** Wrapping setVisibleRange in
`requestAnimationFrame(() => requestAnimationFrame(fn))` (2 frames) was
tried first and mostly worked, but direct instrumentation (monkey-
patching `setVisibleRange`/`setVisibleLogicalRange` to log every call,
its arguments, a timestamp, and a stack-trace line) caught the
charting library's OWN internal code calling `setVisibleRange` with a
stale value ~10ms *after* the 2-frame settle had already re-asserted
the correct one - consistent with a ResizeObserver-driven internal
correction, which runs on its own schedule, not paint frames. Fixed by
ALSO re-asserting the final state once more after a plain
`setTimeout(..., 250)`, so the app's own value is what wins the last
word regardless of when the library's internal correction fires.

**5. A separate, unrelated bug was hiding inside the RSI-side symptom:
truncating an indicator's real data outright (instead of whitespace-
padding the future region, same as the candlesticks) leaves that
chart's timeScale with literally no record that the future time exists
at all.** Discovered while chasing bug #2/#3 above - RSI's own
`rsiSeries` had been truncated to real bars only (a deliberate, correct
fix from earlier in this same session, for a different reason - see
Section 4), but without whitespace points standing in for the removed
future rows. Any later `setVisibleLogicalRange`/`setVisibleRange` call
targeting a position in that unreserved space had nothing to anchor
to, and silently landed somewhere unrelated and wrong. Fixed by
appending whitespace points for RSI's own future region too, exactly
mirroring the candlestick approach (Section 4) - **any indicator pane
that truncates its real line for the future-space policy MUST also
whitespace-pad the same region, or its own chart's coordinate space is
broken for anything positioning against it.**

**6. A page-load init-order race, surfaced by testing this session's
UI work, not by anything sync-related per se.** RSI's own initial data
fetch (kicked off early in the script, inside `createRsiChart()`) can
resolve *before* the main chart's own `loadTimeframe('daily')` call
(much later in the script, both fetching the same local file) does,
leaving `barsData` still `[]` when code that assumed it was ready ran.
`barsData[lastRealBarIndex(barsData)]` on an empty array is
`undefined`, and reading `.time` off that threw - caught by the
enclosing `.then()`'s own `.catch()`, which then incorrectly showed a
misleading "could not load" / No Data state even though the actual
`rsiSeries.setData()` call just above it had already succeeded with
valid data. **General lesson: any code that runs inside page-load-time
async callbacks and reads global state populated by a DIFFERENT
async callback must not assume that other callback has resolved yet -
check bounds/length before indexing, matching the pattern the
PRE-EXISTING code around it already used** (the existing "sync RSI's
initial view to main" logic was already correctly guarded this way;
the new gap-extension code added this session initially wasn't).

**Debugging methodology, reinforced heavily this session - the thing
that actually cracked most of the above, after guessing repeatedly
didn't:** wrap the actual API calls in question
(`chart.timeScale().setVisibleRange`, `setVisibleLogicalRange`, etc.)
to log every call with its full arguments, a timestamp
(`performance.now()`), and a stack-trace line, THEN reproduce the
exact user-reported interaction, THEN read the log as a literal
sequence of events. Reading intent from `getVisibleRange()` snapshots
taken *after the fact* repeatedly gave misleading or contradictory
results (see points 3 and 4 above) until the actual call sequence was
captured directly. This is a stronger, more specific version of the
existing "build a real headless-JS simulation rather than guessing
repeatedly" lesson from previous sessions - when a *sequence and
timing* of events is the suspected issue (not just a wrong value), log
the calls themselves, not just periodic snapshots of state.

**Reproduction environment notes** (about the debugging tools, not the
app - useful if this comes up again): a single scripted mouse drag
(via the browser automation tool, or synthetic `MouseEvent`/
`PointerEvent` dispatch) does not reliably register as a pan gesture
with this charting library - it needs many small intermediate move
events, which chunky discrete jumps don't provide. Real rapid wheel-
zoom bugs (the cascade in point 3) were only caught by dispatching many
small raw `WheelEvent`s in a tight loop (~16ms apart) rather than a few
large discrete scroll calls - the latter completely missed the
cascade. Also: this project's `file://`-based preview tabs were
observed reusing cached JS state across `navigate` calls to the same
URL in this specific automated testing setup (not a real browser
behavior) - closing the tab and opening a genuinely new one, or using
the `http://localhost` dev server instead, was needed for a reliably
fresh reproduction.

### Other architecture notes (from previous sessions, still true, not touched this session)

**The temporal-dead-zone crash class:** a top-level `const` referenced
by code that runs synchronously early in the script (before the
`const`'s own declaration line executes) throws
`ReferenceError: Cannot access 'X' before initialization` and blanks
the whole page. Prefer `function` declarations (fully hoisted) for
anything that might be called early, over `const () => {}` arrows.
`LINE_STYLE_MAP` hit this once; `lastRealBarIndex` and most of this
session's new sync helpers are deliberately `function` declarations
for this reason. **This session added a new `LINE_STYLE_MAP` key,
`tightDotted`** (`LightweightCharts.LineStyle.Dotted`) **alongside the
existing `dotted`** (`LightweightCharts.LineStyle.SparseDotted`, wide
gaps) **- deliberately did NOT repoint the existing `dotted` key**,
since it's already used by saved drawing-tool/RSI-threshold settings
in the wild; changing its meaning would have silently altered how
already-saved user settings render.

**Extending an app's data to include future-dated placeholder entries
breaks any code that assumed "last array item = the current/latest
real one."** Any time historical data gets extended with placeholder/
future/synthetic entries at either end, search for every place indexing
via `.length - 1` before shipping.

**Resizable + repositionable panes** go through a single
`applyLayout()`-style function that sets CSS height, repositions the
drag handle, and calls the chart library's own
`applyOptions({width, height})` to force a redraw - CSS alone does not
make the chart library redraw itself. See `rsiLayout`/`applyRsiLayout`/
`resizeChartsForRsiLayout`.

**Two-step "live preview vs. saved default"** is a repeated pattern
across drawing-tool Style/Timeframe panels and RSI Settings (now
including the new Extension Line section) - every field applies
immediately, "Set as Default" persists it, "Restore Default" discards
unsaved live tweaks. Still no shared helper across the surfaces that
use this pattern - a reasonable consolidation target if a fourth/fifth
surface is added.

**Draggable floating panels share one helper,
`makeDraggable(panelEl, handleEl)`.**

**Synthesizing plausible intraday data from real daily OHLC** (used to
extend 15m's historical range) - reusable directly if 5m ever needs the
same treatment.

**The main chart's logarithmic price scale is a recurring source of
subtle bugs for anything involving slope/extrapolation** - no new
incidents this session, but the underlying risk (linear price math in
log-price space computing a mathematically-straight-but-visually-wrong
line) is exactly the same *category* of bug as this session's
`setVisibleRange` clamping issue: an API that looks like it should
"just work" for out-of-range/non-linear inputs, but doesn't, silently.
Worth the reminder when touching either area again.

**RSI needed a genuinely separate chart instance, not a shared-canvas
trick.**

**A missing `min-height: 0` / `min-width: 0` on CSS Grid items caused
two separate "impossible" layout bugs** previously - keep in mind for
any future "impossible" layout bug.

**When debugging "creates successfully but doesn't render" (or "should
work but doesn't"), build a real headless-JS simulation rather than
guessing repeatedly** - reinforced again this session (see the
dedicated debugging-methodology note above for the more specific
version that emerged this time: log the actual API calls, not just
periodic state snapshots, when timing/sequencing is the suspect).

**Earnings badges are plain HTML, not Lightweight Charts markers, and
use a rounded square, not a pentagon.**

**For "must sit at an exact, predictable pixel position" requirements,
prefer plain CSS over fighting a charting library's own coordinate
system.**

**GitHub Pages URLs are case-sensitive.**

**Browser caching caused repeated false "it's still broken" reports** -
still the first thing to check; the build-number watermark exists
specifically for this.

**The right sidebar was removed entirely.**

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
  browser data. Deliberately deferred - Section 0 item 2.
- **Rectangle, Arrow, Text, Price Range, Date Range, Date and Price
  Range, Path tool, Highlighter, Eraser** - none built yet, Section 6.
- **No standalone Technical Reference doc exists yet** despite being
  flagged as needed for three sessions running now - Section 0 item 1.
  The gap between "what's in this document" and "what would be needed
  to rebuild the app from scratch on another platform" grew
  significantly again this session (the whole sync system in Section
  7 has no home outside this doc and the code itself).
- **The gap-extension line's target doesn't live-recompute when the
  main chart's timeframe changes while RSI is independently on a
  different timeframe** - Section 0 item 8, a deliberately accepted
  gap, not an oversight.
- **The sync system has at least one known unhandled extreme-overshoot
  edge case** (Section 0 item 9) - not reachable through normal
  scrolling (the charting library's own zoom-out cap prevents it), so
  left alone.
