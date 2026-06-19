# App Guide — standing instructions for every run

House rules that apply to **every** synthetic-user session on this app, regardless of persona
or journey. Loaded once at the start of each run and held as standing context, layered
underneath persona × journey.

> ⚠️ **Do NOT put navigation hints here** (no "the Add button is top-right", no routes/URLs to
> visit). That would break the honest-eyes rule — the Participant is supposed to discover the
> UI themselves. Keep this to access, setup, noise-filtering, data-safety, and known issues.

This file is the generic template. Copy it to your project's `synthetic-testing/app-guide.md`
and fill it in for your app; the project copy wins over this one. Delete sections you don't need.

## Access & login
<!-- How a real test user signs in. Quirks only — not a click-by-click map. -->
- e.g. Sign in with the **SSO** button, not the email/password form.
- e.g. Test accounts skip MFA; if prompted, that's a bug.

## Pre-session setup
<!-- Table-stakes steps to do before the journey proper. Setup, not part of the evaluation. -->
- e.g. Dismiss the cookie banner and the first-run welcome modal before starting.

## Console / network noise to ignore (Observer)
<!-- Known third-party output that is NOT a bug. The Observer must not report these. -->
- e.g. `analytics.js` / Segment warnings on every page.
- e.g. Sentry beacon requests; a 401 from `/api/flags` on first paint (retried, harmless).

## Test-data rules
<!-- Keep the staging account clean and safe. -->
- e.g. Prefix anything you create with `TEST-`.
- e.g. Only delete items you created this session; never touch records owned by `admin@…`.

## Out of bounds
<!-- Flows to avoid entirely. -->
- e.g. Do not enter the billing/upgrade flow — it charges a real card.
- e.g. Do not send invites to real email addresses.

## Domain glossary
<!-- App-specific terms so the Observer interprets correctly. -->
- e.g. "Asset" = any tracked item; "Reminder" = a scheduled maintenance task.

## Known issues (don't re-report)
<!-- Things already on the backlog, so the Observer doesn't surface them again. -->
- e.g. Search is case-sensitive (known, ticket APP-1234).
