# chart — Technical Reference

**Purpose of this document.** `CHART_HANDOFF.md` explains *what the
project is, why decisions were made, and what's outstanding* — it is a
narrative for picking the work back up. This document is the opposite:
a **flat, factual specification** of the data formats, storage formats,
constants, and internal model that currently live *only inside
`index.html`*. It exists so the app could be **rebuilt on another
platform** (React, a real framework, a different charting library)
without reverse-engineering 4,400 lines of one HTML file first.

Section 0 item 1 of the handoff has flagged the absence of this
document as the single highest-priority gap for three sessions running.
This is that document.

**Scope boundary.** This file records *structure and contracts* —
schemas, key names, constant values, function signatures, the sync
model's invariants. It does **not** re-explain the reasoning behind
them; that stays in `CHART_HANDOFF.md` (cross-referenced by section
throughout). When the two disagree, the code is authoritative and both
docs should be corrected.

**How to keep this current.** When a schema, storage key, constant, or
core-model invariant changes, update the matching section here in the
same session — same end-of-session doc pass the handoff already
mandates. Line numbers below are written as *approximate anchors*
(`~1372`) because they drift; the surrounding constant/function *name*
is the durable locator, not the number.

All anchors below refer to `index.html` unless stated otherwise. Build
number at time of writing: **130**.

---

## 1. File & data inventory

| File | Kind | Notes |
|---|---|---|
| `index.html` | The entire app | Single self-contained file. Filename fixed (GitHub Pages). Visible build-number watermark at `#build-number` (~L887). |
| `{ticker}_{tf}.json` | Price+indicator series | 84 files = 12 tickers × 7 timeframes. Schema §2. `aapl_*` is real Yahoo data with no future placeholder bars (and `aapl_m15` is only 891 bars, Yahoo's ~60-day intraday cap); the other 11 are synthetic and padded past "today". |
| `chart_watchlist.json` | Watchlist quote summary | Schema §2.3. |
| `chart_earnings.json` | Earnings dates per ticker | Schema §2.4. |
| `.github/workflows/chart_export.yml` | CI | Manual-trigger zip of the repo for handoff. |

**Tickers (12):** `HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS,
CVSA, RMAX, AAPL` (AAPL = real-market reference data, added later).

**Timeframe keys (7), canonical order low→high** — `TF_ORDER` (~L1210):
```
['m5', 'm15', 'hourly', 'daily', 'weekly', 'monthly', 'quarterly']
```
Filenames use these exact keys: `hpq_m5.json`, `hpq_daily.json`, etc.
(ticker lowercased).

---

## 2. Data schemas

### 2.1 Price/indicator series — `{ticker}_{tf}.json`

**Top level: a JSON array** of bar objects, chronologically ascending.
Every bar object:

```json
{
  "time":            "2024-08-09",   // see 2.2 for the two time formats
  "open":            30.0876,
  "high":            31.0307,
  "low":             30.0876,
  "close":           30.7102,
  "volume":          6605500.0,
  "ema_10":          31.0931,
  "ema_20":          31.863,
  "ema_50":          31.7693,
  "ema_100":         30.4907,
  "turnover":        202856272.2425,
  "turnover_ema_3":  198743817.3098,
  "turnover_ema_20": 212021419.2512
}
```

Every field is present on every bar. `ema_200`, raw **volume** EMAs,
and **RSI** are **NOT** in these files — they are computed client-side
(§6). `turnover` is dollar-volume; the on-screen volume panel plots
share `volume` and derives its own EMAs, so the `turnover_ema_*` columns
are effectively unused by the current UI (kept from the export).

**Future placeholder rows.** Each file extends *past its last real bar*
with flat rows: OHLC all equal to the last real close, `volume`/
`turnover` = 0, EMA columns held at their last real value. Counts:
daily +75 trading days, weekly +16w, monthly +6m, quarterly +3q, hourly
+30 trading days, m15 +12 trading days, m5 +6 trading days. **The app
never renders these raw flat values** — see §5 (whitespace conversion)
and handoff §4. The reliable "last real bar" detector is
`lastRealBarIndex(bars)` (§7), **not** `bars.length - 1`.

### 2.2 The two `time` formats (critical)

| Timeframe group | `TIMEFRAME_META[tf].intraday` | `time` type | Example |
|---|---|---|---|
| `m5`, `m15`, `hourly` | `true` | Unix epoch **seconds** (int) | `1785324600` |
| `daily`, `weekly`, `monthly`, `quarterly` | `false` | Date string `"YYYY-MM-DD"` | `"2024-08-09"` |

This split is the root complexity behind the sync system (§4) and every
cross-timeframe overlay. Conversions go through `convertTimeBetweenTF`,
`timeToNumeric`, and `dateStringToTimestamp` (§7). `TIMEFRAME_META`
(~L1794) is the source of truth for which group a key is in:

```js
const TIMEFRAME_META = {
  m5:        { intraday: true,  label: '5-Minute',  short: '5m' },
  m15:       { intraday: true,  label: '15-Minute', short: '15m' },
  hourly:    { intraday: true,  label: 'Hourly',    short: '1H' },
  daily:     { intraday: false, label: 'Daily',     short: 'D' },
  weekly:    { intraday: false, label: 'Weekly',    short: 'W' },
  monthly:   { intraday: false, label: 'Monthly',   short: 'M' },
  quarterly: { intraday: false, label: 'Quarterly', short: 'Q' },
};
```

### 2.3 `chart_watchlist.json`

Top level: **array**, one object per watchlist ticker:
```json
{ "ticker": "HPQ", "name": "HP Inc.", "sector": "Technology",
  "last": 30.05, "change_pct": 6.64 }
```

### 2.4 `chart_earnings.json`

Top level: **object keyed by ticker**; each value an array of dated
entries (past ~3 years + upcoming):
```json
{ "HPQ": [ { "date": "2023-08-29", "is_future": false }, … ],
  "COHR": [ … ], … }
```
`is_future` distinguishes upcoming earnings (rendered in the reserved
future space) from historical ones.

---

## 3. localStorage formats

Several independent keys, all versioned `_v1`, all best-effort (wrapped in
try/catch — private browsing / disabled storage falls back to
in-memory, feature still works for the session). All are keyed by
constants, never bare strings: `rsiSettings_v1`, `rsiLayout_v1`,
`chartToolDefaults_v1`, `macdSettings_v1`, `macdLayout_v1`,
`turnoverStyle_v1`, `indicatorStyles_v1` (§3.8), `richRoadCandles_v1`
(§3.9). **`volumeColors_v1` is legacy** — volume colours moved into
`indicatorStyles_v1` (build 104); the old key is read once and migrated
(§3.6, §3.8).

### 3.1 `rsiSettings_v1` — RSI indicator styling

Const `RSI_SETTINGS_STORAGE_KEY` (~L1372). Factory default
`RSI_SETTINGS_FACTORY_DEFAULT` (~L1373):
```js
{
  line:      { color: '#131722', lineWidth: 1, lineStyle: 'solid' },
  upper:     { enabled: true,  value: 60, color: '#9a9ea6', lineWidth: 2, lineStyle: 'dashed' },
  middle:    { enabled: false, value: 50, color: '#9a9ea6', lineWidth: 2, lineStyle: 'dashed' },
  lower:     { enabled: true,  value: 40, color: '#9a9ea6', lineWidth: 2, lineStyle: 'dashed' },
  extension: { color: '#9a9ea6', lineWidth: 4, lineStyle: 'tightDotted' },
}
```
- `line` = the RSI line itself. `upper`/`middle`/`lower` = the three
  optional threshold levels, each independently styled and toggled.
  `extension` = the gap-bridging line (§4, handoff §4/§5).
- **Load is a per-section deep-merge** over the factory default
  (`loadRsiSettings`, ~L1385), so an older partial saved object still
  fills in missing fields. Any rebuild must preserve this forward-compat
  merge, not a blind `JSON.parse`.
- `lineStyle` values are `LINE_STYLE_MAP` keys (§8.2).

### 3.2 `rsiLayout_v1` — RSI pane geometry

Const `RSI_LAYOUT_STORAGE_KEY` (~L1423). Factory default
`RSI_LAYOUT_FACTORY_DEFAULT`:
```js
{ height: 150, position: 'bottom' }   // position: 'bottom' | 'top'
```
`RSI_MIN_HEIGHT = 80`, max 500 (mouse-resizable). Persists *immediately*
on change — no separate "set as default" step, unlike §3.3.

### 3.3 `chartToolDefaults_v1` — per-drawing-tool defaults

Const `TOOL_DEFAULTS_STORAGE_KEY` (~L1885). Default: `{}` (empty until
the user saves a default). Shape is keyed by tool type
(`trendline` | `ray` | `hray` | `hline`), each holding that tool's own
independently-saved **style** default and **timeframe-visibility**
default. Two-step model: fields apply live, **"Set as Default"**
persists, **"Restore Default"** discards unsaved live tweaks.

Factory style fallbacks (used when a tool has no saved default):
```js
const FACTORY_STYLE_DEFAULT_SHAPE = { color: '#2962ff', opacity: 1, lineWidth: 2, lineStyle: 'solid' };
const FACTORY_STYLE_DEFAULT_HLINE = { color: 'blue', colorHex: '#2962ff', opacity: 1, lineWidth: 2, lineStyle: 'solid' };
```
Shared between the main chart and RSI panes (it's about the tool, not
the pane).

### 3.4 `macdSettings_v1` — MACD indicator styling

Const `MACD_SETTINGS_STORAGE_KEY`. Factory default
`MACD_SETTINGS_FACTORY_DEFAULT`:
```js
{
  macd:   { color: '#2962ff', lineWidth: 2, lineStyle: 'solid' },
  signal: { color: '#ff6d00', lineWidth: 1, lineStyle: 'solid' },
  hist:   { upColor: '#26a69a', downColor: '#ef5350' },
}
```
Same per-section deep-merge load (`loadMacdSettings`) as `rsiSettings_v1`
(§3.1). `macd`/`signal` are line styles (`lineStyle` = `LINE_STYLE_MAP`
key, §8.2); `hist` is the histogram's up/down bar colors, applied at
paint time so a live change re-tints without recompute. Set/Restore
Default via the MACD settings gear.

### 3.5 `rsiLayout_v1` / `macdLayout_v1` — pane geometry

Each pane persists `{ height, position, priority }`. `position` is
`'bottom' | 'top'`; `priority` is 0–99. Factory defaults:
`{ height: 150, position: 'bottom', priority: 10 }` (RSI) and
`{ height: 140, position: 'bottom', priority: 20 }` (MACD). Height is
mouse-resizable (drag handle, 80–500px, persists immediately); position and
priority are controls in each pane's settings gear (Pane section).

**Priority ordering** (build 97): the shared `paneOrderValue({position,
priority})` maps a pane to a CSS flexbox `order` relative to
`#main-chart-area` (order 0): top → `-(priority+1)`, bottom → `priority+1`.
So among panes docked on the SAME side, LOWER priority sits NEARER the main
chart (RSI 10 next to the chart, MACD 20 beyond it), each side ordered
independently. Any future pane reuses `paneOrderValue`.

**Resize handle grip** (build 96): the drag edge shows a faint centred grip
line via the shared `.pane-resize-handle` class (`::after`) so it's
discoverable. Future panes get it by adding that class to their handle.

### 3.6 `volumeColors_v1` — volume bar colours (LEGACY, migrated)

**Superseded build 104.** Volume up/down colours + transparency now live in
`indicatorStyles_v1` under the `turnover` key (§3.8); the inline colour
pickers were removed and volume is styled via the same name-click popup as
every other line indicator. `loadIndicatorStyles()` reads the old
`volumeColors_v1` once and folds `{ up, down }` into `indicatorStyles.turnover`
if no new value exists (one-time migration). `volumeBarColor()` /
`paintVolume()` now read `indicatorStyles.turnover` (`{ up, down, opacity }`,
default `{ '#26a69a', '#ef5350', 0.45 }`).

### 3.7 `turnoverStyle_v1` — turnover info-box styling

`{ visible, bgColor, bgOpacity, textColor }` for the on-chart turnover
boxes (`#turnover-box` current-TF + `#turnover-box-daily`). Default
`{ visible: true, bgColor: '#ffffff', bgOpacity: 1, textColor: '#787b86' }`.
`applyTurnoverStyle()` sets background (`hexToRgba`), text colour (cascaded
onto the `.value`/`.label` spans), and display on both boxes. **UI:** a
show/hide **Turnover** toggle in the Indicators column (`data-series=
"turnoverbox"`); the colour/transparency controls open in a small draggable
popup (`#turnover-settings`) on **right-clicking the Turnover menu row**
(build 103 — was the box itself in 102).

### 3.8 `indicatorStyles_v1` — unified per-indicator styling (build 104)

One store for every styleable line/marker indicator EXCEPT RSI/MACD (own
panels) and the turnover box (§3.7). Const `INDICATOR_STYLE_KEY`; factory
`INDICATOR_STYLE_FACTORY` keyed by indicator id. Per-key deep-merge load
(`loadIndicatorStyles`) over factory, so partial saved objects still fill in.
Live edits mutate `indicatorStyles` and apply immediately; **"Set as Default"**
persists the whole object; **"Restore Default"** re-reads saved-or-factory for
that id (two-step, as §3.3). See §11 for the popup UI and the four style
*kinds*. Keys + factory shapes:
```js
// kind 'line' — EMAs, M10s, Relative M10s, Volume EMA 20:
{ color, opacity, lineWidth, lineStyle }   // applied via colorWithOpacity + LINE_STYLE_MAP
// kind 'volume' — the histogram (id 'turnover'):
{ up, down, opacity }
// kind 'earnings':
{ color, size }
// kind 'cross' — 'ema_cross' and 'close_above_20':
{ color, shape, size }   // shape ∈ circle|square|cross|plus|arrowUp|arrowDown
```
**Factory defaults of note:** the M10s column (`m5…quarterly`) and all
Relative M10s default to **black**; Relative M10s also default `solid`
(rel0) / `dashed` (rel1) / `dotted` (rel2). Event markers default orange
`#ff9800`: `ema_cross` = circle size 2, `close_above_20` = plus (`+`) size 3.
These factory values override the `MTF_EMA_COLORS`/`RELATIVE_EMA_COLORS`
creation-time colours (§8.1) at load via `applyAllIndicatorStyles()`.

### 3.9 `richRoadCandles_v1` — Rich Road Candles (build 113)

Const `RICHROAD_KEY`; factory `RICHROAD_FACTORY`. Shape (values match the original
RichRoad MA v3 Inputs as of build 121):
```js
{ enabled: true,
  enabledTypes: { highCB, lowCB, dc, dc2, redCB, neutralCB },  // per-type on/off
  useVolume:    { highCB:false, lowCB:false },                 // per-type volume gate (added build 119)
  colors:       { highCB, lowCB, dc, dc2, redCB, neutralCB },  // 6 hex
  params: { highAutoMove:9.5, highMinMove:4.8, highBodyRatio:0.8, highWickTol:5,
            lowMinMove:3, lowMaxBodyRatio:1, dcBodyRatio:0.7, dcWickTol:5, redMove:4,
            volEmaPeriod:20, volMultiplier:1.5 } }
```
Section-merged load (`loadRichRoad`). `enabled` is a runtime toggle that only
persists on Set as Default (like `rsiEnabled`). `enabledTypes` switches each
candle type on/off individually (an off type is left at default appearance).
See §9d for the classifier and per-candle recolouring.

- **`useVolume`** (build 119): per-type Require-volume toggle for High CB / Low CB.
  When on, that type also requires `volume >= volMultiplier × its own volume-EMA`
  (period `volEmaPeriod`) on the bar. **Off by default** — the original has no
  volume filter; this is a user enhancement. `paintCandles()` computes the volume
  EMA fresh (independent of the volume panel's EMA) and passes each bar's value
  into `classifyCandle(bar, volEma)`.
- **`lowMaxBodyRatio`** (build 129): the Low-CB/DC boundary dial. Low CB only
  applies when body ratio ≤ this. Default 1 = every ≥3% move is Low CB; lower it to
  hand strong-body ≥3% candles down to DC (0.7 ≈ all of them on AAPL).
- **`Restore Default` (build 128): resets to the shipped `RICHROAD_FACTORY` AND
  `localStorage.removeItem(RICHROAD_KEY)`** — was previously "reload the saved
  snapshot", which let a stale Set-as-Default silently shadow every pushed change.
  Symptom of a stale save: panel STRUCTURE (labels/layout, from code) updates but
  COLOURS/enabled (from the store) don't.
- Panel number fields (build 130): body-ratio (0–1) fields step by 0.05 on the
  spinner; move %/tolerance fields use `step="any"` so an off-step typed value
  doesn't trip the browser's red "!" validation marker.

**Settings panel (build 118).** `renderRichRoadPanel(clientX, clientY)` opens on
left- or right-click of the indicator name via `wirePanelName`, positioned **at
the cursor** like the style popups. The panel is appended to `<body>` (NOT
`#richroad-overlay`, which is inside the positioned `main-chart-area` and would
anchor it off-screen) with `z-index: 1000` (else it hides behind the Indicators
dropdown), and clamped fully on-screen. Position is remembered in
`richRoadPanelPos` across re-renders. `#richroad-overlay` is now unused.

---

## 4. The cross-pane sync model (main ↔ indicator panes)

The hardest and least-obvious subsystem. Full *why* is handoff §7; this
is the *contract* a reimplementation must satisfy. It keeps N independent
Lightweight Charts instances (main + each indicator pane — currently RSI
and MACD) showing the same calendar-time window even when each is on a
**different timeframe**.

### 4.0 N-pane sync — `broadcastRange`
Sync is pane-count-agnostic. A "pane" is a descriptor of three **getters**
— `{ chart(), bars(), getTF() }` — so one descriptor reads live state
forever (`rsiChart`/`macdChart` are reassigned on toggle-off/on, the bars
arrays on every load, the effective TFs on switch). The registry
`syncPanes = [mainSyncPane, rsiSyncPane, macdSyncPane]` holds one per pane.
Each pane's `timeScale().subscribeVisibleLogicalRangeChange` calls
`broadcastRange(thatPane, range)`. **Adding a pane = one descriptor +
one subscription line.**

`broadcastRange(srcPane, range)` (~L1338) is the whole model, in order:

1. **Hard-lock bounds** as logical indices *in the source's own bars*:
   `[minIdx, maxIdx]` = `mainDataBounds()` (main's first/last bar times)
   mapped through `timeToLogicalIndex(convertTimeBetweenTF(…))` into the
   source's indexing. For a same-TF source that's `[0, lastBar]`; for a
   coarser-TF source it's the sub-range overlapping main's data (so a
   weekly pane can't scroll back past main's first daily bar).
2. **Width-preserving clamp** of the incoming `range` into
   `[minIdx, maxIdx]`: when an edge hits a bound, **slide** the whole
   window rather than truncating that edge (only shrink if the window is
   wider than the allowed span). Truncating one edge while the other kept
   moving *shrank* the window, turning a pan at the boundary into a
   runaway zoom-in (build 93 fix — "drag to the edge and it accelerates
   away"). `clamped` = whether the range changed.
3. **Echo check, but only when `!clamped`** (build 91). An out-of-bounds
   (clamped) source MUST spring back regardless of whether the event looks
   like an echo; in-bounds echoes are still dropped so nothing bounces.
4. `syncInProgress = true`.
5. If `clamped`, snap the source itself back to the clamped range
   (`recordApplied` **before** the `setVisibleLogicalRange`, so the
   source's own resulting event is recognised as an echo).
6. Push to every other live pane: **same-TF → the source's exact logical
   range** (identical bar indexing, no rounding); **cross-TF → convert**
   the clamped window's edge times (`logicalIndexToTFTime` →
   `convertTimeBetweenTF` → `timeToLogicalIndex`). `recordApplied` before
   each set.
7. `syncInProgress = false` — **synchronous** (build 90). A deferred
   (rAF) reset could be starved during a rapid wheel-zoom burst and stay
   stuck `true`, freezing every later broadcast (indicators stopped
   following the zoom). `syncInProgress` only needs to block the
   *synchronous* re-entrant broadcasts that step 6's sets trigger; any
   later-frame echo is caught by the guards below instead.

**Echo suppression** (`isEcho`, ~L1310) recognises a pane reporting a
range *we* just set, so it isn't rebroadcast:
- **by value** — `lastAppliedRange` (WeakMap pane→range) records what we
  set; a match within `rangesClose` (tol 0.05 bar) is an echo. Catches
  late echoes regardless of frame timing.
- **by recency** — `lastAppliedAt` (WeakMap pane→timestamp); an event
  from a pane we set within `ECHO_WINDOW_MS` (150ms) is an echo *even if
  the value differs*. Needed when we ask a pane for a range it can't show
  and it **clamps** to a different one (its clamped echo won't value-match)
  — without this the panes fought.

`reassertIndicatorRange(pane, from, to, mainData, mainTF)` (~L1428) is the
settle-then-reassert step used by `loadTimeframe` (one call per pane).

`suppressSync` still hard-mutes all of `broadcastRange` during a
TF-switch transition. The old pairwise `syncingFromMain`/`syncingFromRsi`
booleans are gone (didn't generalize past two panes). `sourceVisibleTime`
and `paneRangeForWindow` remain defined but are **no longer called** (the
build-93 rewrite clamps in logical space directly) — dead code, safe to
delete.

### 4.1 State (~L1300, ~L1683)
```js
let syncInProgress = false;                 // synchronous re-entrancy guard
const lastAppliedRange = new WeakMap();     // pane -> {from,to} we last set on it
const lastAppliedAt    = new WeakMap();     // pane -> timestamp (ms) we last set it
const ECHO_WINDOW_MS   = 150;
let suppressSync = false;                    // hard-mute during TF-switch transitions
let rsiTF = 'daily';  let rsiFollowChart = true;   // effective TF + "Chart" follow mode (§4.5)
let macdTF = 'daily'; let macdFollowChart = true;
```

### 4.2 Invariants a rebuild must uphold
1. **Position by logical index, never by time, for anything possibly out
   of range.** `setVisibleRange()` (time-based) silently *clamps* to the
   pane's own data; `setVisibleLogicalRange()` (index-based) accepts
   out-of-bounds indices. Convert via `timeToLogicalIndex` first.
2. **The hard-lock clamp is WIDTH-PRESERVING** — slide the window at a
   bound, don't truncate an edge, or a boundary pan becomes a runaway
   zoom-in.
3. **Out-of-bounds always springs back; only in-bounds changes are
   echo-suppressed.** (Echo check gated on `!clamped`.)
4. **Reset `syncInProgress` synchronously.** A deferred reset can get
   starved and stick, freezing sync.
5. **Same-TF panes use exact logical-range passthrough; only cross-TF
   converts through calendar time.** Calendar round-tripping a same-TF
   pane adds rounding that at extreme zoom reads as "stretch".
6. **`suppressSync` during a TF swap, then re-assert after settle.** A
   data swap fires a cascade of transitional range events over several
   frames, AND the library applies a late ResizeObserver-driven
   correction outside paint timing — so `loadTimeframe` re-asserts the
   final range after 2 rAFs **and** again after `setTimeout(…, 250)`.
7. **Whitespace-pad the future region on any truncated series** (§5) —
   and pad **index 0** on any per-bar indicator undefined at the first bar
   (RSI), or it desyncs by one bar (§4.2a).

### 4.2a Horizontal alignment — SEPARATE from range sync
Two bugs proved that identical *logical ranges* do **not** guarantee the
panes line up horizontally. Both are rebuild-critical:

- **Equal price-scale width (build 88).** Each chart auto-sizes its right
  price scale to its own labels (price `32.00`→52px, RSI `120.00`→60px).
  Different axis widths → different plot-area widths → the same date maps
  to a different x per pane, offset *growing* toward the right edge. Fix:
  set `rightPriceScale.minimumWidth` to a common value (64) on **all**
  charts. **`minimumWidth` is only honored at `createChart` time in this
  Lightweight Charts build** — runtime `applyOptions` and formatter
  padding both do nothing (verified). So it lives in the creation options
  of the main chart, `createRsiChart`, and `createMacdChart`.
- **RSI leading whitespace (build 89).** `calculateRSI` can't compute a
  value for bar 0 (no prior close). It used to `continue`, so the series
  started at bars[1] and its logical index 0 mapped to bars[1] — the whole
  RSI series shifted one bar, so a same-TF passthrough landed RSI one bar
  off (MACD/main compute a value for *every* bar, so they never shifted).
  Fix: `calculateRSI` emits a **whitespace point** `{ time: bars[0].time }`
  (no value) at index 0 so it aligns with bars[0]. **Any per-bar indicator
  that legitimately has no value at bar 0 must whitespace-pad index 0**,
  or it desyncs by one bar.

### 4.3 Gap-extension line
When RSI's own timeframe is coarser than the main chart's, RSI's last
real bar sits behind the main chart's "today" (e.g. quarterly RSI has no
bar for the open quarter). `rsiExtensionSeries` carries the last real
value flat forward to the main chart's real boundary, built by
`buildLastValueExtension(lastRealPoint, targetTime)` — written
generically (indicator-agnostic) so a future second indicator pane can
reuse it. Styled via `rsiSettings.extension` (§3.1). Returns `[]` when
no bridge is needed. **Known accepted gap:** its target doesn't
live-recompute when the *main* chart's TF changes while RSI is pinned to
a different TF (handoff §0 item 8). MACD does **not** have a gap-extension
line yet (deferred — §4.4).

### 4.4 MACD pane — the second indicator, and what it reused
The MACD pane (`createMacdChart`, `loadMacdData`, `calculateMACD`, ~L1852+)
was built as a near-clone of RSI to test the abstraction. Integration
points, mirroring RSI exactly:
- **DOM:** `#macd-chart-container` — third child of `#chart-pane`, fixed
  140px, below RSI. Own `.macd-tf-btn` timeframe selector.
- **State globals:** `macdChart`, `macdLineSeries`/`macdSignalSeries`/
  `macdHistSeries`, `macdBarsData`, `macdTF` (~L1484+).
- **Sync:** one descriptor (`macdSyncPane`) in `syncPanes`; subscription
  calls `broadcastRange`. Zero sync-specific code (§4.0).
- **Future space:** the two line series whitespace-pad the future region;
  the histogram is real-bars-only (§5).
- **loadTimeframe:** a recompute-on-`macdTF===tf` block + a
  `reassertIndicatorRange(macdSyncPane, …)` call.
- **switchTicker:** `if (macdChart) loadMacdData(currentTF)`.

**Reused cleanly** (the payoff): `broadcastRange`/`reassertIndicatorRange`
(sync), `calculateEMA` (math), `lastRealBarIndex` + whitespace pattern
(future), and the time/index converters. **Hand-mirrored, not shared**
(the remaining parity debt): `loadMacdData` ≈ `loadRsiData`, and the
`loadTimeframe` recompute block — these would benefit from an
`IndicatorPane` factory, the way `createDrawingPane` already unifies the
drawing tools.

**MACD at full parity with RSI on:** Indicators-column row + on/off toggle
(`setMacdEnabled`, `data-series="macd"`; excluded from Chart-State text
like RSI since it's visibly on/off); settings gear + panel
(`renderMacdSettingsPanel`: Pane · Position / MACD Line / Signal Line /
Histogram; persisted to `macdSettings_v1` — §3.4, histogram colors applied
at paint time via `paintMacdHist` + `macdHistRaw`); **pane position +
resize** (`macdLayout_v1`, `applyMacdLayout` / `resizeChartsForMacdLayout`
/ `setupMacdResize` — §3.5); whitespace future padding; sync; and the
**"Chart" follow-TF mode** (§4.5).
MACD also has the full **drawing toolset** (build 95): its own
`createDrawingPane` instance (`macdPane`) on the *shared* toolbar +
`sharedToolState` (same as main/RSI), attached/reset by `setMacdEnabled`,
in the keydown chain and `clearAll`, with `MACD_LINE`/`MACD_TRENDLINE`/
`MACD_RAY`/`MACD_HRAY` protocol twins and a "MACD Drawings" Chart-State
section.
**Still deferred for MACD** (RSI has them): the gap-extension line, and
inclusion in the ✦ screenshot capture. (Style-beyond-color still isn't in
the text protocol for *any* pane — §9.)

### 4.5 "Chart" timeframe (follow mode)
Each indicator's TF row starts with a **"Chart"** button
(`data-tf="chart"`), the default. `rsiFollowChart`/`macdFollowChart`
(default `true`) mean the indicator mirrors the main chart's timeframe:
`rsiTF`/`macdTF` (the *effective* plotted/synced TF) are kept equal to
`currentTF` and updated on every `loadTimeframe` switch, so the indicator
re-plots on the new TF automatically. Clicking a specific TF sets
`follow = false` and pins the indicator to that TF. `switchTicker` reloads
each indicator on its effective TF (chart TF when following, else its
pinned TF). Button highlight tracks `"chart"` while following. Both
booleans are runtime-only (not persisted). **Note:** a pinned indicator
on a much finer TF than the main view (e.g. 5m RSI while the chart shows
2 years of Daily) can only cover its own short data span within the shared
window — expected data-coverage behaviour, not a sync fault.

## 5. Future-space rendering policy

Raw future rows (§2.1) are **never plotted as their flat values**. Per
series:
- **Candlesticks:** each future row → a Lightweight Charts *whitespace
  point* `{ time }` (no OHLC). Reserves the axis time (drawing/scrolling
  into it still works) but renders nothing. Candlestick series also sets
  `priceLineVisible: false`.
- **Price EMAs (10/20/50/100/200), volume/turnover EMA, RSI line:**
  truncated at `lastRealBarIndex()`. Only earnings markers occupy the
  future space.
- **RSI specifically** must *also* append whitespace points for its own
  future region (not just truncate) — see §4.2 invariant 5.
- **Volume histogram:** left as-is (already zero → invisible) but needs
  `priceLineVisible: false`.

`fitContent()` fits *all* loaded bars incl. future ones — use a
computed `setVisibleLogicalRange` instead for initial view (both panes).

---

## 6. Client-side computed indicators

Present in JSON export: `ema_10/20/50/100`, `turnover`,
`turnover_ema_3/20`. Everything else computed live in the browser:
- **EMA 200** — not exported.
- **Volume EMA 3 / 20** — export only had *turnover* EMAs; panel plots
  share volume, so these are recomputed.
- **RSI(14)** — never exported.
- **MACD(12,26,9)** — never exported; `calculateMACD` builds it from
  `calculateEMA` on close (fast/slow EMA → MACD line → EMA-of-MACD signal
  → histogram).
- **Every-Timeframe EMA / Relative EMA overlays** — each re-fetches that
  timeframe's raw file and computes EMA(10) fresh on toggle.

Deferred goal: move these into `fetch_stock_timeframes.py` in the
sibling repo `RichRoadStockScreenerUS` (handoff §0 item 3 / §4).

---

## 7. Function map (durable locators)

Grep by name; line numbers drift. Grouped by role.

**Time / index conversion**
- `convertTimeBetweenTF(time, fromTF, toTF)` ~L1025 — calendar-time
  across the date-string ↔ epoch-seconds boundary.
- `dateStringToTimestamp(dateStr)` ~L1017.
- `timeToNumeric(t)` ~L2762 — normalize either time format to a number.
- `logicalIndexToTFTime(idx, bars, tf)` ~L1044 — index→time, extrapolates
  out of range via average bar spacing.
- `timeToLogicalIndex(time, bars)` ~L1071 — time→index (§4.2 inv. 1).
- `alignTimeToCurrentChart(data)` ~L1158 — overlay another TF's data.
- `extrapolatePrice(t1,p1,t2,p2,target,isLog)` ~L2766 — **log-scale
  aware**; linear price math in log space is a recurring bug class.

**Sync** (§4.0) — all ~L1300–1440
- `broadcastRange(srcPane, range)` — the whole model: width-preserving
  hard-lock clamp, `!clamped` echo gate, same-TF exact / cross-TF convert,
  synchronous `syncInProgress`. Every pane subscription calls this.
- `isEcho(pane, range)` / `recordApplied(pane, r)` / `rangesClose(a, b)` —
  value + recency echo suppression (`lastAppliedRange`/`lastAppliedAt`).
- `mainDataBounds()` — the hard-lock bounds (main's first/last bar time).
- `reassertIndicatorRange(pane, from, to, mainData, mainTF)` — per-pane
  settle-then-reassert used by `loadTimeframe`.
- Descriptors: `mainSyncPane`/`rsiSyncPane`/`macdSyncPane` + `syncPanes`.
- `buildLastValueExtension(lastRealPoint, targetTime)` — §4.3.
- **Dead (defined, uncalled):** `sourceVisibleTime`, `paneRangeForWindow`.

**Indicator panes**
- RSI: `createRsiChart`, `loadRsiData`, `calculateRSI` (whitespace at
  index 0 — §4.2a). Follow mode: `rsiFollowChart`.
- MACD: `createMacdChart`, `loadMacdData`, `calculateMACD`, `paintMacdHist`.
  Follow mode: `macdFollowChart`. Layout: `applyMacdLayout` /
  `resizeChartsForMacdLayout` / `setupMacdResize`.

**Bars / future space**
- `lastRealBarIndex(bars)` ~L1985 — scans backward for first non-zero
  volume. **Use everywhere instead of `bars.length - 1`.**

**View / layout / interaction**
- `candleWindowAround(anchorIdx)` + `CANDLES_EACH_SIDE` — the default-view
  window (§9a).
- `resyncIndicatorsToMain()` — init-sync reconcile (§9a).
- `paneOrderValue({position, priority})` — pane ordering (§3.5).
- `resyncIndicatorsToMain` is scheduled from the init tail; the chart
  context menu is `setupChartContextMenu` (IIFE).

**Volume / turnover styling**
- `paintVolume()` / `volumeBarColor(bar)` / `hexToRgba(hex, a)` — volume bar
  colours (§3.6).
- `applyTurnoverStyle()` — turnover box styling (§3.7).

**Data load**
- `loadTimeframe(tf)` ~L2102 — main chart TF load (fetches
  `{ticker}_{tf}.json`).

**RSI settings/layout**
- `loadRsiSettings` ~L1385, `persistRsiSettings` ~L1407,
  `applyRsiLayout` ~L1448.

**Drawing tools (shared factory)**
- `createDrawingPane(cfg)` ~L3344 — one pane-agnostic implementation
  driving both main and RSI panes. Inside it:
  `addUserDrawing(type,time1,price1,time2,price2,color,source)` ~L3422,
  `addHline(price,colorName,label,source,lineWidth,lineStyle)` ~L3446.
- `matchesRelativeTF(drawing, tfKey)` ~L1857 — relative-timeframe
  visibility test.
- `makeDraggable(panelEl, handleEl)` ~L2455 — shared for all floating
  panels.

**Protocol (§9)**
- `parseAttrs(rest)` ~L2723 — `key=value` parser, quoted values allowed.
- Command dispatch ~L4127 (`MARKER`/`LINE`/`TRENDLINE`/`RAY`/`HRAY` and
  `RSI_` twins).
- `buildDrawingCommandLines(priceLines, userDrawings, prefix)` ~L4189 —
  emits protocol text for Chart State; `prefix` is `''` or `'RSI_'`.
- `rgbToColorName(rgbString)` ~L4185 via `COLOR_NAMES` (§8.3).

### 7.1 In-memory drawing object shape
Produced by `addUserDrawing` (~L3422):
```js
{
  type,                         // 'trendline' | 'ray' | 'hray' | 'hline'
  time1, price1, time2, price2, // geometry (hline uses price only)
  source,                       // 'claude' | 'user'
  color,                        // claude→pasted color; user→tool default
  opacity, lineWidth, lineStyle,
  timeframes, timeframesColumnEnabled,      // absolute per-TF visibility
  relativeTF, relativeTFColumnEnabled,      // relative-to-home visibility
  homeTF,                                   // TF it was drawn on
}
```
`source: 'user'` items are tagged `[added by user on chart]` in Chart
State output. Both paste and direct-draw call the same
`addUserDrawing`/`addHline`, so there is no behavioral divergence.

---

## 8. Constant tables

### 8.1 `MTF_EMA_COLORS` (~L1000) — per-timeframe overlay EMA colors
```
m5 #e91e63 · m15 #fb8c00 · hourly #8e24aa · daily #3949ab
weekly #6d4c41 · monthly #00acc1 · quarterly #7cb342
```
`RELATIVE_EMA_COLORS` (~L1211): `rel0 #00897b · rel1 #f9a825 · rel2 #455a64`.

**These are only the series *creation-time* colours** — since build 104
`applyAllIndicatorStyles()` immediately overrides them from
`INDICATOR_STYLE_FACTORY` (§3.8), which defaults the M10s column and all
Relative M10s to **black** (rel0/1/2 solid/dashed/dotted). The values above are
now effectively dead unless a user restores a per-indicator style that happens
to match; the live defaults are in `INDICATOR_STYLE_FACTORY`.

### 8.2 `LINE_STYLE_MAP` (~L933) — style key → library enum
```
solid       → LineStyle.Solid
dashed      → LineStyle.Dashed
dotted      → LineStyle.SparseDotted   (wide gaps)
tightDotted → LineStyle.Dotted         (close-packed)
```
**Do not repoint `dotted`.** Existing saved settings/drawings store the
string `'dotted'` meaning SparseDotted; `tightDotted` was *added* as a
separate key rather than remapping (handoff §7, temporal-dead-zone note).
These strings are the values persisted in every `lineStyle` field across
§3.

### 8.3 `COLOR_NAMES` (~L4175) — `"r,g,b"` → protocol color name
Reverse lookup used when emitting protocol text from a rendered shape's
RGB. Note two RGB triples both map to `black` (`0,0,0` and `19,23,34`).
Protocol-accepted color names in practice: `green`, `red`, `blue` (the
parser side); the fuller table exists for *emitting* readable names.

---

## 9. Text protocol (unchanged since build 82)

Commands (each also has an `RSI_`-prefixed twin except `MARKER`, which
is main-chart only):
```
MARKER    date=YYYY-MM-DD pos=above|below shape=arrowUp|arrowDown|circle color=green|red|blue text="label"
LINE      price=NUMBER color=green|red|blue label="text"
TRENDLINE time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
RAY       time1=YYYY-MM-DD price1=NUMBER time2=YYYY-MM-DD price2=NUMBER color=green|red|blue
HRAY      time=YYYY-MM-DD price=NUMBER color=green|red|blue
RSI_LINE / RSI_TRENDLINE / RSI_RAY / RSI_HRAY   — same syntax, RSI pane
```
Rules a parser must enforce:
- `key=value` attrs; quoted values may contain spaces (`parseAttrs`).
- **`TRENDLINE`/`RAY` (and `RSI_` twins): `time1` and `time2` must fall
  on different bars.** Two points on the same bar crash the charting
  library — the parser rejects same-bar commands with an error.
- **Style beyond color is NOT in the protocol.** A pasted shape's
  color comes from its `color=`; opacity/thickness/line-type come from
  the tool's saved default, not the command.
- Not yet expressible: Rectangle, Arrow, Text, Price Range, Date Range,
  Date-and-Price Range, Path, Highlighter, Eraser (none built — §6 of
  handoff).

---

## 9a. Default view, context menu & init reconcile (build 98)

**One default-view knob.** `CANDLES_EACH_SIDE = 100` and
`candleWindowAround(anchorIdx)` → `{ from: anchor-100, to: anchor+100 }`.
Used for every default view: initial page load, every timeframe change (in
`loadTimeframe`, anchored on the last real bar), and the two context-menu
actions. This **replaced** the older "preserve start date across a TF
switch" / "half real, half future" logic — every default view now shows a
consistent candle count. Tune the one constant to change the zoom.

**Chart context menu.** Right-click `#main-chart-area` → `#chart-context-
menu` (built at the end of the script). "Reset chart" recenters on the last
real bar; "Center chart" recenters on the current middle bar — both via
`candleWindowAround`, set on the main chart (propagates to indicators
through the sync). Click-away / Escape closes it.

**Init-sync reconcile.** `resyncIndicatorsToMain()` reasserts every live
indicator to the main chart's current visible range. Scheduled at
600/1200/2200 ms after load. Fixes an intermittent race where an
indicator's data fetch resolves and sets its start view before the main
chart's is settled (they briefly share `suppressSync`) — it looked out of
sync until the first scroll; this reconciles it automatically.

## 9b. Settings-panel layout — the 2-column standard (build 100–101)

Indicator settings panels (RSI, MACD) use a short-and-wide **2-column**
layout instead of one tall column that ran off the chart. The render adds a
`.settings-body` (CSS `column-count: 2`) inside a `.settings-2col` panel
(384px); each section is a `.settings-section-group` with
`break-inside: avoid` so it's never split across columns. `addRow` appends
to the current section group. **This is the standard for any future
settings menu** — build it the same way.

---

## 9c. Unified indicator styling popup & event overlays (builds 104–112)

**One popup for every styleable indicator.** Clicking an indicator's **name**
in the Indicators dropdown (left-click the name, or right-click anywhere on the
row) opens `openIndicatorStylePanel(id, x, y)` — the checkbox still toggles
visibility. The popup adapts to the indicator's *kind* (`INDICATOR_KIND`):
`line` (colour + 12-swatch palette + **separate** opacity/transparency +
thickness + line style), `volume` (up/down colour + opacity), `earnings`
(colour + size), `cross` (colour + shape + size). Each carries Set/Restore
Default (§3.8). The palette is the shared 12 hex `IND_PALETTE`. Panel is
appended to `<body>` with **`z-index:1000`** — without it the legend
(`z-index:5`) paints over it and the popup opens but is invisible.

**RSI/MACD have no generic style — they keep their own panels** but are now
opened the *same way*: `wirePanelName('rsi'|'macd', …)` wires the dropdown name
to `renderRsiSettingsPanel`/`renderMacdSettingsPanel`. **The old ⚙ gear buttons
were removed** (build 107) — supersedes the "settings gear" references in §3.4
and §4.4. Turnover still opens its box popup on right-click (§3.7).

**Event-marker overlays — `#crossover-overlay`, `#close-above-overlay`.** The
10/20 EMA cross and Close-above-20-EMA dots are drawn as **custom HTML/SVG
markers**, NOT native chart markers, so shapes beyond the library's four are
possible (adds **cross `X`** and **plus `+`**). One shared engine:
- `computeCrossPoints(data, keyA, keyB, upOnly)` — bars where line A crosses
  line B (ema_10×ema_20 either way; close×ema_20 upward-only). Stores the four
  values `{prevTime, time, pa, pb, ca, cb}` around each crossing.
- `renderEventOverlay(containerId, points, visible, styleId)` — places each dot
  at the **exact PIXEL intersection** of the two drawn segments. Critical: the
  price scale is **logarithmic**, so a price-space interpolation converted to a
  coordinate does NOT lie on the pixel-drawn line (it sat ~16px off). Intersect
  in pixel space instead (`s = (ypb-ypa)/((yca-ypa)-(ycb-ypb))`).
- `refreshEventOverlays()` — renders now AND again via a debounced **double-rAF**
  because a visible-range-change fires its subscription *before* the chart
  repaints its auto-scaled axis; the deferred pass lands after the repaint so
  the dots aren't a frame stale. `loadTimeframe`'s `applyFinalRange` also
  re-renders post-settle. Same pattern any price-anchored HTML overlay needs.

## 9d. Rich Road Candles (build 113)

Recolours each **real** candle by classification. `paintCandles()` rebuilds the
candle data (future bars → whitespace, §5) and is called from `loadTimeframe`
and on any live change. A classified bar gets **`color` = the type colour (body
fill), `borderColor` = black, `wickColor` = black** — coloured body, black
outline + wick. A bar is left at its default appearance when Rich Road is off,
the bar is unclassified, or its matched type is switched off in `enabledTypes`
(§3.9). `classifyCandle(bar, volEma)` returns one of six colour keys using the
adjustable `richRoad.params` (`volEma` is that bar's volume-EMA value, for the
volume gate):
- **move** = body move `(close-open)/open*100`; **body ratio** =
  `|close-open|/(high-low)`; **wick tol** = `(open-low)/low*100`. **`volumeOk(t)`**
  = `!useVolume[t]` OR `volume >= volMultiplier*volEma`.
- Precedence (first match): **High CB auto** (move ≥ `highAutoMove` 9.5 — beats
  everything, ungated) → **Red CB** (bearish, down ≥ `redMove`) → **High CB**
  structural (bullish; `volumeOk('highCB')` + move ≥ `highMinMove` 4.8 + ratio ≥
  `highBodyRatio` 0.8 + wick ≤ `highWickTol` 5) → **Low CB** (move ≥ `lowMinMove` 3
  + ratio ≤ `lowMaxBodyRatio` + `volumeOk('lowCB')`) → **DC** (ratio ≥ `dcBodyRatio`
  0.7 + wick ≤ `dcWickTol` 5) → **DC2** (close > ema_10 or ema_20) → **Neutral CB**.
- **State of validation (2026-08-13):** High CB matches the original spec (locked);
  Low CB is the user's preferred version (locked, intentionally divergent); DC is
  paused pending the user's DC-vs-Low-CB decision (see §0 of CHART_HANDOFF). We are
  building the user's preferred version, not a strict copy of the original — treat
  thresholds as the user's evolving rules, not canonical.

**UI:** the first row of the **Indicators column** (`data-series="richroad"`);
checkbox = whole-indicator on/off, name opens `renderRichRoadPanel()` — a
2-column panel (§9b) whose "Candle types" section gives each of the six types an
**on/off checkbox + colour picker + 12-swatch palette**, plus a number input per
threshold, Set/Restore Default (persisted `richRoadCandles_v1`). The classifier
precedence still assigns Neutral, but a Neutral (or any off) type just means the
candle keeps its default appearance.

---

## 10. What this document deliberately does NOT cover

- **Narrative history / rationale** — in `CHART_HANDOFF.md` (§7 for the
  hard-won sync/debugging lessons especially).
- **The sibling data pipeline** — `RichRoadStockScreenerUS` +
  its own `PROJECT_HANDOFF.md`.
- **The synthesis methodology for future/placeholder bars** — handoff
  §4 (reusable for extending m5 depth).
- **Deferred/unbuilt features and known limitations** — handoff §0 & §8.

When rebuilding on another platform, read this document for *what the
contracts are*, then the handoff §7 for *which of them are landmines*.
```