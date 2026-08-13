# Course Learning Plan — capturing the trading course

**Purpose.** The user is a student in a **video-delivered trading course**
(~**40 videos, 1–2 hrs each**, mostly **English with some Hindi/Hinglish** — the
instructor and some students are Indian). The end goal of the whole effort is to
**teach the assistant the course's method and the user's own trading-decision
logic**, then build tools around it. The **Chart** repo (see `CHART_HANDOFF.md`)
is the first such tool; the Rich Road candle classifier is the first concrete rule
being encoded. This doc is the plan for getting the course content into a form the
assistant can actually learn from. Started 2026-08-13.

---

## The core problem
The assistant can't watch video. So each video has to become **text + visuals**:
- **The words** — a transcript of what the instructor says.
- **The visuals** — because he **free-draws / annotates / scrolls live on charts
  while talking** (dynamic, not slides), we need either screenshots at key moments
  or an AI's description of the on-screen action.

## Two routes (decide after a FREE pilot on lesson 1)
1. **Local — Buzz (Whisper) on Linux Mint.** Free, 100% private, but words-only
   (+ the user grabs screenshots) and slow (days to batch all 40). Settings:
   **Medium** model (better on the Hindi/mixed audio), **Auto-detect** language,
   **Transcribe** task, export **SRT** (timestamps let the assistant sync
   screenshots). Install: Software Manager → search "Buzz" (by Chidi Williams),
   or `flatpak install flathub io.github.chidiwilliams.Buzz`.
2. **Cloud — Gemini via Google AI Studio (aistudio.google.com).** Watches video
   directly (audio AND visuals) → captures the dynamic drawing that screenshots
   miss. Non-technical web UI: upload a video + paste a prompt. **~$15–30 total for
   all 40 on Gemini Flash** (low-res mode ≈ half; Pro ~$100+, not worth it); the
   **pilot is free**. Trade-offs: uploads copyrighted video to Google (vs Buzz
   staying local); long videos may need ~1 hr chunks; it samples ~1 fps so it's
   great for narrative/visuals but NOT exact prices — those get rebuilt in our tool.

## Division of labour
- **Transcription / video-AI = eyes + ears** — video → text + visual description.
- **Assistant = the reasoning** — extract the rules, rebuild the setups on real
  data in the chart tool, store the method in memory + a `notes/` doc, and grow
  the tools to match.
- **User = the bridge for ambiguity** — confirms mangled terms, "which candle he
  meant", freehand drawings.

## Storage & privacy (important)
- Raw videos stay **local / on an external drive** — too big for GitHub (100 MB
  per-file cap; LFS too costly for 40 large files).
- The course is **copyrighted**: keep transcripts/screenshots/notes in a **PRIVATE**
  repo, **not** in the public Chart repo (which is public for GitHub Pages).
- Suggested layout: `~/RichRoad-Course/` → `transcripts/  screenshots/  notes/`
  (name files `lesson-01.srt`, `lesson-01_14-30.png`, etc.). The assistant reads
  the transcript files directly off the machine — no upload needed for it to work.

## Effort estimate
Roughly **~10–16 working sessions of 5 hours** to interpret all 40 lessons and
fold them into tools/notes — about **2–3 lessons per 5-hour session**, depending
on how many chart setups we rebuild per lesson vs just noting the rules. **The
lesson-1 pilot will calibrate the real pace**, after which the estimate can be
made exact. Done incrementally — no big upfront commitment; value accrues each
session.

## Immediate next step (resume here)
Run the **pilot on lesson 1** (free). The assistant owes the user the **AI Studio
prompt** for lesson 1 — ask Gemini for a *timestamped transcript + a running
chart-state / drawing description + flagged rules + Hindi bits marked* — and/or
help finish the Buzz install. Then compare the two routes, read the **actual token
count** AI Studio reports on lesson 1 to give an exact per-video cost, and pick
which route to batch the other 39 with.
