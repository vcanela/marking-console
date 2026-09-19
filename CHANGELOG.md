# Changelog

Semantic versioning (major.minor.patch). The version also shows in the app
header and in `guide.html`; keep all three in step on every change.

## 1.17.1 — 2026-09-19

- Fixed today's quota tally collapsing when you archive a job you finished the
  same day. Archiving dropped the job from the count entirely, so a morning
  spent on it stopped counting toward today: finishing and filing a job could
  take the tally from 13 papers back to 3. Work done today now counts wherever
  it was done, including a job archived since. Only the target drops when a job
  leaves the active list, because a finished job owes nothing more today.

## 1.17.0 — 2026-09-16

- Trip hazards can now belong to a single part, so each question keeps its own
  short list instead of every question sharing one long one. A list long enough
  to cover a whole paper is one you stop reading, and its rarer entries go
  unused.
- Importing into a multi-part job asks which part to add to, defaulting to the
  part you are marking, with "All parts" for mistakes that can appear anywhere
  (units missing, no working shown). Hazards you add by hand while marking
  belong to that part too.
- The Trip hazards column names the part it is showing, so a short list reads as
  scoped rather than as hazards having gone missing.
- The same mistake can be listed under two questions. Importing skips only what
  is already in play where it lands, so a list imported into Q1 can be imported
  into Q2 as well, while a hazard that already applies to the whole paper is
  never duplicated into a part.
- A live set from a multi-part job is offered per part ("Mechanics test · Q1"),
  so marking one class, refining its Q1 list and carrying that into the second
  class still works question by question instead of arriving as one flat pile.
- The hazard summary export names the question on each row.
- Existing hazards, and every hazard in a single-part job, apply to the whole
  paper exactly as before; nothing you have set up changes, and single-part jobs
  show none of this. Removing a part returns its hazards to the whole paper
  rather than leaving them stranded.

## 1.16.0 — 2026-09-09

- Time estimates now cover every job, not only the ones you have already timed.
  A job you have not started is priced from its difficulty: 1.5 minutes per part
  per student at difficulty 1, 3 at difficulty 2, 4.5 at difficulty 3, and a
  paper marked as a whole counts as one bigger cell at 3, 6 or 9 minutes. Before
  this, an untimed job contributed nothing, so the headline looked like a total
  while describing a fraction of the work: 26 papers waiting could read "about
  10m of marking left".
- "About X of marking ahead" now includes work that has not been sat yet, with
  the not-yet-sat share named separately, since a batch of exams next week is
  real marking coming at you even though you cannot start it.
- Estimates move to your own times gradually rather than switching over at a
  threshold. Each level is trusted in proportion to how many papers back it: a
  new job starts at the difficulty estimate, is about half yours after three
  papers, and is entirely yours after ten. A second part inherits the job's
  measured pace instead of falling back to the default. The figures creep like a
  journey time rather than lurching by hours when a threshold is crossed.
- One unusually long paper still barely moves anything: typical times are
  medians, so a 20 minute paper among 2 minute ones shifts the estimate by
  seconds, where an average would more than double it.
- Hover any time estimate to see how much of it comes from your own timings and
  how much is still assumed.
- The timer strip keeps its gauge and its "typical" reading measured-only; they
  still say "learning your pace" until three papers are timed. Assumed numbers
  are for planning totals, not for pacing you against a figure you never set.
- Fixed Today's quota headline counting jobs that have not been sat yet, so it
  disagreed with the sum of the per-job bars beneath it.

## 1.15.0 — 2026-09-09

- Days left now counts whole days to the deadline, so "1 day left" means due
  tomorrow. It used to mean due today, because the count ran to the end of the
  due date and rounded up.
- A due date now means due at the **start** of that day, so the last day you can
  mark is the day before. You often see the class first period, and the old
  maths handed you the due date itself as marking time, which paced everything a
  day too slowly. Targets are correspondingly tighter: a job due Friday now
  spreads over Wednesday and Thursday, not Wednesday to Friday. If a job really
  does give you the whole due date, enter the following day as the due date.
- A job due today, or one whose remaining days are all crossed out as rest days,
  puts its whole remainder on today rather than dropping out of Today's quota.
- Overdue jobs say "overdue" instead of "due today", which is what they used to
  show once the date had passed.
- On the runway, the deadline dot now sits on the due date itself rather than a
  day early, and an overdue or due-today job loads onto today, so the first
  square still agrees with Today's quota.

## 1.14.0 — 2026-09-09

- Marking timer above the paper you are on. It times the paper (or the part),
  and when you mark it done that becomes a lap, recorded on the paper itself so
  your typical times build up across sessions and devices.
- A gauge fills toward your typical time for that part, with a tick where the
  typical sits. Past the tick it keeps filling in a muted tone rather than
  turning red: the papers that run long are usually the ones that deserve the
  thought, so the mark is a reference, not a verdict.
- Pace estimates, once three papers in a part have been timed: time left on the
  current part and on the whole job in the strip; "about 3h 20m of marking left
  at your pace" under Overall progress; the cost of today's target beside
  Today's quota; and a "~1h 10m" chip on each job card. Below three timed
  papers it says "learning your pace" and estimates nothing.
- Typical times use the median, so one paper interrupted by a phone call does
  not skew every estimate that follows.
- Focus block (15, 25, 45 or 60 minutes) with a countdown. It ends in green
  with "take a break", and the browser tab title changes so you see it from
  another tab.
- Walking away costs nothing: after ten minutes with no keyboard or pointer
  activity the timer ends the interval at your last activity rather than
  counting the gap, so an interrupted paper is not billed for the interruption.
  There is a pause button as well.
- Papers ticked with "Mark all remaining", or from the roster box, record no
  time, since they were never timed; they are left out of the typical.

## 1.13.0 — 2026-09-07

- Overall progress now counts only jobs that have been sat. A batch of exams
  entered ahead of time (a future sit date) no longer dumps its full paper
  count into the ring and drags the percentage down against work you cannot
  start yet. The upcoming papers stay visible as a note beside the ring, for
  example "20 upcoming in 1 job", so you keep sight of what is coming. When
  every sat job is finished the ring reads 100% with that note; before anything
  is sat it reads "nothing to mark yet". The tiles, quota and runway are
  unchanged (the quota and runway already left upcoming jobs out).

## 1.12.2 — 2026-08-31

- Switching parts now keeps the student you are on, instead of jumping to the
  first one. So you can mark every part for a single student without scrolling
  back each time. Marking a part across the whole class is unchanged (Enter and
  Next unmarked still carry you through, and auto-jump to the next part).

## 1.12.1 — 2026-08-31

- Fixed roster drag-to-reorder: the handle selected but the row would not move.
  The drag captured the pointer on the handle, and reordering reparents the row
  (which holds the handle), releasing the capture and freezing the drag. It now
  captures on the stable list container, so dragging works on mouse and touch.

## 1.12.0 — 2026-08-31

- Edit a class roster after creation, from the pencil in Set up: add students
  (paste box), remove a student who has left (× on their row), and drag ⠿ to
  reorder. Every marking job for that class updates from the list, so a new
  student appears unmarked in each job and a removed one drops out. Removals are
  tombstoned so they do not reappear on sync; reorders sync too. Renaming shown
  labels still works, and existing labels are kept when you add students.

## 1.11.0 — 2026-08-31

- Job difficulty weight (1 Mechanical / 2 Average / 3 Tricky, default 2), set in
  the job modal. The daily quota and the runway now count load points (papers ×
  weight), so a tricky investigation pulls harder than a quick chunk; each job
  shows its weight as a badge. Completion percent stays a plain paper count.
- Cross out rest days on the runway: tap a day and choose "Rest day"; it greys
  out with an X, carries no load, and its work shifts onto the days you keep.
  The daily target then paces over working days, not calendar days. Rest days
  sync across your devices.

## 1.10.2 — 2026-08-28

- Fixed a timezone bug where a job whose sit date had arrived still read
  "Upcoming" (and stayed out of the count, quota and runway) until about midday.
  "Today" was computed in UTC, which is a day behind local time for much of the
  New Zealand day; it now uses the local calendar date, so a job flips to current
  the moment its sit date arrives. "Marked today" is derived the same way.

## 1.10.1 — 2026-08-25

- Fixed the keyboard-shortcut hint overlapping the button label (most visible on
  "Mark part done" / Enter). Buttons now reserve symmetric space sized to the
  hint, so the label stays centred and clear.

## 1.10.0 — 2026-08-25

- Reworked the Current Paper view for clearer grouping: the Sat-on date moves up
  under the name; the logged trip hazards and the optional note sit together
  under a Feedback heading; the two everyday actions (Mark done, Next unmarked)
  are paired; and the rarely-used controls (Flag for moderation, and Absent
  follow-up / not-sitting) are separated under a "Less often" heading.
- Copy feedback prompt is renamed "Copy feedback" and tucked below the note. It
  is unchanged: still the whole-assessment prompt for one student.
- Each action button shows its keyboard shortcut in a tiny corner (Enter, n, u,
  f) to help you learn them; hidden on touch devices.

## 1.9.0 — 2026-08-25

- More colour on the dashboard, all from the Okabe-Ito palette and each keeping
  its meaning. The three tiles carry a hue each (Active jobs blue, Remaining
  orange then green at zero, Follow up yellow when any are outstanding). Today's
  quota bars are coloured by urgency (due today vermillion, within two days
  orange, comfortable blue, met green). Job cards gain a left status stripe in
  the same language, so the list reads as a colour-coded status column. The
  colourblind-safe reads hold and both themes are covered.

## 1.8.1 — 2026-07-30

- Overall progress now shows as a ring with the percentage in the centre,
  instead of a wide bar; it reads at a glance and frees a little space. The
  runway and the per-job quota bars are unchanged.

## 1.8.0 — 2026-07-30

- "Mark all" button in the Papers header: ticks every remaining paper in the
  current part in one go, for when you marked a batch on paper and are recording
  it on return. It skips absent students and already-done papers, asks to
  confirm with the count (and part name in a multi-part job), and offers to
  archive if it completes the job. Un-ticking stays per student.

## 1.7.0 — 2026-07-30

- Archive finished jobs into their own collapsible "Archived" section, so the
  active list stays only what you are still marking. When marking completes a
  job you are asked whether to archive it (with a note if a follow-up or a
  moderation flag is still outstanding), so nothing is filed away before you are
  ready; "Not yet" keeps it active with an Archive button for later.
- Archived jobs leave every headline figure: Overall progress, Today's quota,
  the runway and the tiles all count active work only. Un-archive any time.

## 1.6.1 — 2026-07-30

- Added `i` to jump the cursor into the notes box (caret at the end), the
  inverse of Esc which steps out. Marking a paper and writing a note now stay
  fully on the keyboard.

## 1.6.0 — 2026-07-30

- Keyboard shortcuts for faster marking (desktop). While marking a paper:
  Enter marks done and jumps to the next unmarked, `]` / `[` step between
  students, `n` skips to the next unmarked, `u` un-marks, `f` flags for
  moderation. Anywhere: `?` shows the list, `d` goes to the dashboard, Esc
  closes a dialog. A "Keys" link in the header opens the cheat-sheet too.
- Shortcuts pause while you type, so writing notes is untouched; press Esc to
  step out of the notes box and use the keys again.

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
