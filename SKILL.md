---
name: synthetic-user
description: Run a moderated usability session on a deployed/staging web app — a persona-driven synthetic USER drives the app and thinks aloud while an OBSERVER watches over their shoulder, interprets the behavior, and notes technical bugs the user can't see (console errors, failed requests, 4xx/5xx, broken states). Produces an over-the-shoulder transcript plus an analytical findings report. Use when the user wants to QA an app from a real-user's perspective, observe usability/think-aloud behavior, find friction, or do exploratory bug-hunting via the browser.
---

# Synthetic User

This skill simulates a **moderated usability session**: the kind of insight you get from
sitting behind a real user while they think aloud. It runs two roles at once.

- 🧑 **The Participant** — a persona who *is* the live user. They drive the app through the
  browser, narrate a continuous think-aloud stream (expectations, confusion, emotions,
  decisions), and experience the product honestly. They do **not** analyze, and they do **not**
  see anything a normal user couldn't (no console, no DOM, no source).
- 🔬 **The Observer** — you, watching over the Participant's shoulder. You read their behavior,
  pull the instrumentation the user can't see (`browser_console_messages`,
  `browser_network_requests`), and **interpret**: why did they hesitate, what did the label
  mislead them into, what silently broke that they never noticed. You produce the analysis and
  the technical bug findings.

The same agent plays both roles, switching voice deliberately: act + think aloud *as the
Participant*, then step back and analyze *as the Observer*. Keep the two voices distinct — the
Participant never says "this is a 500 error"; the Observer does.

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

## App-wide standing instructions (`app-guide.md`)

Beyond *who* (persona) and *what* (journey), an app has **house rules that apply to every
run**. These live in a single `app-guide.md` **in the project folder**
(`synthetic-testing/app-guide.md`), loaded at the start of every session and treated as
standing context layered *underneath* persona × journey. The skill does **not** ship its own
app-guide — it's inherently app-specific. If the project folder has no `app-guide.md`, proceed
without one (and you may offer to scaffold one for the user from the structure below).

What belongs in it (and what the skill should honor):
- **Access & login quirks** — e.g. "sign in via the SSO button, not the email form."
- **Pre-session setup** — table-stakes steps to do before the journey proper, e.g. "dismiss
  the cookie banner and welcome modal." These are *setup*, not part of what's being evaluated.
- **Console/network noise to ignore (Observer)** — known third-party warnings/requests that
  are NOT bugs (analytics, Sentry, etc.). The Observer must not report these as findings.
- **Test-data rules** — naming conventions (e.g. prefix `TEST-`), what's safe to delete, and
  records/accounts to never touch.
- **Out-of-bounds areas** — flows to avoid entirely (e.g. billing that charges a real card).
- **Domain glossary & known issues** — so the Observer doesn't re-report what's already known.

**Hard boundary — `app-guide.md` must NOT contain navigation hints.** It cannot tell the
Participant where buttons/menus are or which route to visit; that would break the honest-eyes
rule, which is the whole point. It's limited to access, setup, noise-filtering, data-safety,
and known issues — table-stakes context and Observer-side knowledge, never a UI map. If a
specific journey/persona instruction conflicts with the guide, the more specific one wins.

## Required inputs (fail fast if missing)

Do NOT guess these. If any is unset, stop and ask the user:

- **`base_url`** — the staging/deployed URL to test (e.g. `https://staging.example.app`).
- **`login`** — a test account's email + password. The Participant logs in through the normal
  sign-in form, like a real user would.

**Where credentials come from:** read them from the project folder's `.env.local`
(`synthetic-testing/.env.local`), falling back to a `.env.local` next to this SKILL.md
(`SYNTHETIC_USER_BASE_URL`, `SYNTHETIC_USER_EMAIL`, `SYNTHETIC_USER_PASSWORD`). That file is
gitignored — never committed. If it's missing, copy `.env.example` to `.env.local` and tell
the user to fill it in (or let them paste creds inline for a one-off run). Never hardcode
credentials in this skill or echo the password into reports/logs.
- **`persona`** — *who* the Participant is and *how* they behave (from the merged `personas/`
  pool — skill defaults + project folder). Default: run all of them in sequence.
- **`journey`** — *what* use case to perform: a file from the merged `journeys/` pool (e.g.
  `track-farm-equipment`). The journey supplies the goal and the "done" bar; the persona
  supplies mindset, pace, and quirks. If no journey is given, fall back to the persona's own
  default goal. You invoke this skill as **persona × journey**, e.g. "run the *confused-user*
  through the *upload-receipt* journey."
- **`artifacts_dir`** *(optional)* — folder of REAL files the Participant can upload (photos,
  receipts, PDFs, etc.). **Defaults to the project folder's `artifacts/`, else the skill's own
  `artifacts/`.** When a flow asks for a file, list the folder and pick an appropriate one (by
  extension/name), then upload it via `browser_file_upload`. If the folder has no usable
  files (e.g. only its README), the Observer logs the missed upload as a friction note and the
  Participant moves on — do NOT fabricate or generate files.

Mutating actions (create / edit / delete) are **allowed** — this runs against staging with a
dedicated test account. Still announce destructive deletes in the report so the user knows
what test data was touched.

## The loop

**First, once per session: load `app-guide.md`** from the project folder
(`synthetic-testing/app-guide.md`) if present, and hold its house rules as standing context
for everything below — honoring the hard boundary above (no navigation hints). If there's
none, continue without it.

For each persona × journey:

1. **Read the persona file** (`personas/<name>.md`) for the Participant's mindset, pace, and
   quirks, **and the journey file** (`journeys/<name>.md`) for the goal, steps, and success
   criteria. The journey is *what* the Participant is doing; the persona is *how*. With no
   journey, use the persona's default goal.
2. **Honest-eyes rule (Participant).** The Participant navigates using ONLY what
   `browser_snapshot` / screenshots show on screen. They do NOT read the app's source, DOM
   selectors, or routes to find their way. If the Participant can't find the button, that's a
   finding — not a problem to engineer around. (Use selectors only when the *tooling* requires
   a ref to click; never to "know" where things are.)
3. **Think aloud, continuously (Participant).** This is the heart of the session. Before,
   during, and after each action, narrate in first person — not just *what* you'll click, but
   first impressions, what you expect to happen, what you're hunting for, your reaction when
   reality differs, and your emotions. Verbalize the messy middle: "okay this is… a dashboard?
   I just want to add my tractor… I'd expect a plus button top-right… nope… maybe this menu?
   ugh, nothing." Capture hesitation at decision points ("two buttons both look right, I'm
   guessing"), wrong turns, and backtracks. Tag emotional beats: 🤔 confused, 😤 frustrated,
   😀 delighted, 😲 surprised, ↩️ backtracked. **Screenshot the moment** the persona is
   reacting to and cite the filename next to the beat (e.g. "I can't find Add — 02-dashboard.png").
4. **Observe and interpret (Observer).** After each meaningful action, step out of the
   Participant and put on the researcher hat. **Every finding cites a screenshot** as its
   evidence (take a fresh one if the relevant state isn't already captured):
   - 🐛 **Technical bug** — pull `browser_console_messages` and `browser_network_requests`
     (the Participant can't see these). Record console errors/warnings, failed requests
     (4xx/5xx), long hangs, broken/empty layout, unhandled states. Flag **silent failures**
     especially: cases where the request failed but the Participant thinks it succeeded.
     Evidence: the screenshot of the broken/misleading state + the `observer-console.log` line.
   - 😕 **Usability finding** — interpret the behavior you just watched. Don't just restate it;
     diagnose it. "She hesitated 8s and guessed wrong → the 'Submit' vs 'Save draft' labels are
     ambiguous." Tag severity: **blocker** / **annoyance** / **nitpick**, and cite the screenshot.
5. **Screenshot every beat — observations are backed by images whenever possible.** Take a
   screenshot (`browser_take_screenshot`) at each meaningful state, and ALWAYS on anything that
   looks broken or that any observation refers to. The rule: **if the Participant felt it or the
   Observer flagged it, there should be an image of it.** Capture it at (or immediately after)
   the moment, name it in sequence (`01-landing.png`, `02-add-form.png`, …), and reference the
   filename in whichever document logged the observation. Dense frames + the think-aloud
   transcript should replay like a screen recording. Only skip a frame when a screenshot is
   genuinely impossible (e.g. mid-navigation) — note that rather than leaving the claim
   unbacked.
6. **Don't give up silently (Participant).** If stuck, try the 2–3 things the persona would
   plausibly try, think aloud through each dead end, then move on. Getting stuck IS the finding
   — and the Observer notes *where* and *why*.

## Evidence & output

Create a run folder under the **project folder's** `reports/` (`synthetic-testing/reports/`)
when one exists, else the skill's own `reports/`. **Name it prescriptively** — do not invent a
freeform slug:

```
reports/<persona>__<journey>__<YYYY-MM-DD>/
```

- `<persona>` — the persona filename stem (e.g. `confused-user`).
- `<journey>` — the journey filename stem (e.g. `upload-receipt`); use `default-goal` when no
  journey was given.
- `<YYYY-MM-DD>` — today's date.
- Segments are joined with **double underscores** (`__`) so the single-hyphen names inside each
  segment stay unambiguous.
- **On collision** (same persona × journey already run today), append `-2`, `-3`, … — never
  overwrite a prior run.

Example: `reports/confused-user__upload-receipt__2026-06-19/`. The folder contains **two
distinct documents** plus evidence:

- **`user-journal.md`** — the **Participant's** over-the-shoulder transcript. Pure first-person,
  experiential, timestamped think-aloud. This is the "sitting behind the user" recording. No
  analysis, no technical jargon — just what the user thought, felt, and did, beat by beat, with
  emotion tags and screenshot references.
- **`observer-notes.md`** — the **Observer's** analytical report. Third-person interpretation of the
  Participant's behavior plus the technical findings, cross-linked to `user-journal.md` by timestamp
  and to screenshots by filename. The verdict goes at the top (see below).
- **`observer-console.log`** — raw console + network errors captured by the Observer.
- screenshots (`01-landing.png`, `02-add-form.png`, …).

**Cross-linking is what makes the split work:** `user-journal.md` and `observer-notes.md` share timestamps,
and both reference the same screenshot filenames, so any finding traces back to the exact
moment the user lived it.

The Observer writes the **verdict** at the top of `observer-notes.md` (and summarizes it back to the
user):

```
## Verdict — <persona>, journey: <journey>
Result: completed / partially / blocked
Task success: ✅ / ⚠️ / ❌   ·   Time-on-task: ~Nm   ·   Attempts to succeed: N   ·   Got stuck: <where>

### 🐛 Bugs (by severity)
- [blocker] Save silently failed — 500 on POST /equipment, no error shown; user believed it saved.
  Evidence: 03-after-save.png, observer-console.log:L42, user-journal.md @00:04:12

### 😕 Usability findings (by severity)
- [annoyance] User hunted 20s for "Add", expecting it top-right; it was a floating button bottom-left.
  Evidence: 02-dashboard.png, user-journal.md @00:01:30

### 🗣️ Standout quotes
- "wait, did that even save? it just… went back to the list" — user-journal.md @00:04:20

### 👍 What felt good
- <a couple of genuine positives, in the user's voice>
```

Keep findings specific and evidence-linked — no vague "could be improved." If the Observer
didn't see it happen and capture it, don't claim it.

## When done

- Close the browser (`browser_close`).
- Summarize per-persona verdicts to the user, lead with blockers and the standout quotes, and
  link both `user-journal.md` (the over-the-shoulder transcript) and `observer-notes.md` (the analysis).
