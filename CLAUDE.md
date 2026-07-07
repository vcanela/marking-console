# CLAUDE.md — marking-console

## What this is

A single-file HTML marking tracker for a physics/science teacher working through
a large holiday marking load (multiple classes, ~150 papers, hard deadline).
It tracks per-paper completion, captures observed mistakes as clickable tags,
and exports ready-made Claude prompts for writing bespoke student feedback.

The design philosophy behind it: AI's role in marking is preventing
fatigue-induced degradation; every judgement stays with the teacher. The tool
offloads memory (which mistakes have I seen) and drafting (feedback wording),
never the marking itself.

## Owner and working style

- Owner is a teacher with a physics PhD, not a developer. He reads HTML/JS with
  context and debugs from behaviour. Skip basics (Git, deployment, localStorage);
  explain framework internals or async behaviour before assuming knowledge.
- Explain reasoning before building. For anything non-trivial, propose a plan
  and wait for approval. Never make changes that were not asked for.
- Disagree openly before complying if an approach has a problem.
- Feedback arrives as numbered points; respond to each point specifically.
- Prose style in any written output: commas, full stops, colons, semicolons
  only. No dashes.

## Hard constraints

- One self-contained HTML file: `marking-console.html`. No frameworks, no build
  step, no external dependencies except the Google Fonts import (IBM Plex,
  degrades gracefully to system fonts offline).
- Deployable to GitHub Pages as a static file. All data lives in the browser's
  localStorage; the file itself never contains student data, so public hosting
  is safe.
- Keep the code readable enough that the owner can maintain it by reading it.
- Portability and shareability beat any particular technology. Single-file HTML
  is the current expression of that, not a dogma; propose alternatives only
  with a clear reason.

## Architecture

Everything is in `marking-console.html`: CSS in `<style>`, app in one
`<script>` block, vanilla JS, no modules.

- **State**: single object `S`, persisted to `localStorage` under key
  `markingConsole.v2` on every mutation via `save()`. `load()` reads v2, and
  if absent falls back to the old `markingConsole.v1` and converts it via
  `migrateV1()` (the v1 record is left in place as a safety net, not deleted).
  `normalize()` backfills any missing keys so partial states never crash.
- **Two-layer model**: a **class** is a persistent roster; an **assessment**
  is a markable job attached to a class, holding that job's per-student marks
  in a `marks` map keyed by student id. A class carries no marking state.
- **Views**: `ui.view` toggles between the `dashboard` (landing) and the
  `workspace` (marking). Forced to `dashboard` on every launch. Body classes
  `view-dashboard` / `view-workspace` drive which `<main>` shows.
- **Two modals**: the Set up modal (`#settingsOverlay`) is class setup only,
  plus the rare calibration and backup/reset config; it never touches jobs.
  Marking jobs are created and edited from the dashboard via the job modal
  (`#jobOverlay`, `openJobModal(editId?)` / `saveJob()`), and deleted from a
  card control (`deleteAssessment`). On edit the class is fixed.
- **Rendering**: full re-render per region (`renderHeader`, `renderDashboard`,
  `renderChips`, `renderRoster`, `renderPaper`, `renderTags`), all called by
  `render()`. No virtual DOM, no diffing; the data is small enough that this
  is fine.
- **Per-assessment figures are derived** via `assessmentStats(a)`: the single
  source of truth for a job's marked/remaining/missing/target numbers, used by
  the header, dashboard, and settings. Tag counts are derived too
  (`tagCountInAssessment`); do not introduce a stored counter.
- **Marks accessors**: `markOf(a, sid)` reads (returns a fresh default when
  absent, not persisted); `ensureMark(a, sid)` creates on first write.

### Data model

```js
S = {
  classes: [{
    id, name,
    students: [{ id, first, last, label }]   // roster only; entry order kept
  }],
  assessments: [{
    id, classId, name,
    dateSat,                 // '' or 'YYYY-MM-DD'; a future date marks the job "upcoming"
    dueDate,                 // '' or 'YYYY-MM-DD'; drives that job's target
    createdAt,               // ISO string
    marks: {                 // keyed by student id
      [studentId]: {
        status,              // 'unmarked' | 'marked' | 'missing'
        tags: [tagId, ...],  // mistakes observed on this paper
        note,                // free text, included in feedback prompt
        markedAt,            // ISO string or null; drives "marked today"
        satOn                // '' or 'YYYY-MM-DD'; per-student sit override (default = dateSat)
      }
    }
  }],
  tags: [{ id, name }],      // global tag library, shared across everything
  ui: { currentClass, currentAssessment, currentStudent, view, theme },
  settings: { calibrateEvery }   // calibrateEvery: 0 = off
}
```

## Key behaviours (do not break these)

1. **Labels**: roster keeps the order entered or pasted (no sorting). Label
   is first name + shortest unique surname prefix within the class
   (`buildLabels`). Collisions extend the prefix; identical full names get a
   numeric suffix. Privacy by construction: full names exist only in
   localStorage, labels are what render.
2. **Missing ≠ unmarked**: three paper states. Missing papers are excluded
   from an assessment's progress denominator so a job can reach 100% with
   non-submissions; missing count shows separately.
3. **Daily target** (per assessment): `ceil((remaining + markedToday) /
   dueDaysLeft)`, computed in `assessmentStats`. Papers marked today count
   toward today, so the target stays stable through the day instead of
   shrinking as you work. Each job has its own due date; the dashboard sorts
   jobs into three groups (markable now, then upcoming, then complete). A job
   whose `dateSat` is in the future is "upcoming" (nothing to mark yet) and
   sinks below active jobs. A student can sit late: `satOn` on their mark
   overrides the job's `dateSat`, edited from the paper (student) view.
4. **Mark done flow**: tick animation (respects `prefers-reduced-motion`),
   green row flash, auto-advance to next unmarked paper after 350 ms. The
   satisfying tick is a feature requirement, not decoration.
5. **Tag interaction**: clicking a sidebar tag toggles it on the current
   paper. Adding a new tag while a paper is selected auto-attaches it (you
   just saw the mistake). Sidebar sorts by frequency within the current
   assessment, most common first, with proportional bars.
6. **Feedback prompt export** (`copyPrompt`): clipboard text containing first
   name, class, assessment name, tagged issues as a list, optional marker's
   note, and fixed constraints (address student directly, encouraging but
   honest, ~80 words, do not invent issues beyond those listed). The owner
   pastes this into Claude to generate the comment.
7. **Assessment tag summary** (`copyAssessmentSummary`): frequency table for
   the current assessment, used for planning reteaching of that topic.
   **Overall summary** (`copyOverallSummary`, dashboard): anonymous text
   across all jobs, grouped by assessment name so the same test sat by several
   classes lines up for comparison; no student names, safe to share or feed to
   an AI. Includes a per-class "N sat on another date" count from `satOn`.
8. **Calibration check**: optional, off by default. Every N marked papers in
   an assessment, a banner suggests re-reading the first marked paper for
   drift.
9. **Backup**: JSON export/import of the whole state. localStorage is
   fragile (Safari eviction); this is the safety net. Import accepts a v2
   backup (has `assessments`) or an old v1 backup (has `classes`), converting
   the latter via `migrateV1()`.
10. **All user text is escaped** through `esc()` before hitting innerHTML.
    Keep it that way for any new rendering code.

## Design language

Dark instrument-panel aesthetic, consistent with the owner's other tools
(Jaynes–Cummings simulation, lesson planner). IBM Plex Sans for UI, IBM Plex
Mono for data readouts and labels. Palette in CSS custom properties at the
top of the stylesheet: amber (`--amber`) is the primary accent and active
state, green strictly for completion, red for missing, cyan for copy/export
actions. Reuse the variables; do not introduce new hex values inline.

Two themes: night (default) and a cream/pastel light mode, toggled from the
header and persisted in `ui.theme`. Light mode is a second palette under
`body.theme-light` that overrides the same variables, so every rule inherits
it; keep new colours as variables so both themes stay in sync.

## Roadmap (agreed direction, not yet built)

- **Tags as reteaching data**: generate a targeted starter quiz from a
  class's tag frequency table.
- **Tags as distractor metadata**: tag naming should stay consistent enough
  to transfer into the owner's org-mode IB exam question database, where
  observed mistakes become distractor candidates for future questions.
- Possible later extension to multi-user (colleagues marking shares); the
  current build is deliberately single-user.

## Testing

No test framework. Sanity check after changes:
`node --check` on the extracted script block, then manual test of: first-run
setup (add a class, then an assessment), roster paste with duplicate first
names, opening a job from the dashboard, mark/unmark/missing cycle, tag
toggle, both clipboard exports, JSON export/import round-trip, theme toggle,
v1→v2 migration (load with only a `markingConsole.v1` key present), and a
reload to confirm persistence.
