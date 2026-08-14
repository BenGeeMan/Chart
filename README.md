# Chart

A TradingView-style charting web app (a single self-contained `index.html`, served
via GitHub Pages) that acts as a **two-way text + visual bridge** between the owner
and an AI assistant (Claude): the assistant can describe drawings/markers in a
simple text protocol that the app renders, and the app can export its full state
(and a screenshot) back in a form the assistant can read.

It's the first tool in a larger effort: **capture a video trading course into a
structured method, then build tools that apply it** — find setups on real charts,
prepare trades for the owner to execute, and pose questions back to the teacher on
annotated charts.

Current build: **130** (watermark on the chart confirms a hard refresh picked up
new code).

## Documents
- **[CHART_HANDOFF.md](CHART_HANDOFF.md)** — the narrative: what's built, why,
  what's outstanding. Start here (§0 has the current priorities).
- **[CHART_TECHNICAL_REFERENCE.md](CHART_TECHNICAL_REFERENCE.md)** — the flat spec:
  data/localStorage schemas, the cross-pane sync model, constants, function map —
  enough to rebuild the app on another platform.
- **[COURSE_LEARNING_PLAN.md](COURSE_LEARNING_PLAN.md)** — the blueprint for turning
  the trading course into a reference library the assistant can reason with (the
  "north star" for where this is all heading).
- **[WORKING_WITH_ME.md](WORKING_WITH_ME.md)** — how the owner likes to work
  (not project-specific; read alongside the handoff).
- **[PROJECT_HANDOFF.md](PROJECT_HANDOFF.md)** — handoff for the sibling repo
  `RichRoadStockScreenerUS`, the daily stock-scanning data pipeline this chart is
  built on top of.

## Layout (high level)
- `index.html` — the entire app (keep this filename; GitHub Pages needs it).
- `chart_watchlist.json`, `chart_earnings.json` — watchlist quotes and earnings.
- `{ticker}_{timeframe}.json` — price + indicator data (12 tickers × 7 timeframes).
- `.github/workflows/chart_export.yml` — manual "zip the repo" action for handoff.

## Running it
It's static — open it through a web server (GitHub Pages serves it live), or locally:
```bash
python3 -m http.server 8777
```
then visit `http://localhost:8777/index.html`. Opening the file directly via
`file://` won't work because the app fetches the JSON data files.
