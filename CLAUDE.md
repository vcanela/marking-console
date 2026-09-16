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

- The app is one self-contained HTML file: `marking-console.html`. No
  frameworks, no build step, no external dependencies except the Google Fonts
  import (IBM Plex, degrades gracefully to system fonts offline). Two sibling
  static pages ship with it: `index.html` (redirects to the app) and
  `guide.html` (the user guide, same self-contained rules and visual language).
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
    weight,                  // 1 mechanical | 2 average (default) | 3 tricky; scales load points in the quota/runway (behaviour 4)
    archived,                // bool; filed away from the active list and all headline figures (behaviour 16)
    createdAt, updatedAt,    // ISO strings
    hazardSet,               // id into S.hazardSets; assessments can share one set
    parts: [{ id, name, updatedAt }],   // [] means one implicit part (PART_ALL)
    marks: {                 // keyed by student id
      [studentId]: {
        absence,             // '' | 'followup' (absent, chase) | 'notsitting' (excused); either excludes all cells
        flag, flagNote,      // moderation flag (whole paper) and its comment
        satOn,               // '' or 'YYYY-MM-DD'; per-student sit override (default = dateSat)
        updatedAt,           // ISO; student-level fields merge by this
        parts: {             // keyed by part id; one cell per part
          [partId]: {
            done,            // bool
            tags: [tagId, ...],  // trip hazards observed on this part (ids into the assessment's hazard set)
            note,            // free text
            markedAt,        // ISO or null; drives "marked today"
            ms,              // active marking time for this cell, stamped by markDone (behaviour 17); 0 = never timed
            updatedAt        // ISO; drives sync merge (newer wins per cell)
          }
        }
      }
    }
  }],
  hazardSets: { [setId]: [{ id, name, part, deleted, updatedAt }] },  // per-assessment trip-hazard sets; assessments may share a set; deleted is a soft-delete tombstone; part scopes a hazard to one part, absent/'' = the whole paper (see behaviour 6)
  hazardBank: [{ id, name, year, hazards: [{ id, name }], updatedAt }],  // reusable hazard lists prepared ahead of marking; seeded into a job's set by copy, no student data
  daysOff: { ['YYYY-MM-DD']: { off, updatedAt } },  // rest days crossed out on the runway; synced (per-date newest wins); off 0|1 (behaviour 14)
  tombstones: { [id]: deletedAtISO },// deleted class/assessment/bank-entry ids, so a merge cannot resurrect them
  ui: { currentClass, currentAssessment, currentStudent, currentPart, view, theme, jobSort, archivedOpen, focusMin, focusEnd },  // device-local; NOT synced
  settings: {}   // currently unused
}
```

Every entity and cell carries `updatedAt`, stamped on each mutation, so sync
merges at the per-cell level (marking different parts on two devices both
survive). `ui` is deliberately device-local (theme, current selections and the
current part differ per device) and is excluded from the synced document.

**Dates vs timestamps**: `nowISO()` stores UTC ISO strings for sync ordering,
but any *calendar-day* comparison must use `localDay()`/`todayStr()` (LOCAL
date), never `iso.slice(0,10)` (UTC). `dateSat`/`dueDate`/`satOn` come from
`<input type="date">` and are local; comparing them to a UTC "today" reads a day
behind for much of the NZ day (UTC+12/13), which once mislabelled a job sat
today as "upcoming". Derive "marked today" the same way (`localDay(markedAt)`).
Day *counts* must be whole-day differences between local date strings on noon
anchors (see `dueDaysLeft`), never a millisecond division against `new Date()`:
measuring to `dueDate + 'T23:59:59'` and rounding up once made "1 day left" mean
due today and handed the deadline day itself out as marking time.

## Key behaviours (do not break these)

1. **Labels and minimal identity**: rosters store only first name + last
   **initial** (a pasted full surname is reduced to its initial in `addClass`);
   entry order is kept (no sorting). Label is first name + initial
   (`buildLabels`); students who would share a label are numbered. The **Set up
   class editor** (`classLabelEditor`, pencil on a class) is a full roster
   editor: rename any shown `s.label`; **remove** a student (`removeStudent`,
   tombstoned so a sync merge cannot resurrect them — the class merge filters
   students by `alive`); **add** students (`addStudentsToClass`, appends with
   fresh ids, keeps existing labels, numbers clashes via `uniqueLabel`); and
   **drag to reorder** (`wireDragHandle`, pointer events, mouse+touch; bumps
   `class.updatedAt` so the order syncs — it captures the pointer on the stable
   `rows` container, not the handle, because reordering reparents the dragged
   row and moving a captured element releases its capture and freezes the drag). Assessments read the roster live
   (`assessmentStats` iterates `classOf(a).students`, `markOf` returns a default
   for new ids), so a roster change flows to every job at once; a removed
   student's marks orphan in `a.marks` and are ignored. Existing pre-1.1.0
   classes keep whatever surname they had. Privacy by construction: only what
   renders is stored, and only in localStorage.
2. **Parts and part-by-part marking**: an assessment optionally splits into
   parts, edited from the job modal (one name per line; blank = single-part).
   In a multi-part job a **part bar** (`renderPartBar`) selects the current
   part; you mark that part across all students, then move on, keeping one
   meter stick per part. When a part is finished, `gotoNextUnmarked` auto-jumps
   to the next part with unmarked papers. Clicking a part tab (`selectPart`)
   **keeps the current student** (only picks one if none is set or they have
   left the class), so marking every part for one student needs no
   re-navigating; the across-the-class flow is driven by `gotoNextUnmarked`, not
   the tabs. Single-part jobs show no part bar and behave as before.
3. **Absence, two kinds** (`mark.absence`, student-level; both keep the
   student out of the marking denominator, via `absent(m)`): `followup` means
   absent on the day and needs chasing, surfaced in **amber** (roster "!"
   marker, a dashboard "Follow up" tile and count, listed in exports);
   `notsitting` means an accepted non-sit, kept **quiet** (muted, struck
   through in the roster, absent from the dashboard, still in the data export).
   Red is retired from absence. Set from the paper view; a followed-up student
   who later sits is marked present and given a `satOn` late date.
4. **Daily target** (per assessment): `ceil((remaining + markedToday) /
   workDaysLeft)`, computed in `assessmentStats` in **cells** (student × part).
   **A due date means due at the START of that day** (the class is often seen
   first period), so the last day you can mark is the day before: `dl` =
   `dueDaysLeft` = whole calendar days from today to `dueDate`, where **0 = due
   today (or overdue) and 1 = due tomorrow**, and `workDaysLeft` spans
   `[today, dueDate)`, which is exactly that window. Do not "fix" this back to
   counting the due date as markable; entering the following day is the escape
   hatch for a job that really does give you the whole day. `dl` drives
   chips/urgency; `wdl` = `dl` minus **rest days** (`isDayOff`) drives the paper
   target, so crossing out days concentrates the load onto the days you keep.
   With **no working days at all** (`wdl === 0`: due today, overdue, or every
   remaining day crossed out) the whole remainder falls due now rather than
   dropping out of the quota, which is why the fallback tests `dl !== null` and
   not `dl > 0`. `isOverdue(a)` (a plain `dueDate < todayStr()` comparison)
   separates "overdue" from "due today" in the chip, since `dl` clamps at 0.
   Cells marked today count toward today, so the target stays stable through the
   day. **Load points** = paper target ×
   the job's `weight` (1/2/3); `dailyQuota` sums load points across active dated
   jobs so a tricky job counts more, and the runway shades by effort the same
   way. Weight only scales pacing: **percentage and progress stay a plain paper
   count** (`marked/denom`). Each job has its own due date; the dashboard sorts
   jobs into three groups (markable now, then upcoming, then complete). A future
   `dateSat` marks a job "upcoming" (nothing to mark yet). A student can sit
   late: `satOn` on their mark overrides the job's `dateSat`, edited from the
   paper view.
5. **Mark done / un-tick**: `markDone` marks the current cell (tick animation,
   respects `prefers-reduced-motion`; green row flash; auto-advance after
   350 ms). It is reversible: a done cell shows an **Un-mark** button
   (`unmarkCell`, no advance), and the roster status box toggles done directly
   (`toggleDoneFor`) for quick corrections after moderation. **Mark all**
   (`markAllRemaining`, a button in the Papers column head shown only while
   papers remain) ticks every not-done, non-absent cell in the **current part**
   at once (for marking taken on paper and recorded on return), behind a confirm
   naming the count and part; it can trigger `maybePromptArchive` on completion.
   Un-ticking stays per student.
6. **Trip hazards**: the canonical term everywhere (input, hints, the
   **Copy hazard summary** button), coloured purple to tie to the Trip hazards
   column. **Per assessment**, not global: each assessment points at a hazard
   set in `S.hazardSets` (`hazardsOf`/`ensureHazardSet`), so the sidebar,
   counts and summary are that assessment's own. Hazards attach to the current
   **cell** (student-part); clicking a sidebar hazard toggles it; adding a new
   one auto-attaches it. Sidebar sorts by frequency within the current part.
   `tagName(a, tid)` resolves within the assessment's set. Migration: the old
   global `tags` becomes one shared `legacy` set kept by existing assessments.
   **Scoped to a part** (`t.part`): on a multi-part job a hazard can belong to
   one part, so each question keeps a short list of its own; a list long enough
   to cover a whole paper is one the owner stops reading, leaving its rarer
   entries unused. `part` absent or `''` means **the whole paper** (every part),
   which is what every pre-1.17 hazard is, and what single-part jobs and
   new-job-draft imports always produce, so nothing already set up changes.
   **Two accessors, do not conflate them**: `allHazards(a)` is the whole live
   set and must be used to resolve a name or to export, since a hazard logged on
   Section A still has to resolve while Section B is on screen (this is why
   `tagName` uses it); `hazardsOf(a, pid)` is what is in play on one part
   (scoped to it, plus the whole-paper ones) and drives the sidebar and every
   dedupe. Called with no `pid` it means "all". The same mistake may exist as
   two entries under two parts, deliberately, so **deduping is always against
   what is in play at the destination, never the whole set** (`addTag` and
   `doImport` both do this): a list imported into Q1 can then be imported into
   Q2, while a whole-paper hazard is never duplicated into a part. `renderTags`
   names the part in the column head (`#tagScope`) so a short list reads as
   scoped rather than as hazards gone missing. `reconcileParts` calls
   `promoteOrphanHazards`, so removing a part returns its hazards to the whole
   paper instead of stranding them against a dead part id, where `hazardsOf`
   would filter them out of every list and they would look deleted. The `part`
   field rides `mergeSet`'s whole-entry newer-wins and `normalize` mutates
   hazard entries in place rather than rebuilding them, so it syncs and persists
   with no extra plumbing.
   A hazard can be **deleted while marking** (`deleteHazard`, a delete control
   on each sidebar row): this is a **soft delete** (`t.deleted = true` with a
   fresh `updatedAt`), not a splice, so a sync merge (which unions tags by id)
   cannot resurrect it; `hazardsOf` filters `deleted` out and the tag is
   stripped from every cell that logged it.
   **Hazard bank** (`S.hazardBank`, edited in Set up: `renderHazardBank`,
   `saveBankEntry`, `editBankEntry`, `deleteBankEntry`): reusable named lists
   labelled by assessment name and optional year level, prepared ahead of
   marking from the schedule. The bank holds no student data, so it syncs; bank
   entries carry `updatedAt` and delete via `tombstones` like
   classes/assessments.
   **Import** (`openImport`/`renderImport`/`doImport`, the `#importOverlay`
   modal): brings hazards into a job by **copy**, from two clearly separated
   sources: **the bank** and a **live set** (another job's current non-deleted
   hazards). Importing is additive: it adds only names not already in play at
   the destination (case-insensitive, ignoring deleted) and never repoints or
   removes. It is
   reachable from three places, all sharing the modal: creating a job (staged
   into `jobHazardDraft`, an array of names built into the new set in `saveJob`),
   editing a job (straight into the job's real set), and while marking (the
   `#tagImportBtn` control in the Trip hazards column). `importTarget` routes to
   the draft or an assessment. On a multi-part target the modal shows an
   **Add to** selector (`#importScopeRow`, `renderImportScope`,
   `importScopeTarget`): "All parts" or a named part, defaulting to `curPart()`
   when the target is the job on screen, so importing mid-marking needs no
   choice. A **draft** (new job) always imports whole-paper, because its parts
   have no ids until `saveJob` runs; per-question prep happens from the pencil or
   on reaching each part. A multi-part **live set** is listed **one entry per
   part** plus its whole-paper hazards ("Mechanics test · Q1"): flattening every
   question into one pile would defeat the workflow the live set exists for,
   which is refining class 1's Q1 list and carrying it into class 2's Q1.
   There is deliberately no UI for re-scoping an existing hazard; delete and
   re-import covers it. This replaced the old linked "share a set"
   option; two jobs pointing at one `hazardSet` id still work, but new setups
   are copies, so the workflow is: mark class 1, refine, then import class 1's
   live set into class 2 to carry the refinements across (each class keeps its
   own frequencies).
7. **Feedback prompt export** (`copyPrompt`, the **Copy feedback** button under
   the note in the paper view): clipboard text with first name,
   class, assessment name, and the student's tagged issues and notes gathered
   across every part (grouped by part name when multi-part), plus fixed
   constraints (address student directly, encouraging but honest, ~80 words, do
   not invent issues beyond those listed). It is whole-assessment for that one
   student, not per-part. The owner pastes this into Claude.
8. **Two assessment exports, both in the workspace** (`class-summary-row`):
   **hazard summary** (`copyAssessmentSummary`) is the trip-hazard frequency
   table for reteaching, each row naming its part on a multi-part job (a scoped
   hazard can only ever be logged while its own part is on screen, so the counts
   are already question-partitioned; the label just makes that visible);
   **assessment data** (`copyAssessmentData`) is a full
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
    sides by per-entity/per-cell `updatedAt` (hazard sets merge by set id, tags
    within a set by `updatedAt`; hazard-bank entries merge by id, newer entry
    wins wholesale, deleted via tombstones; `daysOff` merges per date, newest
    `updatedAt` wins; a class's students merge by id, union preserving order,
    with removed students dropped by the shared tombstone `alive` filter) with
    tombstones for deletions, so two devices converge with no data loss and no
    forced conflict choice (mark Section A on one device and Section B on another
    and both survive).
    Auto-sync runs on open and debounced after edits; a header button and a Set
    up section drive it manually. Comparison and the gist body use a **canonical
    serialisation** (`docString`: sorted keys, entity arrays sorted by id,
    deviceId excluded) so identical data never ping-pongs between devices. The
    gist holds student names, so it is secret by construction (`public:false`).
12. **Moderation**: a whole-paper flag (`mark.flag`) plus a comment
    (`mark.flagNote`), toggled from the paper view (yellow `--flag`). Flagged
    students show a ⚑ in the roster; the flag and comment appear in the
    assessment data export as a moderation record.
13. **Daily quota and motivation**: the dashboard's **Overall progress** is a
    ring (`progressRing(pct, size)`, percentage centred) over **sat**,
    non-archived cells: `renderDashboard` splits `list` into `started`
    (`!upcoming`) and `upcoming`, and the ring sums marked/denom over `started`
    only, so a future batch of exams (a job whose `dateSat` has not arrived) does
    not dump its full paper count into the denominator and read low against work
    that cannot begin. The excluded upcoming papers stay visible as a subline
    note (`upCells` = their remaining, with the job count); no started work reads
    100% ("nothing to mark yet"). Completed sat jobs stay in the ring (genuine
    progress). The per-job quota below it stays as sorted bars (rings lose the
    at-a-glance comparison there), and the runway stays linear (a countdown reads
    best as a line). `dailyQuota()` sums cells marked today and
    today's targets across all active jobs; a second header bar shows progress
    toward it, green when met (replaces the old "missing" header stat). On the
    dashboard the quota section breaks the day down **per job** (`quotaRowsHtml`):
    one mini-bar per dated active job of today-done vs its own daily target,
    sorted most-urgent-first, so a Sunday deadline visibly needs more today
    than a Tuesday one. The Marking Jobs cards are compact (no progress bar;
    overall % as text) with a Due date / Class sort toggle (`ui.jobSort`,
    `setJobSort`). A curated, attributed literary quote (`QUOTES`, `quoteLine`)
    sits high on the dashboard and shows once when the quota is met. Keep the
    quotes literary and unfussy: no self-help, no exclamation marks. Completed
    but non-archived jobs sink to the bottom of the active list; archived jobs
    leave it entirely (behaviour 16).
    **Dashboard colour**: colour is applied by meaning, never for decoration, so
    the colourblind-safe reads hold. The three tiles carry a hue each
    (`t-info`/`t-accent`/`t-success`/`t-flag`: Active jobs blue, Remaining orange
    then green at zero, Follow up yellow when any); quota bars are coloured by
    urgency (`u-danger` due today, default `--accent` within two days, `u-later`
    blue further out, `.met` green); job cards get a left stripe by state
    (`s-danger`/`s-soon`/`s-info`/`s-success`/`s-idle`) in the same language.
    `dailyQuota()` excludes `upcoming` jobs exactly as the per-job bars do, so
    the headline always equals the sum of its own breakdown.
    Hues are soft `color-mix` washes over `--panel`; if you add a dashboard
    element, colour it by state from the semantic palette, do not invent a hue.
14. **Marking runway** (`runwayHtml`/`runwayPick`, dashboard, below the quota):
    a **forward** heat strip. For each active dated job it spreads
    `remaining + markedToday` evenly across its **working days** (indices
    `[0, dl)`, i.e. today through the day *before* the due date, skipping
    `daysOff`) and sums the **effort** (papers × `weight`) per day, so a tricky
    job burns hotter. A job with **no working days left** (due today, overdue, or
    all its days crossed out) puts its whole remainder on index 0, so **the first
    square keeps agreeing with Today's quota** — the invariant to preserve if you
    touch this. Horizon runs today to the last due date, clamped to [14, 35]
    days. Shade is
    `ceil(load/max * 4)` (relative to the busiest day) over a `color-mix` ramp of
    `--accent` (a single-hue sequential scale, kept clear of `--danger`); today is
    ringed, the `--info` dot sits on `days[dl]`, the due date **itself** (which
    carries no load, since marking ends the day before), weekends render
    as smaller centred squares (an inner `.rw-sq` holds the shade), and **rest
    days** show a crossed-out `.rw-off` square with no load. Hover shows a `title`;
    click/tap toggles a caption open and shut (`runwayPick`/`runwayPicked`,
    mobile-friendly) that carries a **Rest day / Working day** toggle
    (`toggleDayOff`, writes `S.daysOff`, synced). Per-day papers are split across
    jobs by largest-remainder (`allocateCells`) so the parts sum to the day total.
    It is a standing suggestion, not a target: it recomputes from live stats, so
    resting a day (crossing it out, or letting `dl` fall) raises the other
    squares. Hidden when no dated active job exists.
15. **Keyboard shortcuts** (`onHotkey`, one global `keydown` listener; desktop):
    bare single keys, in the spirit of the lesson planner. Ignored while typing
    (input/textarea/select/contentEditable) and when any Ctrl/Cmd/Alt is held, so
    text entry is untouched. `Esc` steps out of a focused field (`blur`), else
    closes the topmost open overlay (`closeTopOverlay`). Action keys fire only in
    the workspace and never while a dialog is open: `Enter` = `markDone`, `i` =
    `focusNote` (caret at end; the inverse of `Esc`), `]`/`[` = `stepStudent`,
    `n` = `gotoNextUnmarked`, `u` = `unmarkCell`, `f` = `toggleFlag`; anywhere:
    `?` opens the cheat-sheet (`#hotkeyOverlay`, also the
    header **Keys** link), `d` = `goDashboard`. Trip hazards and parts are
    deliberately not hotkeyed (a long/short click list; keeps the set small). The
    notes box does not autofocus, so keys are live on arrival at a paper. Each
    action button shows its key in a tiny corner (`.kbd-hint`, hidden on touch
    via `@media (hover: none)`) to help learn them. The paper view groups its
    controls: header + Sat-on, then a **Feedback** section (logged hazards, note,
    Copy feedback), then the paired everyday actions (Mark done, Next unmarked),
    then a separated **Less often** group (Flag, Absent follow-up / not-sitting).
16. **Archiving** (`a.archived`, a synced bool on the assessment;
    `archiveAssessment`/`unarchiveAssessment`): finished jobs are filed into a
    collapsible **Archived** section at the foot of the dashboard (`toggleArchived`,
    `ui.archivedOpen`, collapsed by default, muted cards with Unarchive). It is
    always the teacher's choice, never automatic: when marking crosses a job into
    complete (`markDone`/`toggleDoneFor` compare `wasComplete`), `maybePromptArchive`
    offers to archive it and names anything still outstanding (follow-ups, a
    moderation flag) so **Not yet** is easy; a complete non-archived job also
    carries an Archive button. Archived jobs leave the active list and **every
    headline figure** — Overall progress, the Active-jobs/Remaining/Follow-up
    tiles, Today's quota (`dailyQuota` filters them), and the runway (already
    excluded as complete). `renderDashboard` splits `all` into non-archived
    `list` (drives the metrics) and `archived`. The flag rides the assessment
    base in `mergeDocs` (newer `updatedAt` wins), so archiving syncs with no
    extra merge code; `normalize` backfills `archived: false`.
17. **Marking timer and pace** (the `T` object plus `tick`, `paintTimer`,
    `timerBarHtml`; a strip at the top of the paper view). Two clocks: a **cell
    clock** timing the student-part on screen, and a **session clock** driving
    the focus block. Both bank **active intervals only**: `tStop(endAt)` folds
    the running interval into `T.cellMs`/`T.sessionMs`, `tRun()` resumes.
    - **Idle rollback is the thing that keeps the data honest**: when the tick
      sees no activity for `IDLE_MS` (10 min) it calls `tStop(T.lastAct)`, so a
      walk-away is ended at the last real input and the gap costs nothing rather
      than being billed to whichever paper was open. Activity = keydown,
      pointerdown, wheel and throttled pointermove (movement counts, so silence
      really means away and not reading a long piece of working).
    - **One navigation hook**: `tSyncCell()` runs at the top of `renderPaper`
      and compares an `assessment|student|part` key; any change (roster click,
      Next unmarked, part tab, step) banks the old lap and starts a fresh clock.
      Do not hook the individual navigation functions. An absent student or the
      dashboard clears the key, so nothing is timed there.
    - **`markDone` writes the lap** onto `cell.ms`; `unmarkCell` and a roster
      un-tick clear it and restart the clock. `markAllRemaining` and
      `toggleDoneFor` deliberately record **no** time (never timed), so they
      cannot distort the typical.
    - **The tick must never call `render()`**: `renderPaper` rebuilds the notes
      textarea through `innerHTML`, so a per-second re-render would destroy it
      and steal the caret mid-sentence. `paintTimer()` only writes `textContent`
      and bar widths onto existing nodes. Elapsed time is derived from
      `Date.now() - since`, never accumulated by the tick, so a throttled hidden
      tab keeps exact data and only the display lags; `visibilitychange`
      repaints on return.
    - **Pace is a median, not a mean** (`median`, `typicalMs`): one paper that
      ran long would otherwise skew every estimate after it. Do not add outlier
      clipping on top; the median already resists them, and discarding slow
      papers would bias every estimate optimistic.
    - **Every job is priced, including untimed and not-yet-sat ones**
      (`cellPriceMs`). An untimed job falls back to `defaultCellMs(a)` =
      `DEFAULT_PART_MS` (1.5 min) × weight per part, or `DEFAULT_WHOLE_MS`
      (3 min) × weight for a single-part paper, the owner's own figures. A job
      marked in parts is therefore priced higher in total than the same job
      marked whole, deliberately: each paper is handled once per part.
      Previously an untimed job contributed nothing, so the headline read as a
      total while describing a fraction of the work (26 papers waiting once read
      "about 10m left").
    - **The default gives way to measurement by a glide, never a threshold**:
      `price = w_part·median(part) + (1 − w_part)·[w_job·median(job) +
      (1 − w_job)·default]`, with `w = min(1, samples / TRUST_N)` and
      `TRUST_N` = 10. Switching outright at `MIN_SAMPLE` would lurch the
      headline by hours the moment a third paper was timed, and again on every
      part; the glide makes it creep like a journey time. A second part inherits
      the job median (`w_job` = 1) rather than reverting to the default.
      `cellTrust` returns the weight resting on measurement, which
      `paceSourceTitle` turns into the "X% from your own times" tooltip.
    - `pacePartMs`/`paceJobMs` multiply the price by remaining cells;
      `paceTotals()` returns `left` (everything unmarked, **including
      `upcoming`** jobs, since a batch sat next week is real work coming),
      `upcoming` (that share, named separately in the dashboard line), `today`
      (the cost of today's targets, excluding upcoming since none of it is
      markable today) and `measuredPct`. Estimates render in `--info`
      (`.dash-pace`, `.cc-time`, `.mt-est`) as derived information.
    - **`MIN_SAMPLE` = 3 still governs the strip's gauge and its "typical"
      readout only** ("learning your pace" until then). Those two claim to know
      *your* pace, so they stay measured-only; a gauge filling against an assumed
      figure is the one place a guess could push the owner's marking around.
      Keep defaults out of them.
    - **Colour is deliberately not alarming**: the gauge fills `--accent` to the
      typical tick (at 62.5%, the bar running to 1.6× typical so an overrun
      stays visible) and continues in `--faint` beyond it. `--danger` is
      **never** used here: the papers that run long are usually the ones that
      deserve the thought, and this tool's premise is preventing fatigue-induced
      degradation, so the tick is a reference and not a threshold. The focus
      block ends in `--success` with a break, since stopping on time is its
      point.
    - **Focus block**: `ui.focusMin`/`ui.focusEnd` (device-local, an absolute
      end time so a reload resumes the block), `startFocus`/`stopFocus`/
      `checkFocus`. On expiry it toasts and sets the tab title to "Break time",
      visible from another tab. No audio.
    - Times stay out of both clipboard exports on purpose: time per paper reads
      too easily as a judgement on the student rather than on the marking.

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

## Versioning and the guide

The app is versioned with semver, held in one place: the `VERSION` constant at
the top of the script in `marking-console.html`. It renders in the header
(`#verLink`, links to the guide's changelog). **On every change, bump `VERSION`
and add an entry in three places kept in step**: `CHANGELOG.md` (canonical),
the header `Guide · vX.Y.Z` line and the `#changelog` section in `guide.html`.
Patch for fixes, minor for features, major for a big shift. The guide is
example-led and succinct by design (scenario, then the exact taps); keep it
that way, and add a recipe when a feature genuinely needs one.

## Testing

No test framework. Sanity check after changes:
`node --check` on the extracted script block, then manual test of: first-run
setup (add a class, then an assessment), roster paste with duplicate first
names, editing a roster (pencil in Set up: add students appear unmarked in the
class's jobs, remove drops a student from the jobs and is tombstoned so sync
does not resurrect, drag reorders and the order syncs, custom labels survive an
add), a single-part job (mark/un-tick, absence follow-up vs not-sitting,
hazard toggle), a
multi-part job (part bar, mark a part across students, auto-jump to the next
part, per-part hazard counts), un-ticking from the roster box, Mark all
remaining (ticks the current part's unmarked non-absent cells, skips absences,
confirms with the count/part, hides at full), the hazard bank
(add a bank list in Set up), importing hazards (from the bank and from a live
set, at job creation, on edit, and while marking; confirm it is an additive
copy that stays independent of the source and dedupes on re-import),
part-scoped hazards (on a multi-part job the sidebar shows only that part's list
plus the whole-paper ones and names the part in the column head; the import
"Add to" selector defaults to the part being marked; a whole-paper name is not
duplicated into a part but a part-scoped one does copy into another part on
re-import; a hazard typed while marking joins that part only; a multi-part live
set is offered per part; removing a part promotes its hazards to the whole paper
instead of hiding them; the summary export names the part per row; a single-part
job shows no scope label and no selector; pre-existing hazards with no part show
everywhere), deleting a
hazard while marking (it clears from every paper and does not resurrect on
sync), flagging a
student for moderation with a comment, the due-date arithmetic (a job due
tomorrow reads "1 day left" and puts its whole remainder on today; one due today
reads "due today" with everything due now and still appears in the quota; one
already past reads "overdue", not "due today"; a job due in 3 days paces over
today plus the next two, not three; the runway's deadline dot sits on the due
date and the first square equals Today's quota), job difficulty weight (1/2/3 in the job
modal; the quota reads in load points = papers × weight, badges show it,
completion % stays a paper count), the marking runway (dated jobs shade a
forward strip by effort, per-day parts sum to the day total, hover/tap caption,
crossing out a rest day greys it with an X and lifts the other days, rest days
sync, hides with no dated jobs), the marking timer (the clock runs on arrival at a
paper and resets on every navigation path; Mark done banks the lap onto the cell and
it survives a reload and a sync merge; the gauge tick sits exactly at the typical and
overruns fill the muted segment without ever going red; estimates stay hidden below
three timed papers and use the median so an outlier does not skew them; an untimed
job is still priced from its difficulty and a not-yet-sat job still counts in the
"marking ahead" total; the estimate glides toward your own times as papers are
timed instead of jumping at the third; the quota headline equals the sum of its
bars; idle rollback
banks only the active time and charges nothing for a walk-away; Mark all and roster
un-ticks record no time; a focus block counts down, ends green and changes the tab
title, and survives a reload; **typing in the notes box keeps focus and caret while
the clock ticks**), the keyboard shortcuts (Enter marks and advances, [ ] step
students, ? opens the sheet, Esc blurs the notes box then keys work again, keys
paused while typing), archiving (marking the last paper prompts to archive and
names outstanding follow-ups/flags; Not yet keeps it active with an Archive
button; archived jobs drop out of Overall progress/quota/runway/tiles and sit in
the collapsible Archived section; un-archive restores), the daily quota bar
filling and turning
green (and the quote on meeting it), both clipboard exports, JSON
export/import round-trip, theme toggle, v2→v3 and v1→v3 migration (load with
only the older key present),
and a reload to confirm persistence. For sync, the merge (`mergeDocs`) can be tested
without a token by mocking `gistGet`/`gistPatch`/`gistCreate` in the console:
check convergence (`docString(mergeDocs(A,B)) === docString(mergeDocs(B,A))`),
idempotence, that two devices' marks on different papers both survive, and that
deletions do not resurrect. The live gist round-trip needs a real token.
