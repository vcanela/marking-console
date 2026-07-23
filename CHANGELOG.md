# Changelog

Semantic versioning (major.minor.patch). The version also shows in the app
header and in `guide.html`; keep all three in step on every change.

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
