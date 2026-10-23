# Every issue belongs to an epic

> **Revision 2 (2026-10-23, task #430).** This revision absorbs
> `vergil-project/.github#429` (formalize intake review) into this epic. The
> original design contained intake in per-repo intake epics and kept the three
> intake kind labels. This revision replaces that with a two-stage **intake →
> planning** pipeline: **one** intake epic and **one** planning epic per org, both
> in `<org>/.github`. The parent epic is an issue's state, so the kind labels are
> retired. None of the original implementation tasks had started, so the
> breakdown is rewritten rather than patched (see *Task breakdown*).

## Problem

The issue framework has two gaps that turn out to be one.

**Orphans.** Issues can exist with **no epic parent**:

1. **Intake created unlinked by design.** `vrg-triage-create` mints `triage` /
   `idea` / `research` issues with no parent, as a "capture now, groom later"
   queue.
2. **Issues born outside the tools.** Anything holding the app credential (a
   direct API call, the web UI, a script, CI) can open a bare issue the agent
   sandbox never sees. This is how `vergil-tooling#2461` and `#2468` appeared.

Orphans don't roll up to any epic and, linked with `Ref`, never auto-close. The
framework's core promise, that every unit of work lives under an epic that
manages it, was only half enforced.

**Unprocessed intake has no defined lifecycle.** Captured work collects under
three ad hoc labels with no end state. `triage-review` reads only `triage`, so on
2026-10-23 the org had 30 open `idea` and 3 `research` issues nobody had
reviewed, many over a year old. Epic processing works well. Intake processing
does not. It isn't possible to look at the org's issues and tell which are
ready to build and which are still raw. And there is no place to queue, or track,
the deliberate multi-hour (sometimes multi-day) work of expanding an idea into a
full epic.

## Goals

- **The invariant:** in any repo with a `vergil.toml`, every issue is an epic or
  has an epic parent. Asserted, universal, no opt-out.
- **Every issue's state is visible from its parent:**
  1. **Intake:** captured, not reviewed. Lives under the org's intake epic.
  2. **Planning:** reviewed, judged to need a full epic, waiting for (or
     undergoing) `epic-create`. Lives under the org's planning epic.
  3. **Ready:** under a repo's ad-hoc epic (a one-off) or under a fully expanded
     finite epic (part of a collection of work). Only ready work is buildable.
- **Capture has near-zero friction.** A one-line "don't forget X" is a valid
  intake item. No category decision at capture time.
- **Intake review is one skill** that reviews everything in intake, oldest first,
  and routes each item.
- **Epic creation is always tracked** in a planning issue, so a planning session
  can pause and resume days later from a written record.
- The invariant holds regardless of how an issue is born, by auto-healing rather
  than rejecting.
- Retro-attach every existing orphan, and migrate the existing intake backlog,
  across all managed repos in every org.

## Non-goals

- **Perfect placement by the sweep.** The sweep attaches every orphan to intake.
  Putting it in the right place is intake review's job.
- **Rejecting issue creation.** Enforcement heals (auto-attach). It never fights
  creation.
- **A stale-item mechanism.** Oldest-first ordering is the pressure. There is no
  separate stale flag or auto-close.
- **Reviewing the wider label set.** That is the follow-on brainstorm bookend
  (see *Bookends*).
- **Intake review cadence** (when and how often it runs). Tracked elsewhere.
- **Cross-org linkage.** Each org has its own `.github` and its own intake and
  planning epics.
- **Public → private visibility changes.** Unsupported by policy (see *Private
  repos and visibility*). The audit detects them; nothing migrates or retracts.
- **Reviving closed intake.** A closed item stays searchable. Bringing one back
  is a manual act with no tooling.

## Anti-goals

- **Making capture costly.** Any required field, category, or triage step at
  capture time defeats the purpose. Ideas get lost when capturing them is work.
- **More planning states.** Waiting / in progress / completed. Adding a fourth
  (blocked, deferred, needs-research, …) is out. Detail goes in the progress
  comments.
- **Labels that duplicate the parent.** State is carried by the parent epic. A
  label that restates it can drift out of sync with the parent.

## Invariant (the assertion)

**Every issue is either a legitimate top-level kind or has an epic parent.** The
legitimate top-level kinds are:

- **epics** (`epic` label), including the standing intake, planning, and ad-hoc
  epics;
- **archives** (`archive` label): the per-quarter ad-hoc, intake, and planning
  archive buckets;
- **release tracking issues** (`release: X.Y.Z`, recognized by
  `release.is_release_tracking_issue`), which `vrg-release` creates top-level and
  keeps open for the length of a release.

These are defined once, in a single predicate shared with `epic_audit`'s existing
exemptions (`stray_dotgithub_issue` already exempts archives, and task drift
already exempts release tracking issues), so the sweep and the audit cannot
disagree. Legitimate top-level issues are never swept. There is no per-repo
opt-out and no "unlinked" terminal state.

## Design

### Standing epics: intake and planning (one each per org)

Generalize the existing ad-hoc machinery in `lib/epics.py` (`ensure_adhoc_epic`)
into a shared standing-epic helper, then add two **org-level** standing epics,
both in `<org>/.github`:

| Epic | Title | Labels | Holds |
| --- | --- | --- | --- |
| Intake | `Epic (intake): <org>` | `epic`, `intake` | every unreviewed item, and every orphan the sweep finds |
| Planning | `Epic (planning): <org>` | `epic`, `planning` | items that need a full epic, waiting for or undergoing `epic-create` |

Both are perpetual and never auto-close, reusing the ad-hoc epic's
never-rolls-up handling (the `ad-hoc` branch of the closure logic generalizes to
"a standing epic"). Both are created and reused idempotently.

Unlike ad-hoc epics, which exist per member repo, there is **exactly one** intake
epic and **one** planning epic per org. No member repo has either.

**The 100-sub-issue cap.** GitHub caps an issue at 100 sub-issues, which the
ad-hoc epics already handle with the quarterly archive-and-drain
(`_ADHOC_ARCHIVE_SEGMENT_CAP`, #2872). Intake and planning reuse that drain
unchanged: closed children (dropped, split, completed) move into per-quarter
`Archive (intake): <org> …` / `Archive (planning): <org> …` buckets (labels
`archive` + `intake`/`planning`), so only open items count against the cap. If
attaching would push an epic's open children past the cap, the sweep and the
migration **stop and report loudly** (the overflowing issues, the count). They
never drop or skip silently. A full intake queue is a signal to run intake
review, not something to engineer around. *(Builder's call, recorded here.)*

### Private repos and visibility

**Private work stays private.** Intake and planning follow the existing epic-home
rule (`resolve_epic_home`): a public repo's items live in the org's `.github`
intake and planning epics, while a **private repo self-homes its own** `Epic
(intake): <repo>` and `Epic (planning): <repo>`, exactly as it already self-homes
its epics. "One intake and one planning epic per org" therefore means one pair
for the org's public repos, plus one pair in each private repo. Capture, the
orphan resolver, intake review, the queue view, and `epic-create` all resolve the
home from the item's repo:

- `vrg-intake-create` run inside (or `--repo`-targeted at) a private repo files in
  that repo. Everything else files in `<org>/.github`.
- A private repo's orphans go to that repo's own intake epic, never to `.github`.
- Intake review never routes a private repo's item to a public repo or epic.
  (GitHub also refuses private → public transfers.)

**Visibility is one-way: private → public only.**

- **Private → public is supported.** Making a repo public discloses nothing new.
  Its existing self-homed epics finish where they are. New work homes in
  `.github`. Its self-homed intake and planning items are moved into the `.github`
  queues (a public → public transfer) and its now-empty standing epics are closed.
- **Public → private is not supported.** Once a repo has been public, its
  contents (and anything about it in the public `.github`, including epic
  bodies, specs, and plans) may already have been cloned and redistributed.
  Making it private cannot undo that, and pretending otherwise gives false
  assurance. Decide at repo creation whether something is private. If it is
  public, it stays public. **To take public work private, create a new
  repository that starts life private and migrate the code into it.** The new
  repo self-homes its epics and queues from day one, and the public original is
  left as it was (archived if appropriate). This is not expected to happen often,
  but it is the only supported path, and the architecture docs must say so.
  `vrg-epic-audit` reports a private repo that has
  issues linked under `.github` epics as a likely unsupported flip, loudly, for a
  human to resolve. Tooling never tries to migrate or retract anything.

### Capture: `vrg-intake-create` / `intake-capture`

`vrg-triage-create` is renamed `vrg-intake-create`, with a one-release deprecated
`vrg-triage-create` alias that warns and forwards. The new command:

- takes `--title` and a body (`--body` or `--body-file`); **`--kind` is removed**
  (the deprecated alias accepts and ignores it, with a warning);
- files in `<org>/.github` and links the issue under the intake epic when it is
  created; there is no unlinked path.

The `triage-capture` skill becomes `intake-capture` and drops the "pick the kind"
step. Paraphrasing voice-to-text input is unchanged.

### The orphan resolver: every orphan goes to intake

An orphan is by definition something nobody has reviewed, so the resolver has
one rule: **every orphan is attached to the intake epic of its home**: the org's
`.github` intake epic for a public repo, the repo's own intake epic for a private
one (see *Private repos and visibility*). GitHub sub-issue links may cross repos within an org, so a
member-repo orphan stays in its repo, linked under `.github`'s intake epic, until
intake review routes it.

**Sweep domain:** open issues that are not pull requests, not a legitimate
top-level kind (see *Invariant*), and not already epic-parented, in **managed**
repos only: non-archived repos with
a `vergil.toml`, matching the invariant's scope. Unmanaged repos in the org
(forks, archived or experimental repos) are never written to. Closed issues are
left alone. A parent declared only by a body `Parent:` line counts as parented,
but the migration converts such links to native sub-issue links, so intake
review's child listing and the PR refusal's parent check agree. *(Builder's
call.)*

**Enforcement.** Two layers:

1. **Client:** `vrg-intake-create` creates every issue already linked (above).
   With `vrg-issue-create` requiring `--epic`, no sanctioned tool can create an
   orphan.
2. **Safety net:** extend the existing org-wide `vrg-epic-audit` sweep (driven on
   a schedule by the reusable `ops-epic-sweep` workflow with an org-wide app token
   and `VRG_EPIC_SWEEP=1`) to attach orphans via the resolver. Orphan-attach is
   part of the existing `--close` write pass, the one the scheduled workflow
   already runs, so the workflow needs no change. Without `--close` the audit
   lists the orphans it would attach (preview). It runs on a schedule and is
   eventually consistent. No new workflow and no per-repo rollout. *(Builder's
   call.)*

### Intake review: `intake-review`

`triage-review` is renamed `intake-review` and redesigned.

**Collect.** List every open child of the intake epic, **sorted by creation
date, oldest first**. Report the count and a one-line list.

**Present in batches of about four.** For each item the agent first **checks
current state** (has the bug already been fixed? is there an overlapping issue
or epic?) and then writes a short paragraph: what it is, why it matters, where it
stands now, and a recommended route. The human confirms or redirects the whole
batch in one reply. The agent applies the decisions and moves to the next batch.
The human can stop at any point. Unreviewed items simply stay in intake, so there
is nothing to resume.

**Routes:**

| Route | Effect |
| --- | --- |
| **Drop** | The agent closes the item with the reason. Closed is archived and stays searchable. "Good idea, not feasible for years" is a drop. |
| **Defer** | Leave it in intake and add a dated comment with the reason. It keeps its age, so it comes up first again next time. Not a new state. |
| **One-off** | Move it into the target repo and link it under that repo's ad-hoc epic, or under an existing finite epic where it clearly belongs. |
| **Split** | Create several ready tasks (ad-hoc or under existing epics, across repos as needed), each linking back to the original, then close the original with links to them. |
| **To planning** | Reparent it from the intake epic to the planning epic. It stays in `.github`. |

**Moving an issue to another repo** transfers it (`vrg-gh issue transfer`),
keeping its history. If transfer is unavailable or fails, the agent recreates it
in the target repo with a link back, and closes the original with a pointer.
Either way the resulting issue is re-linked under its new parent (a transfer may
not keep the sub-issue link). A one-off whose PR will land in `.github` itself
stays put and is just reparented.

**Agents close issues.** The current `triage-review` skill says closing is a
human action and that agents are denied `issue close`. That is wrong: `vrg-gh
issue close` works for the user identity. `intake-review` closes dropped and
split items itself, after the human confirms the batch.

The routing mechanics (transfer-or-recreate, reparent, re-link, close with
pointer) are deterministic. A single routing command should own them so the
skill does not hand-roll them.

### Planning: the planning queue and `epic-create`

**A planning issue records its own epic-creation process.** It stays open and
unchanged in substance until the epic is fully expanded. `epic-create` records
progress as comments: what has been done, what is blocked ("waiting on research
from X"), and where to resume. A planning session that pauses for a day or a week
resumes from that record.

**Three states, nothing more:**

| State | Representation |
| --- | --- |
| Waiting | Open under the planning epic, no `in-progress` label |
| In progress | Open under the planning epic, `in-progress` label; detail in the latest progress comment |
| Completed | Closed with "planned as #N". Also closed, with the reason, if planning concludes the idea is not worth doing. |

**Machine-readable link.** When `epic-create` creates the epic, it records the
link on both sides: an `Epic: <org>/.github#N` line in a planning-issue progress
comment, and a `Planning: <org>/.github#M` line in the epic body. The queue view
uses that link to check the epic's docs task (closed means the docs PR merged)
and whether implementation tasks exist under the epic. That is how it computes
"ready to close". *(Builder's call.)*

**Completed means fully expanded:** the epic exists, its spec + plan docs PR is
**merged**, and its implementation tasks are **filed**. The docs merge is a human
action, so `epic-create` may finish its work before it can close the planning
issue. It then records "waiting on docs PR merge" and leaves the item in
progress. Closure happens on the next `epic-create` touch, or the planning-queue
summary flags it as ready to close.

**`epic-create` always works through a planning issue.** Its entry points:

| Invocation | Behaviour |
| --- | --- |
| `epic-create <issue>`, issue is in planning | Start it, or resume it from its progress comments |
| `epic-create <issue>`, issue is in intake | Reparent it to planning (a one-item intake review), then start |
| `epic-create` with no argument | Summarize the planning queue (waiting, in progress with latest note, ready to close) and ask which one to work on |
| `epic-create` with a free-text idea | Create a planning issue for it first (skipping intake), then start |

**Visibility.** A read-only queue view shows each standing queue at a glance:
intake (count, oldest first) and planning (grouped by state, with each in-progress
item's latest note). `epic-create`'s no-argument summary and `intake-review`'s
collect step both use it.

### Intake and planning items are not PR-workable (closes `#175`)

An item under the intake or planning epic must not acquire a PR and close as
though it were groomed work. `vrg-pr-workflow report-ready` and `vrg-submit-pr`
(and, upstream, `issue-implement`) refuse an issue **whose parent is the org's
intake or planning epic**, with "route this through intake review / epic-create
first". This mirrors the existing operational-task refusal, but checks the
**parent** rather than a label. Routing out of those epics is the only way to
lift the refusal. This answers and closes `vergil-project/.github#175`.

### Labels: parent is state

- The `triage`, `idea`, and `research` labels are **retired**. The migration
  removes them from every issue it moves, and no tool or skill applies them after
  this epic. Their definitions stay in `labels.json` (so the label sync still
  creates them) until the label-set follow-on decides the namespace. Removing
  them is that follow-on's job, deliberately not this epic's.
- `intake` and `planning` label only the two standing epics, so tooling can find
  them, the way `ad-hoc` labels the ad-hoc epics.
- `in-progress` is the single new per-item label, used only on planning items.
- Code that currently keys off `_INTAKE_LABELS` (e.g. `epic_audit`'s drift skip
  and stray-issue check) keys off the parent instead.

### Migration

The migration is not a separate mechanism. It is the same `vrg-epic-audit`
orphan-attach, run once org-wide for immediacy, plus a one-time conversion of the
existing intake backlog:

- every open **unparented** `triage`/`idea`/`research` issue is attached under the
  intake epic and loses its kind label, whatever repo it lives in;
- a kind-labelled issue that **already has an epic parent** keeps its parent and
  only loses the label (someone already routed it, as with `#428` → `#139`);
- every other open orphan is attached under the intake epic.

It is idempotent and re-runnable, reports a tally, and never closes anything. The
one-shot run is triggered by a human through `workflow_dispatch` on
`ops-epic-sweep` (the app identity, `VRG_EPIC_SWEEP=1`), not from an agent
session. *(Builder's call.)*
After it runs, the scheduled `ops-epic-sweep` maintains the invariant. Because it
mutates live GitHub state across the org, it is proven by running it and
recording the outcome (an operational task), not by a code PR. It runs per org
(`vergil-project`, `logical-minds-foundry`, …).

Existing finite epics are untouched. Planning issues for them are not created
retroactively.

### Bootstrapping

This epic builds the planning queue, so it cannot be planned through one. Its own
revision is tracked the old way (this spec, docs task `#430`).

## Data model: what varies and what does not

- **Varies (data-driven):** standing-epic titles and labels, the batch size, and
  the routing table. These live as constants beside the existing `_ADHOC_EPIC_*`
  definitions.
- **Does not vary (the invariant):** every issue is a legitimate top-level kind
  or has an epic parent; every orphan resolves to intake; nothing under intake or planning is
  PR-workable.

## Testing

- Standing epics: intake and planning are idempotent to create and reuse, both
  homed in `<org>/.github`, and perpetual (never auto-close, never gate rollup).
- Resolver: a sweepable orphan in a public repo resolves to the org's `.github`
  intake epic, and one in a private repo to that repo's own intake epic. PRs,
  closed and already-parented issues, legitimate top-level kinds (epics, open
  ad-hoc/intake/planning archives, an in-flight `release: X.Y.Z` tracking issue),
  and unmanaged repos are all excluded.
- Visibility: a private repo's capture, orphans, and routes never reach a public
  repo or epic; private → public migration moves the queue items; the audit flags
  a private repo with issues under `.github` epics.
- `vrg-intake-create`: creates every issue already linked under intake, with no
  kind label; the deprecated alias warns, ignores `--kind`, and forwards.
- Routing command: transfer + re-link; the recreate + close fallback;
  reparent to planning; split; drop. Each leaves the issue (or its replacement)
  correctly parented.
- PR refusal: `report-ready` / `vrg-submit-pr` refuse a child of intake or
  planning, and accept the same issue once it has been routed to an ad-hoc or
  finite epic.
- Queue view: intake is ordered oldest first; planning is grouped into waiting,
  in progress, and ready to close.
- Orphan-attach in `vrg-epic-audit`: org-wide enumeration, correct exclusions,
  idempotent re-run, never closes, gated by `VRG_EPIC_SWEEP`.
- `epic_audit` code that keyed off intake labels now keys off the parent.

## Task breakdown (this epic)

Bookends: `#177` (original docs, done), `#430` (this revision), the docs review
`vergil-tooling#2474`, the retrospective (to seed), and `#178`, retargeted to the
label-set follow-on brainstorm (see *Bookends*).

The original implementation tasks have not started. Each is rewritten in place
where its scope survives, and new tasks are filed for new scope. All are filed in
the repo where their PR lands:

1. **`lib/epics.py`:** shared standing-epic helper; org-level `ensure_intake_epic`
   and `ensure_planning_epic`; the every-orphan-to-intake resolver; the shared
   legitimate-top-level predicate (adopted by `epic_audit`'s existing
   exemptions); the sweep-domain predicate; parent-based `is_intake_item` / `is_planning_item`
   (`vergil-tooling`; rewrites `#2481`).
2. **Rename + capture:** `vrg-triage-create → vrg-intake-create` (no `--kind`,
   linked under intake when created, deprecated alias) (`vergil-tooling`; merges
   `#2484` with the tool half of `#2482`).
3. **PR refusal by parent** for intake and planning items; closes `#175`
   (`vergil-tooling`; rewrites `#2485`).
4. **Orphan-attach + migration in `vrg-epic-audit`,** including the kind-label
   conversion, moving `_INTAKE_LABELS` consumers to parent checks, the
   private → public queue move, and the unsupported-flip report
   (`vergil-tooling`; rewrites `#2483`).
5. **Routing command and queue view:** the deterministic intake routes
   (transfer-or-recreate, reparent, split, drop) and the read-only intake/planning
   queue view (`vergil-tooling`; new).
6. **Skills:** `triage-capture → intake-capture`, `triage-review → intake-review`
   (the redesign above, including the fix for the wrong claim about closing
   issues), and the `epic-create` planning-issue entry points and progress
   tracking. Also update **every other intake producer**, so no skill applies a
   retired label or names a retired tool:
   - `migrate-repo` routes uncertain items to intake (`vrg-intake-create`, or
     links them under the intake epic) instead of labelling them `triage` in
     place;
   - `memory-audit`, `issue-validate`, `issue-deploy`, and
     `issue-validate/references/operational-task-lifecycle.md` switch to
     `intake-capture`;
   - the plugin docs (`docs/development/skills-architecture.md`,
     `docs/site/docs/skills/index.md`) use the new names.

   (`vergil-claude-plugin`; takes the skill half of `#2482`.)
7. **Operational: run the one-shot migration per org.** A `validation` task that
   runs the orphan-attach + conversion and confirms no open, non-PR, non-epic,
   unparented issue and no kind-labelled issue remains, blocked by tasks 1, 4,
   and their deployment (rewrites `#185`).

`ops-epic-sweep` needs no change: it already runs `vrg-epic-audit` with
`VRG_EPIC_SWEEP=1` (`vergil-actions#786`, closed not planned).

## Bookends

- **Docs review** (`vergil-tooling#2474`): the site docs describing the issue
  model, intake, and `epic-create` reflect intake → planning → ready.
- **Follow-on brainstorm** (`#178`, retargeted): **review the wider label set.**
  With state moved to parents, the remaining labels need a deliberate, minimal
  namespace. This chains directly from this epic.
- **Retrospective:** terminal, authored with `epic-retrospective`.
