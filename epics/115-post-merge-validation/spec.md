# Post-merge validation follow-on tasks

**Status:** approved design (brainstorm output).
**Epic:** `vergil-project/.github#115`
**Date:** 2026-07-07
**Origin:** `vergil-project/.github#114`; the **originating incident** was a lab
epic where a verification task auto-closed on its runbook PR (`.github#550`,
re-gated by hand). That is history that motivated the pattern — a bespoke
anecdote, not the template. This spec designs the feature **generically**, tied
to no particular target.

## Problem

One-PR-one-task auto-close closes an implementation task the moment its code PR
merges. But some tasks' true acceptance can only be confirmed **after** merge —
a cold VM rebuild, a live-lab check, a deploy smoke test. Today those tasks
close (and the epic rollup reads "done") before the validation is ever run. We
have already been bitten: a lab epic auto-closed a verification task on a
*documentation* PR, so the actual cold-build verification was never gated and
the epic could roll up as complete without anyone running the checklist.

The fix is to make post-merge validation a first-class, gating part of the
epic/task framework, with the judgment of when to apply it built into the
planning and implementation skills.

## Concept and vocabulary

A **validation task** is a first-class task type whose acceptance is proven by
*running* a validation and recording the result as an issue **comment** — not by
merging a code PR. It re-establishes the acceptance gate that one-PR-one-task
auto-close removes for work whose real acceptance is only observable
post-merge / post-deploy.

The mental model: a validation task behaves like a pull request that cannot be
merged until it is green. Passing closes it; failing holds it open.

**Incidental benefit — forensic history.** Because each run records its PASS/FAIL
and evidence as issue comments, validation tasks also accrue as durable
forensic/audit data we can mine later. Where and how we extract that is not yet
decided; for now the value is simply in *capturing* it, which this pattern does
by construction.

## Invariants

1. **One-PR-one-task is unchanged.** A validation task is not a code task and is
   **not PR-workable** — it has no code PR, the PR tooling refuses to operate on
   it (see Tooling), and it never auto-closes.
2. It **closes only** when the validation is executed and a PASS result is
   recorded as a comment, then the task is closed (by agent or human).
3. It links to its dependency implementation task(s) as a **blocked-by**
   dependency (merge-first) — via GitHub's native issue-dependency if our App
   token can write it, otherwise a `Blocked-by: #N` body reflink (the same
   fallback pattern `lib/epics.py` already uses for sub-issues on forges without
   native support). A feasibility spike (Task 1) decides which; the concept and
   the runnable-vs-blocked rollup are identical either way — only the storage
   differs.
4. It **gates epic rollup** simply by staying open — the existing auto-close
   rollup already holds the epic open until every sub-issue closes.
5. Its body is **self-contained and executable**: concrete commands plus a
   PASS/FAIL results template, so an agent (including cloud/CI agents that
   cannot easily push code) can run it and record the result without touching
   code.

## Granularity and gating

A validation task can gate at any level, chosen by agent judgment:

- **1:1** — validate one task before starting the next.
- **N:1 (epoch)** — one validation over a group of tasks, `blocked-by` all of
  them.
- **Epic-level closing bookend** — a final "validate the deployed system,"
  joining the existing documentation-review bookend as a closing gate.

## Judgment doctrine — when to add one, and where it is discovered

**Add** a validation follow-on when acceptance requires a cold rebuild, a
live-lab check, or a deploy smoke test — i.e. the real acceptance path cannot be
exercised by pipeline unit/integration tests. Infra/provisioning-shaped epics
carry a cold-rebuild validation by default.

**Do not add** one for pure documentation, or for code fully covered by pipeline
tests, where merge equals done.

The concept is caught at **two distinct entry points**, which is why two skills
must understand it:

- **Planning time (`epic-create`).** While decomposing the work, the agent can
  already see which tasks span a batch-validation opportunity and seed the
  epoch validation up front.
- **Implementation time (`issue-implement`).** The agent discovers mid-build
  that pipeline tests genuinely cannot prove the change works — it must be
  merged and deployed first. That discovery is only visible while implementing,
  so the skill must be able to mint a validation follow-on on the spot and
  refuse to declare the task/epic done while it is outstanding.

## Validation task anatomy

The issue body is a self-contained, executable scaffold with these sections:

1. **Preconditions (self-check).** The task **declares its own preconditions**
   in its body, and the runner checks them *first*. If any precondition is
   unmet, the runner records *"blocked: preconditions not met"* and stops — it
   never fabricates or partially fakes results. That behavioral rule is the only
   firm requirement here.
   The precondition set is **generic and author-defined; the framework does not
   prescribe a mechanism.** A precondition may be a machine-checkable probe (a
   health/status command, a check that the dependency change is actually
   deployed) *or*, commonly today, a **human-attested statement** ("the target
   has been rebuilt to include #N"). Expressing preconditions in a uniform,
   universally machine-checkable form is deliberately **left open** — that shape
   is not settled, still leans on the human interface, and this spec does not
   force it. Keep this section generic: no tool, environment, or git-vs-deploy
   mechanism is baked into the framework.
2. **Commands to run** — the concrete validation checklist.
3. **Acceptance criteria** — explicit pass/fail conditions.
4. **Results template** — PASS/FAIL plus evidence, to be posted as a comment.

Symmetry worth noting: `blocked-by` is the *issue-graph* gate; the preconditions
self-check is the *runtime* gate for the same dependency.

## FAIL semantics

- **PASS** → record the result comment, close the task.
- **FAIL** → the task **stays open**. Record the failure evidence as a comment,
  file follow-on fix task(s), and leave the epic open. A gate that opens on
  failure is not a gate; this mirrors a PR that cannot merge.

## Provisioning boundary (human-driven now)

Some validation targets require **heavy, out-of-band setup** — a full
environment rebuild or provision — before the check can run. For now that setup
**remains human-driven and deliberately un-mechanized**, sometimes necessarily
so (a rebuild can tear down the very session that would run it). This is a
generic boundary, not tied to any particular target.

Concretely, for now:

- There is **no provisioning task and no `blocked-by` to one** in this epic.
- The human performs the setup out-of-band and the validation **assumes** it was
  done.
- `issue-validate` then runs the task's declared precondition self-check before
  the checklist — which today is frequently a human-attested precondition (see
  anatomy). The framework does not prescribe how that check is mechanized.

**Forward-looking (documented, not built).** A mechanized version would model
the setup as its own "stand-up" task that the validation is `blocked-by`; the
`blocked-by` model and the runnable-vs-blocked rollup below already accommodate
it when we are ready. That is a later epic — the shape of the generic
machine-checkable precondition is not settled, and this epic does not force it.

## Tooling changes (`vergil-tooling`)

First-class validation task type.

1. **`vrg-issue-create` gains a validation path** (`--kind validation`):
   - applies the `validation` label (idempotent via `vrg-ensure-label`),
   - sets `blocked-by` links to the named dependency task(s),
   - links the task under its epic (`--epic`),
   - stamps the standard executable body scaffold (preconditions self-check,
     commands, acceptance criteria, results template).
2. **Validation tasks are not PR-workable** — the direct fix for the
   `.github#114` failure, and simpler than guarding the close keyword.
   `vrg-submit-pr` **and** `vrg-pr-workflow report-ready` hard-refuse a task
   carrying the `validation` label: no PR development against a validation task,
   period. Because no PR ever references a validation task, **both auto-close
   vectors are moot by construction** — `vrg-submit-pr`'s `Closes` keyword has
   no PR to attach to, and the scheduled `ops-epic-sweep`
   (`vrg-epic-audit --close`) closes tasks only from its `task_drift` set
   ("merged PRs whose Ref'd task is still open"), which a PR-less validation
   task can never enter. Belt-and-suspenders: `vrg-epic-audit`'s report-only
   invariant checks flag any validation task found *closed without* a recorded
   PASS result comment (catches a human closing one by hand).
3. **Validation-aware rollup/audit** — `vrg-epic-rollup` / `vrg-epic-audit`
   report an epic as **"code-complete, validation-pending"** when all code
   tasks are closed but validations remain open, and list each outstanding
   validation as **runnable** (all dependencies closed) vs **blocked**, reading
   whichever dependency storage the Task 1 spike selects (native issue
   dependency or the `Blocked-by:` reflink). This runnable-vs-blocked worklist
   is precisely what a future automator would poll, so building it now gives the
   machine-readable signal for free.

## Skills (`vergil-project/vergil-claude-plugin`)

- **`epic-create`** — extend the bookend doctrine with the validation task type;
  default cold-rebuild-for-infra seeding; batch/epoch identification; document
  the validation task anatomy and the judgment doctrine.
- **`issue-implement`** — *discover-and-create* during coding; **never execute**;
  refuse "done" while a validation is outstanding. Add a **wrong-skill
  redirect**: on picking up a `validation`-labeled issue, detect it and redirect
  ("this is a validation task — use `issue-validate`"), killing the observed
  confusion at the door.
- **`issue-validate`** *(new)* — the execution lifecycle, deliberately split from
  `issue-implement` because it has a different goal, workspace, terminal state,
  and failure handling (execute a recorded procedure vs. write code; no
  worktree/PR; comment-and-close vs. PR handoff; record-and-file-fix vs.
  fix-in-place). Flow:
  1. **Reachability / precondition gate** — run the body's self-check
     (lab-up + commit-floor). If unmet: record *"blocked: preconditions not
     met"* and stop. Never fabricate.
  2. **Run** the recorded commands.
  3. **Record** PASS/FAIL with evidence as an issue comment.
  4. **PASS → close.** **FAIL → stay open**, file follow-on fix task(s), leave
     the epic open.

## Automation trajectory (non-goal for this epic)

Longer term, *where you validate* can differ from *where you implement* (one
current example: implement on macOS, run the check on a cloud x86 VM — an
illustration, not a design constraint this epic depends on). The end state is an
automator that sits on the epic and
notices validations moving from blocked to runnable as their dependencies close,
then executes the self-contained body wherever it can reach the target. This
epic does not build that runner, but three of its pieces make the runner
possible later and are therefore built now: (a) `blocked-by` is machine-readable
and load-bearing, (b) the validation body carries preconditions/prep, not just
assertions, and (c) rollup/audit already computes runnable-vs-blocked.

## Repos touched (epic shape)

- **`vergil-tooling`** — `vrg-issue-create` validation path, `vrg-submit-pr`
  anti-close guard, `vrg-epic-rollup` / `vrg-epic-audit` validation-awareness.
- **`vergil-project/vergil-claude-plugin`** — `epic-create`, `issue-implement`,
  and the new `issue-validate` skill.
- **`vergil-project/.github`** — the epic issue and its `spec.md` / `plan.md`;
  the bookend and validation tasks whose PRs (or comments) land there.

## Task shape and sizing

The plan sizes tasks by true weight, not as equal bullets:

- **Task 1 — `blocked-by` feasibility spike** (gates the storage decision):
  determine whether GitHub's native issue-dependency is writable/readable via
  our App token; if not, adopt the `Blocked-by: #N` body-reflink fallback. Its
  outcome is a prerequisite for the rollup work.
- **Heavier tasks** — the runnable-vs-blocked rollup/audit logic, and the new
  **`issue-validate`** skill, which gets its own design section in the plan
  (novel lifecycle: precondition gate, reachability, run, record, FAIL-stays-open,
  fix-task filing).
- **Lighter tasks** — the `validation` label + body scaffold in
  `vrg-issue-create`, the PR-workability guard, and the `issue-implement`
  discover/redirect doctrine.

## Testing

- **Tooling (`vergil-tooling`, unit tests, existing patterns):**
  - `vrg-issue-create` validation path — asserts the `validation` label, the
    body scaffold sections, and the dependency link (native or `Blocked-by:`
    reflink, per the Task 1 spike).
  - PR-workability guard — asserts `vrg-submit-pr` **and**
    `vrg-pr-workflow report-ready` both refuse a `validation`-labeled task.
  - rollup/audit — asserts "code-complete, validation-pending" state and the
    runnable-vs-blocked classification; asserts `vrg-epic-rollup` holds an epic
    **open** while a validation child is open; asserts the report-only invariant
    flags a validation task closed without a PASS comment.
- **Skills:** prose doctrine. This epic's own validation is a **lab-free tooling
  dogfood** (plan Task 10) — it exercises the PR-workability guard,
  `--kind validation`, and the runnable-vs-blocked rollup end-to-end with no live
  target. The lab incident (`.github#550`) is the originating anecdote, not a
  required dependency for closing this epic.

## Non-goals

- Mechanizing or formalizing the human-driven lab rebuild interface.
- Building the future validation automator/runner.
- Any change to one-PR-one-task or the existing auto-close rollup.
