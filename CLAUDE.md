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
  `markingConsole.v3` on every mutation via `save()`. `load()` reads v3, else
  the old `markingConsole.v2` (flat marks, upgraded on read), else
  `markingConsole.v1` via `migrateV1()`. `normalize()` backfills missing keys
  and, via `upgradeMark()`, converts old flat marks to the part-based shape;
  older records are left in place as a safety net.
- **Two-layer model**: a **class** is a persistent roster; an **assessment**
  is a markable job attached to a class. An assessment can optionally be split
  into **parts**; the unit of marking is a **cell** (one student's one part).
  With no parts it uses one implicit part `PART_ALL`, so single-part
  assessments behave exactly as a whole-paper mark. A class carries no marking
  state.
- **Views**: `ui.view` toggles between the `dashboard` (landing) and the
  `workspace` (marking). Forced to `dashboard` on every launch. Body classes
  `view-dashboard` / `view-workspace` drive which `<main>` shows.
- **Two modals**: the Set up modal (`#settingsOverlay`) is class setup plus the
  rare backup/reset and sync config; it never touches jobs. Marking jobs
  (including their parts) are created and edited from the dashboard via the job
  modal (`#jobOverlay`, `openJobModal(editId?)` / `saveJob()`), and deleted
  from a card control (`deleteAssessment`). On edit the class is fixed;
  `reconcileParts` preserves a part's cells across renames/reorders.
- **Rendering**: full re-render per region (`renderHeader`, `renderDashboard`,
  `renderChips`, `renderPartBar`, `renderRoster`, `renderPaper`, `renderTags`),
  all called by `render()`. No virtual DOM, no diffing; the data is small
  enough that this is fine.
- **Per-assessment figures are derived** via `assessmentStats(a)`, counted in
  cells (student × part, missing students excluded): the single source of truth
  for a job's marked/remaining/target numbers. `partStats(a, pid)` does the
  same for one part. Tag counts are derived (`tagCountInPart`,
  `tagCountInAssessment`); do not introduce a stored counter.
- **Accessors**: `markOf`/`ensureMark` for a student's mark; `cellOf`/
  `ensureCell` for a student-part cell; `partsOf(a)` returns the real parts or
  the implicit single part; `curPart()` is the device-local part being marked.
  Read accessors return fresh defaults (not persisted) when absent.

### Data model

```js
S = {
  meta: { deviceId, updatedAt },   // deviceId per browser; updatedAt tracks non-entity changes (settings/import)
  classes: [{
    id, name, updatedAt,
    students: [{ id, first, last, label, updatedAt }]   // roster only; entry order kept
  }],
  assessments: [{
    id, classId, name,
    dateSat,                 // '' or 'YYYY-MM-DD'; a future date marks the job "upcoming"
    dueDate,                 // '' or 'YYYY-MM-DD'; drives that job's target
    createdAt, updatedAt,    // ISO strings
    parts: [{ id, name, updatedAt }],   // [] means one implicit part (PART_ALL)
    marks: {                 // keyed by student id
      [studentId]: {
        missing,             // bool; non-submission excludes all this student's cells
        flag, flagNote,      // moderation flag (whole paper) and its comment
        satOn,               // '' or 'YYYY-MM-DD'; per-student sit override (default = dateSat)
        updatedAt,           // ISO; student-level fields merge by this
        parts: {             // keyed by part id; one cell per part
          [partId]: {
            done,            // bool
            tags: [tagId, ...],  // mistakes observed on this part
            note,            // free text
            markedAt,        // ISO or null; drives "marked today"
            updatedAt        // ISO; drives sync merge (newer wins per cell)
          }
        }
      }
    }
  }],
  tags: [{ id, name, updatedAt }],   // global tag library, shared across everything
  tombstones: { [id]: deletedAtISO },// deleted class/assessment/tag ids, so a merge cannot resurrect them
  ui: { currentClass, currentAssessment, currentStudent, currentPart, view, theme, jobSort },  // device-local; NOT synced
  settings: {}   // currently unused
}
```

Every entity and cell carries `updatedAt`, stamped on each mutation, so sync
merges at the per-cell level (marking different parts on two devices both
survive). `ui` is deliberately device-local (theme, current selections and the
current part differ per device) and is excluded from the synced document.

## Key behaviours (do not break these)

1. **Labels**: roster keeps the order entered or pasted (no sorting). Label
   is first name + shortest unique surname prefix within the class
   (`buildLabels`). Collisions extend the prefix; identical full names get a
   numeric suffix. Privacy by construction: full names exist only in
   localStorage, labels are what render.
2. **Parts and part-by-part marking**: an assessment optionally splits into
   parts, edited from the job modal (one name per line; blank = single-part).
   In a multi-part job a **part bar** (`renderPartBar`) selects the current
   part; you mark that part across all students, then move on, keeping one
   meter stick per part. When a part is finished, `gotoNextUnmarked` auto-jumps
   to the next part with unmarked papers. Single-part jobs show no part bar and
   behave as before.
3. **Missing ≠ unmarked**: `mark.missing` is student-level (non-submission).
   A missing student contributes no cells to the denominator, so a job can
   reach 100% with non-submissions; missing count shows separately.
4. **Daily target** (per assessment): `ceil((remaining + markedToday) /
   dueDaysLeft)`, computed in `assessmentStats` in **cells** (student × part).
   Cells marked today count toward today, so the target stays stable through
   the day. Percentage and progress are cell-based too; for a single-part job a
   cell equals a paper, so the numbers match the old behaviour. Each job has
   its own due date; the dashboard sorts jobs into three groups (markable now,
   then upcoming, then complete). A future `dateSat` marks a job "upcoming"
   (nothing to mark yet). A student can sit late: `satOn` on their mark
   overrides the job's `dateSat`, edited from the paper view.
5. **Mark done / un-tick**: `markDone` marks the current cell (tick animation,
   respects `prefers-reduced-motion`; green row flash; auto-advance after
   350 ms). It is reversible: a done cell shows an **Un-mark** button
   (`unmarkCell`, no advance), and the roster status box toggles done directly
   (`toggleDoneFor`) for quick corrections after moderation.
6. **Trip hazards**: the canonical term everywhere (input, hints, the
   **Copy hazard summary** button), coloured purple to tie to the Trip hazards
   column. Hazards attach to the current **cell** (student-part); clicking a
   sidebar hazard toggles it; adding a new one auto-attaches it. Sidebar sorts
   by frequency within the current part, most common first, with bars.
7. **Feedback prompt export** (`copyPrompt`): clipboard text with first name,
   class, assessment name, and the student's tagged issues and notes gathered
   across every part (grouped by part name when multi-part), plus fixed
   constraints (address student directly, encouraging but honest, ~80 words, do
   not invent issues beyond those listed). The owner pastes this into Claude.
8. **Two assessment exports, both in the workspace** (`class-summary-row`):
   **hazard summary** (`copyAssessmentSummary`) is the trip-hazard frequency
   table for reteaching; **assessment data** (`copyAssessmentData`) is a full
   anonymous dump, every paper broken down by part with status/hazards/note
   (papers numbered, no names) plus hazard totals and moderation flags, for
   sharing or AI analysis.
9. **Backup**: JSON export/import of the whole state. localStorage is
   fragile (Safari eviction); this is the safety net. Import accepts a v3 or v2
   backup (has `assessments`) or an old v1 backup (has `classes`); `normalize`
   upgrades older shapes on load.
10. **All user text is escaped** through `esc()` before hitting innerHTML.
    Keep it that way for any new rendering code.
11. **Sync** (optional, off until configured): whole state syncs through one
    secret GitHub gist. The token and gist id live in their own localStorage
    key (`markingConsole.sync`), never in `S`, so they never reach the gist or
    a backup. Sync is pull, merge, push in one action (`syncNow`); the merge
    (`mergeDocs`) is deterministic, commutative and idempotent, combining both
    sides by per-entity/per-cell `updatedAt` with tombstones for deletions, so
    two devices converge with no data loss and no forced conflict choice (mark
    Section A on one device and Section B on another and both survive).
    Auto-sync runs on open and debounced after edits; a header button and a Set
    up section drive it manually. Comparison and the gist body use a **canonical
    serialisation** (`docString`: sorted keys, entity arrays sorted by id,
    deviceId excluded) so identical data never ping-pongs between devices. The
    gist holds student names, so it is secret by construction (`public:false`).
12. **Moderation**: a whole-paper flag (`mark.flag`) plus a comment
    (`mark.flagNote`), toggled from the paper view (yellow `--flag`). Flagged
    students show a ⚑ in the roster; the flag and comment appear in the
    assessment data export as a moderation record.
13. **Daily quota and motivation**: `dailyQuota()` sums cells marked today and
    today's targets across all active jobs; a second header bar shows progress
    toward it, green when met (replaces the old "missing" header stat). On the
    dashboard the quota section breaks the day down **per job** (`quotaRowsHtml`):
    one mini-bar per dated active job of today-done vs its own daily target,
    sorted most-urgent-first, so a Sunday deadline visibly needs more today
    than a Tuesday one. The Marking Jobs cards are compact (no progress bar;
    overall % as text) with a Due date / Class sort toggle (`ui.jobSort`,
    `setJobSort`). A curated, attributed literary quote (`QUOTES`, `quoteLine`)
    sits high on the dashboard and shows once when the quota is met. Keep the
    quotes literary and unfussy: no self-help, no exclamation marks. (No archive
    yet: completed jobs sink to the bottom of the list.)

## Design language

Dark instrument-panel aesthetic, consistent with the owner's other tools
(Jaynes–Cummings simulation, lesson planner). IBM Plex Sans for UI, IBM Plex
Mono for data readouts and labels.

Colours are the Okabe-Ito colourblind-safe palette, held in CSS custom
properties at the top of the stylesheet with semantic names: `--accent` =
orange (primary accent / active), `--success` = green (completion), `--danger`
= vermillion (missing / danger), `--info` = sky blue on dark / blue on light
(copy, export, info), `--tag` = purple (trip-hazard system, and the hazard
controls carry this colour so the concept is found by colour), `--flag` =
yellow (flagged for moderation); each has a `-dim` companion where needed.
Reuse the variables;
do not introduce new hex values inline. The light theme (`body.theme-light`)
overrides these same variables with darker shades so text keeps contrast on
cream; the hue relationships that carry the colourblind distinction are kept.

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
names, a single-part job (mark/un-tick/missing cycle, hazard toggle), a
multi-part job (part bar, mark a part across students, auto-jump to the next
part, per-part hazard counts), un-ticking from the roster box, flagging a
student for moderation with a comment, the daily quota bar filling and turning
green (and the quote on meeting it), both clipboard exports, JSON
export/import round-trip, theme toggle, v2→v3 and v1→v3 migration (load with
only the older key present),
and a reload to confirm persistence. For sync, the merge (`mergeDocs`) can be tested
without a token by mocking `gistGet`/`gistPatch`/`gistCreate` in the console:
check convergence (`docString(mergeDocs(A,B)) === docString(mergeDocs(B,A))`),
idempotence, that two devices' marks on different papers both survive, and that
deletions do not resurrect. The live gist round-trip needs a real token.
