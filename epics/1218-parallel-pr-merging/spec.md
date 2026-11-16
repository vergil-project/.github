# Parallel PR merging — design

- **Epic:** vergil-project/.github#1218
- **Planning issue:** vergil-project/.github#1213 (review history is in its
  comments)
- **Status:** draft

## 1. Intent

A batch of ready PRs should merge into `develop` in parallel, not one after
another. The human submits the batch once. Every PR opens with GitHub
auto-merge on and lands as soon as its own CI is green. One watcher then waits
for the whole batch, brings the local `develop` up to date, cleans up and
proves that `develop` is still healthy.

What has to stay true is that **released code is fully tested** and that
**`develop` stays stable**. That `develop`'s CI ran on the exact combined
commit before each merge no longer has to be true.

This epic also folds in vergil-project/.github#1184. Finalizing becomes the
default for `vrg-submit-pr`, and installing becomes the default for
`vrg-release`, because those are how both commands are used in practice.

## 2. Why

Today `vrg-submit-pr --all --yes --finalize` is serial. For each branch in
turn it rebases onto `develop`, opens the PR, waits for CI, merges, then moves
to the next. `develop`'s ruleset requires a branch to be up to date before it
merges, so PRs opened side by side go `BEHIND` after every merge and must update
and run CI again. With several epics landing in vergil-tooling at once, this
queues serialized CI runs for an hour or more. On the self-hosted Forgejo
runners, runner capacity is finite, so the repeated CI runs cost real capacity
there too.

The up-to-date rule was kept (org governance spec, "Why 'Require Up to Date' Is
Retained") so that CI tests the exact code that lands on `develop`. The
guarantee that matters is the one about released code, and the release path
already gives it: the `develop` → `main` release PR runs full CI, `main` keeps
the up-to-date rule, and the release enforces CI evidence before publishing.
The combined-change check moves later; it does not disappear.

- A textual conflict still surfaces, at merge time instead of at a pre-submit
  rebase.
- A semantic break between two individually green PRs is caught by the local
  `vrg-validate` that finalize runs on the synced `develop` after the batch
  merges, and in any case before release.
- Changes known to touch the same files are still kept apart by whoever
  assembles the batch, as agents already do.

The CD wait in finalize is relaxed for the same reason. Its blocking check made
sense one PR at a time. In a batch it mostly buys waiting. On
vergil-containers that wait is up to about 80 minutes, behind a 40-minute
publish already in progress. The human has been skipping it with
`--no-wait-cd` to get changes in faster.

## 3. Goals, non-goals, anti-goals

### Goals

1. Develop no longer requires PRs to be up to date before merging, in every
   repo, including the `docs-single-branch` repos that have no `main`. Main
   still does. The daily config audit enforces both, fleet-wide.
2. `vrg-submit-pr` opens every selected PR with auto-merge on, through one code
   path for one PR or many, without rebasing.
3. `vrg-finalize-pr` watches a list of PRs to the end. If any merged, it syncs
   `develop`, cleans up and validates once, and reports every unmerged PR with
   its reason.
4. Concurrent `vrg-finalize-pr` runs on one machine cooperate safely.
5. CD no longer blocks finalize by default, and the release refuses to cut from
   a `develop` whose CD is not green.
6. The #1184 defaults: finalize by default on submit, install by default on
   release.
7. A pr-watch fix to an open PR goes through merge review before it is pushed,
   with a clean rebase exempt.

### Non-goals

- **A merge queue.** The batch-pipeline spec's research still holds: GitHub's
  queue spends more CI, needs a paid plan, and Forgejo has none.
- **Changing who may do what.** `vrg-submit-pr` stays the human's tool.
  Mechanized PRs keep their own automation path. Agents still cannot create,
  edit, merge or close PRs through `vrg-gh`.
- **Attributing a broken `develop` to a batch.** Finalize does not try to work
  out which PR or which concurrent batch broke `develop`; see §6.4.
- **Changing `main`'s rules or the release PR's CI.**
- **Enforcing that a branch is rebased at handoff.** See §6.2.

### Anti-goals (the epic fails if any of these happen)

- A finalize run reports success while the local `develop` it leaves behind
  fails `vrg-validate`.
- Two finalize runs interleave their sync, cleanup or validation and leave a
  checkout or worktree half-updated, or a crashed run's lock blocks every later
  run.
- A chained release (`--release`) runs after a batch that did not fully merge
  or whose validate failed.
- Any release is cut from a `develop` whose CD run failed.
- A failed PR goes unreported, or is reported without the reason it failed.
- A PR that already merged makes finalize fail.

## 4. Current state (facts the design builds on)

Paths are in vergil-tooling unless named otherwise.

- **Rulesets.** `lib/github_config.py` `desired_ci_gates_ruleset` (about
  lines 391–512) builds one "CI gates" ruleset whose `ref_include` covers both
  `refs/heads/main` and `refs/heads/develop`, with
  `strict_required_status_checks_policy: True`.
  - `_stale_classic_ruleset` (about lines 1215–1238) is not a copy of that
    ruleset. It is a per-branch stand-in built from the live state to report
    stale classic-protection checks, with `strict` hard-coded.
  - The audit never reads classic protection's `strict` flag.
  - The "Branch protection" ruleset is also shared, and allows all three merge
    methods.
  - `desired_repo_settings` already sets `allow_auto_merge=True` fleet-wide.
- **`vrg-submit-pr`** (`bin/vrg_submit_pr.py`).
  - There are three entry paths: template mode, `--issue` CLI mode, and relay
    mode (branch positionals). `--all`/`--select` take the serial batch path,
    `_run_submit_batch`, which is the only path that rebases
    (`worktrees.rebase_onto`).
  - `--install` implies `--release`, which implies `--finalize`. Finalize runs
    as a subprocess. A batch runs `vrg-finalize-pr <url> --skip-post-checks`
    once per PR, then one post-checks step and one `vrg-release`.
  - A batch PR (one PR, several `Closes`, one commit per issue) is checked at
    submit by `batch_rules`.
- **`vrg-finalize-pr`** (`bin/vrg_finalize_pr.py`).
  - The stages run in this order: provenance, merge
    (`pr_merge.wait_and_merge`), close-refs, cleanup (switch to `develop`,
    fast-forward, delete branches and worktrees, sweep stragglers), provision,
    validation, cd-check. The release chain runs after them.
  - A comma list or `--all` runs one subprocess per PR, serially and
    fail-fast.
  - It picks the strategy: rebase for a batch PR, merge for `release/*`,
    squash otherwise. Before rebase-merging a batch PR it re-checks one commit
    per issue and the closing references, and merges only at the head it
    checked (`bin/vrg_finalize_pr.py` about lines 1325–1408, epic #1019).
  - Other code calls it too: `vrg-release` runs
    `vrg-finalize-pr --cleanup-only --no-wait-cd` (`lib/release/finalize.py`),
    and `lib/pr_workflow/batch.py` runs `--cleanup-only`. Release's
    update-deps stage also fast-forwards `develop` in the main worktree.
  - It holds no lock; it only has to run from the main worktree. Exit codes
    are 0 and 1.
  - The CD check waits for the run whose head is the merge commit (120 s for it
    to register, then up to `$VRG_CD_TIMEOUT`, default 5400 s). A
    concurrency-supersede cancel is a non-fatal warning. `--no-wait-cd` opts
    out.
- **`lib/pr_merge.py`.** `wait_and_merge` and `wait_until_merged` handle
  `BEHIND` with update-branch (capped at 5), wait for required checks to
  register, and pin batch merges to the head commit. Its callers are finalize,
  `lib/release/merge.py` (release PRs, strategy `merge`, including the merge
  into `main`), `lib/update_deps/pr.py` (strategy `merge`, base `develop`) and
  `lib/mechanized_pr.py`.
- **`lib/pr_provenance.py`.** It classifies who acted on the PR. Its only
  caller is finalize's provenance stage, with `--allow-provenance-violation`.
- **`lib/mechanized_pr.py`.** Opens the PR and enables squash auto-merge
  through GraphQL `enablePullRequestAutoMerge` with `expectedHeadOid`, reads
  the setting back, then waits in `pr_merge.wait_until_merged`. It is the only
  auto-merge user today.
- **Locks.** The only lock in the tooling is `fcntl.flock` in
  `lib/trivy_scan.py`.
- **`vrg-release`.**
  - Preflight refuses when local `develop` differs from `origin/develop`
    (`lib/release/preflight.py` about line 640).
  - `confirm-develop` (`lib/release/confirm.py`) watches the develop CD run of
    the back-merge *after* the release. It treats a develop docs-publish
    failure as deferred, not fatal.
  - Nothing checks develop's CD *before* the release is cut.
- **CD workflows.** vergil-containers' `cd.yml` runs on pushes to `develop` and
  `main`. On `develop` it deploys docs and publishes the `dev-` images; the
  release job is `main`-only.
  - `cd-docker-publish.yml` has the concurrency group `docker-publish-<prefix>`
    with `cancel-in-progress: false`. That group is shared with the nightly
    no-cache rebuilds (`ops.yml`). Its comment records why: with cancelling
    on, a cancelled publish was never retried (vergil-containers#879, #820),
    and two publishes of one prefix race on its tags.
  - GitHub keeps at most one running and one pending run per group, so the
    queue never exceeds two.
  - vergil-actions' `cd`, `cd-release` and `cd-docs` groups are shared by
    `develop` and `main`.
  - The containers PR CI (`ci-images.yml`) already builds, scans and
    smoke-tests every affected image on both architectures without pushing.
- **Dev images.** The fleet's CI defaults to `prod-` images. Containers' own PR
  CI builds a child FROM the published `dev-` parent when the parent is not
  rebuilt. The dev images are rebuilt nightly.
- **pr-watch** (vergil-claude-plugin `skills/pr-watch`) already treats a
  conflict with an advanced `develop` as routine: rebase and force-push.
  **merge-review** (`skills/merge-review`) runs only inside `issue-implement`,
  on a task branch before report-ready.

## 5. Approaches considered

- **A. Auto-merge at submit; finalize only watches.** (Chosen.) Merging happens
  on GitHub's side whether or not anything runs locally. It reuses
  `mechanized_pr`'s auto-merge code, and it ports to Forgejo, which has
  auto-merge.
- **B. Finalize does the merging, in parallel.** Submit opens the PRs; finalize
  polls them all and merges each the moment it goes green. Nothing lands unless
  a human-run finalize is alive, and we keep owning the merge and retry code.
  Rejected: the human does not need merges tied to a running process (§6.3).
- **C. Keep the serial pipeline, minus the rebase.** Smallest change, but each
  PR still waits for the one before it to merge. Rejected: it misses the point.

For CD, three options were weighed:

- keep blocking, on the final head only;
- stop blocking by default, move the gate to release, and let `develop` CD
  cancel stale runs;
- (chosen) the same, but leave the concurrency settings alone.

Cancelling was first chosen, then dropped. It would bring back the lost
publishes recorded in vergil-containers#879 and #820. It would let a develop
push kill the nightly rebuild or, through vergil-actions' shared groups, a
`main` release. Once finalize no longer waits on CD, it gains nothing either:
the queue is already capped at two.

## 6. Design

### 6.1 Rulesets (vergil-tooling)

- **Split "CI gates" into two rulesets**, one per branch, and record each name
  in `github_config.py`.
  - The **develop** ruleset requires the same status checks with
    `strict_required_status_checks_policy: False`.
  - The **main** ruleset keeps `True`.
- **The rule applies to every repo's `develop`,** `docs-single-branch` repos
  included. Those repos have no release path, so the combined-commit check is
  removed for them rather than moved later. Finalize's post-merge validate
  still covers them.
- **Classic protection.** The audit reads classic branch protection's `strict`
  flag on `develop`, reports `strict: true` as drift and clears it. Otherwise
  a repo with classic protection would keep requiring up-to-date branches.
  `_stale_classic_ruleset` is already per branch and stays as it is.
- **Audit migration.** The config audit's apply must replace the old shared
  ruleset with the pair. That means creating the two new rulesets and deleting
  the old one, without a window where either branch has no required checks.
  The audit's drift report must name the change.
- **No other ruleset changes.** The "Branch protection" ruleset is unchanged.
  `allow_auto_merge=True` is already set.

This lands and is applied fleet-wide **before** anything that relies on it
(§6.12). Auto-merge never updates a branch that is `BEHIND`, so under the old
rule an auto-merged PR would wait forever.

### 6.2 `vrg-submit-pr` (vergil-tooling)

- **One path.** Template mode, `--issue` mode, relay mode, and `--all`/
  `--select` all reach the same submit loop. For each selected PR it:
  1. applies today's ready gate and freeze check unchanged;
  2. pushes the branch;
  3. opens the PR;
  4. turns auto-merge on, pinned to the pushed head (`expectedHeadOid`), with
     the merge method: **rebase** for a batch PR, **squash** otherwise;
  5. records the submission.

  `release/*` PRs are not submitted through `vrg-submit-pr` and are
  unaffected.
- **No rebase at submit.** `_run_submit_batch`'s rebase, and the
  merge-commit refusal that supports it, are removed. Rebasing and validating
  before handoff stays the agent's job, as today, but it is not enforced at
  handoff. `develop` moves between report-ready and submit anyway, so such a
  check would prove little. A conflict surfaces when the PR tries to merge, and
  finalize reports it (§6.3).
- **Auto-merge is shared code.** The enable-and-read-back logic moves out of
  `mechanized_pr` into one helper that both use. Enabling auto-merge can fail
  in two ways:
  - **GitHub refuses because the PR is already mergeable.** The helper merges
    it at once with the same method and head pin.
  - **Any other refusal.** The helper fails the submit for that PR, loudly,
    naming the PR. The other PRs in the selection continue, and the summary
    lists the failure.
- **The batch-PR check moves earlier.** Finalize no longer re-checks a batch
  PR (one commit per issue, closing references) at merge time. The check now
  runs in two places:
  - at submit, as today;
  - in pr-watch, before any push to a batch PR (§6.8).

  The auto-merge head pin covers only the head submitted. A later push is
  covered by pr-watch's check.
- **Flags (#1184).**
  - **Default: finalize.** After opening every PR, submit runs **one**
    `vrg-finalize-pr <PR>,<PR>,…` over all of them and passes its exit status
    through.
  - **`--no-finalize`.** Opens every PR with auto-merge on and returns without
    watching. It prints the exact `vrg-finalize-pr <PR>,<PR>,…` command that
    would have watched them. The PRs still merge on their own.
  - **`--finalize` and `--install` are removed.** Passing either is a fatal
    error that says what is now the default.
  - **`--release` stays opt-in.** No flag implies it. It is passed to finalize
    and runs only on full success (§6.3). `--release` with `--no-finalize` is
    rejected.
  - **`--no-install`** is passed through to `vrg-release`. Without `--release`
    it is an error: `--no-install requires --release`.
  - **`--yes` and `--dry-run`** keep their meaning. The single up-front plan
    and confirmation from the batch-pipeline design remain.

### 6.3 `vrg-finalize-pr` (vergil-tooling)

- **Input.** A comma list of PR numbers or URLs, `--all` (every open PR whose
  branch has a worktree in `.worktrees/`), or the interactive pick. One PR is a
  list of one; there is one path.
- **Watch phase** (no lock held). The run polls every listed PR, in parallel,
  until each reaches a terminal state:
  - **merged:** this includes a PR that was already merged when finalize
    started, which is not an error;
  - **failed: CI**, naming the required check that failed;
  - **failed: conflict**: the PR can no longer merge cleanly;
  - **failed: closed** without merging;
  - **failed: auto-merge off**: the PR is open and auto-merge is no longer
    enabled;
  - **timed out:** the batch deadline passed before the PR reached any of the
    states above.

  The batch deadline covers the whole watch phase. It defaults to 2 hours and
  can be overridden with an environment variable, which must be a positive
  whole number. A malformed value is a configuration error, as with
  `$VRG_CD_TIMEOUT`.
  - **Auto-merge stays on for a failed PR.** An agent fixes it, CI goes green
    and it lands by itself; the human then reruns finalize, which counts it as
    merged.
  - **Restoring auto-merge.** When finalize starts, any listed PR that is open
    with auto-merge off has it turned back on through the shared helper
    (§6.2). If GitHub reports the PR already mergeable, the helper merges it
    directly. Finalize is run by the human, so this is a human merge. Only a
    PR whose auto-merge goes off *during* the watch is reported as *failed:
    auto-merge off*.
- **Settle phase.** This runs only if at least one PR merged. Steps 1–5 hold
  the lock (§6.4); steps 6 and 7 run after it is released. In order:
  1. **close-refs** for each merged PR, as today;
  2. **sync:** switch the main worktree to `develop` and fast-forward it to
     `origin/develop`, which picks up everything merged by anyone;
  3. **cleanup** of the merged PRs' branches and worktrees, plus today's
     straggler sweep;
  4. **provision**, as today;
  5. **validate:** one `vrg-container-run -- vrg-validate` on the `develop`
     head;
  6. **CD report** (§6.5);
  7. **release**, if `--release` was given and the run is a full success. A
     full success means every PR merged, the validate is green and, with
     `--wait-cd`, CD is green. `--no-install` is passed through.
     - The release syncs again and cuts from the current `develop` head. That
       head may include merges that landed after this run's validate.
     - That is accepted. Finalize's validate checks the health of `develop`.
       It is not what guarantees the release: the release PR into `main` runs
       full CI, `main` keeps the up-to-date rule, and the release enforces CI
       evidence.
     - The release is not pinned to the validated SHA, and it does not
       validate again.

  If no PR merged, the settle phase is skipped entirely and the run goes
  straight to the report.
- **Report.** The report lists every PR under *merged* or *not merged (reason)*.
  It states the validate result and the CD run's link and status. It names
  what to do next for each unmerged PR: hand it to an agent, then rerun
  finalize on it.
- **Exit status.**

  | Code | Meaning |
  | --- | --- |
  | 0 | Full success. |
  | 1 | A tool or usage error, as today. |
  | 3 | Develop is healthy, but at least one PR did not merge. |
  | 4 | **Develop is broken:** the validate failed. Precedes 3. |
  | 5 | Full success, but the chained release failed. |

  With `--wait-cd`, a failed or timed-out CD run is also 4. A broken `develop`
  stops everything until it is fixed. The report says so loudly, and gives the
  `develop` head SHA and the list of commits the sync brought in.
- **Flags removed:**
  - `--allow-provenance-violation`;
  - `--skip-post-checks`, which existed only for the per-PR subprocess loop
    that goes away;
  - `--strategy`, because submit chooses the merge method;
  - finalize's own `--install`, which is a fatal error, as on the other
    commands.
- **Flags added:** `--no-install`, passed through to the chained release.
- **Kept:** `--cleanup-only`, which takes the lock for its sync and cleanup
  (§6.4).

### 6.4 Concurrent finalize runs (vergil-tooling)

- **The lock.** Several finalize runs may run at once on one machine, one per
  batch. Their watch phases run fully in parallel. Their settle phases are
  serialized by an exclusive `fcntl.flock` on a lock file under the repository
  clone's `build/.vergil-state/`, one lock per clone.
  - The OS releases the lock when a process exits, so a crashed run never
    wedges later runs.
  - A run waiting for the lock says so, with the holder's PID if the lock file
    records it.
  - **The lock covers only local checkout work:** the sync, cleanup, provision
    and validate. It never covers waiting on GitHub, a CD run or a release.
  - **The lock is not re-entrant, and nothing nests under it.** Every command
    that syncs or cleans up the main worktree takes it for just that section:
    finalize's settle steps, `vrg-finalize-pr --cleanup-only` (including when
    `vrg-release` calls it), and release's update-deps fast-forward. A chained
    release starts only after finalize has released the lock, so release's own
    use of it cannot deadlock.
  - The wait has no timeout of its own. Every section that holds the lock is
    bounded by its own timeouts.
- **What is validated.** A run validates whatever `develop` head it synced,
  including other batches' merges. That is deliberate. The question finalize
  answers is "is `develop` healthy?", not "did these commits pass?".
  - A second run that gets the lock after the first fast-forwards again and
    validates the new head. When nothing has landed in between, that validate
    repeats on the same head, and that cost is accepted. Finalize does not
    cache results by SHA.
  - **No attribution.** A validate failure means `develop` is broken (§6.3,
    exit 4), whoever caused it. Finalize does not try to pin it on a batch.
- **One lock helper.** The lock helper is shared, not a copy of trivy's. If
  `trivy_scan` can use it unchanged, it does; otherwise it keeps its own.

### 6.5 CD (vergil-tooling and the CD workflows)

- **Finalize reports, and waits only with `--wait-cd`.**
  - **The default.** After the validate, finalize looks up the develop CD run
    for the synced head. It prints the run's link and its status at that moment
    (queued, in progress, succeeded, failed, cancelled or superseded) and does
    not wait.
  - **`--wait-cd`** restores today's blocking watch, on that one run, with the
    supersede handling unchanged. It is for work where the publish itself is
    being checked.
  - **No CD run registered** for the head within the existing 120 s dispatch
    window is reported as "no CD run for this head", as today.
  - **`--no-wait-cd` is removed.** Passing it is a fatal error that says not
    waiting is now the default. `vrg-release`'s own call to
    `vrg-finalize-pr --cleanup-only --no-wait-cd` drops the flag.
- **The CD workflows do not change.** Concurrency stays
  `cancel-in-progress: false` everywhere (§4, §5).
  - Because finalize no longer waits, the queue of at most two runs blocks
    nobody.
  - The accepted cost: after a burst of merges, the `dev-` images can lag the
    `develop` head by up to one publish.
- **Epic rule.** An epic that depends on freshly published `dev-` images states
  it in its plan and carries an explicit validation task for the CD build.

### 6.6 `vrg-release` (vergil-tooling)

- **The develop-CD gate (new).**
  - Before cutting a release, `vrg-release` checks the develop CD run for the
    `develop` head it is releasing, the same way finalize finds it.
  - **A run in progress** is waited on, bounded by `$VRG_CD_TIMEOUT`.
  - **A run that failed or timed out** stops the release, with the run's link.
  - **What counts as failed.** The gate classifies jobs the way
    `confirm-develop` does. A failure that `confirm-develop` defers, such as a
    develop docs-publish failure, is a loud warning, not a stop. Finalize's
    `--wait-cd` uses the same classification.
  - **No run for that head:** the release uses the latest completed develop CD
    run whose head is an ancestor of the release head, and requires it to be
    green.
  - **A run superseded** by a newer pending run, which GitHub cancels before it
    starts, is skipped in favour of the newer run. Any other cancellation fails
    the gate.
- **Install by default (#1184).**
  - After the release, the project's consumer install step runs if the project
    defines one.
  - `--no-install` skips it.
  - Passing `--install` is a fatal error that says install is now the default.

This change is independent of the merge work and is its own task.

### 6.7 Removals (vergil-tooling)

- **Provenance.** `lib/pr_provenance.py`, finalize's provenance stage,
  `--allow-provenance-violation` and their tests are removed. Approvals do not
  gate merges (develop requires none), auto-merge performs the merge, and
  `vrg-gh` already stops agents from creating, editing, merging or closing PRs.
- **The merge path in finalize.** Finalize no longer calls
  `pr_merge.wait_and_merge` or `_resolve_strategy`. Submit picks the method
  instead (§6.2).
- **The submit-time rebase** (§6.2) is removed, along with the per-PR finalize
  subprocess loop and its `--skip-post-checks` plumbing in both commands.
- **`lib/pr_merge.py` stays whole.** Its `BEHIND` and update-branch handling
  still matters for the release PR into `main`, which keeps the up-to-date
  rule. For `develop` it is harmless: once the rule is gone the path does not
  fire. This keeps vergil-project/.github#1168 (upstream-refresh PRs that
  merge themselves, open) compatible: its mechanized PRs wait through
  `wait_until_merged` unchanged. `mechanized_pr` changes only to use the shared
  auto-merge helper (§6.2). Its docstring is updated so it no longer says
  develop requires up-to-date branches.

### 6.8 pr-watch and merge review (vergil-claude-plugin)

- **Routine rebase.** A PR that conflicts because another PR merged first is
  routine: rebase onto `origin/develop` and force-push. This is pr-watch's
  existing rule, kept and restated for the auto-merge flow. A clean rebase that
  resolves no conflict changes no code of the branch's own, and needs no
  review.
- **Batch PRs.** Before any push to a batch PR, pr-watch runs the batch-PR
  check (one commit per issue, closing references, the `batch_rules` check
  submit uses), and refuses to push on a violation. A clean rebase is checked
  too.
- **Every other fix is merge-reviewed before it is pushed.** That covers a
  resolved conflict, a CI fix or any code change. merge-review gains an entry
  for a fix to an open PR, with the same reviewer, outcomes and ruling flow as
  today, scoped to the fix on top of the reviewed head.
- **Why it matters.** Auto-merge stays on, so a pushed fix that turns CI green
  merges without a further human step. The merge review is the check before
  that happens.
- **Finalize failure handoff.** pr-watch, and the finalize report it reads, use
  the new unmerged reasons (§6.3). After a fix, pr-watch tells the human to
  rerun `vrg-finalize-pr <PR>`, not to merge.

### 6.9 Specs superseded (vergil-tooling docs)

- **Org governance spec.** `docs/specs/2026-05-11-org-governance-design.md`,
  "Why 'Require Up to Date' Is Retained", gets a dated supersession note
  pointing at this spec. Its "Rules for `develop`" list drops the up-to-date
  line. Its risks-table line about merge serialization is also updated.
- **Batch-pipeline spec.** `docs/specs/2026-06-17-batch-pr-pipeline-design.md`
  gets a dated supersession note on its zero-waste serial design. Its merge
  queue research still stands.
- **Epic #1168's spec.** That spec states that develop requires up-to-date
  checks. The note goes in this epic's retrospective, not as an edit to that
  spec.

### 6.10 Callers, docs and messages

Every caller that passes a removed flag is updated, so that none hits the new
fatal errors. The removed flags are:

- `--finalize` and `--install` on submit;
- `--install` on release;
- `--install`, `--no-wait-cd`, `--skip-post-checks`, `--strategy` and
  `--allow-provenance-violation` on finalize.

The callers to update are:

- the code itself: `lib/release/finalize.py`'s `--no-wait-cd`, and
  `lib/pr_workflow/batch.py`;
- skills in vergil-claude-plugin;
- hook messages;
- docs;
- `vrg-submit-pr`'s printed hints. A search for each flag across the fleet
finds them. The site docs sweep is the documentation-review task's job
(vergil-project/vergil-tooling#4529).

### 6.11 Fixture runs

The change reaches release tooling (`vrg-release`'s default) and the PR flow
every fixture uses, so the plan seeds a fixture run on every language fixture.
Each is blocked by the vergil-tooling release that ships the change.

### 6.12 Rollout order

1. **The ruleset split (§6.1)** is released and applied fleet-wide by the
   config audit, and confirmed live on `develop` and `main`.
2. **The submit and finalize rework (§6.2–§6.4, §6.7), the release gate (§6.6)
   and the pr-watch and merge-review change (§6.8)**, released together. They
   need step 1 live: an auto-merged PR under the old rule waits forever at
   `BEHIND`.
3. **A live batch validation:** a batch of at least three independent PRs in
   one repo, submitted with the released tooling, merges without any `BEHIND`
   update. One finalize settles it, and a second finalize run on another batch
   at the same time waits for the lock and then validates.

## 7. Failure model

- **A PR's CI fails.** That PR is reported *failed: CI*, and the others are
  still watched. Exit 3, unless develop is broken.
- **A PR conflicts after another merges.** It is reported *failed: conflict*.
  pr-watch rebases and pushes, and the human reruns finalize.
- **Auto-merge cannot be turned on at submit.** That PR's submit fails loudly,
  the rest continue, and the summary names it.
- **The batch deadline passes.** Pending PRs are reported *timed out*, and
  their auto-merge stays on. The settle phase runs for any that merged.
- **A PR merged before finalize ran.** It counts as merged.
- **The validate fails.** Exit 4: develop is broken, and there is no release.
  The report gives the head SHA and the commits brought in.
- **A finalize run crashes while holding the lock.** The OS frees the lock,
  and the next run proceeds and syncs again.
- **Develop CD fails.** Finalize reports it, and with `--wait-cd` exits 4. The
  next `vrg-release` refuses to cut.
- **A pending develop CD run is replaced by a newer one.** It is reported as
  superseded, never as a failure.
- **The chained release fails.** Exit 5. The batch's merges and validate
  stand.
- **Develop moves between the validate and a chained release.** The release
  cuts from the newer head. The release PR's full CI covers it.
- **A malformed deadline or CD timeout variable.** A configuration error,
  raised before any waiting.

No failure is swallowed. Every unmerged PR, every skipped release and every
CD state that is not green is named in the output.

## 8. Constraints

- **Wrappers.** Use `vrg-git` and `vrg-gh` only. Run every check through
  `vrg-container-run`.
- **Forge-agnostic.** Auto-merge with a head pin and a merge method must go
  through the existing GitHub abstraction, so a Forgejo transport can supply
  the same operation later.
- **Human actions stay human.** PR submission, merge (by enabling auto-merge)
  and release are human actions. Agents never run `vrg-submit-pr` or
  `vrg-finalize-pr`.
- **Correctness first.** No known-broken intermediate state ships. That is why
  the rollout order in §6.12 is binding.

## 9. Related work

- **vergil-project/.github#1184.** Folded in and closed.
- **vergil-project/.github#1168** (upstream-refresh PRs merge themselves, open).
  Kept compatible by §6.7. Its mechanized PRs gain nothing and lose nothing.
- **vergil-project/.github#1088** (fixture-scoped agent rights). It asked for a
  per-repo lock for agent releases. §6.4's lock helper may serve it later.
- **The batch-pipeline design** (vergil-tooling#1673) and **the org governance
  design.** Superseded in part by §6.9.

## 10. Left to the builder

- **The exact names** of the split rulesets and of the deadline environment
  variable.
- **How the audit swaps the ruleset** (create both, then delete the shared one)
  without a gap in required checks.
- **Recognizing an "already mergeable" refusal.** The exact way GitHub's
  GraphQL API signals that it refused auto-merge because the PR can merge now,
  and the direct merge that follows.
- **Auto-merge after a push.** Confirm whether GitHub keeps auto-merge enabled
  when a write-access account pushes to the PR. The design assumes it does;
  test it during the live validation. If it does not, finalize's restore step
  (§6.3) recovers it on the human's next run.
- **The poll interval** for the watch phase, within GitHub's API rate limits
  for a batch of about ten PRs.
- **Where the shared auto-merge and lock helpers live** in `lib/`.
