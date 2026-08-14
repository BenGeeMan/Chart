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

## Immediate next step (resume here)
Run the **lesson-1 pilot** (free): assistant owes the user the **AI Studio prompt**
(ask for a timestamped transcript + running chart-state/drawing description +
flagged rules + key-moment timestamps + Hindi marked), writes the **frame-extraction
script**, and stands up the **repo skeleton + first concept node + concept-map**.
Do lesson 1 end-to-end, compare Gemini vs Buzz, read the real token cost, then pick
the route to batch the rest. (Separately parked: the DC-vs-Low-CB modelling
question — see `CHART_HANDOFF.md` §0 / `RICHROAD_FACTORY`.)
