---
name: synthetic-user
description: Browse a deployed/staging web app as a persona-driven synthetic user to surface BUGS (console errors, failed requests, 4xx/5xx, broken states) and UX FRICTION (confusing labels, dead ends, layout surprises). Use when the user wants to QA an app from a real-user's perspective, find usability problems, or do exploratory bug-hunting via the browser.
---

# Synthetic User

Drive a real browser through a deployed app *as a human persona would*, and produce an
evidence-backed report of two things at once: **bugs** and **UX friction**.

This skill uses the Playwright MCP browser tools (`browser_navigate`, `browser_snapshot`,
`browser_click`, `browser_type`, `browser_fill_form`, `browser_file_upload`,
`browser_console_messages`, `browser_network_requests`, `browser_take_screenshot`,
`browser_wait_for`).

## Where personas, journeys, and data live (two sources, merged)

Personas and journeys are loaded from **two** places and combined into one pool:

1. **Skill defaults** — `personas/` and `journeys/` next to this SKILL.md. These are the
   generic, reusable starting points (e.g. `new-user`, `confused-user`, `power-user`, plus the
   journey `_template.md`). They ship with the skill and apply to any project.
2. **Project folder** — a `synthetic-testing/` directory in the user's current repo, holding
   `personas/` and `journeys/` for *that* project (app-specific personas, real use-case
   journeys, the project's `.env.local`, and its `reports/`). Find it by searching the current
   working directory and its parents for a `synthetic-testing/` folder; if there isn't one,
   just use the skill defaults.

**Merge rules:**
- The pickable set of personas/journeys is the **union** of both folders (skill defaults +
  project). When the user names one (e.g. "the apple-reviewer persona"), resolve it from
  whichever folder has it.
- On a **name collision** (same filename in both), the **project folder wins** — it's the
  app-specific override.
- **Credentials, artifacts, and reports are project-scoped:** prefer the project folder's
  `.env.local`, `artifacts/`, and `reports/`. Fall back to the skill's own only when no project
  folder exists.
- When the user says "run all personas," run the union of both folders.

## Required inputs (fail fast if missing)

Do NOT guess these. If any is unset, stop and ask the user:

- **`base_url`** — the staging/deployed URL to test (e.g. `https://staging.example.app`).
- **`login`** — a test account's email + password. The persona logs in through the normal
  sign-in form, like a real user would.

**Where credentials come from:** read them from the project folder's `.env.local`
(`synthetic-testing/.env.local`), falling back to a `.env.local` next to this SKILL.md
(`SYNTHETIC_USER_BASE_URL`, `SYNTHETIC_USER_EMAIL`, `SYNTHETIC_USER_PASSWORD`). That file is
gitignored — never committed. If it's missing, copy `.env.example` to `.env.local` and tell
the user to fill it in (or let them paste creds inline for a one-off run). Never hardcode
credentials in this skill or echo the password into reports/logs.
- **`persona`** — *who* is using the app and *how* they behave (from the merged `personas/`
  pool — skill defaults + project folder). Default: run all of them in sequence.
- **`journey`** — *what* use case to perform: a file from the merged `journeys/` pool (e.g.
  `track-farm-equipment`). The journey supplies the goal and the "done" bar; the persona
  supplies mindset, pace, and quirks. If no journey is given, fall back to the persona's own
  default goal. You invoke this skill as **persona × journey**, e.g. "run the *confused-user*
  through the *upload-receipt* journey."
- **`artifacts_dir`** *(optional)* — folder of REAL files the personas can upload (photos,
  receipts, PDFs, etc.). **Defaults to the project folder's `artifacts/`, else the skill's own
  `artifacts/`.** When a flow asks for a file, list the folder and pick an appropriate one (by
  extension/name), then upload it via `browser_file_upload`. If the folder has no usable
  files (e.g. only its README), log the missed upload as a friction note and move on — do
  NOT fabricate or generate files.

Mutating actions (create / edit / delete) are **allowed** — this runs against staging with a
dedicated test account. Still announce destructive deletes in the report so the user knows
what test data was touched.

## The loop

For each persona × journey:

1. **Read the persona file** (`personas/<name>.md`) for mindset, pace, and quirks, **and the
   journey file** (`journeys/<name>.md`) for the goal, steps, and success criteria. The journey
   is *what* you're doing; the persona is *how*. With no journey, use the persona's default goal.
2. **Honest-eyes rule.** Navigate using ONLY what `browser_snapshot` / screenshots show on
   screen. Do NOT read the app's source, DOM selectors, or routes to find your way. The whole
   point is to experience the app like the persona — if *you* can't find the button, that's a
   finding, not a problem to engineer around. (Use selectors only when the *tooling* requires a
   ref to click; never to "know" where things are.)
3. **Pursue the goal step by step.** Before each action, narrate in first person what the
   persona expects ("I want to add my dishwasher — I'll look for an Add button, probably
   top-right"). Then act. Then observe what actually happened.
4. **Log two channels at every meaningful step:**
   - 🐛 **Bug** — after key actions, pull `browser_console_messages` and
     `browser_network_requests`. Record: console errors/warnings, failed requests (4xx/5xx),
     long hangs, broken/empty layout, unhandled states, anything that looks crashed.
   - 😕 **Friction** — first-person notes on confusion, surprise, bad/missing copy, unexpected
     layout, too many steps. Tag each with severity: **blocker** / **annoyance** / **nitpick**.
5. **Screenshot evidence.** Take a screenshot at each meaningful state (and ALWAYS on anything
   that looks broken) into the run folder. Reference filenames in the report.
6. **Don't give up silently.** If stuck, try the 2–3 things the persona would plausibly try,
   log each dead end, then move on. Getting stuck IS the finding.

## Evidence & output

Create a run folder under the **project folder's** `reports/` (`synthetic-testing/reports/`)
when one exists, else the skill's own `reports/`: `reports/<persona>-<a short slug>/` containing:
- `notes.md` — the running first-person log (timestamped steps, both channels).
- screenshots (`01-landing.png`, `02-add-form.png`, …).
- `console.log` — raw console + network errors captured.

Then write a final **verdict** at the top of `notes.md` (and summarize it back to the user):

```
## Verdict — <persona>, goal: <goal>
Result: completed / partially / blocked

### 🐛 Bugs (by severity)
- [blocker] <what> — evidence: 03-crash.png, console.log:L42

### 😕 Friction (by severity)
- [annoyance] I expected save at the top; it was hidden at the bottom — 04-form.png

### 👍 What felt good
- <a couple of genuine positives>
```

Keep findings specific and evidence-linked — no vague "could be improved." If you didn't see
it happen and capture it, don't claim it.

## When done

- Close the browser (`browser_close`).
- Summarize per-persona verdicts to the user, lead with blockers, link the run folders.
