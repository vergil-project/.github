# Dynamic, version-agnostic CI gates — Implementation Plan

> **For agentic workers:** This is an **epic plan**. Each task below becomes a
> GitHub implementation task (issue) filed under epic `vergil-project/.github#338`
> and implemented via `vergil:issue-implement` (its own task-by-task TDD). Steps
> use checkbox (`- [ ]`) syntax. Tasks are dependency-ordered; the **Depends-on**
> line on each task is authoritative. Tasks land in the repo named on each task
> (placement law).

**Goal:** Make CI versions dynamic and version-agnostic so a `[ci].versions`
change just works fleet-wide — no `--admin`, no hand-edited `ci.yml`.

**Architecture:** Two independent tracks. **Track 1 (unblock):** point
branch-protection required checks at the **already-existing** `<kind> / evidence`
aggregate gates instead of per-version legs — near-atomic, kills the deadlock.
**Track 2 (de-drift):** the vergil-actions reusable workflows read `[ci].versions`
from `vergil.toml`, so `ci.yml` stops carrying the matrix. Plus a governance
backfill so every repo is self-canonicalizing.

**Tech Stack:** Python 3.14 (vergil-tooling), GitHub Actions reusable workflows
(vergil-actions), `vergil_tooling.lib.github_config` / `config` / `repo_init` /
`fleet_sweep`.

**Spec:** `epics/338-dynamic-ci-gates/spec.md` (in `vergil-project/.github`).

## Global Constraints

- **No new gate jobs.** The `evidence` job per reusable workflow already
  aggregates the matrix; this epic **requires** it, never rebuilds it.
- **Coverage gate:** vergil-tooling holds `--cov-fail-under=100`; every Python
  task keeps it.
- **Shell wrappers:** `vrg-git`/`vrg-gh`, never raw. Validate with
  `vrg-container-run -- vrg-validate`.
- **Commits:** `vrg-commit`; agents stop at `vrg-pr-workflow report-ready`.
- **Placement law:** each task's PR lands in its own repo.
- **Back-compat during rollout:** the `versions:`/`container-tag:` reusable-
  workflow inputs stay **accepted but optional** until every repo has dropped
  them; only then are they removed.

---

## Track 1 — Require the evidence gates (unblock the deadlock)

### Task 1: Ruleset requires `<kind> / evidence` instead of per-version names

**Repo:** vergil-tooling · **Depends-on:** none

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` — `desired_ci_gates_ruleset`.
- Test: `tests/vergil_tooling/test_github_config.py`.

**Interfaces:**

- Produces: for the matrixed kinds (`AUDIT`, `LINT`, `TYPECHECK`, `TEST`) the
  desired ruleset now lists the **stable** contexts `audit / evidence`,
  `quality / evidence`, `test / evidence` (one per reusable workflow), not
  `audit / dependencies / {version}` etc. Run-once kinds already use their single
  `<kind> / evidence` gate — align all kinds on the same shape.

- [ ] **Step 1: Write the failing test** — assert the desired CI-gates ruleset
  contains `audit / evidence`, `quality / evidence`, `test / evidence` and does
  **not** contain any `/ {version}`-suffixed context, for a config with
  `[ci].versions = ["3.12","3.13","3.14"]`.

```python
def test_ci_gates_require_evidence_not_per_version():
    cfg = _cfg(ci_versions=["3.12", "3.13", "3.14"])
    checks = {c["context"] for c in desired_ci_gates_ruleset(cfg.ci, ...).required_status_checks}
    assert {"audit / evidence", "quality / evidence", "test / evidence"} <= checks
    assert not any(re.search(r"/ 3\.\d+$", c) for c in checks)
```

- [ ] **Step 2: Run to verify it fails** — `uv run pytest …::test_ci_gates_require_evidence_not_per_version -v` → FAIL.
- [ ] **Step 3: Implement** — replace the per-version loop in `desired_ci_gates_ruleset` (`github_config.py:340-346`) with one stable `<kind> / evidence` context per matrixed kind. Keep the run-once kinds unchanged. The function no longer reads `ci.versions` to build names (it may still take it for the container/primary logic elsewhere).
- [ ] **Step 4: Run to verify pass** → PASS.
- [ ] **Step 5: Full validate** — `vrg-container-run -- vrg-validate` green at 100%.
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope github-config --message "require <kind>/evidence gates, not per-version checks (#338)"`.

### Task 2: Promote `unproducible_required_contexts` into the live audit

**Repo:** vergil-tooling · **Depends-on:** Task 1

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` (or `repo_config.py`) — move
  the `unproducible_required_contexts` cross-check from repo-init
  (`repo_init.py:1414`) into `compute_desired_state`/the audit path.
- Test: `tests/vergil_tooling/test_github_config.py`.

**Interfaces:**

- Produces: `unproducible_required_contexts(required: set[str], produced: set[str]) -> set[str]` invoked during audit; a non-empty result is a reported non-compliance (a required check no workflow can produce).

- [ ] **Step 1: Write the failing test** — audit flags a required context that the repo's workflows do not produce (e.g. a stale `audit / dependencies / 3.12` still required) and passes when required == `<kind> / evidence` gates that the reusable workflows produce.
- [ ] **Step 2: Run → FAIL.**
- [ ] **Step 3: Implement** — call the existing helper from the audit; produced-set derived from the known reusable-workflow gate names. Report each unproducible context as a diff item.
- [ ] **Step 4: Run → PASS.**
- [ ] **Step 5: Validate green.**
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope github-config --message "audit flags unproducible required contexts (#338)"`.

### Task 3: Read classic branch protection with scoped cleanup

**Repo:** vergil-tooling · **Depends-on:** Task 1

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` — `fetch_actual_state`
  (`:762`) reads classic `branches/{branch}/protection`; extend
  `_cleanup_classic_branch_protection` (`:1187-1202`) to scoped removal.
- Test: `tests/vergil_tooling/test_github_config.py`.

**Interfaces:**

- Produces: `fetch_actual_state` includes any classic-protection required-check
  contexts; cleanup removes **only** the stale version-suffixed CI contexts owned
  by the evidence ruleset, and returns a report of every other classic setting
  left untouched.

- [ ] **Step 1: Write failing tests** — (a) a classic protection carrying
  `audit / dependencies / 3.12` is detected as drift; (b) cleanup removes exactly
  that context and **preserves** an unrelated classic setting (e.g. a review
  requirement), returning it in the report.
- [ ] **Step 2: Run → FAIL.**
- [ ] **Step 3: Implement** — read classic protection (fail-soft on 404/permission); scope the cleanup to the CI-context set; never delete the whole protection object; report the rest.
- [ ] **Step 4: Run → PASS.**
- [ ] **Step 5: Validate green.**
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope github-config --message "read + scoped-clean classic branch protection (#338)"`.

### Task 4: (deployment) Roll the evidence-gate ruleset to the fleet

**Repo:** vergil-tooling · **Kind:** deployment · **Depends-on:** Tasks 1–3 merged

Operational, run via `vergil:issue-deploy`. Precondition (attested): Tasks 1–3
released. Steps: run `vrg-github-repo-config apply` across the fleet (nightly
`ops.yml` also does this) so every repo's required checks become the `<kind> /
evidence` gates. Acceptance: sampled repos show `audit / evidence` etc. required,
no `/ {version}` contexts. Record `Outcome: SUCCESS`. This closure is the
"deadlock removed fleet-wide" signal.

---

## Track 2 — Dynamic matrix (de-drift the source)

### Task 5: `[ci].primary-version` config + `primary_version` derivation

**Repo:** vergil-tooling · **Depends-on:** none (parallel with Track 1)

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` — `CiConfig` gains
  `primary_version`.
- Test: `tests/vergil_tooling/test_config.py`.

**Interfaces:**

- Produces: `CiConfig.primary_version: str` = explicit `[ci].primary-version` if
  set, else the **highest** entry of `versions` (semantic version sort). Allowed
  `[ci]` keys extend to include `primary-version`.

- [ ] **Step 1: Write failing tests**

```python
def test_primary_version_defaults_to_highest():
    assert _cfg(ci_versions=["3.12", "3.14", "3.13"]).ci.primary_version == "3.14"

def test_primary_version_explicit_override():
    assert _cfg(ci_versions=["3.12"], primary="3.12").ci.primary_version == "3.12"

def test_primary_version_non_string_raises():
    with pytest.raises(ConfigError):
        _cfg(ci_versions=["3.14"], primary=314)
```

- [ ] **Step 2: Run → FAIL.**
- [ ] **Step 3: Implement** — parse optional `primary-version` (string; `ConfigError` otherwise); default = `max(versions, key=Version)` via `packaging`/tuple split. Register the key.
- [ ] **Step 4: Run → PASS.**
- [ ] **Step 5: Validate green.**
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope config --message "add [ci].primary-version (default = highest) (#338)"`.

### Task 6: Setup action outputs `versions` + `primary-version`

**Repo:** vergil-actions · **Depends-on:** none

**Files:**

- Modify: `actions/shared/setup/vergil/action.yml` (it already reads
  `vergil.toml`) — add outputs `versions` (JSON array) and `primary-version`.
- Test: workflow-level (a smoke job asserting the outputs for a fixture repo).

- [ ] **Step 1:** Add a step that reads `[ci].versions` from `vergil.toml` and sets `versions` (JSON) and `primary-version` (highest, or `[ci].primary-version`) as action outputs. Mirror the tooling logic from Task 5 so both agree.
- [ ] **Step 2:** Add a smoke workflow that calls the action against a multi-version fixture and asserts the two outputs.
- [ ] **Step 3:** Run the smoke workflow (or `act`/manual) → outputs correct.
- [ ] **Step 4: Commit** — `vrg-commit --type feat --scope actions --message "setup action outputs versions + primary-version from vergil.toml (#338)"`.

### Task 7: Matrix + single-container jobs read from the setup output

**Repo:** vergil-actions · **Depends-on:** Task 6

**Files:**

- Modify: `.github/workflows/ci-audit.yml`, `ci-quality.yml`, `ci-test.yml` (the
  `matrix` job), and `ci-security.yml`, `ci-version-bump.yml`, `ci-docs.yml`
  (single-container).

- [ ] **Step 1:** In each `matrix` job, source versions from the setup-action output when `inputs.versions` is empty (keep `inputs.versions` honored when supplied — back-compat). Single-container jobs use `inputs.container-tag` when supplied, else the setup `primary-version`.
- [ ] **Step 2:** Run each workflow against a multi-version and a single-version fixture (and one non-Python fixture) → matrix resolves from `vergil.toml`.
- [ ] **Step 3: Commit** — `vrg-commit --type feat --scope ci --message "reusable workflows read [ci].versions from vergil.toml (inputs optional) (#338)"`.

### Task 8: `evidence` gate fails red on a bad leg

**Repo:** vergil-actions · **Depends-on:** none

**Files:**

- Modify: the `evidence` job in `ci-audit.yml`/`ci-quality.yml`/`ci-test.yml`.

- [ ] **Step 1:** Give `evidence` `if: always()` and a first step asserting every needed matrix result is `success` (fail otherwise), so a red/skipped leg makes `evidence` **fail** (red), not skip.
- [ ] **Step 2:** Prove it: a workflow run with a deliberately failing leg shows `evidence` = failure (not skipped); an all-pass run shows success.
- [ ] **Step 3: Commit** — `vrg-commit --type fix --scope ci --message "evidence gate fails red on a failed/skipped leg (#338)"`.

### Task 9: (deployment) Publish the vergil-actions release

**Repo:** vergil-actions · **Kind:** deployment · **Depends-on:** Tasks 6–8 merged

Operational (`issue-deploy`). Human-gated precondition: the vergil-actions
release/tag (e.g. `v2.x`) is cut (the agent never cuts a release). Steps: verify
the published tag resolves the new reusable workflows and the setup outputs.
Acceptance: a consumer repo on the new tag runs CI with a `vergil.toml`-derived
matrix. `Outcome: SUCCESS` is the "dynamic matrix available fleet-wide" signal
that gates Task 11.

### Task 10: `render_ci_workflow` emits the thin caller

**Repo:** vergil-tooling · **Depends-on:** Task 5

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py` — `render_ci_workflow`
  (`:587,595-597,635,648,710`) stops emitting `versions:`/`container-tag:`.
- Test: `tests/vergil_tooling/test_repo_init.py`.

- [ ] **Step 1: Write failing test** — the rendered `ci.yml` for a new repo contains **no** `versions:` or `container-tag:` input under any reusable-workflow call.
- [ ] **Step 2: Run → FAIL.**
- [ ] **Step 3: Implement** — drop those inputs from the template; the caller just `uses:` the workflow at the pinned tag.
- [ ] **Step 4: Run → PASS.**
- [ ] **Step 5: Validate green.**
- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope repo-init --message "render ci.yml as a thin caller (no versions/container-tag) (#338)"`.

### Task 11: `vrg-ci-sync` — fleet sweep strips hardcoded matrix from `ci.yml`

**Repo:** vergil-tooling · **Depends-on:** Task 9 (dynamic matrix live), Task 10

**Files:**

- Create: `src/vergil_tooling/bin/vrg_ci_sync.py` (a `fleet_sweep` consumer,
  mirroring `vrg_fleet_sync.py`).
- Create: `src/vergil_tooling/lib/ci_yml_applicator.py` — remove the
  `versions:`/`container-tag:` inputs from a repo's `ci.yml`.
- Modify: `pyproject.toml` `[project.scripts]`.
- Test: `tests/vergil_tooling/test_ci_yml_applicator.py`.

**Interfaces:**

- Produces: `ci_yml_applicator.strip_matrix_inputs(worktree: Path) -> AppResult`
  — removes the two inputs from each reusable-workflow call; idempotent; for a
  `ci.yml` shape it cannot parse, makes no edit and flags `needs_followup` (no
  silent failure). **NOTE:** consume absolute `--repos` paths (avoid #2979's
  relative-path bug) and refresh `uv.lock` is N/A here (yaml-only).

- [ ] **Step 1: Write failing tests** — strips both inputs from a multi-call `ci.yml`; idempotent when already stripped; unparseable `ci.yml` → `needs_followup`, no edit.
- [ ] **Step 2: Run → FAIL.**
- [ ] **Step 3: Implement** the applicator + the `run_sweep` CLI (loud follow-up report). `--dry-run` touches no git/GitHub state.
- [ ] **Step 4: Run → PASS; validate green.**
- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope fleet --message "add vrg-ci-sync: strip hardcoded matrix from ci.yml (#338)"`.
- [ ] **Step 6 (rollout, human-run):** run `vrg-ci-sync` over the fleet (absolute paths); humans submit/merge one PR per repo.

### Task 14: Remove the deprecated `versions:` / `container-tag:` inputs

**Repo:** vergil-actions · **Depends-on:** Task 11 (fleet sweep complete — no repo
still passes the inputs). *Terminal Track-2 cleanup.*

**Files:**

- Modify: `.github/workflows/ci-audit.yml`, `ci-quality.yml`, `ci-test.yml`,
  `ci-security.yml`, `ci-version-bump.yml`, `ci-docs.yml` — drop the now-unused
  `versions`/`container-tag` `inputs:` declarations and their `inputs.*` fallback
  branches.

- [ ] **Step 1:** Confirm no consuming `ci.yml` still passes `versions:` or
  `container-tag:` (fleet grep / `vrg-ci-sync --dry-run` reports clean).
- [ ] **Step 2:** Remove the two `inputs:` blocks and the `inputs.versions` /
  `inputs.container-tag` fallback branches; the matrix + single-container jobs now
  source only from the setup action / `vergil.toml`.
- [ ] **Step 3:** Run each reusable workflow against a fixture consumer → matrix
  still resolves from `vergil.toml`; a caller that erroneously passes `versions:`
  now fails fast (unknown input) instead of silently overriding.
- [ ] **Step 4: Commit** — `vrg-commit --type refactor --scope ci --message "drop deprecated versions/container-tag inputs; vergil.toml is sole source (#338)"`.

---

## Track 3 — Governance

### Task 12: `ops.yml`-missing audit signal

**Repo:** vergil-tooling · **Depends-on:** none

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` — `_check_required_workflows`
  (`:89-119`) already flags a missing/unwired `ops.yml`; ensure it is a
  first-class, fleet-visible non-compliance (and covers a present-but-cronless
  `ops.yml`).
- Test: `tests/vergil_tooling/test_repo_config.py`.

- [ ] **Step 1: Write failing tests** — missing `ops.yml` → non-compliant; present-but-no-`cron` → non-compliant; canonical `ops.yml` → compliant.
- [ ] **Step 2: Run → FAIL** (add the cron-trigger check if absent).
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run → PASS; validate green.**
- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope repo-config --message "flag missing/cronless ops.yml as non-compliant (#338)"`.

### Task 13: (deployment, human-run) Backfill `ops.yml` across the fleet

**Repo:** vergil-tooling · **Kind:** deployment · **Depends-on:** Task 12

Operational. Sweep every managed repo missing `ops.yml` (the
`logical-minds-foundry` repos are already done) and open one backfill PR per repo
(reuse the ops.yml template). Humans submit/merge. `Outcome: SUCCESS` when the
`ops.yml`-missing audit signal is clean fleet-wide.

---

## Closing bookends (own skills)

- **Documentation review** (`vergil-tooling#2981`) — sweep human-facing docs
  (esp. `docs/site`) for the dynamic-versions + evidence-gate model; spawn
  per-repo doc tasks (vergil-actions, docs).
- **Validation** — acceptance: a matrix-reducing change merges via the normal
  gate with no `--admin`, on a Python **and** a non-Python vergil-project repo.
  Seed with `--kind validation`, blocked-by Task 4 (+ Track 2 for the de-drift
  half).
- **Retrospective** (`.github#340`) — terminal; via `vergil:epic-retrospective`.

## Dependency summary

```text
Track 1:  Task 1 ─┬─▶ Task 2
                  ├─▶ Task 3
                  └─▶ Task 4 (deploy: ruleset rollout)  ◀── acceptance validation

Track 2:  Task 5 ─┬─▶ Task 10 ─▶ Task 11 (vrg-ci-sync + rollout) ─▶ Task 14 (drop deprecated inputs)
          Task 6 ─▶ Task 7 ─┐
          Task 8 ───────────┴─▶ Task 9 (deploy: actions release) ─▶ Task 11

Track 3:  Task 12 ─▶ Task 13 (ops.yml backfill)
```
