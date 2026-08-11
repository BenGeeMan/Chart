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
number at time of writing: **82**.

---

## 1. File & data inventory

| File | Kind | Notes |
|---|---|---|
| `index.html` | The entire app | Single self-contained file. Filename fixed (GitHub Pages). Visible build-number watermark at `#build-number` (~L887). |
| `{ticker}_{tf}.json` | Price+indicator series | 77 files = 11 tickers × 7 timeframes. Schema §2. |
| `chart_watchlist.json` | Watchlist quote summary | Schema §2.3. |
| `chart_earnings.json` | Earnings dates per ticker | Schema §2.4. |
| `.github/workflows/chart_export.yml` | CI | Manual-trigger zip of the repo for handoff. |

**Tickers (11):** `HPQ, COHR, ABNB, RKLB, IREN, NTRA, NEM, TTWO, KGS,
CVSA, RMAX`.

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

Three independent keys, all versioned `_v1`, all best-effort (wrapped in
try/catch — private browsing / disabled storage falls back to
in-memory, feature still works for the session). All three are keyed by
constants, never bare strings.

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

---

## 4. The cross-pane sync model (main ↔ indicator panes)

The hardest and least-obvious subsystem. Full *why* is handoff §7; this
is the *contract* a reimplementation must satisfy. It keeps N independent
Lightweight Charts instances (main + each indicator pane — currently RSI
and MACD) showing the same calendar-time window even when each is on a
**different timeframe**.

### 4.0 Generalized N-pane sync
Sync is now pane-count-agnostic. A "pane" is a descriptor of three
**getters** — `{ chart(), bars(), getTF() }` — so one descriptor reads
live state forever (`rsiChart`/`macdChart` are reassigned on toggle-off/on,
the bars arrays on every load, the TFs on switch). The registry
`syncPanes = [mainSyncPane, rsiSyncPane, macdSyncPane]` (~L1228) holds one
descriptor per pane. Three functions carry the model:
- `applyCrossPaneRange(src, dst, range)` (~L1123) — maps one pane's
  visible range onto another (same-TF passthrough vs. cross-TF
  calendar conversion). The pane-agnostic core.
- `broadcastRange(srcPane, range)` (~L1245) — each pane's timeScale
  subscription calls this; it pushes the range to every *other* live pane
  via `applyCrossPaneRange`. One `syncInProgress` flag (reset one frame
  later) guards the whole burst — replacing the old pairwise
  `syncingFromMain`/`syncingFromRsi` booleans, which did **not**
  generalize (a change from RSI would reach main but not propagate on to
  MACD). `suppressSync` still hard-mutes during TF-switch transitions.
- `reassertIndicatorRange(pane, from, to, mainData, mainTF)` (~L1262) —
  the settle-then-reassert step in `loadTimeframe`, one call per indicator
  pane instead of a copy-pasted block.

**Adding an indicator pane = one descriptor in `syncPanes` + its
subscription calling `broadcastRange`.** MACD is exactly that. This is
what handoff §0 item 3 asked for; the sync layer is now genuinely
reusable. Still hand-mirrored per pane (not yet shared): the data-load +
init-view function (`loadMacdData` ≈ `loadRsiData`) and `loadTimeframe`'s
recompute-on-match block — see §4.4.

### 4.1 State flags (~L1345)
```js
let syncingFromMain = false, syncingFromRsi = false; // per-direction re-entrancy guards
let suppressSync   = false;   // hard mute of BOTH directions during a TF-switch transition
let rsiTF          = 'daily'; // RSI's own timeframe, independent of the main chart's
```

### 4.2 Invariants a rebuild must uphold
1. **Position by logical index, never by time, for anything possibly
   out of range.** `setVisibleRange()` (time-based) silently *clamps*
   to the pane's own loaded data; `setVisibleLogicalRange()`
   (index-based) accepts out-of-bounds indices and shows blank margin.
   Convert time→index via `timeToLogicalIndex(time, bars)` first, then
   call `setVisibleLogicalRange`.
2. **Read each visible edge independently** via `sourceVisibleTime()`
   — `getVisibleRange()`'s clamping is asymmetric between the past
   (`from`) and future (`to`) edges. Trust it only for whichever edge
   is still within the pane's own bars; extrapolate the other via
   `logicalIndexToTFTime`.
3. **A single synchronous boolean guard is insufficient.** A TF switch
   fires a *cascade* (~10) of transitional range-change events over
   several frames. Reset the per-direction guard a frame later
   (`requestAnimationFrame`), AND use `suppressSync` to mute both
   directions across the whole transition, then re-assert the final
   state explicitly.
4. **"Settled" is not reliably 2 animation frames.** The library
   applies internal corrections (ResizeObserver-driven) outside paint
   timing. Re-assert the final range once more after
   `setTimeout(…, 250)` so the app's value wins last.
5. **Any pane that truncates its real line for the future-space policy
   MUST whitespace-pad the same region** (§5) — otherwise its timeScale
   has no anchor for the future time and sync positioning lands
   somewhere wrong.

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

**Reused cleanly** (the payoff): `applyCrossPaneRange`/`broadcastRange`/
`reassertIndicatorRange` (sync), `calculateEMA` (math),
`lastRealBarIndex` + whitespace pattern (future), and the time/index
converters. **Hand-mirrored, not shared** (the remaining parity debt):
`loadMacdData` ≈ `loadRsiData`, and the `loadTimeframe` recompute block —
these two would benefit from an `IndicatorPane` factory next, the way
`createDrawingPane` already unifies the drawing tools. **Deferred for
MACD** (RSI has them, MACD doesn't yet): settings panel, drawing tools,
gap-extension line, on/off toggle + legend row, resize/reposition/persist
layout, protocol twins (`MACD_*`), Chart-State output, screenshot capture.

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

**Sync** (§4.0)
- `applyCrossPaneRange(src, dst, range)` ~L1123 — pane-agnostic core.
- `broadcastRange(srcPane, range)` ~L1245 — push a source pane's range to
  all other live panes; `syncInProgress` guard. Every pane subscription
  calls this.
- `reassertIndicatorRange(pane, from, to, mainData, mainTF)` ~L1262 —
  per-pane settle-then-reassert used by `loadTimeframe`.
- `sourceVisibleTime(sourceChart, range, bars, tf)` ~L1114 — §4.2 inv. 2.
- `buildLastValueExtension(lastRealPoint, targetTime)` ~L1147 — §4.3.
- Descriptors: `mainSyncPane`/`rsiSyncPane`/`macdSyncPane` +
  `syncPanes` ~L1228.

**Indicator panes**
- RSI: `createRsiChart`, `loadRsiData`, `calculateRSI`.
- MACD: `createMacdChart`, `loadMacdData`, `calculateMACD` ~L1852+.

**Bars / future space**
- `lastRealBarIndex(bars)` ~L1985 — scans backward for first non-zero
  volume. **Use everywhere instead of `bars.length - 1`.**

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

## 9. Text protocol (build 82)

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