# Changelog

Semantic versioning (major.minor.patch). The version also shows in the app
header and in `guide.html`; keep all three in step on every change.

## 1.5.1 — 2026-07-30

- Marking runway polish: weekends now read as smaller centred squares (clearer
  than the old underline); deadlines are marked with a dot above the square
  instead of a coloured outline; tapping a day toggles its summary open and
  shut; and the strip no longer clips at the left edge.

## 1.5.0 — 2026-07-30

- Marking runway on the dashboard: a forward heat strip of the coming days, each
  square shaded by how many cells you would mark that day to keep a flat load to
  every deadline (the same maths as Today's quota, projected across the whole
  horizon). Near days run hottest where deadlines overlap; the strip cools as
  each due date passes. Today and each due date are ringed; hover or tap a day
  for its load and which jobs drive it. It is a standing suggestion, not a
  target: rest a day and the remaining squares quietly rise.

## 1.4.1 — 2026-07-30

- Import trip hazards from two clearly separated sources: the bank (lists you
  prepared) and a live set (another job's working list). Importing copies the
  hazards in, adding only ones not already there and removing nothing.
- Import is now available in three places: creating a job, editing a job, and
  while marking (an "Import…" control in the Trip hazards column).
- This replaces the old linked "share a set" option with a copy: mark the first
  class, refine its hazards, then import that live set into the second class and
  pick up your refinements. Jobs already sharing a set keep working.

## 1.4.0 — 2026-07-30

- Trip hazard bank: prepare reusable hazard lists ahead of marking (in Set up),
  each labelled by assessment name and optional year level, from the marking
  schedule. When you create a job you can seed it from a bank list; the job gets
  its own copy, so editing it while marking never changes the bank.
- You can now delete a trip hazard while marking: each hazard in the sidebar has
  a delete control; removing one clears it from every paper that logged it.
- The job's trip-hazard picker now offers three starts: empty, seed a copy from
  the bank, or share another job's live set (same test in another class).

## 1.3.1 — 2026-07-24

- A follow-up can now be cleared: click the status box in the roster to send an
  absent student back to present, or use "Will sit — clear" / "Not sitting" on
  the paper. Clearer wording for resolving an absence.
- Streamlined the Current Paper view: removed Prev/Next (the roster is the
  student list), leaving a single "Next unmarked"; moved the moderation flag in
  beside the absence controls.

## 1.3.0 — 2026-07-24

- Trip hazards are now per assessment, not one global pile. Each job keeps its
  own set; when creating a job you can share another job's set (for the same
  test given to a second class). Existing data keeps one shared "legacy" set,
  so nothing logged is lost.
- Marking Jobs cards show sat / due / days-left as separate chips, with the
  days-left chip coloured by urgency, so each fact is quick to find.

## 1.2.0 — 2026-07-23

- The dashboard "Follow up" tile is now clickable: it opens a popup listing
  every student to follow up, with their class and assessment. Clicking one
  jumps straight to that student's paper.

## 1.1.1 — 2026-07-23

- Set up now shows a "N share a label" hint on any class with clashing
  students, so shared labels are visible without opening the editor; clicking
  it opens the label editor.

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
