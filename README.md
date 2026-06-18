# synthetic-user

A [Claude Code](https://claude.com/claude-code) skill that runs a **moderated usability session** on a deployed/staging web app — the closest thing to sitting behind a real user while they think aloud. It runs two roles at once:

- 🧑 **The Participant** — a persona who *is* the live user. Drives the app, thinks aloud continuously (expectations, confusion, emotions, decisions), and experiences the product honestly. Sees only what a real user sees — no console, no source.
- 🔬 **The Observer** — the researcher watching over the Participant's shoulder. Reads their behavior, pulls the instrumentation the user can't see (console errors, failed requests, 4xx/5xx), and **interprets**: why they hesitated, what a label misled them into, what silently broke that they never noticed.

It drives a real browser (via the Playwright MCP tools) and produces two artifacts: an **over-the-shoulder transcript** (`user-journal.md`, the Participant's voice) and an **analytical findings report** (`observer-notes.md`, the Observer's voice) — cross-linked by timestamp and screenshot.

## How it works

You invoke the skill as **persona × journey** — e.g. "run the *confused-user* through the *upload-receipt* journey." The journey supplies the goal and the "done" bar; the persona supplies mindset, pace, and quirks.

A key rule is the **honest-eyes rule**: the Participant navigates using only what's visible on screen (snapshots/screenshots), never the app's source or routes. If they can't find the button, that's a finding — not a problem to engineer around. Meanwhile the Observer catches what the Participant *can't*: silent failures, technical bugs, and patterns across the user's hesitations.

## Layout

| Path | Purpose |
|------|---------|
| `SKILL.md` | The skill definition Claude Code loads. |
| `personas/` | Default personas (`new-user`, `confused-user`, `power-user`). |
| `journeys/` | Default journeys + `_template.md`. |
| `artifacts/` | Real files personas can upload (photos, receipts, PDFs). |
| `reports/` | Run output: notes, screenshots, console logs. |
| `.env.example` | Template for credentials — copy to `.env.local` (gitignored). |

## Personas, journeys, and data — two sources, merged

The pickable set is the **union** of:

1. **Skill defaults** — the `personas/` and `journeys/` shipped here.
2. **Project folder** — a `synthetic-testing/` directory in the repo under test, holding app-specific personas, journeys, `.env.local`, `artifacts/`, and `reports/`.

On a name collision, the **project folder wins**. Credentials, artifacts, and reports are project-scoped, falling back to the skill's own only when no project folder exists.

## Setup

1. Copy `.env.example` to `.env.local` and fill in:
   - `SYNTHETIC_USER_BASE_URL` — the staging/deployed URL to test.
   - `SYNTHETIC_USER_EMAIL` / `SYNTHETIC_USER_PASSWORD` — a dedicated test account.
2. (Optional) Drop uploadable files into `artifacts/`.
3. Invoke the skill, naming a persona and journey (or run all personas in sequence).

`.env.local` is gitignored and never committed — credentials are never hardcoded or echoed into reports.

> **Note:** Mutating actions (create/edit/delete) are allowed — this runs against staging with a dedicated test account. Destructive deletes are announced in the report.

## Output

Each run writes `reports/<persona>-<slug>/` containing:

- **`user-journal.md`** — the Participant's over-the-shoulder transcript: pure first-person, timestamped think-aloud with emotion tags (🤔 😤 😀 😲 ↩️). Reads like sitting behind the user.
- **`observer-notes.md`** — the Observer's analytical report. The top carries a **verdict**: task success, time-on-task, attempts, where they got stuck, bugs by severity, usability findings by severity, standout quotes, and what felt good.
- **`observer-console.log`** — raw console + network errors.
- screenshots (`01-landing.png`, …) — a dense frame at every beat, so journal + screenshots replay like a recording.

`user-journal.md` and `observer-notes.md` share timestamps and reference the same screenshot filenames, so any finding traces back to the exact moment the user lived it.


## Example invocations

Invoke in natural language — name a **persona**, a **journey**, or both. A few to show the range:

**Persona × journey (the core form)**
- "Run the **new-user** through the **track-farm-equipment** journey on staging."
- "Send the **confused-user** through **upload-receipt** and see what breaks."
- "Have the **power-user** do a batch session — add a few assets, attach a file, edit, search, delete a test item."

**First-impression / onboarding check**
- "Be a brand-new user seeing this app for the first time — can you figure out what to do and add your first thing?"
- "Watch a first-time user try to reach their first 'aha' moment and tell me where they stall."

**Bug & edge hunting**
- "Take the **confused-user** persona and try to break the create-reminder flow — messy input, back button, double-clicks."
- "Exploratory bug hunt: run all the experiential personas and surface any console errors, 4xx/5xx, or silent failures."

**Expert reviews**
- "Run the **qa-analyst** over the checkout flow and give me defects with repro steps and severity."
- "Have the **ux-evaluator** do a heuristic evaluation of the dashboard — Nielsen's 10, with severity ratings and suggested fixes."

**Run everyone**
- "Run **all personas** through the **invite-a-teammate** journey and summarize the blockers across all of them."

**Ad-hoc / no journey**
- "Just poke around the settings page as an impatient power user and tell me what's annoying."
- "Here are creds for our preview env — `<url>`, `<email>`/`<pw>`. Walk the signup-to-first-asset flow as a skeptical new user."

In every case you get back the over-the-shoulder transcript (`user-journal.md`), the analytical findings + verdict (`observer-notes.md`), screenshots, and `observer-console.log` — with blockers and standout quotes summarized up front.