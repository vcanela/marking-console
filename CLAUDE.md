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
  `markingConsole.v1` on every mutation via `save()`.
- **Rendering**: full re-render per region (`renderHeader`, `renderChips`,
  `renderRoster`, `renderPaper`, `renderTags`), all called by `render()`.
  No virtual DOM, no diffing; the data is small enough that this is fine.
- **Tag counts are derived**, never stored: `tagCount()` scans all students'
  `tags` arrays. Single source of truth; do not introduce a stored counter.

### Data model

```js
S = {
  classes: [{
    id, name,
    students: [{
      id, first, last,
      label,                 // display label, e.g. "Sophia K"
      status,                // 'unmarked' | 'marked' | 'missing'
      tags: [tagId, ...],    // mistakes observed on this paper
      note,                  // free text, included in feedback prompt
      markedAt               // ISO string or null; drives "marked today"
    }]
  }],
  tags: [{ id, name }],      // global tag library, shared across classes
  ui: { currentClass, currentStudent },
  settings: { endDate, calibrateEvery }   // calibrateEvery: 0 = off
}
```

## Key behaviours (do not break these)

1. **Labels**: students sorted by surname (matches physical marking order).
   Label is first name + shortest unique surname prefix within the class
   (`buildLabels`). Collisions extend the prefix; identical full names get a
   numeric suffix. Privacy by construction: full names exist only in
   localStorage, labels are what render.
2. **Missing ≠ unmarked**: three paper states. Missing papers are excluded
   from the progress denominator so a class can reach 100% with
   non-submissions; missing count shows separately.
3. **Daily target**: `ceil((remaining + markedToday) / daysLeft)`. Papers
   marked today count toward today, so the target stays stable through the
   day instead of shrinking as you work.
4. **Mark done flow**: tick animation (respects `prefers-reduced-motion`),
   green row flash, auto-advance to next unmarked paper after 350 ms. The
   satisfying tick is a feature requirement, not decoration.
5. **Tag interaction**: clicking a sidebar tag toggles it on the current
   paper. Adding a new tag while a paper is selected auto-attaches it (you
   just saw the mistake). Sidebar sorts by global frequency, most common
   first, with proportional bars.
6. **Feedback prompt export** (`copyPrompt`): clipboard text containing first
   name, class, tagged issues as a list, optional marker's note, and fixed
   constraints (address student directly, encouraging but honest, ~80 words,
   do not invent issues beyond those listed). The owner pastes this into
   Claude to generate the comment.
7. **Class tag summary** (`copyClassSummary`): frequency table per class,
   used for planning reteaching in the first lessons back.
8. **Calibration check**: optional, off by default. Every N marked papers in
   a class, a banner suggests re-reading the first marked paper for drift.
9. **Backup**: JSON export/import of the whole state. localStorage is
   fragile (Safari eviction); this is the safety net. Preserve import
   validation (`classes` and `tags` keys must exist).
10. **All user text is escaped** through `esc()` before hitting innerHTML.
    Keep it that way for any new rendering code.

## Design language

Dark instrument-panel aesthetic, consistent with the owner's other tools
(Jaynes–Cummings simulation, lesson planner). IBM Plex Sans for UI, IBM Plex
Mono for data readouts and labels. Palette in CSS custom properties at the
top of the stylesheet: amber (`--amber`) is the primary accent and active
state, green strictly for completion, red for missing, cyan for copy/export
actions. Reuse the variables; do not introduce new hex values inline.

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
setup, roster paste with duplicate first names, mark/unmark/missing cycle,
tag toggle, both clipboard exports, JSON export/import round-trip, and a
reload to confirm persistence.
