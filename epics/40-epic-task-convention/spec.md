# Epic / Task issue convention, docs lifecycle, and project observability

- **Date:** 2026-06-28
- **Status:** Draft design (pushback-reviewed; pending final user review)
- **Supersedes:** vergil-tooling#714 (define lifecycle for spec/plan documents),
  vergil-tooling#1641 (automated issue lifecycle management — its substance is
  folded into the in-scope migration workstream of this epic)
- **Related (follow-on, not folded):** vergil-tooling#681 (issue readiness state
  machine)
- **Positioning:** This is a deliberate **midterm solution** — good while we are
  on GitHub, intentionally un-over-engineered, to be revisited with better
  (likely open-source) project-management tooling alongside the eventual forge
  migration (Forgejo/Codeberg). Design choices favor portability and avoid
  bolting on attributes GitHub cannot natively view or represent.

---

## 1. Problem

We have aggressively adopted GitHub issues to track work and correlate every
pull request with the issue documenting it. That correlation works. The
breakdown is downstream: we do a poor job of **closing issues correctly**.

The ad hoc rule "a PR merged, so close its issue" is wrong for two reasons:

1. **Issues are not 1:1 with PRs today.** A single issue often absorbs several
   PRs — most commonly a brainstorm → pushback → plan → docs-publish cycle that
   produces a PR *before any implementation has started*. Closing the issue when
   that first PR merged silently dropped the remaining work.
2. **Merge is not the end of the story.** A PR merging to `develop` is necessary
   but not sufficient; the work must also survive the post-merge release
   pipeline. Closing on merge bypasses that verification.

To remove the first ambiguity we previously banned auto-close keywords
(`Fixes`/`Closes`/`Resolves`) on PRs and require `Ref`-only linkage. That made
closing a manual human responsibility — which then slipped. The result is repos
(e.g. the MQ Resiliency Lab) with many merged PRs whose little issues sit open,
and ~100-issue backlogs that are not cobbled together: a large fraction are
"don't forget X" seeds thrown over the wall and forgotten.

There is a deeper opportunity behind the immediate pain. We have been operating
at the altitude of individual issues ("here is a problem, fix it"). We are ready
to manage work that spans days or weeks — bodies of work delivered across a
sequence of PRs, interleaved with other initiatives — and to make that work
*visible*: a roadmap of where the project is going, and a ledger of where it has
been. This design introduces a light formalization that makes closing
mechanical, makes the larger units of work first-class, makes the whole project
observable, and then **migrates the existing backlog into the new framework**.

## 2. Goals and non-goals

### Goals

- A two-tier issue hierarchy (epic / task) invented on GitHub's flat model, with
  mechanical, deterministic closing rules.
- A strict, enforced 1:1 task↔finalizing-PR mapping so closing can be automated
  safely.
- A single place — the org `.github` repo — that holds every cross-cutting
  initiative and serves as the roadmap source of truth.
- A lossless intake net for uncurated ideas, bugs, and quick fixes, with a
  periodic triage workflow.
- A standardized label namespace as an orthogonal categorization axis.
- A spec/plan documentation lifecycle keyed to epic state (open/closed), with no
  state encoded in file paths.
- Mechanically generated observability: a roadmap (forward) and an activity log
  (backward), derived from issue/PR metadata.
- **Migration of the existing backlog, repo by repo, into this framework** — the
  epic is not done until every repo is migrated.

### Non-goals (this spec)

- The readiness-gating state machine (follow-on, #681).
- **Cross-org epics.** The `.github` project layer is **per-org**; the "virtual
  project" is a single org (here, `vergil-project`, with its home at
  `vergil-project/.github`). Epics and tasks never link across orgs — that is a
  different mechanism, out of scope. Orgs may share code and conventions, and a
  change developed in one org may serve another, but they remain independent at
  the epic/issue level.
- **A private `.github` fronting public member repos.** A private `.github` is
  read as "the whole org is private," so everything routes to `.github`. The
  mixed topology — a private `.github` with public normal repos — is
  deliberately unsupported; it is what keeps the visibility-based home rule a
  simple three-row table (epic #130).
- **Autonomous task implementation.** Making tasks discrete enough that
  `issue-implement` can run them unattended is an *aspiration* the
  task-granularity discipline serves; it is not delivered here.
- Any stored epic "status" attribute (proposed/in-progress). Active-vs-parked is
  *derived* from reporting timestamps, not stored (see §3.7).

## 3. The model

### 3.1 Two issue types

**Epic** — an umbrella issue. Never linked from a PR. Holds tasks as child
issues (mechanism in §4). Two flavors:

- **Finite epic** (label `epic`) — a body of work spanning multiple PRs, born
  from a brainstorm/plan. **Auto-closes** when its last open child task closes.
- **Standing epic** (labels `epic` + `standing`) — a perpetual category bucket
  for small unplanned work that is genuinely not part of any initiative.
  **Exempt from auto-close.** Seeded minimally: one per repo to start,
  `Ad-hoc maintenance`. Standing epics are intended to shrink to the exception.

**Task** — a unit of work with a **strict 1:1 mapping to one finalizing PR**.
Always a child of exactly one epic. Linked from its PR with `Ref` only.

**Precise 1:1 definition (load-bearing for mechanical close):**

- A task is closed by **exactly one finalizing PR**. A task may have multiple PR
  *attempts* (a closed-without-merge PR, then another), but only the PR that
  **finalizes** — merges to `develop` and passes the post-merge hygiene — closes
  it.
- **A PR finalizes exactly one task.** Bundling two tasks into one PR is
  forbidden and rejected by the linkage gate (§4).

There are no standalone tasks: every issue in the formal model is an epic or a
task, and every task has a parent epic. Issues still in *intake* (§3.4) are
exempt until triaged.

**Task granularity (a design value).** Tasks should be small and discrete, for
two reasons: a human reviewer can consume one in a sitting, and an AI agent is
far likelier to get a small, well-scoped task right in one pass. This discipline
is what eventually makes autonomous implementation reachable (a non-goal here).

Text-prefix conventions (`Epic:` / `Epic (standing):` in titles) are retained as
a human scannability affordance. **Labels are the machine source of truth.**

### 3.2 Two-layer placement

**Project layer — the resolved epic home (the roadmap).** A finite epic's *home*
— where its issue and `spec.md`/`plan.md` live — is derived automatically from
repository visibility (`epics.resolve_epic_home`, epic #130):

| `.github` | target repo | epic home |
| --- | --- | --- |
| public | public | `<org>/.github` |
| public | **private** | **the target repo itself** |
| private | (any) | `<org>/.github` |

The default is the org's `.github` repo: a **public** repo's epics live centrally
in **its own org's** `<org>/.github`, so going there shows every active *public*
initiative across that org, regardless of how many of *its* repos the work
touches. Rationale: cross-repo (within-org) work is the practical default — an
epic that *looks* single-repo usually grows a cross-repo change, so uniform
placement avoids a mid-flight migration — and it makes the org's `.github` the
single source for the derived roadmap (§3.7).

A **private** member repo (with a public `.github`) instead **self-homes** its
epics in its own issue list, so nothing about private work — issue titles, spec
text, roadmap rows — ever reaches the public `.github`. Such an epic is
self-contained: its epics, tasks, roadmap, and audit stay scoped to that repo,
and the org-level roadmap omits them by design. The home is chosen automatically
by visibility with no configuration; the creating and reporting commands take an
explicit `--repo` target (defaulting to the current repo) and echo the resolved
home before acting. Epics never span orgs (see non-goals).

**Repo layer — each member repo.** Each repo's issue list becomes a **task
queue**: tasks (1:1 PRs) plus the repo's one standing `Ad-hoc maintenance` epic.
Clean task queues are the substrate for a future capability — an agent that picks
discrete, fully-specified tasks off the queue and implements them — *because the
epic did the planning upfront*.

```text
.github/                         (org repo — project layer)
  epics/
    <N>-<slug>/                  N = epic issue number in .github
      spec.md
      plan.md
  roadmap.md                     (generated; not hand-edited)
  activity-log.md                (generated; not hand-edited)

<member repo>/                   (repo layer)
  docs/specs|plans/              ad-hoc / standing work only
  (issues: tasks + one standing `Ad-hoc maintenance` epic)
```

(A **private** member repo additionally holds its own `epics/<N>-<slug>/` and its
own epic issues — the project layer folded into the repo.)

**Cross-visibility linkage is asymmetric (epic #130).** A task may hard-link
(native sub-issue / `Parent:` reflink) to an epic only if the task's repo is **no
more publicly visible than the epic's home** (order: `PUBLIC` > `INTERNAL` >
`PRIVATE` — the task must be ≤ the home). A private child under a public parent is
fine; a **public** task under a **private** epic is refused by `vrg-epic-link` —
it would leak the private repo's name into a public issue and break
cross-boundary roll-up. Express that dependency instead as a soft
`Blocked-by: <org>/<public-repo>#<N>` line in the **private** epic's body: the
reference lives in the private repo and points *at* a public thing, so nothing
leaks. This generalizes the cross-org non-goal from the org boundary to the
visibility boundary.

### 3.3 Lifecycle and mechanical closing

**Linkage stays `Ref`-only.** Auto-close keywords remain forbidden. A PR merging
is explicitly *not* the close event.

**A task closes programmatically at the end of `vrg-finalize-pr`**, after the
work is safely merged to `develop`, the post-merge workflows reach success, and
hygiene is complete. If a release-step workflow fails after merge, the task stays
open until addressed. Stragglers are the acceptable failure mode; a prematurely
closed issue is not.

**Epic rollup.** Immediately after a child task closes during finalize, the
tooling fetches the parent epic and its children's states and closes the epic
iff: the epic does **not** carry `standing` **and** every child is closed.

**Reopen on late child (symmetric inverse).** Adding a task to an already
**closed** finite epic must **reopen** that epic; when the new child is later
finalized, the rollup closes it again. This is the preferred handling of "a
follow-up to a finished initiative" — the work stays correctly attributed.

**Reopen applies to epics only — never to tasks.** A new problem (including a
revert/fix for something a closed task broke) becomes a **new issue**, typically
a `bug`, optionally reference-linked to the culprit change to preserve the
"this is where we broke it" relationship. Whether to *reopen the origin epic*
(to keep the bug in that initiative's history) or route the bug to a tactical
standing epic is a per-case judgment, intentionally not nailed down.

### 3.4 Intake and triage (the capture net)

The formal path (brainstorm → spec → epic → tasks) is the *preferred* on-ramp
but cannot be the only one. Most issues arrive uncurated — a dog-walk epiphany, a
production emergency, a "we should probably…". The intake must be **lossless**
and lower-friction than formal planning.

**The net is a `triage` label.** Any captured idea/bug/quick-fix becomes a raw
issue labeled `triage`, meaning *"not yet in the formal model"* — explicitly
exempt from the every-task-needs-an-epic invariant. Capture is deliberately
frictionless (e.g. "hey Claude, don't forget X" from a phone) and **distributed**:
seeds land in the repo where they arise by default, or in `.github` when the seed
is inherently project-level or its home is unknown.

**The queue is a label, not a place.** One intake *state* (`triage`), not
separate `inbox`/`seed` states. The "queue" is a **derived cross-repo view** —
`label:triage` queried across all repos and `.github`.

**The queue is worked by a `triage-review` skill**, run on a human-initiated
cadence (~weekly). It collects every `triage` issue project-wide and walks the
human through dispositioning each to exactly one outcome:

1. **Drop** — not a problem / duplicate / wontfix → close.
2. **Assign to an existing finite epic** as a task (prefer-existing-epic).
3. **Route to the repo's standing `Ad-hoc maintenance` epic** as curated small
   work.
4. **Promote to a new finite epic** in `.github` — the on-ramp into a
   brainstorming session.

On disposition the `triage` label is removed and the issue enters the formal
model.

**The emergency path, handled honestly.** A production fix under a gun is
*allowed*: create the task under the standing epic, label it `hotfix`, ship it.
`hotfix` makes the bypass **visible and auditable** rather than hidden, so the
formal path remains the path of least resistance and emergencies get retroactive
scrutiny.

### 3.5 Label namespace

Labels are standardized into orthogonal axes, replacing unused GitHub defaults
(`help wanted`, `question`, `wontfix`, etc.):

| Axis          | Labels                                                     | Notes                                                   |
| ------------- | ---------------------------------------------------------- | ------------------------------------------------------- |
| **Role**      | `epic`, `standing`                                         | Task = *no* role label; `standing` modifies `epic`      |
| **Stage**     | `triage`                                                   | Pre-model intake; removed on disposition                |
| **Kind**      | `bug`, `enhancement`, `docs`, `chore`, `refactor`, `idea` | `idea` = a seed destined to become an epic              |
| **Exception** | `hotfix`                                                   | Emergency path: skipped planning, flagged for review    |

No `status:*` labels. Epic state is open/closed only; active-vs-parked is derived
in reporting (§3.7). A future `deprecated`-style label may be revisited, but is
out of scope now. The final label set is refined during implementation; the
*axes* (role, stage, kind, exception) are the design commitment.

### 3.6 Documentation lifecycle (folds in #714)

Each finite epic **owns its spec and plan**, co-located at a stable path
`.github/epics/<N>-<slug>/`. Repo-local `docs/specs|plans/` are used **only** for
ad-hoc/standing work.

**State is metadata, never encoded in file paths.** The earlier idea of moving
docs between `proposed/`/`in-progress/`/`completed/` subdirectories is rejected:
relocating files churns paths and breaks links. Epic state is the source of
truth and it is binary — **open** (the work is in flight) or **closed**
(complete). The "is it actively worked or parked?" question is answered by the
generated reporting (§3.7), not by a stored attribute.

**Durable-knowledge integration (folds in #714's best idea).** Before a finite
epic is closed, its completion step integrates durable content (design rationale,
interface contracts, decision records) into the relevant published site
documentation. The spec/plan then remain in place as the archived record — the
*knowledge* lives in the site docs, the *record* lives with the epic.

### 3.7 Observability outputs (generated)

A single metadata-driven job — analogous to changelog/release-note generation —
produces two views, both regenerated nightly, neither hand-edited:

- **Roadmap (`roadmap.md` + published project page).** Where we are going.
  Derived from **open** finite epics. Each epic row shows: affected repos, child
  issue counts, **epic creation date**, and the **rolled-up last-activity
  timestamp** of its child issues (optionally broken down per repo). Those
  timestamps make a stored "in progress vs. proposed" status unnecessary — an
  epic created weeks ago with no recent child activity is visibly parked; one
  with fresh activity is visibly in flight.
- **Activity log (`activity-log.md` + page).** Where we have been. A rolling
  ledger of what shipped in the last day / week / month — completed epics,
  finalized PRs, issue churn — rolled up at PR → task → epic level, drillable,
  across every repo. This quantifies throughput that is otherwise invisible.

Both are pure functions of issue/PR metadata. The `.github/epics/` directory is
merely backing store and carries no dynamic state.

### 3.8 Plans evolve append-only

A plan is **frozen at execution start** (when its first task ships).
Mid-execution additions, drops, and rescopes are **not** edited into the planned
task list — that hides how the plan actually evolved. They go in an append-only
`## Evolution during execution` section at the bottom of `plan.md`: dated entries
of *what* changed and *why*. The epic's GitHub sub-issues remain the
authoritative live task list; the addendum carries the *reasoning* for deltas, so
a reader sees what was foreseen up front versus adapted in flight. Log meaningful
deviations only — a new or dropped task, a discovered dependency, a scope shift —
not trivial mechanics. (Surfaced and adopted during epic #45.)

## 4. Mechanics and enforcement

**Parent→child link mechanism (defined abstractly for portability).** The
contract is conceptual: *an epic is an umbrella over tasks that may live in other
repos.* The implementation is swappable:

- **GitHub today:** native cross-repo sub-issues are **supported** within an org
  (verified 2026-06-28: up to 100 sub-issues per parent, 8 nesting levels, with
  parent/sub-issue progress surfaced in Projects). Use them as the primary
  mechanism, queried via the GraphQL `parent`/`subIssues` fields.
- **Portable baseline / future forges (Forgejo, Codeberg):** a structured
  cross-repo reference (`Parent: <org>/.github#<N>`), which we already use
  routinely. The rollup query degrades to this where native sub-issues are
  unavailable.

The rollup logic is written against the abstract relationship so swapping the
mechanism does not re-architect the design.

**Tooling changes (in `vergil-tooling`).**

- `vrg-gh`: sub-issue / cross-repo query support; sub-issue-add with **link-time
  reopen** of a closed parent epic.
- `vrg-finalize-pr`: programmatic task close + **cross-repo** epic rollup into
  `.github`.
- Batch backlog tooling (absorbed from #1641): batch auto-close helpers, orphan
  detection, and **premature-close detection** (verify acceptance criteria were
  actually met; flag/reopen the epic if a task closed wrongly).
- Seed step: create the standardized label set (§3.5) and the per-repo standing
  `Ad-hoc maintenance` epic; retire default label cruft.

**Credential / trust boundary (acknowledged, not re-architected).** The
cross-repo `.github` write performed by the rollup runs inside `vrg-finalize-pr`,
which is **human-side tooling** — agents are never granted issue-close access;
the human (or human-driven automation) runs finalize, often batched. Because the
**org is already the unit of access** for that human-side tooling, the cross-repo
write does not expand the *agent* trust boundary. It is acknowledged as a
human-side, org-scoped responsibility. A least-privilege identity scoped to
`.github` issues remains good practice but is not required to ship.

**Enforcement split — machinery vs. judgment.**

| Rule                                                      | Enforcement                                  |
| -------------------------------------------------------- | -------------------------------------------- |
| PR links exactly one issue, `Ref` only                   | existing `pr-issue-linkage` gate (extend)    |
| PR-linked issue must **not** be an `epic`                | **new** check in the linkage gate/hook       |
| PR `Ref`s **exactly one** non-epic task                  | **new** check in the linkage gate/hook       |
| Task closes only at finalize, never on merge             | `vrg-finalize-pr` (already `Ref`-only)       |
| Epic auto-close / reopen-on-late-child                   | `vrg-finalize-pr` + sub-issue-add tooling    |
| Closed-issue sanity / premature-close detection          | batch backlog tooling + audit reporting      |
| Which epic a task belongs to; when to create an epic     | **human + AI judgment** (not mechanized)     |
| Triage disposition                                       | `triage-review` skill (human-in-the-loop)    |
| Enforcement applies only to **new-taxonomy** issues      | hooks scoped so legacy issues never trip them|

The dividing line: *closing and linking* are deterministic machinery;
*categorization and curation* are human+AI judgment.

## 5. Transition (cutover)

The end state is clean; getting there from the current ~100-issue backlogs is
itself in scope.

- **Cutover principle.** New work follows the model on day one. The existing
  backlog is **grandfathered** and reconciled by the migration workstream below.
- **Enforcement scoping.** Linkage hooks and rules apply only to issues created
  or labeled under the new taxonomy, so legacy issues do not trip them.
- **In-flight work.** Issues mid-implementation at rollout are exempt until
  closed; they may be retro-folded into an epic during their repo's migration.
- **Label migration ordering.** Seed the new labels **additively** first; retire
  old labels only after a repo's migration pass, so nothing is orphaned mid-flight.

**Migration workstream (in scope — one child task per repo).** This brainstorm
is the proof-of-concept instance. After the framework lands, we go **repo by
repo** (~one per evening), each a brainstorm-and-reconcile pass that:

- identifies existing issues that are really **epics** waiting to be labeled, and
  retroactively creates the epics that should have existed for collections of
  related issues;
- identifies **ideas** that need fleshing out, and **issues implicitly part of a
  larger epic**, and folds them in;
- **batch auto-closes** issues that are already done (e.g. the fast patching work
  in the lab repos).

**Done-definition.** This epic is **not closed until every repo is migrated**.
It is therefore long-lived and open-ended by design: as long as one repo's
migration child task is open, the rollup keeps the epic open; repos added later
are handled by reopen-on-late-child. The framework thus dog-foods its own
mechanics during its own rollout.

## 6. Follow-on epic (linked, not folded)

- **Readiness gating** (**#681**) — the spec → pushback → plan → aligned state
  machine that gates implementation. Builds on the epic/task substrate defined
  here; it gates work, it does not define the hierarchy.

(#1641's substance is folded into §4 tooling and §5 migration, not deferred.)

## 7. Feasibility notes for writing-plans

1. **Cross-repo native sub-issues — resolved.** Supported on GitHub today
   (verified 2026-06-28); reference fallback covers future forges. No risk to the
   architecture.
2. **`vrg-gh` cross-repo scope.** The rollup writes to `.github` from a member
   repo's finalize run; extend the wrapper allowlist for the GraphQL +
   cross-repo issue-close calls (human-side tooling — see §4 trust boundary).
3. **`.github` publishing.** Confirm the roadmap/activity-log pages can be
   published from `.github` to the project site.
4. **Label migration tooling.** Idempotent, additive-first seeding across many
   repos (see §5 ordering).

## 8. Out of scope

- Readiness-gating state machine (follow-on, #681).
- Autonomous task-picking / unattended `issue-implement` (future; this design
  only creates the substrate via task-granularity discipline).
- Any stored epic status attribute (derived from reporting instead).
