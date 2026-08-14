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

## Capture workflow (frees the user's time)
1. **Gemini (Google AI Studio)** watches each video → timestamped transcript +
   description + a list of key visual moments. (Alternative: local Buzz/Whisper —
   free, private, words-only + manual screenshots, slower. Decide after the pilot.)
2. A **script (ffmpeg) the assistant writes** pulls those exact frames from the
   local video into `frames/` — automatic, no manual screenshotting.
3. Assistant reads transcript + frames → writes `notes.md`, updates the relevant
   `concepts/` nodes + `concept-map.md` + `quiz.md`.

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

## Cloud vs local + cost
Gemini Flash for the whole ~40 videos ≈ **$15–30** (low-res ~half; pilot free).
Trade-off: uploads copyrighted video to Google vs Buzz staying fully local. The
lesson-1 pilot gives an exact per-video token count to firm this up.

## Effort estimate
~**10–16 sessions of 5 hours** to interpret all 40 and fold them into
concepts/tools (~2–3 lessons per session). The pilot calibrates it exactly.
Incremental — value accrues each session, no big upfront commitment.

## Pilot progress & learnings (2026-08-13)
Lesson-1 pilot is **in progress**. Practical learnings so far:
- **The course repo now exists:** `~/GitHub/richroad-course/` (git-init'd) — skeleton +
  `ai-studio-prompt.md` (the reusable transcription prompt) + `sessions/001-lesson-01/`.
  Raw videos are `.gitignore`d; they live in `~/Downloads/Course/`.
- **Recording (SimpleScreenRecorder on Linux Mint):** full screen 1080p / 30fps /
  H.264 / MKV / superfast / CRF 18, audio source = **"Monitor of…"** (system sound).
  **Tip: set SSR's audio codec to compressed (Vorbis/AAC), NOT Uncompressed** — PCM
  made lesson 1's MKV 465 MB; converting to AAC dropped it to 122 MB.
- **Post-record steps (assistant does these on the machine):** trim black tail
  (`ffmpeg -to <end> -c copy`; find the real end with `blackdetect`), convert MKV→MP4
  for upload (`-c:v copy -c:a aac`), and **split into ~11-min chunks** (see next).
- **AI Studio free tier = model `Gemini 3 Flash Preview`** (Gemini 2.5 is PAID — do
  not upgrade). The full 33-min video (and even a single 11-min chunk) shows a red
  **"Token count failed"** — that's just the pre-flight counter; the run itself can
  still work once the video **"extracting"** step finishes. Uploading the whole
  33-min file was unreliable, so we **split into ~11-min chunks** (`ffmpeg -f segment
  -segment_time 00:11:20 -reset_timestamps 1`), one per **fresh chat**, run one at a
  time (free-tier rate limits). Each chunk's timestamps are local 0:00 — assistant
  offsets them when stitching (part0=0:00, part1=11:20, part2=22:43).

## Immediate next step (resume here)
Finish transcribing lesson 1 in AI Studio (3 chunks → paste each output back,
assistant stitches + offsets timestamps). Then: extract the flagged key frames →
`frames/`, write `notes.md`, build the first `concepts/` node + `concept-map.md` +
`quiz.md`, and do the test-each-other loop. After lesson 1, read the real token
cost/effort and decide whether to keep the Gemini route or fall back to local Buzz.
(Separately parked: the DC-vs-Low-CB modelling question — `CHART_HANDOFF.md` §0.)
