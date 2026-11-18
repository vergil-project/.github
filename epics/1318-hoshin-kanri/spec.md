# Hoshin kanri: a review cadence and logbook for the fleet — design

- **Epic:** vergil-project/.github#1318
- **Planning issue:** vergil-project/.github#471 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-claude-plugin (the `hoshin` skill), docs (the logbook
  section and the hoshin background page)

## Intent

Vergil now does a large share of its work unattended. Lifecycle and
upstream-refresh PRs open and merge themselves, the develop CD builds publish
container images, and scheduled and event-driven jobs (`ops.yml` in every repo,
the epic rollup and sweep, package cleanup, docs refresh) run whether or not
anyone is looking. More of that is coming.

The human's working day has no point at which any of that gets looked at.
Each morning goes straight to the most interesting open development effort.
Nothing asks what ran overnight, what failed and why, how stable the past week
has been, or whether the work is still pointed the right way. Mid-term and
strategic concerns (priorities, strategy, capacity) have no regular moment at
all. The faster development goes, the more those gaps show: at this speed the
small instabilities that were invisible when work was slow surface many times a
day.

This epic adds a **management cadence** to complement the fast, ad hoc
engineering work: a fixed set of reviews at the start and end of each day, and
at week, month, quarter and year boundaries, each leaving a written record. The
reviews are meant to become the primary driver of the human's working day, run
from one dedicated, reused session.

### The model: 方針管理 hōshin kanri

The human's strategic decision: **until a better framework is found, Vergil
scales hōshin kanri down to one person working with agents, and adopts its
practices in spirit where not to the letter.** The skill, its documents and the
logbook use the method's Japanese terms freely, in content. (Vergil is a
personal, single-user stack; it takes no outside contributions, so the
vocabulary is chosen for its one reader.)

Background, for the builder and for the published background page:

- **The word.** 方針 *hōshin* is policy or direction; the characters are
  literally "direction" and "needle", as in a compass needle. 管理 *kanri* is
  management. In English it is usually called *policy deployment*.
- **Origin.** It grew out of Japan's total quality control movement in the late
  1950s and 1960s, building on Deming's plan–do–check–act cycle and Drucker's
  management by objectives. Bridgestone Tire is usually credited with naming
  and first using it in 1965 (one history dates the company's idea to
  systematize it to 1962); Komatsu and Toyota were early adopters.
  Hewlett-Packard brought it to the US in the 1980s, and Danaher made it central
  to the Danaher Business System.
- **How it works.**
  1. A three-to-five-year direction yields a small number of yearly
     breakthrough objectives: **the vital few**, deliberately not a long list.
  2. Each level turns the objectives above it into its own targets and the
     means of reaching them.
  3. **キャッチボール catchball**: targets and means pass back and forth
     between levels until both sides agree they are achievable. Direction
     comes from the top; the means are agreed with the people doing the work.
  4. Two halves: **hōshin**, which drives change toward the direction, and
     **日常管理 nichijō kanri** (daily management), which keeps normal
     operation stable and standardised. Hōshin assumes daily management is
     under control.
  5. A nested review cadence (monthly, quarterly), and a yearly review by top
     management, 社長診断 *shachō shindan* (the president's diagnosis). Each
     review's findings feed the next cycle's plan.
- **References.**
  - Akao, Y. (ed.), *Hoshin Kanri: Policy Deployment for Successful TQM*,
    Productivity Press, 1991 (English edition).
  - Lean Enterprise Institute lexicon, "Hoshin Kanri":
    <https://www.lean.org/lexicon-terms/hoshin-kanri/>
  - Akao's book, Appendix 1, "History of Hoshin Kanri" (2020 reissue):
    <https://www.oreilly.com/library/view/hoshin-kanri/9781000286656/xhtml/18_Appendix01.xhtml>
  - All About Lean, "The Evolution of Strategic Management — Hoshin Kanri":
    <https://www.allaboutlean.com/evolution-of-strategic-management-hoshin-kanri/>
  - Wikipedia, "Hoshin Kanri": <https://en.wikipedia.org/wiki/Hoshin_Kanri>

  The Bridgestone dates come from the Akao and All About Lean histories; the
  Lean Enterprise Institute and Wikipedia pages give no dates. The term
  社長診断 for the yearly review is the human's own, from years of working in
  Japanese; English sources call it the president's diagnosis.

### Terms used in this design

- **方針管理 hōshin kanri:** the whole system; the skill is `/vergil:hoshin`.
- **日常管理 nichijō kanri:** the daily review, run on every invocation.
- **朝礼 chōrei:** the daily `start` review (morning assembly).
- **終礼 shūrei:** the daily `end` review (closing assembly).
- **方針 hōshin:** the direction document, `hoshin.md`, holding the vital few.
- **反省 hansei:** reflection; the `end` review at the week, month and quarter
  levels.
- **社長診断 shachō shindan:** the yearly `end` review; in a one-person
  company the president is the human.
- **キャッチボール catchball:** agreeing objectives and means; the agent
  proposes, the human adjusts, until both accept.
- **改善 kaizen:** notes, in each entry, on improving the reviews themselves.

## Goals

- One command the human runs at the start of each working day,
  `/vergil:hoshin start`, which works out for itself which reviews are due,
  closes whatever period was left open, and runs them; and an optional
  `/vergil:hoshin end` to close the day (or a larger period) when the human
  chooses to.
- A daily review (nichijō kanri) of what ran unattended since the last review:
  what failed, why, and what needs action.
- Week, month, quarter and year reviews that set direction at the start of each
  period and reflect on it at the end.
- A direction document holding the fleet's vital few objectives, which every
  weekly review checks the work against.
- A public, published **logbook**: one dated file per review, always written,
  so the record proves each review ran and later reviews can read earlier ones.
- A kaizen loop: the reviews record how they should change, and that feeds
  back into the skill.
- A path to mechanization: every check the skill runs ad hoc is described well
  enough to be the brief for a `vrg-*` tool that later does it
  deterministically.

## Non-goals

- **Building the status and history tools.** `vrg-org-status`
  (vergil-project/.github#1140), `vrg-fleet-status` (#1316) and the
  `vrg-repo-history`, `vrg-org-history` and `vrg-fleet-history` family (#1317)
  are separate planning issues. This epic feeds their designs; it does not
  build them.
- **Development-process observability and metrics** (PR rates, gate failure
  rates by cause, token spend): vergil-project/.github#798.
- **Ops scheduling.** How scheduled jobs are split and triggered
  (vergil-project/.github#493, closed to be rethought).
- **A full content design for each review.** The first checklists are
  deliberately thin; the cadence and the habit matter now, and the content
  grows through kaizen over the first weeks of use.
- **Org- or repo-level logbooks.** The logbook is fleet-level. Org and repo
  journals wait until a real need shows up.
- **Seeing inside agent sessions.** Reviews report on durable state only.

## Anti-goals

- **A review that only happens when remembered.** If the human has to work out
  which reviews are due, the discipline fails. The calendar is the skill's job.
- **A silently skipped review.** A run that finds nothing still writes its
  entry. A period left open is detected and closed, by a late review or a
  "not performed" entry, and a week or longer period with no entries at all
  is named in the next entry; nothing is quietly dropped. (A day with no
  entries is a day off, not a gap.)
- **Heavy structure.** The logbook is Markdown with a minimal header, not a
  schema. Over-structuring it before it has been used would freeze the wrong
  shape.
- **Security details in public.** The logbook is published. Nothing a security
  record keeps private may appear in it.
- **An agent poking around where a tool exists.** Where a mechanized `vrg-*`
  tool answers the question, the skill uses it.
- **Non-ASCII paths.** Japanese belongs in content; every file and folder name
  is ASCII.

## Constraints

- **Durable state only.** A review reports on what is recorded durably: git
  and GitHub today (repos, issues, PRs, workflow runs, releases, packages), and
  later other durable systems the fleet adds, starting with the observability
  stack (#798). It never reports on what an agent session is doing; any
  session can crash and recover from durable state, and the review works from
  the same state.
- **Prefer mechanized tools.** Vergil's architectural rule is to maximize
  determinism by mechanizing everything that can be deterministically
  mechanized. The skill uses `vrg-*` tools wherever they exist (today
  `vrg-repo-status`, `vrg-queue`, `vrg-epic-audit`, `vrg-roadmap`,
  `vrg-activity-log` and others). Where none exists, the agent gathers the data
  ad hoc with `vrg-gh` and `vrg-git`, which is allowed but costly.
- **The fleet** is every GitHub organization owned by Vergil's primary human
  (six today). Personal repositories under the human's own account are out of
  scope. An inactive org is still in the fleet and is marked inactive. The
  fleet may hold private repositories (one today); the public logbook never
  publishes their details (see "Security").
- **Human gates are unchanged.** The skill runs as the `user` agent. It hands
  off each logbook PR with `vrg-pr-workflow report-ready`; the human submits
  and merges.
- **Security records.** Security findings follow the existing security-records
  model: details go into the private record (`vrg-security-record`), and the
  logbook refers to the issue only by its public ref, abstractly.
- **ASCII path names**, everywhere this epic creates files.

## Approaches considered

- **A. One skill with `start` and `end` modes; checklists inside the skill
  (chosen).** The skill computes which levels are due from the date and the
  logbook, runs them, and writes one file per review in one PR.
  One command to remember, the calendar in one place, catch-up for free. Its
  cost: changing a checklist is a plugin change and release.
- **B. Five skills, one per level.** Simplest to build, but the human must
  track the calendar, which is exactly the discipline this epic exists to
  remove. Rejected.
- **C. Like A, with the checklists read from the docs repo beside the
  logbook.** Refining a checklist becomes a docs PR, without a plugin release.
  Rejected for now: skill instructions belong in the plugin, where they are
  reviewed and released, and the kaizen loop turns logbook notes into plugin
  issues.

The human chose **A**.

## Design

### The skill: `/vergil:hoshin start|end [<level>]`

A new skill in vergil-claude-plugin, `skills/hoshin/`. It works out the date,
the levels due and the window to review for itself.

- **`start`** is the main event, run at the start of each working day. It
  closes whatever was left open, then opens the new day and any new week,
  month, quarter or year (see "The calendar").
- **`end`** is optional. Plain `end` closes the day. `end week`, `end month`,
  `end quarter` or `end year` closes that period now, together with every
  smaller period still open beneath it (so `end month` also closes the week
  and the day), when the human decides it is time to wrap it up. `end` always
  closes the latest open period at each level it touches (the most recent
  `start` with no `end`), whatever the current date, so an `end` run after
  midnight still closes the day that was worked; it refuses if nothing is
  open. A period not closed by `end` is closed by the next `start`.

Each run:

1. **Preflight.** Confirms it is the `user` agent. Locates the docs repo clone
   and confirms its `develop` is current with `origin/develop`, so it reads the
   latest logbook. Looks for any open logbook PR (or logbook task still open);
   if one exists, it stops and tells the human to submit and merge it first,
   since the logbook on `develop` must be the full record before a run can
   reason about it. Refuses if the entry this run would write already exists
   (the same mode run twice in one day).
2. **Reads the logbook** to find, for each level, the latest entry, its period,
   and the end of the window it covered.
3. **Decides what is due** (see "The calendar" below): open periods to close,
   new periods to open, and periods with no entries to name.
4. **Closes, then opens, in a fixed order.** First any closing: `end`
   reviews (late, or recorded as not performed) run smallest level first, since
   reflection rolls up (the day into the week, the week into the month). Then,
   for `start`, the daily 朝礼 chōrei runs first, so the human knows operations
   are stable before setting direction, and the boundary `start` reviews follow
   from the largest level down (yearly, quarterly, monthly, weekly), since
   direction flows top-down and each level plans against the one above it.
   Each review gathers its data, presents it, and holds the conversation with
   the human that the level calls for.
5. **Files follow-up work** the conversation decides on, through the existing
   paths (`intake-capture` / `vrg-intake-create`, `vrg-issue-create`, a
   security record), and lists each in the entry. Follow-up work about a
   private repository is filed in that repository, never in a public one.
6. **Writes one logbook file per review,** closing ones included.
7. **Hands off one docs PR** for the run: a task issue, a worktree, a commit,
   `vrg-container-run -- vrg-validate`, `merge-review`, `vrg-pr-workflow
   report-ready`. Before the commit, once validation is green, a gate screens
   the final entry text mechanically against the rules under "Security" and
   shows the human the full text for approval, so nothing that breaks a rule
   is ever pushed to the public repo. Any later change to entry text, a
   merge-review or `pr-watch` fix included, goes back through validation,
   the screen and the approval before it is pushed. A logbook run then goes
   through merge review like any task, and the human submitting the PR is
   the review's sign-off.

#### What each level does

The two halves:

- **Nichijō kanri (daily)** runs on every invocation. `start` is the 朝礼
  chōrei, `end` the 終礼 shūrei.
- **Hōshin (week, month, quarter, year)** runs only at boundaries, or when the
  human asks to close a period with `end <level>`. A `start`
  review sets direction for the period; an `end` review is 反省 hansei,
  reflection on the period just finished. The yearly `end` is the 社長診断
  shachō shindan.

The first checklists are thin on purpose. Each is a starting point, to be grown
by kaizen:

- **Daily `start` (chōrei):** unattended work since the last entry, across
  the fleet: failed or cancelled workflow runs (CD builds, `ops.yml`, the other
  scheduled and event-driven workflows) and why each failed; automation-authored
  PRs merged, still open or stuck; releases cut; new GitHub-config audit and
  security alerts (reported only as "Security" below allows). What needs
  action today.
- **Daily `end` (shūrei):** what the durable record shows changed today (PRs
  merged, issues and epics closed, releases); what is waiting on the human
  (handed-off PRs awaiting submission or merge, rulings); anything that failed
  during the day; a note for tomorrow's chōrei.
- **Weekly `start`:** the vital few in `hoshin.md`; open epics and their
  progress; choose the week's focus, a short list tied to the vital few.
- **Weekly `end` (hansei):** the week's focus against what happened; how
  stable the week was, from the week's daily entries; kaizen.
- **Monthly `start`:** the month's direction against the vital few; the size
  and age of the intake, planning and deferred queues.
- **Monthly `end` (hansei):** the month against its direction; the month's
  kaizen notes gathered and turned into plugin issues; a first, rough look at
  capacity.
- **Quarterly `start`:** set or refresh the vital few by catchball; capacity
  planning for the quarter.
- **Quarterly `end` (hansei):** the quarter's objectives against what was
  delivered; what to carry, change or drop.
- **Yearly `start`:** the three-to-five-year direction; the year's
  breakthrough objectives.
- **Yearly `end` (shachō shindan):** the year as a whole, against its
  direction: what the system delivered, how it changed, what it should become.

Every review ends with a kaizen prompt: what should this review do differently?

#### Checks as mechanization briefs

Each data-gathering step in a checklist is written as a named **check**: what
it answers, which durable sources it reads, the window, and what its output
shows. Where a `vrg-*` tool already answers it, the check names the tool and
the agent runs that. Where none does, the check is precise enough to be
handed, unchanged, to a planning issue as the brief for the tool that will
mechanize it.

When a check is first written, its brief is posted as a comment on the
planning issue it belongs to: #1316 (`vrg-fleet-status`), #1140
(`vrg-org-status`), #1317 (the history commands), or a new planning issue when
none fits. When the tool ships, the check is changed to call it. The skill
also tells the agent to point out, in the entry's kaizen notes, any ad hoc
query it found itself repeating.

### The calendar

The calendar follows how the human actually works: `start` is run on most
working days, weekends included; `end` is often skipped, because the day ends
when it ends. So `start` carries the boundaries, and `end` is the human's
choice.

- **Time zone.** Every date and time is computed in one configured IANA time
  zone (for example `America/New_York`), recorded in the logbook's `index.md`,
  never from the host clock, which may run on UTC.
- **Days.** A day with a `start` entry is a day worked. A day with no `start`
  entry is a day off, and nothing about it is flagged.
- **Weeks** run Monday to Sunday.
- **Every period comes from the week.** A week belongs to the month its Monday
  falls in, and so to that month's quarter and year. A run computes the
  current week, month, quarter and year from the current week, never from the
  calendar date alone. Example: the week of Monday 2025-12-29 belongs to
  December 2025, Q4 2025 and the year 2025, so January 2026, Q1 2026 and 2026
  begin with the week of Monday 2026-01-05.
- **Boundaries.** For each level, a `start` run compares the current period
  with the period of that level's latest `start` entry. If they differ, the
  level has crossed a boundary, and the new period's `start` review is due.
  A level with no entries at all is due too.
- **Closing an open period.** If, at a level, the latest `start` is for a
  period earlier than the current one and has no matching `end` (the day,
  week, month, quarter or year was opened and never closed), the run pauses
  and says so, for example "Thursday was started but never closed". A period
  still in progress is never offered for closing by `start`. The human chooses
  either to **close it now**, running the missed `end` review late (its entry
  records that it ran late, and when), or to **record it as not performed**, a
  short entry in the `end` slot saying the review was not done, with an
  optional note on why. At the daily level the pause is one quick prompt whose
  default is "not performed", so an unclosed day costs one keystroke. Either
  entry goes into the same PR as the run's new `start`, so every `start` ends
  up with a matching `end`.
- **Gaps.** At the week level and above, periods that have neither a `start`
  nor an `end` entry (after a long absence, say) are named in the new `start`
  entry ("no entries: weeks of 2026-12-07 and 2026-12-14"), so the gap is on
  record. Days are not gaps: a day with no entries is a day off.
- **Windows.** Every entry that reviews something has a window, from and to.
  - A `start` review's window runs from the end of the window of the latest
    entry at its level that has one, to now. So a skipped day widens the next
    window rather than losing time, and weekend work falls into the next
    weekly review.
  - An `end` the human runs covers from the end of its matching `start`'s
    window to whichever is earlier, now or the end of its period. It never
    records a time in the future, so an `end` at 18:00 leaves the evening for
    the next `start`, and an `end` after midnight still stops at midnight.
  - A late `end` review covers only its own period: its window runs from the
    end of its matching `start`'s window to the end of that period (midnight
    ending that day, in the configured time zone, or the end of that week,
    month, quarter or year). It never runs to now, so the overnight window is
    left for the 朝礼 chōrei that follows.
  - A "not performed" entry has no window. The next `start` window therefore
    starts where the last reviewed window ended, and no time goes unreviewed.
- **The first run.** On an empty logbook there is nothing open to close, and
  every level's `start` review is due. The first `/vergil:hoshin start` is
  therefore the bootstrap: it runs the daily, then the yearly, quarterly,
  monthly and weekly `start` reviews, and sets the first `hoshin.md` by
  catchball. No separate bootstrap mode is needed.

### The logbook

A new top-level section of the published docs site in `vergil-project/docs`:

```text
docs/site/docs/logbook/
  index.md     what the logbook is, the entry format, the time zone
  hoshin.md    the direction document: the vital few
  fleet.md     the fleet roster: each org, active or inactive
  daily/       index.md, and one file per daily review
  weekly/      index.md, and one file per weekly review
  monthly/     index.md, ...
  quarterly/   index.md, ...
  yearly/      index.md, ...
```

**File names** are ASCII and name the period the review covers, plus the mode.
A week is named by its Monday's date, so a week's name can never fall in a
different year from the week itself:

| Level | Example |
| --- | --- |
| Daily | `daily/2026-11-18-start.md` |
| Weekly | `weekly/2026-11-16-end.md` |
| Monthly | `monthly/2026-11-start.md` |
| Quarterly | `quarterly/2026-Q4-end.md` |
| Yearly | `yearly/2026-start.md` |

**An entry** is free-form Markdown under a minimal header. The header carries
only what later runs depend on:

- the level and the mode;
- the period it covers;
- the review window (from and to, with times), except on a "not performed"
  entry, which has none;
- when it ran, and whether it ran late or records a review not performed.

The body is the review: what was found, what was decided, the follow-up issues
filed, and the kaizen notes. A run where nothing happened still writes an
entry, saying so. A "not performed" entry is a few lines: that the review was
not done, and the human's note on why, if any. The entry format is described
in the logbook's `index.md` and kept loose on purpose.

**Indexes.** The site navigation gets a top-level **Logbook** section, linking
each level's `index.md`. Each `index.md` lists that level's entries, newest
first, and the run adds its own entries to the indexes in the same PR. Because
preflight refuses while a logbook PR is open, two runs never edit an index at
once. If maintaining the indexes by hand proves error-prone, a small generator
the run calls is the fallback; it is not built up front.

**The direction document, `hoshin.md`,** holds the vital few: three to five
fleet objectives, each with a line on why and how progress will be judged. It
is set at the yearly and quarterly `start` reviews by catchball, edited in those
reviews' PRs, and read by every weekly review. It is a current document, not a
log; its history is its git history and the entries that changed it.

**The fleet roster, `fleet.md`,** lists the fleet's orgs and marks each active
or inactive. Reviews read it to know where to look. Until `vrg-fleet-status`
exists it is the fleet's only definition; that tool's design (#1316) decides
whether the roster moves into configuration.

### The logbook PR

Every run hands off exactly one PR to `vergil-project/docs`, holding all of
that run's entries and index updates (and, at the quarterly and yearly
levels, any `hoshin.md` change). Because the next run refuses while a logbook
PR is open, the human submits each one as the review's sign-off; the docs
repo's checks are quick, so it merges well before the next run.

Vergil's rule is one task, one PR, so each run files a task in the docs repo's
ad-hoc epic (`vrg-issue-create --epic adhoc --repo vergil-project/docs`),
and works it on its own worktree. Logbook PRs are expected to be background
noise alongside the fleet's other automated and routine PRs, and are easy to
tell apart from development PRs by their fixed wording.

**Fixed wording.** Every public record a run creates besides the entries
themselves uses fixed wording that carries no review content:

- the task issue's title, and the commit message's description, are
  `Logbook: <date> <mode>` (for example `Logbook: 2026-11-18 start`), with the
  commit typed `docs(logbook)`;
- the task issue's body, and the `vrg-pr-workflow report-ready` title, summary
  and notes, say only which entries the run wrote, by path; the notes also
  carry the merge review's eligibility line.

Findings, failures and decisions appear only in the entries, where the rules
under "Security" apply.

### Security

The logbook is public. The skill's prose states the rule plainly:

- A security finding raised in a review goes into a security record
  (`vrg-security-record`) with its details; the entry refers to it only by its
  public issue ref and in abstract terms.
- Security alerts and advisories a review reads appear in an entry only as the
  public refs of their security records' neutral issues. An entry carries no
  alert or advisory IDs and no per-repo alert counts or severities; at most a
  single fleet-wide count. Their details stay in the review conversation.
- A failed run whose cause is a security finding (a security scan, a
  dependency audit, the GitHub-config audit) is reported only as "a
  security-related failure" with the neutral ref of its security record. If
  it has no record yet, the review creates one. The cause stays in the review
  conversation and the private record.
- The skill never pastes raw output from a security tool into an entry.
- These rules cover every public record a run creates, not only the entries;
  "Fixed wording" under "The logbook PR" keeps the others free of review
  content.
- **Private repositories.** Before writing, the skill checks each repo's
  visibility. A public repo is reported in detail. A private repo appears only
  as counts per org (for example "1 failed run in a private repo"): no name,
  no PR or issue titles, no failure details. Its follow-up work is filed in the
  private repo itself, and the entry gives only the count of such follow-ups.

### The hoshin background page

A page in the docs site's Management section explains the model, so a later
reader understands why the reviews are shaped as they are: the method's
background and references (from "The model" above), the terms table, the two
halves, the calendar, and how Vergil scales the method down to one person and
agents. The logbook's `index.md` links to it.

## Delivery shape

The plan sizes the tasks; the expected shape is:

- the logbook section and the background page (docs repo);
- the `hoshin` skill and its checklists (vergil-claude-plugin);
- posting the check briefs to #1140, #1316 and #1317;
- making the skill usable: a plugin release (a human-gated precondition) and
  its install, as a deployment task;
- the bootstrap: the first `/vergil:hoshin start` on the empty logbook,
  blocked by the deployment;
- the bookends (documentation review, retrospective).

## Left to the builder

- The skill's internal file layout (for example one reference file per level)
  and the exact wording of each check.
- The exact header format of an entry (front matter or a short field list),
  within the "minimal header" rule above.
- How the skill locates the docs repo clone, its ad-hoc epic and open logbook
  PRs.
