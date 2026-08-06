# Enforce no-optional PR status checks + orphaned-check resilience

**Epic:** vergil-project/.github#224
**Home:** vergil-project/.github
**Status:** Spec (pushback-resolved 2026-08-06)

## 1. Summary

Two independent bugs share one theme — *how we manage the status checks that gate
a PR merge*:

1. **Optional PR gates are structurally possible.** `vrg-github-repo-config`
   derives each repo's required-status-checks ruleset from a **hand-maintained
   enumeration** in `desired_ci_gates_ruleset()` (`github_config.py`). Any check
   the reusable CI emits that is not typed into that function is **silently
   optional**: it runs, it can fail, and it never blocks a merge. `docs / docs`
   is exactly this — a universal CI check that was never added, so a broken docs
   build could merge unnoticed.

2. **`vrg-finalize-pr` can hang forever on a GitHub-orphaned check.** Its wait
   loop calls `gh pr checks --watch`, which blocks until *every* check reaches a
   terminal state — required or not. When GitHub leaves a check-run non-terminal
   after its workflow run has already completed (an **orphaned check-run**),
   finalize blocks indefinitely even though GitHub itself considers the PR
   mergeable.

This epic makes an optional PR gate impossible (enforced by the config tool) and
makes finalize resilient to an orphaned check (raise loudly, never hang, never
merge past it).

### The incident that surfaced it

`vrg-finalize-pr 934` on `logical-minds-foundry/mq-resiliency-lab-for-linux` sat
on "Waiting for checks on 934..." indefinitely. Evidence:

- `mergeable: MERGEABLE`, `mergeStateStatus: CLEAN` — GitHub says merge it.
- Every check `SUCCESS` **except** `docs / docs` = `IN_PROGRESS` (`pending`).
- Workflow run `31101639453`: `status: completed`, `conclusion: success` — the
  run finished, yet its `docs / docs` job was still `in_progress`. A completed
  run cannot have a genuinely-executing job → the check-run was **orphaned**.
- `docs / docs` is **not a required check** (proof: a pending *required* check
  forces `BLOCKED`, not `CLEAN`), so GitHub's merge gate ignored it — but
  `gh pr checks --watch` did not.

The two root causes are orthogonal: making `docs / docs` required would have
turned this into a permanent `BLOCKED` instead of a hang. Both must be fixed.

## 2. Invariants

- **I1 — No optional PR gate.** Every check that can appear on a `pull_request`
  is a required status check. If we are unwilling to block on a check, we do not
  run it on PRs. Enforced by `vrg-github-repo-config`, not by discipline.
- **I2 — Finalize never hangs on, and never merges past, a non-terminal check.**
  A check that is genuinely still running is waited on; a check that GitHub has
  orphaned (non-terminal after its run completed) is surfaced as a loud error.

*Scope of I1:* PR-gating checks only. Post-merge workflows (publish, Docker image
builds on push-to-develop/main, releases) never produce a `pull_request`
check-run, so they are excluded by construction — no special-casing.

*Assumption (boundary):* reusable-CI PR checks are **unconditional** — every job
runs on every PR (verified: `docs`/`quality`/`security`/`version` in
`vergil-actions` carry no `if:`/`paths` guard). **Path-filtered per-PR required
checks are out of scope**: they would both deadlock GitHub's merge gate (a
required-but-skipped check stays "expected" forever) and confuse empirical
observation, and would need separate handling. Documented here so a future path
filter is a conscious decision, not a silent break.

## 3. Task A — No optional PR gates (config side)

Home of the fix: `src/vergil_tooling/lib/github_config.py` and
`src/vergil_tooling/bin/vrg_github_repo_config.py`.

### 3.1 Immediate miss — `docs / docs` is universal

The reusable CI aggregator (`vergil-actions/.github/workflows/ci.yml`) invokes the
`docs:` job (`ci-docs.yml`, job `name: docs` → check context `docs / docs`) with
**no `if:` guard**, exactly like `quality`, `security`, and `version`. The job
*skips the build but still reports* when a repo has no `docs/site/mkdocs.yml`, so
the `docs / docs` check appears on **every** repo's PRs. It is therefore a
universal gate and must be added to `desired_ci_gates_ruleset()`
**unconditionally**, alongside `quality / common` / `security / trivy` /
`security / semgrep`.

### 3.2 Enforcement is two layers — contract (apply-time) + empirical (daily)

**The contract layer reuses the existing audit machinery.** `vrg-github-repo-config`
already computes the *desired* required-check set and its `audit` subcommand
already **hard-fails on required-set drift** (exit 1; see §3.4). So the contract
enforcement of I1 is mostly *completeness of the desired set*, not new gating
code:

- **(a) Complete the desired set.** Add `docs / docs` (§3.1). Add a **completeness
  test** that pins the universal required-check set against the vergil-actions
  reusable-CI contract, so a future universal check added to CI without a matching
  entry here fails the tooling's own test suite (not at fleet apply-time).

  Because enforcement is contract-based, it works at **config-apply time with no
  PR needed** — critical for new/quiet repos, which have no PR check-runs to
  observe. (This resolves the empirical-source feasibility gap: the required
  ruleset is written before any PR exists, so the ground truth must be the known
  reusable-CI contract the tool already computes — not observed PR check-runs.)

- **(b) Empirical drift detection (daily-job layer).** For **repo-local** checks
  the contract does not know about, compare the check contexts a repo actually
  **produces on a `pull_request`** (read from PR check-runs via `gh pr checks` /
  check-runs API) against the required set, and flag any produced-but-not-required
  PR check. This is the **report-first** layer the daily ops job (§5) runs; it
  needs at least one PR's worth of data and is naturally scoped to `pull_request`.

### 3.3 Rollout — hard-fail immediately; deploy then re-apply the fleet

Hard-fail is immediate (single-owner fleet; fully auditable). The blast radius is
**narrow and precise** — see §3.4 — so the deployment sequence *is* the
remediation:

1. **Code:** add `docs / docs` + completeness test (contract layer) and the
   empirical reconciler (daily layer).
2. **Deploy** the tooling.
3. **`vrg-github-repo-config apply --repo <each>` across the fleet** (deployment
   task, §6) — brings every repo's required set current. `apply`/`audit` operate
   on the GitHub API (`--repo OWNER/REPO`), so only the **list** of repos is
   needed, not local checkouts.
4. `audit` is then green fleet-wide and releases are unblocked.

### 3.4 What actually hard-fails (blast radius)

- **`vrg-github-repo-config audit`** — exit `0` compliant / `1` drift / `2`
  could-not-complete (`vrg_github_repo_config.py:277`). This is the component that
  fails on drift.
- **`vrg-release`** consumes that exit code as a **fail-fast preflight gate**
  (`release/preflight.py:193`, `release/orchestrator.py:116`; `--skip-audit`
  escape hatch). So a non-compliant repo **cannot be released** until re-applied.
- **`apply` never self-blocks** — it writes only when non-compliant
  (`:282`) and *is* the remediation.
- **Nothing else** invokes audit: `vrg-validate`, `vrg-submit-pr`,
  `vrg-finalize-pr`, and PR CI are unaffected. Deploying the code breaks no
  day-to-day workflow; only an explicit `audit` or a `vrg-release` goes red until
  the fleet is re-applied.

### 3.5 Acceptance (Task A)

- `docs / docs` is required on all managed repos after the deployment re-apply.
- `audit` exits non-zero on a repo missing a contract-required check (existing
  machinery, now with a complete desired set).
- A completeness test ties the universal required set to the reusable-CI contract.
- The empirical layer flags a produced-but-not-required repo-local PR check.
- Post-merge-only checks are never flagged.

## 4. Task B — Orphaned-check resilience (finalize side)

Home of the fix: `src/vergil_tooling/lib/pr_merge.py` and
`src/vergil_tooling/lib/github.py` (`wait_for_checks`).

### 4.1 The orphan signature

An orphaned check is detectable because two GitHub views disagree:

- `gh pr checks` reports the check-run non-terminal (`IN_PROGRESS` / `pending`).
- The backing workflow run/job reports terminal (`status: completed`).

A genuinely-running check has both non-terminal; an orphan has a non-terminal
check-run over a **completed** run/job.

### 4.2 Algorithm

Replace the unbounded `gh pr checks --watch` wait with a bounded, sanity-checked
loop:

1. Watch with a **long timeout** (minutes, not seconds — a legitimately slow job
   must never trip a false alarm).
2. On timeout, for each still-pending check, **cross-check its backing run/job**:
   - backing run/job still running → resume waiting (re-poll).
   - backing run/job **completed** while the check-run is still non-terminal →
     **orphan** → raise a clear GitHub-orphan error and stop. Never merge.

The loop never merges past a non-terminal check; the only terminal outcomes are
"all checks reached terminal" (proceed to the existing failed-check gate) or
"raised" (orphan / true timeout).

### 4.3 Liveness probe — `gh run view`, identity-robust

`vrg-finalize-pr` is **human-run**, so the probe uses the human identity — the
earlier user-identity `gh api .../jobs` denial does **not** constrain the design.
Probe via **`gh run view`**, which is how this orphan was detected in the first
place (it showed `docs / docs` as in-progress while the run header was completed):

- The check's `link` field yields the run id (`/actions/runs/<id>/job/...`).
- `gh run view <id> --json jobs` returns per-job `status`/`conclusion`.
- Allowlisted and identity-robust; the raw `/actions/runs/{id}/jobs` API is **not**
  needed.

**App-posted status contexts** (CodeQL, Semgrep OSS, Trivy — `link` is `/runs/...`,
not `/actions/runs/.../job/...`) have no Actions job to probe. For those the
run/job cross-check does not apply, so the **timeout itself is the signal and
"raise" is the safe default**.

### 4.4 Acceptance (Task B)

- A simulated orphaned check (non-terminal check-run over a completed run) makes
  finalize **raise** with an actionable message, not hang.
- A genuinely-running check is still waited on to completion.
- A real failing check still fails via the existing `failed_check_names` gate.
- An app-posted status that never terminates raises on timeout (does not hang).

## 5. Enforcement path & follow-on

The intended production enforcement of I1 is running `vrg-github-repo-config`
**daily as an ops job that reports on failure** — designed but never built. It
consumes Task A's empirical layer (§3.2b). This is **out of scope here** and is
captured as the epic's follow-on brainstorm bookend (`.github#227`); its outcome
is recorded in the retrospective §5.

## 6. Component checklist (to be detailed by the plan)

- `github_config.py`: add `docs / docs` unconditionally; empirical
  produced-vs-required PR-check reconciler; completeness test vs reusable-CI
  contract.
- `vrg_github_repo_config.py`: surface the empirical reconciler result (the
  contract layer already fails via existing `audit`).
- `pr_merge.py` / `github.py`: bounded watch + `gh run view` backing-run
  cross-check + orphan raise; app-posted-status timeout fallback.
- Tests — Task A: contract completeness, empirical drift, post-merge-not-flagged.
  Task B: orphan / genuinely-running / failing / app-posted-timeout.
- **Deployment task** (operational, `--kind deployment`, blocked-by Task A):
  `vrg-github-repo-config apply` across all managed repos to bring required
  checks current. Seeded at task-filing time (step 9) so blocked-by wires to a
  real issue.

## 7. Out of scope

- The daily drift-reporting ops job itself (follow-on `.github#227`).
- Full derivation of the required set from workflow YAML (matrix / reusable /
  dynamic-name mapping). Approach chosen is contract completeness + empirical
  reconciler, not YAML derivation.
- Path-filtered per-PR required checks (§2 boundary).
- Any change to which checks CI *runs* — this epic changes which are *required*
  and how finalize *waits*, not the CI job set.
