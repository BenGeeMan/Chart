# Course Learning Plan — capturing the trading course

**Started 2026-08-13.** The user is a student in a **video-delivered trading
course** and wants to turn it into a system the assistant can learn from, reason
with, and build tools around. This doc is the blueprint. The **Chart** repo (see
`CHART_HANDOFF.md`) is the first tool; the Rich Road candle classifier is the first
encoded rule.

---

## ★ NORTH STAR (the end goal everything points at)
Turn the course into a **living reference library** the assistant can use to:
1. **Find trades** — apply the encoded method to real charts and flag setups that
   match ("demand candle into a demand zone, with the entry trigger he teaches").
2. **Support execution** — find, judge, size, and lay out the exact entry/stop/
   target. **HARD BOUNDARY: the assistant does NOT place real orders** (moving
   money is a line it won't cross). It hands the user a decision-ready setup; the
   user — or an automated system the user owns and controls — pulls the trigger. If
   automated later, the assistant helps *build* that code, paper-traded/tested
   first. The assistant is a tool applying the user's own learned method, **not a
   financial adviser**.
3. **Ask the teacher good questions — on a chart.** When understanding has a gap,
   the assistant draws the exact setup on a chart, marks the candle/zone in
   question, annotates the question on it (with the lecture timestamp), and the
   user shows the teacher. The teacher likes a chart with questions. His answer
   feeds back into the concept graph.

**The flywheel:**
```
   reference library → apply rules to real charts → find setups + surface questions
          ↑                                                             │
          └──────────────  teacher answers → graph refined  ←──────────┘
```
Every loop sharpens the library → the tools → the questions.

**Sequencing:** library first → scanning + teacher-questions → execution *logic*
last and carefully. The question-on-a-chart loop can start early (the chart tool is
~80% there already: markers, annotations, "copy image for Claude").

---

## What we're capturing — two content types
- **Lessons** — the linear teaching; each lesson introduces a **new concept that
  builds on earlier concepts** (a dependency progression).
- **Live chats** — integrative sessions that reference the lesson before and after,
  and increasingly **draw from many lessons** as the course goes on.

~40 videos, 1–2 hrs each, mostly **English with some Hindi/Hinglish** (instructor +
some students are Indian). The assistant reads/understands Hindi, so a mixed
transcript is fine.

## Storage architecture — a PRIVATE repo, three layers
Plain **Markdown + images in a private git repo**, cloned to the user's machine so
the assistant reads it directly. No database. Readable, diffable, renders in
GitHub's web view.

```
richroad-course/              ← PRIVATE repo (copyrighted course content)
  sessions/                   ← RAW material, ONE chronological timeline
    001-lesson-01/
      transcript.md           ← timestamped, cleaned text
      frames/00-14-30.png     ← AI-extracted screenshots, named by timestamp
      annotated/              ← assistant's marked-up copies (arrows/boxes/labels)
      notes.md                ← distilled rules; top line: references: [00X, 00Y]
      quiz.md                 ← the test-each-other log (Qs, gaps, what's solid)
    002-lesson-02/
    003-livechat-2024-05-10/  ← references 002 (before) & 004 (after) ...
    ...
  concepts/                   ← THE MODEL: a dependency graph of the method
    demand-candle.md          ← builds_on: [candle-basics]; introduced 002 @14:30;
                                 refined livechat-006 @22:10; applied 011 @08:00
    demand-zone.md            ← builds_on: [demand-candle, support-resistance]
    glossary.md
  concept-map.md              ← auto-kept Mermaid diagram of the graph (renders on GitHub)
  INDEX.md                    ← two views: the timeline, and the concept map
  progress.md                 ← dependency-aware mastery tracker (per concept)
```

**Why three layers:**
- **`sessions/`** keeps the raw material on one timeline, so "before/after" is just
  adjacency and the reference web stays intact.
- **`concepts/`** is the real prize: each concept declares `builds_on:`
  (prerequisites), so the method becomes a **navigable dependency graph**. When a
  live chat "draws from many lessons," those threads converge in the concept docs
  rather than scattering. **The tools read from this layer.**
- **`concept-map.md`** is the picture of the graph, grown lesson by lesson — you
  watch the method assemble.

**Foundation-first testing:** because later concepts sit on earlier ones, verify the
foundations are solid before building on them; a shaky prerequisite flags every
concept downstream of it.

## Capture workflow (frees the user's time) — SETTLED after the lesson-1 pilot
1. **Local faster-whisper** transcribes each ~11-min chunk → timestamped Markdown:
   `/home/ben/whisper-venv/bin/python tools/transcribe.py <chunk> <offset_s> small <out.md>`
   (~8.5 min CPU per chunk; the offset makes chunk-local times lesson-global).
   Runs unattended and harness-tracked, so it notifies on completion.
2. Assistant stitches the chunks, reads the transcript, and picks the key timestamps
   itself; **ffmpeg** pulls exactly those frames into `frames/` — no manual
   screenshotting.
3. Assistant reads transcript + frames → writes `notes.md`, updates the relevant
   `concepts/` nodes + `concept-map.md` + `quiz.md` + `doubts.md`.

## Session workflow (we do this together)
- We **watch a lesson together**; I **show the relevant frames** inline and
  **annotate** them (arrows/boxes/labels) to explain a point.
- **Time references throughout** — timestamped transcript + timestamp-named frames
  mean we're always looking at the same moment.
- **We test each other** — I quiz the user's understanding; the user quizzes mine;
  gaps go into `quiz.md`/`progress.md` and become teacher-questions where needed.

## Storage & privacy
Raw videos stay **local / external drive** (too big for GitHub). Transcripts,
frames, notes → the **PRIVATE** repo only, never the public Chart repo. The course
is copyrighted; this is personal-study use.

## Cloud vs local — DECIDED: local, £0
The Gemini/AI Studio route (≈$15–30 for all 40) was **dropped**. Local Whisper is
free, nothing leaves the machine, and lesson 1 came out clean (English 0.98–0.99).
Cloud's one real advantage — describing visuals — proved unnecessary: his recordings
are **screen shares** (Excalidraw for teaching, TradingView for charts) with the
**cursor and crosshair visible in frame**. Excalidraw drawings accumulate, so one
frame at the end of a segment holds the whole diagram; the TradingView crosshair
prints exact price and date, so frames are precise enough to rebuild a moment on real
data. Gesture is therefore a non-issue.

## Effort estimate
~**10–16 sessions of 5 hours** to interpret all 40 and fold them into
concepts/tools (~2–3 lessons per session). The pilot calibrates it exactly.
Incremental — value accrues each session, no big upfront commitment.

## Pilot COMPLETE (2026-08-14) — lesson 1 processed end to end
The course repo is live and private: **https://github.com/BenGeeMan/richroad-course**
(`~/GitHub/richroad-course/`). **Read its `INDEX.md` first** — it's the front door.
Lesson 1 produced: `transcript.md` (407 segments), 18 frames, `notes.md`, 5 concept
nodes, `concept-map.md`, `quiz.md` (answered, 4½/5) and `doubts.md`.

**Things learned that changed the plan:**
- **The course opens with few rules on purpose** — rules sharpen over time but stay
  loose, always applied through perception. So doubts are **logged, not chased**;
  `doubts.md` tracks them with **lecture 19** as the sweep checkpoint (his marker that
  many should have gone by then). 3 of 8 already resolved.
- **An indicator's job is to make the chart readable to a human** — not a rule engine;
  he warns against leaning on indicators instead of watching price action unfold.
  So Rich Road's thresholds are a **colouring convention**: tune for **legibility**,
  not for accuracy or agreement with TradingView.
- **DC = Demand Candle, CB = Committed Buyers candle** — both defined later in the
  course, so the DC-vs-Low-CB tuning stays **paused** rather than guessed at (D006).
- Raw videos are `.gitignore`d; they live in `~/Downloads/Course/`.
- **Recording (SimpleScreenRecorder on Linux Mint):** full screen 1080p / 30fps /
  H.264 / MKV / superfast / CRF 18, audio source = **"Monitor of…"** (system sound).
  **Tip: set SSR's audio codec to compressed (Vorbis/AAC), NOT Uncompressed** — PCM
  made lesson 1's MKV 465 MB; converting to AAC dropped it to 122 MB.
- **Post-record steps (assistant does these on the machine):** trim black tail
  (`ffmpeg -to <end> -c copy`; find the real end with `blackdetect`), convert MKV→MP4
  for upload (`-c:v copy -c:a aac`), and **split into ~11-min chunks** (see next).
- **Chunking** (`ffmpeg -f segment -segment_time 00:11:20 -reset_timestamps 1`) is
  still worth doing — it keeps each transcription run short and restartable. Each
  chunk's timestamps are local 0:00, so pass the offset (part0=0, part1=680,
  part2=1363) and they come out lesson-global.

## Immediate next step (resume here)
**Record lesson 2** → `~/Downloads/Course/`, then run the pipeline above (chunk,
transcribe locally, stitch, frames, notes, concepts, doubts).

**Then: the comprehension test.** After lecture 2 the user draws on a chart and the
assistant says what it shows in the language of the lectures — the real check that
the library transfers. Works today via screenshot (the Chart tool has trendline /
hline / ray / hray with editing). Note drawings are **in-memory only**
(`userDrawings`, `index.html:5081`), so they don't survive a reload and can't be read
directly; if eyeballing screenshots proves too imprecise, add a drawings→JSON export
for exact price/time.

(Still parked: the DC-vs-Low-CB modelling question — `CHART_HANDOFF.md` §0 — now
waiting on the course to define Demand Candle and Committed Buyers.)
