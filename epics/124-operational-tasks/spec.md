# Deployment tasks and the operational-task family

**Status:** approved design (brainstorm output).
**Epic:** `vergil-project/.github#124`
**Date:** 2026-07-08
**Origin:** follow-on of epic `vergil-project/.github#115` (post-merge
validation). The motivating instance: the #115 dogfood needed a manual
release + `uv tool install` + `vrg-ensure-label --sync` before its validation
could run — that manual step *is* a deployment task.

## Problem

Epic #115 shipped the **validation task** — a not-PR-workable task whose
acceptance is proven by running a check and recording the result as a comment.
But there is a step that frequently sits *between* an implementation task (code
merged to develop) and a validation task: **deployment** — actually taking the
merged changes and releasing/installing/deploying them so the new functionality
is usable in the environment. Today that step is invisible to the framework: an
epic cannot express "the next task can't run until the previous changes are
**deployed**, not merely merged." That gap makes the interdependencies between
tasks impossible to reason about — or eventually automate.

## Concept — the operational-task family

Generalize the not-PR-workable machinery from #115 into a shared **operational
task** family: a task whose acceptance is proven by *running a procedure* and
recording an `Outcome:` comment, not by merging a PR. It has two **kinds** now,
and the design is deliberately open to more:

- **`validation`** — *verify* that prior work is correct (shipped in #115).
- **`deployment`** *(new)* — the active act of taking merged PRs and
  releasing/installing/deploying them so the new functionality is usable.

All operational kinds share one mechanism; each kind supplies only its own
label, scaffold, run skill, and semantics.

## Merged-vs-deployed becomes graph structure

The key payoff. A deployment task, like validation, closes only when it
*succeeds* — so **the deployment task's closure is the canonical "it's deployed"
signal**:

```text
impl task(s) ──Blocked-by──▶ deployment task ──Blocked-by──▶ validation / next impl
   merged ✓                    deployed ✓ (= task closed)        runs against deployed
```

- A **deployment task** is `Blocked-by` its implementation tasks → runnable once
  they are **merged** (an impl task closes when its PR merges).
- A downstream task that needs the thing **deployed** is `Blocked-by` the
  **deployment task** → runnable once deployment **closes**. One that needs it
  only **merged** stays `Blocked-by` the impl task.

The existing `Blocked-by` + `all_blockers_closed` machinery computes runnability
unchanged — no new precondition mechanism. This turns "is it deployed?" from a
human-attested precondition into structure, and so **partially delivers idea
#123** (machine-checkable preconditions) by making deployed-ness a closeable
task rather than an attestation.

## Shared operational-task invariants

Both kinds (and any future kind):

1. **Not PR-workable.** No code PR; the PR tooling (`vrg-submit-pr`,
   `vrg-pr-workflow report-ready`) refuses an operational-labelled task. It is
   *run* by its run skill, not implemented.
2. **Closes only on success**, recorded as an `Outcome: SUCCESS` comment. On
   failure it stays open.
3. **Gates the epic** by staying open — an ordinary open child holds the epic
   until it succeeds.
4. **Records dependencies as `Blocked-by:` reflinks** (merge-first / deploy-first),
   which the audit reads to report each as runnable vs blocked.
5. **Self-contained executable scaffold**: author-defined preconditions
   self-check, the procedure, acceptance criteria, and an `Outcome:` results
   template. The **framework prescribes no mechanism** — the deploy commands
   (release, install, label sync, …) are author-specified, exactly as validation
   kept its precondition mechanism generic.

## Result marker

The machine-read result is unified across all operational kinds — the *kind* is
already known from the label, so the word needn't re-encode it:

- **`Outcome: SUCCESS`** or **`Outcome: FAILURE`**.
- The audit's success invariant recognizes the legacy `Outcome: PASS` as an
  alias, so validation tasks closed before this epic (e.g. the #115 dogfood
  #120) are not falsely flagged.

## Deployment-specific semantics

Deployment differs from validation in what it *does* and how it fails:

- **It changes state** (releases/installs/deploys), where validation observes.
- **It is idempotent and retriable.** A failure is often transient (network, a
  not-yet-ready target).
- **Failure handling — retry first.** On failure the deployment task stays open
  (gating the epic and any downstream validation, exactly like validation). But
  the `issue-deploy` skill's failure path **leads with retrying** the idempotent
  deployment; only if it cannot succeed without a code change does it file a fix
  task (an impl task), then re-deploy once that lands. It closes only on
  `Outcome: SUCCESS`.

## Tooling changes (`vergil-tooling`) — generalize validation → operational

1. **Label.** Add a `deployment` label to the registry (description ≤ 100 chars —
   heeding the #2203 lesson). `validation` stays.
2. **`vrg-issue-create`.** `--kind {task,validation,deployment}`;
   `_render_validation_body` becomes `_render_operational_body(kind)` selecting a
   per-kind scaffold (`validation_task_body.md` + a new `deployment_task_body.md`);
   applies the right label and renders `Blocked-by:` reflinks.
3. **Guard (generalize).** `epics.is_validation_task` → `is_operational_task`
   (carries the validation *or* deployment label; `is_validation` / `is_deployment`
   are the per-label predicates). The two guards (`_reject_if_validation_task` →
   `_reject_if_operational_task` in `vrg-submit-pr`; `_reject_validation_issue` →
   `_reject_operational_issue` in `vrg-pr-workflow`) refuse any operational task,
   naming the kind in the message.
4. **`Blocked-by` plumbing — unchanged.** `render_blocked_by` / `blockers_of` /
   `all_blockers_closed` are already generic and are reused as-is.
5. **Audit (generalize).** `validation_status` / `validation_pending` /
   `closed_validation_without_pass` → `operational_*`, each tagging kind; the
   success invariant reads `Outcome: (SUCCESS|PASS)` (PASS = legacy alias). The
   `vrg-epic-audit` report's "Validation pending" section becomes
   "Operational tasks pending" and shows each item's kind.
6. **Marker migration.** The validation scaffold and `issue-validate` record
   `Outcome: SUCCESS/FAILURE` going forward; the legacy-alias audit covers
   already-closed validations.

## Skills (`vergil-project/vergil-claude-plugin`)

- **`issue-deploy`** *(new)* — the deployment run skill: preflight (USER +
  deployment label) → preconditions gate (block, don't fabricate) → run the
  recorded deploy commands → record `Outcome:` → **close on SUCCESS / retry-first,
  then fix-task-if-needed, on FAILURE**. Emphasizes idempotency, retriability,
  and "you *are* changing the environment."
- **Shared operational-task lifecycle** — the common lifecycle (preflight →
  preconditions gate → run → record → close-on-success / hold-open-on-failure)
  lives in **one** referenced place that both `issue-validate` and `issue-deploy`
  point to, so N kinds cost N thin skills, not N copies of the doctrine. Exact
  home (a shared reference file vs. an `epic-create` section) is settled in the
  plan.
- **`epic-create`** — generalize its "Validation tasks" section to **"Operational
  tasks"** (validation + deployment): the impl→deploy→validate ordering, when to
  add a deployment task (the next step needs it *deployed*, not just *merged*),
  and creation via `--kind deployment`.
- **`issue-implement`** — broaden the wrong-skill redirect: any operational-labelled
  issue routes to its run skill (validation → `issue-validate`, deployment →
  `issue-deploy`); the discover-and-create rule extends to minting a deployment
  task when a step must be deployed before the next can run or be validated.

## Docs

Generalize the GitHub Issue Standards site doc's "Validation tasks" section to
**"Operational tasks"**, covering both kinds and the impl→deploy→validate model
(this is the doc-review closing bookend's authoring target).

## Repos touched (epic shape)

- **`vergil-tooling`** — label, `--kind` path + deployment scaffold, guard
  generalization, audit generalization, marker migration.
- **`vergil-project/vergil-claude-plugin`** — the new `issue-deploy` skill, the
  shared lifecycle, and the `epic-create` / `issue-implement` doctrine updates.
- **`vergil-project/.github`** — the epic issue and its `spec.md` / `plan.md`;
  the bookend tasks.

## Testing

- **Tooling (unit tests, existing patterns):** the generalized guard/audit/creation;
  a **regression pass** proving the validation→operational rename does not change
  the just-shipped validation behavior; the deployment scaffold; the marker
  migration and legacy-`PASS` recognition.
- **Skills:** prose doctrine; dogfooded with a real deployment task in the
  framework (the impl→deploy→validate chain exercised end-to-end).

## Relationship to deferred ideas

- **Idea #123 (machine-checkable preconditions)** — partially delivered:
  deployed-ness becomes a closeable task, removing one common human-attested
  precondition. #123 may shrink as a result.
- **Idea #122 (validation automator)** — advanced: the automator's
  runnable-vs-blocked worklist now includes deployment tasks, so it could run
  deploys as their deps land, not just validations.

## Non-goals

- Building the validation/deployment automator (#122).
- A general machine-checkable precondition standard (#123) beyond what
  deployed-as-a-task delivers incidentally.
- An open plugin framework for arbitrary operational kinds — the design supports
  a third kind cheaply, but we scope this epic to validation + deployment.
- Prescribing a deploy mechanism — deploy commands stay author-specified.
