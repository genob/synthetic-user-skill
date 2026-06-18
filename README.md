# synthetic-user

A [Claude Code](https://claude.com/claude-code) skill that browses a deployed/staging web app **as a persona-driven synthetic user** to surface two things at once:

- 🐛 **Bugs** — console errors, failed requests (4xx/5xx), broken/empty states, hangs.
- 😕 **UX friction** — confusing labels, dead ends, layout surprises, too many steps.

It drives a real browser (via the Playwright MCP tools) through the app the way a human would, then produces an evidence-backed report with screenshots, console/network logs, and a per-persona verdict.

## How it works

You invoke the skill as **persona × journey** — e.g. "run the *confused-user* through the *upload-receipt* journey." The journey supplies the goal and the "done" bar; the persona supplies mindset, pace, and quirks.

A key rule is the **honest-eyes rule**: navigation uses only what's visible on screen (snapshots/screenshots), never the app's source or routes. If the synthetic user can't find the button, that's a finding — not a problem to engineer around.

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

Each run writes `reports/<persona>-<slug>/` containing `notes.md` (first-person timestamped log), screenshots, and `console.log`. The top of `notes.md` carries a **verdict**: result (completed / partial / blocked), bugs by severity, friction by severity, and what felt good.
