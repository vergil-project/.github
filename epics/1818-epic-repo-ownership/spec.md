# epic-repo-ownership: epic ownership of repos for coordinated releases — design

- **Epic:** vergil-project/.github#1818
- **Planning issue:** vergil-project/.github#1678 (review history is in its
  comments)
- **Status:** draft

## 1. Intent

Today the human is three rubber stamps in the fleet: the release lock (no
merges while a release runs), the release-block manager (deciding what ships
together) and the only identity that runs `vrg-release`. Three epics
mechanize those stamps so that agents can, later, run their own releases:

1. **The release lock** (vergil-project/.github#1664): a running release
   holds back merges into `develop`.
2. **Batches as the unit of release** (vergil-project/.github#1672): tasks
   that must ship together in one repo are one batch PR, so that submitting a
   PR asserts it is ready to merge **and** ready to release at any time.
3. **Epic ownership of repos** (this epic): the exception to that invariant.
   When work in one repo must not release until its partner work in another
   repo is ready, the epic takes **ownership** of the repos involved, and
   only it decides when they release.

The goal beyond the three is an epic that is planned with `epic-create`,
started with `epic-implement` and a single confirmation, and next heard from
when its retrospective asks for approval, with the human interrupted only for
rulings, outages and triage. Agent-run releases are the next milestone after
these three; this epic builds the mechanism they will rest on, and stops
short of granting them.

A common epic changes vergil-tooling and vergil-claude-plugin together, or
vergil-tooling and vergil-actions; an epic for another org's product can
need a vergil-tooling change in the same effort. Each repo releases on its
own, so without coordination a release can ship half of an epic's work, and
two epics can race for one repo.

## 2. Goals, non-goals and anti-goals

### 2.1 Goals

- An epic can take, show and give up ownership of a group of repos, across
  orgs, with a tool whose operations are idempotent and safe to re-run.
- While an epic owns a repo, no release of it runs except one made for that
  epic.
- The decision that work needs ownership is made at planning time, as a
  planned task the human reviews, never inferred at run time.
- Ownership is held only for the window it protects: from the submission of
  the group's first batch to the release of its last repo.
- An epic that cannot take ownership blocks and says which epic holds which
  repo; it never holds some repos while waiting on others.
- Every take, give-up and break is recorded; nothing is overridden silently.
- The design leaves agent-run releases a small follow-on change.

### 2.2 Non-goals

- **Agent-run releases.** `vrg-release` stays human only. Which permissions
  change when agents release, and where the agent–human boundary sits, is a
  separate epic of its own.
- **Enforcing a release order in the tooling.** While the human runs every
  release, the agent states any order and the human carries it out. A
  mechanism for it (such as `vrg-release` waiting on another task's
  release) is designed with agent-run releases.
- **Releasing ad-hoc work** (an agent asking the human for permission to run
  a release when the fleet is quiet). It needs agent-run releases; it goes to
  that epic.
- **Ownership gating merges.** Other epics keep merging into an owned repo;
  their work, releasable by the invariant, goes out in the owner's release.
  The release lock stays the only thing that holds back merges.
- **Atomic acquisition across repos.** Each repo is its own remote, groups
  can span orgs, and cross-repo groups are rare enough that the human's
  oversight of the rare half-taken group is accepted (§4.2).
- **Ownership of other admin operations.** Likely later; the record is
  named for repos, not releases, so that it can grow.
- **Automatic reclaim of stale ownership.**

### 2.3 Anti-goals

- Any step that still needs the human for something the tooling could
  decide.
- Run-time judgment, in skill prose, of whether work must ship together.
- Ownership held for a whole epic's life, locking the human out of the repo
  for days.
- A holder that waits while holding, so that two epics can deadlock.
- An ownership record that silently disappears or is silently overridden.

## 3. Current state

- **`vrg-release`** is human only and fail-closed against every agent
  identity and signal (`bin/vrg_release.py`). It takes no epic argument.
- **The release lock** (epic #1664, approved, being built) adds
  `refs/vergil/release-lock`, taken by a create-only push
  (`--force-with-lease=<ref>:`) and updated by compare-and-swap, with a
  history ref, a human-only `break --reason`, a `status` any identity can
  run, "a ref that cannot be read is held" and `vrg-git` refusing manual
  pushes to its refs. Its pre-check (#1664 spec §5.3, step 1.1) runs before
  it takes the lock.
- **Other `refs/vergil/*` refs** (the report-ready relay, ruling and scope
  drafts) are force-pushed one-file commits, handled in vergil-tooling's
  `src/vergil_tooling/lib/pr_workflow/github_transport.py` and
  `src/vergil_tooling/lib/rulings/draft_ref.py`.
- **Operational task kinds** are `validation` and `deployment`, each with a
  label, a scaffold and a run skill (`issue-validate`, `issue-deploy`), not
  PR-workable, closed only on `Outcome: SUCCESS`. A new kind must join every
  place in vergil-tooling that lists the kinds or keys a map by them:
  `_OPERATIONAL_KINDS`, the `--kind` choices and the scaffold maps
  `_SCAFFOLD_INTRO` and `_SLOT_PLACEHOLDERS` in `bin/vrg_issue_create.py`,
  with a per-kind body template in `data/` beside
  `deployment_task_body.md` and `validation_task_body.md`;
  `_OPERATIONAL_KINDS` in `lib/queue_view.py`; `_OPERATIONAL_LABELS` in
  `lib/epics.py` (the check `vrg-submit-pr`, `report-ready` and the batch
  rules use for "not PR-workable"); the refusal text in
  `lib/pr_workflow/batch_rules.py` that names the operational kinds; and
  `data/labels.json`.
- **`vrg-release-record check <task>`** proves that a task's change is in a
  release, from its `Released:` record cross-checked against git ancestry.
- **Batch PRs** shipped with epic #1019 (`lib/pr_workflow/batch_rules.py`):
  one PR, one commit per task, for 2 to 8 issues of one epic in one repo.
  Epic #1672, being planned in parallel, makes the batch the unit of
  release. A repo's share of a coordinated group may be one task, so this
  design speaks of the group's PRs, single or batch.
- **Proxy epics** let one primary epic file tasks in other orgs.

## 4. Approaches

### 4.1 Where the decision to take ownership lives

1. **A planned coordinated release task (chosen).** The plan names each
   group of batches across repos that must ship together and files it as a
   new operational task kind; `epic-implement` runs it, and ownership is held
   only for its window. The decision is reviewed in plan pushback, where it
   is cheapest to correct, and agent-run releases change only one of its
   steps.
2. **`epic-implement` infers it** from cross-repo dependencies at run time.
   Less planning, but it is run-time judgment in skill prose, and a wrong
   guess ships half an epic. Rejected.
3. **Own the repos for the epic's whole life.** Simplest, but it locks the
   human's releases out for days and starves other epics. Rejected.

### 4.2 Where ownership is recorded

1. **A ref in each repo (chosen; the human's direction).** `vrg-release`
   reads one ref in the repo it releases, groups work across orgs, and the
   release lock's transport and rules are reused. A group take is not one
   atomic write, so a crash partway can leave a group half-taken; a re-run or
   a break repairs it, the history records it, and the human, overseeing a
   handful of epics at once, resolves the rare case. Cross-repo atomicity is
   judged not worth its cost at the fleet's scale.
2. **One record in the org's `.github`** listing owned repos. A single
   compare-and-swap would make a group take atomic, but only within one org,
   and epics span orgs (an epic for another org's product that needs a
   vergil-tooling change). `vrg-release` would read another repo's ref, and
   every take in an org would contend on one record. Rejected.
3. **Issue labels or fields.** Not atomic, writable by anyone with triage,
   and a release would have to search issues. Rejected.

### 4.3 How a group is taken

1. **All or nothing, in a fixed order (chosen).** Take the repos in sorted
   order; on the first one held by another task, give back those just taken
   and report. No holder ever waits holding, so there is no deadlock; a large
   group can starve, which the human resolves.
2. **Hold what is taken, in a fixed order, and wait.** Avoids starvation and
   avoids deadlock, but holds repos idle while waiting, blocking the human's
   and other epics' releases. Rejected.

## 5. Design

### 5.1 The ownership ref

- One ref per repo, `refs/vergil/repo-owner`, on the repo's origin, separate
  from the release lock. It points at a one-file commit whose JSON document
  carries at least: the owning epic (the primary epic's full ref), the
  coordinated release task's full ref, the group (every repo the task
  takes, as full names, in take order), the run that took it (§5.5: the
  identity, the host and the run's session state directory), and the time
  taken.
- **Taking it** is a create-only push. A failed push means it is held: read
  it.
- **Giving it up** and **breaking it** are compare-and-swap deletes against
  the value read, so a give-up never removes a record it did not read.
- **Reading it** takes it as written, with no trusted-author check, as the
  release lock does (#1664 spec §5.10). A ref that cannot be read or parsed
  is treated as held and reported, never as absent.
- **The history** is a second ref, `refs/vergil/repo-owner-history`: one
  commit per take, give-up and break, carrying the record, who acted, when,
  and for a break its reason. It is appended by compare-and-swap.
- Only the ownership tool writes these refs: `vrg-git` refuses a manual push
  to either, a delete included, as it does for the release lock's.
- **The ref primitives are generalized from the release lock's.** The
  release lock's module (`lib/release_lock/`, #1664 plan Task 17) is tied
  to `refs/vergil/release-lock` and records only overrides in its history.
  This epic factors ref-agnostic helpers out of it: a create-only push, a
  compare-and-swap update and delete, and an appended history whose
  actions each caller defines (here `take`, `give-up` and `break`, every
  one recorded). The release lock then uses the same helpers, and the
  ownership module joins the push guard's allowed sites
  (`test_branch_push_guard.py`). This is one task, blocked by #1664's Task
  17; #1664's own plan is unchanged.
- **Transport: each repo through its own main clone.** The tool reads and
  writes a repo's ownership refs from that repo's main clone, the
  convention `epic-implement` already uses for task repos, across orgs
  included. Every
  repo of a group must have a main clone; a missing one stops the
  operation, naming the repo, before anything is taken.

### 5.2 The ownership tool

A new command, named here `vrg-repo-owner` (the builder may rename it), with
four operations:

- **`take --task <ref> --repo <owner>/<repo> [--repo …]`.** Takes every
  named repo for the task, in sorted order of full name, binding each record
  to the calling run. A repo whose ref already names this task, with the
  same group and the same run, counts as taken (a re-run adopts it). On the
  first repo held by anything else (another task, this task with a
  different group or another run, or an unreadable ref) it gives back, by
  compare-and-swap, every repo whose ref is bound to this run, those adopted
  from an earlier attempt included, so a re-run never holds some repos
  while waiting on others. It never gives back a record bound to another
  run. This is safe because `take` runs only before any PR of the group is
  submitted (§5.5, steps 1–2); a resumed run past that point never runs it.
  It reports each kind of hold machine-readably, on its own exit code or a
  `held_by` field of its `--json` output: held by another task, held by
  another run of this same task, or unreadable, naming for each held repo
  its owning epic, task and run. The run reads that kind, never the text:
  it waits only on a hold by another task (§5.5, step 2), and stops for the
  human on every other kind.
  A failure while giving back is reported, naming the repos still held,
  never swallowed.
- **`status [--repo …]`.** Prints each repo's owner (epic, task, group,
  taker, age) or that it is unowned, and whether the ownership is stale
  (§5.6), as text or `--json`. Any identity may run it. With no `--repo` it
  covers the fleet's repos, as hoshin and the audit need.
- **`give-up --task <ref> [--repo …]`.** Deletes each ref that names the
  task; a repo whose ref names another task is refused, never deleted.
  Only the run the record is bound to gives it up: `give-up` refuses any
  other caller, naming the bound run and pointing to `break`, by the same
  rule `take` applies when it adopts. Every give-up is recorded in the
  history with its caller. With
  no `--repo`, it gives up every repo of the task's group. A repo already
  unowned counts as given up.
- **`break --repo <owner>/<repo> --reason "<why>"`.** The human's escape
  hatch, for a lost session or abandoned work: human only, refused under
  every agent identity and signal, as `vrg-release` is. It records the old
  record and the reason in the history, comments on the owning task, then
  deletes the ref.

The user agent may run `take`, `status` and `give-up`; it never breaks
ownership.

### 5.3 `vrg-release` checks ownership

`vrg-release` gains `--epic <ref>` and `--task <ref>`, given together, and an
ownership check. An epic can hold several coordinated release tasks, so the
check matches the owning task as well as the epic. It runs twice:
first in its pre-check, before it takes the release lock, so a release that
cannot run fails fast; and again once the release lock has disarmed merges
(#1664 spec §5.3, after step 1.4), when `develop` can no longer move. The
second check is the one that guards the cut: ownership can be taken, and an
owned batch merged, while the pre-check waits on a develop CD run. If the
second check fails, the release aborts through the release lock's full
unlock path. Each check applies these rules:

- **With `--epic X --task T`:** the release runs only if the repo's
  ownership names epic X and task T, and aborts otherwise (unowned, owned by
  another epic or task, or unreadable), telling the human to reconcile with
  the owner.
- **Without them:** the release runs as today when the repo is unowned,
  and fails, naming the owning epic and task, when it is owned or its ref
  cannot be read.
- It never takes, changes or ends ownership. A release does not end it.
- The checks run on a fresh release and on `--supersede`, which is a new
  release. A `--resume` finishes a release that already passed them and does
  not check again, so a break during a running release does not stop that
  release; the two refs stay independent.

`vrg-release` stays human only. For an agent-run release later, the
authorization rule is already fixed: an agent may release only a repo that
the coordinated release task it is running owns.

### 5.4 The coordinated release task

A new operational task kind, `coordinated-release` (the builder may rename
it), with its own label, scaffold and run skill (named here
`issue-coordinated-release`). It shares every operational-task rule: not
PR-workable, closed only on `Outcome: SUCCESS` recorded as a comment, an
open child gating its epic's rollup.

- **Its group** is the plan's PRs, one in each repo, a batch or a single
  task (by plan task ID in the plan). Work in one repo that must ship with
  the group, or that builds on another of the group's tasks in that repo,
  is one batch, so no PR of the group is written on top of another
  unmerged one; plan pushback refuses a group with two PRs in one repo.
  Batches are planned in
  `epic-create` (§6); `epic-implement` may change them when implementation
  calls for it.
- **The `Coordinated release:` lines are the only record of the group.**
  Each task in the group carries a `Coordinated release: <ref>` line,
  naming the coordinated release task that submits it. The tooling finds
  the group by reading those lines across the primary epic's tasks and its
  proxies' tasks, as `vrg-epic-audit` lists them, never from the plan's
  text or the task's body, so a re-batch keeps them in step; the run and
  `status` show the group so found. The line is read through the trust
  layer wherever it is read (the run, `issue-implement`, `vrg-submit-pr`,
  `vrg-epic-audit`), as `Blocked-by:` lines are, and untrusted content
  fails closed.
- **The task's body carries only the release order**, keyed by repo, where
  there is one. The releases of a group normally run in parallel; an order
  (a plugin release that would break current users if it went out before
  the tooling release, say) should be rare, and the plan states it only
  where it is real.
- **It is filed in the primary epic's home** (`<org>/.github`), since it
  has no PR and may span orgs.
- **It carries no `Blocked-by:` link to a task of its own group**: the
  `Coordinated release:` lines are that link, and its group's tasks close
  only inside its window. `vrg-issue-create` and `vrg-epic-audit` refuse
  such a link. It may be blocked by tasks outside its group, such as a
  deployment.
- **A batch task in the group is done at report-ready.** For
  `epic-implement`, report-ready is its hand-off to the coordinated release
  task: `issue-implement` stops it there with a named outcome (held for
  coordinated release `<ref>`), which `epic-implement` treats as the task's
  gate passed, not a stop. The issue itself still closes when its PR
  merges, inside the coordinated release task's window.
- **A `Blocked-by:` link between two tasks of one group is met at
  report-ready.** A task blocked by a task of its own group (a plugin task
  that uses a new tooling command, say) becomes runnable once its blocker
  is recorded report-ready, not once it closes. `vrg-epic-audit`'s frontier
  applies this rule, reading both tasks' `Coordinated release:` lines; no
  skill decides it.
- **On re-entry, a group's reported-ready task is handed off.**
  `epic-implement` treats a reported-ready task that carries a
  `Coordinated release:` line as handed off to that coordinated release
  task: it never submits it, whatever its re-entry rule for other
  reported-ready tasks says, and shows it under that task's state. It
  reads the line, not a refusal from `vrg-submit-pr`.
- **The coordinated release task is runnable** once every batch of its
  group is recorded report-ready, read from the tooling's records (the
  report-ready relay), not from the batches' open or closed state.
  `vrg-epic-audit` reports it runnable or blocked on that rule.
- **Downstream work that needs the group released** is blocked by the
  coordinated release task, not by the batches.

### 5.5 Running it

A **run** is one execution of the run skill on one coordinated release task,
through the steps below. A **resumed run** is the run skill taking up the
same task again after its session stopped partway (a cleared context, a
crash, a lost VM, or the human stopping it), whether `epic-implement` or
the human invokes it.

A resumed run first asks whether it can pick the task up. It can only while
no PR of the group has been submitted: steps 1 and 2 are then re-run from
the start, and nothing has shipped. Once any PR of the group has been
submitted, a resumed run does not continue. It reports what it finds (the
repos owned and by which run, each PR's state, open, merged or released,
and the ownership history) and stops for the human, with ownership left as
it is. Failures inside the window are too varied to enumerate; each is an
exception the human and the agent resolve together, using the tools as the
case needs (a break, a release, an emergency patch), and this design does
not script them.

1. **Preconditions.** Every task of the group is implemented and recorded
   report-ready, none has been submitted, and each has a local, frozen
   worktree at its recorded head in its repo's main clone, from which the
   user agent can submit it. A missing worktree, or a PR already
   submitted, stops the run for the human before anything is taken.
2. **Take ownership** of every repo in the group (§5.2). On a hold by
   another task, the run waits, like a process blocked on a lock; on any
   other kind of hold it stops for the human:
   - it says so where the human will see it, in its session and in a
     comment on the task, naming which epic and task hold which repo;
   - it sleeps and retries the whole `take`, on the order of every few
     minutes, never in a tight loop, holding nothing between attempts;
   - when the repos are free it takes them and goes on. Two waiting epics
     race, and the first `take` to succeed wins; the other keeps waiting.

   The wait needs no human and runs unattended; the human watching may
   intervene at any time. A holder that is stale (§5.6) cannot end the wait
   by itself, so the run reports it as such, and the human breaks it. How
   the wait is built (a blocking mode of `take`, a monitoring subagent, a
   skill to block on) is left to the builder.

   Once the group is taken, the run records each task's `Coordinated
   release:` line as it read it, and refuses to submit a PR whose line has
   changed since.
3. **Submit the group's PRs** with `vrg-submit-pr` and watch each to its merge,
   as `issue-implement` does. A batch that cannot merge is fixed forward
   with `pr-watch`, as any submitted PR is, while ownership stays held, so
   a partner batch that has already merged cannot release alone. A failure
   that cannot be fixed forward stops the run for the human with ownership
   still held: it is an exception the human and the agent resolve together,
   out of band, and this design does not script it.
4. **Release each repo.** For now, hand the human the paste-ready
   `vrg-release --epic <epic ref> --task <task ref>` command for each repo, as a human-run
   step, stating any release order the group names; the human carries the
   order out, and the tooling does not check it. Later, when agents run
   releases, this step changes, and so does step 5, whose release records
   count today only when the human role wrote them.
5. **Confirm the releases** with `vrg-release-record check` on every task
   of the group, exit 0 for each. The tooling's record, not the human's
   word, proves the release.
6. **Give up ownership** of the group, then record `Outcome: SUCCESS` and
   close the task. Before giving up, the run reads the group again; if it
   differs from the group recorded at take time (a task added to it or
   removed since), it stops for the human with ownership still held.

**Submission is guarded in the tooling, not only in prose.**
`issue-implement` stops a task carrying a `Coordinated release:` line at
report-ready, and `vrg-submit-pr` refuses to submit it unless the repo is
owned by the task that line names. The guard covers every PR, single or
batch: a PR with any member carrying the line submits only inside the window, and
tasks that came along with it, releasable by the invariant, ship in the
owner's release. So a batch cannot leave the window early, whoever runs the
submit.

### 5.6 Visibility, staleness and audit

- **Stale ownership** is ownership whose coordinated release task or epic is
  closed. Ownership spans an interactive session that can last days, so it
  has no heartbeat, and nothing reclaims it: the human breaks it.
- **`vrg-repo-status`** shows the repo's owner alongside the release lock.
- **Hoshin's daily review** lists every owned repo in the fleet with its
  epic, task and age, flagging stale ownership.
- **The daily agent audit** reports stale ownership and unreadable
  ownership refs, report-only, never removing either.

### 5.7 Planning guidance

`epic-create`'s plan step gains the rule:

- When the plan has PRs in different repos that must ship together, it
  adds a coordinated release task for each such group, naming its PRs and
  any release order between its repos, and lists it like any other
  operational task, with its plan task ID.
- A step that happens outside the tooling (something the human does in
  the physical world) is planned as its own task, which the human attests
  and closes, ordered with `Blocked-by:` like any other. Ownership has no
  part in it.
- Most epics need none: their batches are releasable whenever they merge.
- Plan pushback holds each group to the smallest set of repos that must
  ship together.

`epic-implement` runs coordinated release tasks with their run skill, like
the other operational kinds.

## 6. Constraints

- Builds on the release lock's ref module (#1664, generalized as §5.1
  says) and on batches
  (#1672); tasks that need either are blocked by the tasks that deliver it,
  and only those. The ownership ref, the tool and `vrg-release`'s check
  need only #1664; the task kind and its run skill, the submit guard and
  the planning rule also need #1672.
- Relies on batches being planned in `epic-create`'s plan, with
  `epic-implement` free to adjust them as implementation requires: a change
  to #1672's design, coordinated with its planning session. The three epics
  are built in series (#1664, then #1672, then this one), so #1672's spec
  is published before this epic's tasks are filed. This epic's plan is not
  filed until that spec records planned batches; if it does not, this
  spec's group source is revised first.
- Git and GitHub access goes through the tooling's sanctioned paths; the
  ownership refs use the helpers generalized from the release lock's
  (§5.1).
- `vrg-release` and `vrg-repo-owner break` stay human only.
- No silent failures: an unreadable ownership ref is held, not absent; a
  failed give-back or give-up is reported, naming what is still held.
- Ownership never holds back merges, and never interacts with the release
  lock beyond `vrg-release` reading both (§5.3).

## 7. Left to the builder

- The tool's, task kind's and run skill's final names.
- How the wait on `held` is built, and its retry interval (§5.5).
- How the run identity in the record is formed and compared (§5.1, §5.2).
- Exit codes, within §5.2's rule that each kind of hold is told apart
  machine-readably.
- The exact JSON fields of the ownership record and history commits.
- How `status` enumerates the fleet's repos.
- How the coordinated release task's body records its release order,
  within the operational scaffold's sections.

## 8. Delivery shape

One epic, in phases, since neither half is useful alone: first the
ownership ref, the tool and `vrg-release`'s checks; then the task kind, its
run skill and the skill changes; then visibility (`vrg-repo-status`,
hoshin and the audit).
