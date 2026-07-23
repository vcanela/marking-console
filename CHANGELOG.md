# Changelog

Semantic versioning (major.minor.patch). The version also shows in the app
header and in `guide.html`; keep all three in step on every change.

## 1.1.0 — 2026-07-23

- Rosters now ask for first name and last initial only; a pasted full surname
  is reduced to its initial, so full surnames are no longer stored.
- Duplicate labels (same first name and initial) are numbered, and a class
  label editor in Set up lets you rename any shown label to tell students
  apart without adding full surnames.
- Existing classes are untouched; the change applies to newly added classes.

## 1.0.2 — 2026-07-21

- First-run welcome banner on the dashboard linking the guide; it disappears on
  its own once the first marking job exists.

## 1.0.1 — 2026-07-21

- Guide: recipes are now individual pages (less clutter, more room to explain).
- Guide: AI steps name no single assistant (Claude, ChatGPT, Gemini, etc.);
  clearer scenario phrasing.

## 1.0.0 — 2026-07-21

First shared release. Everything built to date:

- Two-layer model: classes are persistent rosters; assessments are markable
  jobs attached to a class, created and edited from the dashboard.
- Optional parts per assessment, with part-by-part marking across the class
  and auto-jump to the next part.
- Dashboard: overall progress, and a Today's quota that splits the day across
  jobs by deadline (per-job mini-bars); compact job cards with a due/class
  sort toggle.
- Trip hazards (per-part frequency) with a shared library; feedback-prompt,
  hazard-summary and full assessment-data exports.
- Absence split into follow-up (chase, surfaced amber) and not-sitting
  (accepted, quiet); moderation flags with comments.
- Un-tick from the paper or roster; per-student late sit dates; light and dark
  colourblind-safe themes; cross-device sync through a secret GitHub gist.
- A quiet, attributed literary quote on the dashboard and on meeting the quota.
