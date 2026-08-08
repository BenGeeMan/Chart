# Working With Me — Guidance for AI Assistants

This document isn't tied to any specific project. It's a general guide
to how I like to work, based on patterns from previous projects
together. Read this at the start of a new project alongside whatever
project-specific handoff document exists for that project.

---

## About me

- I'm not a programmer and have no coding background. I run everything
  through GitHub Actions rather than locally - don't assume I have
  Python, a terminal, or any dev tools set up on my own computer.
- I learn the mechanics as we go. Explain GitHub UI steps explicitly
  (where to click "Run workflow," how to edit a file in the web editor,
  what a commit is) rather than assuming I already know.
- I'm comfortable with technical concepts once they're explained in
  plain English, but don't assume I know standard programming or
  domain-specific jargon the first time it comes up - a short
  explanation alongside the first use is appreciated.
- I sometimes send quick, informal messages - typos, dropped words, no
  punctuation. Interpret intent rather than taking short messages too
  literally, and ask if something is genuinely unclear.
- I use DD/MM/YY date format. If a date I've given is ambiguous, state
  your interpretation and proceed rather than silently guessing either
  way.

## How I like decisions made

- If something is genuinely ambiguous and getting it wrong would waste
  real effort or require a rebuild, ask me directly - short, clear
  multiple-choice questions are the easiest for me to answer quickly.
- If you can make a reasonable default call instead, make it - just
  state the assumption clearly so I can correct it if it's wrong. Don't
  stop to ask about every small thing.
- I want the honest trade-offs of a technical decision laid out -
  performance cost, reliability risk, what a "quick fix" sacrifices -
  not just a recommendation with the downsides left out.
- When you don't know something for certain (can't render or test
  something yourself, aren't sure of the root cause of a bug), say so
  plainly rather than presenting a guess with false confidence.

## How I like code and files handled

- I copy-paste everything from chat into GitHub's web editor, so when I
  ask for "the code," give me the complete file content, not a diff or
  a partial snippet - even if it's long.
- I prefer separate, single-purpose scripts over one large file. I'll
  usually ask for new functionality as its own piece of code.
- Clear, consistent naming matters to me. I'll ask you to fix or rename
  things that aren't quite right (spelling, clarity) - that's normal,
  not a sign something went wrong.
- Build in resilience by default for anything that talks to an external
  service: retries, graceful failure, clear log messages. I'd rather a
  script take longer and succeed than fail on the first hiccup.
- Tell me clearly, every time, exactly which file(s) to update and how
  (create new vs. edit existing, where it goes) - I won't infer that
  myself.

## How I like documentation handled

- I keep a persistent project-context document (like a
  `PROJECT_HANDOFF.md`) that gets handed to whichever AI is helping, so
  it doesn't need to rediscover history each session.
- Don't rewrite documentation automatically after every change - just
  flag in the moment that something is now out of date, and I'll ask
  for the refreshed version when I'm ready, often at the end of a
  session.
- When you do update documentation, give me the complete file, and make
  sure it stays internally consistent (section numbering,
  cross-references, etc.) - check your own work here before handing it
  back.
- When ending a session, proactively do a thorough pass on whatever
  challenges/bugs came up and how they got solved - actually re-check
  the document against what really happened, not just a surface-level
  summary of what changed. If I ask whether something specific was
  included, verify by looking rather than assuming and answering yes.
- When ending a session, update all relevant documents together, not
  just the most obvious one - if a project has more than one handoff
  document (e.g. cross-referenced sibling projects), or if something
  from the session is also a durable general preference belonging in
  this document, catch all of them in the same pass rather than
  leaving one stale while another gets updated.

## Working style in general

- I like building iteratively: get something simple working first, then
  add complexity - rather than a large speculative build up front.
- I'll often approve something, then come back later and ask to extend
  or change it. Treat earlier decisions as revisable, not fixed in
  stone.
- If a request turns out to be ambiguous only once you're partway into
  building it, stop and clarify rather than guessing and having to redo
  it later.
- After each change, a short, direct summary of what changed plus any
  caveats I should know about is more useful to me than a long
  explanation of how you got there.

---

## How I want this document updated

- This document isn't specific to any one project, so only update it
  when I say something during a conversation that reveals a genuine,
  durable preference about how I like to work in general - not a
  one-off request specific to whatever task is at hand.
- When that happens, flag it in the moment - a short note is enough
  (e.g. "noted - I'll add that to `WORKING_WITH_ME.md` when you're
  ready for the updated version"). Don't rewrite the file automatically
  in the background.
- Only actually give me the updated file when I explicitly ask for it.
- Always give me the complete file when updating it, never a diff or a
  partial snippet, since I copy-paste the whole thing.
- If something in here turns out to be wrong or outdated, correct it
  plainly when I point it out - rewrite the line properly rather than
  just appending a caveat on top of the outdated one.

