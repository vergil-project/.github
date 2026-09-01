# Dynamic, version-agnostic CI gates — Specification

**Epic:** `vergil-project/.github#338`
**Tracking research:** `vergil-project/.github#337`
**Status:** design approved (brainstorm + pushback), pending plan.

## 1. Problem

The CI version matrix does not cascade from `vergil.toml [ci].versions`, and
branch protection requires **version-suffixed** check names that the matrix must
happen to produce. Two failures compound:

- **`ci.yml` matrix is frozen at repo-init.** `repo_init.render_ci_workflow`
  writes `versions: '<json>'` (and `container-tag:`) as literal inputs to the
  vergil-actions reusable workflows once, at creation. Nothing re-derives them,
  so each repo's matrix is hand-maintained and drifts from `[ci].versions`.
- **Required checks are per-version.** `github_config.desired_ci_gates_ruleset`
  requires `audit / dependencies / {version}`, `quality / lint / {version}`,
  `quality / typecheck / {version}`, `test / unit / {version}`. Enforced from the
  **base** branch, a PR that *reduces* the matrix can never produce the retired
  legs → the required checks sit "expected, never reported" → the PR is
  **permanently blocked**, with no `--admin` escape (admin bypass isn't enabled).

A governance gap compounds it: repos missing `ops.yml` (the nightly caller of
`vrg-github-repo-config apply`) never get canonicalized, so their rulesets
silently diverge.

Full root-cause with file:line evidence: `#337`.

## 2. Goal

Make CI versions **dynamic and version-agnostic**, for **every language**, so a
`[ci].versions` change *just works* fleet-wide — no `--admin`, no hand-editing.

## 3. Design

### 3.1 Require the existing version-agnostic `evidence` gates

**The aggregate gate already exists.** Each matrixed reusable workflow
(`ci-audit`, `ci-quality`, `ci-test`) already ends in an **`evidence` job** that
`needs` the per-version matrix and runs only when every leg succeeds, emitting a
`gate:` artifact (`vergil-actions/.github/workflows/ci-audit.yml` — the
`evidence` job; built by vergil-actions#764). Its check name is **stable and
version-free**: `audit / evidence`, `quality / evidence`, `test / evidence`
(`security / evidence` already exists for the run-once kind). These are exactly
the version-agnostic gates this epic needs — **we do not build new gate jobs.**

The change is to **require the `evidence` gates instead of the per-version legs**
(see 3.3). Because a gate's name is independent of the matrix size, any matrix
change — including a reduction — merges through the normal gate. The deadlock
class is eliminated with **no new gate mechanism**.

**One refinement to the existing gate.** Today `evidence` uses `needs: <matrix>`
without `if: always()`, so a failed leg *skips* the gate (it shows "expected",
not "failure"). Once `evidence` is the required check, make it run with
`if: always()` and an explicit `needs.*.result == 'success'` assertion so a red
leg makes the gate **fail red** rather than hang as "expected" — clearer signal,
same blocking behavior.

### 3.2 Dynamic matrix from `vergil.toml`

The `matrix` job in each reusable workflow today reads `${{ inputs.versions }}`
(passed from `ci.yml`). Change it to **read `[ci].versions` from `vergil.toml`**:

- The shared setup action already parses `vergil.toml`; extend it (or add a
  sibling step) to output `versions` (JSON array) and a derived `primary-version`.
- **`primary-version` = the highest version** in `[ci].versions` (semantic max —
  `3.14` from `[3.12,3.13,3.14]`), matching every repo's current hand-set
  `container-tag`. Robust to list order. An explicit **`[ci].primary-version`**
  key is the documented escape hatch when "primary" must *not* be the max.
- **`ci.yml` stops passing `versions:` and `container-tag:`** — it becomes a thin
  caller. Single-container jobs (`ci-security`, `ci-version-bump`, `ci-docs`)
  take the derived `primary-version`.

The version set then exists as a stored value **only in `vergil.toml`**, read at
run time.

### 3.3 Ruleset generator + live reconciliation (vergil-tooling)

- **`github_config.desired_ci_gates_ruleset` requires `<kind> / evidence`** for
  the matrixed kinds (audit/quality/test), replacing the per-version names — so
  the required-check set is version-independent and the ruleset stops churning
  when versions change.
- Promote the `unproducible_required_contexts` cross-check (today only in
  repo-init) into the **ongoing** audit: assert each required check is producible
  by the repo's workflows. With evidence gates this is "the gate check exists,"
  and it guards against future name drift.
- **Read classic branch protection**, not only the rulesets API, in
  `fetch_actual_state`. **Cleanup is tightly scoped:** remove only the stale
  version-suffixed CI-check *contexts* that the evidence ruleset now owns;
  **report** any other classic setting (review rules, push restrictions) rather
  than touching it. Minimal blast radius; closes the invisible-required-check
  blind spot without dropping intentional protections.

### 3.4 Governance backfill + drift guard

- **Backfill `ops.yml`** to every managed repo missing it, across all orgs (the
  `logical-minds-foundry` repos are already done as fallout of `#337`).
- **Guard against future gaps:** repo-init already renders `ops.yml`; add a
  fleet-level audit signal flagging any repo lacking `ops.yml` (or a `cron`
  trigger), so an un-canonicalized repo can't stay silent.

### 3.5 Rollout

Two independent tracks; the gate track is near-atomic because the `evidence`
gates already run on every repo via `@v2.1`.

- **Gate/ruleset track (near-atomic):** switch `desired_ci_gates_ruleset` to
  `<kind> / evidence`; nightly `apply` swaps each repo's required checks from
  per-version to evidence. Safe on in-flight PRs — the evidence checks are
  already reporting on every PR, so they satisfy the new requirement immediately.
- **Dynamic-matrix track (phased, non-breaking):**
  1. vergil-actions: `matrix` job reads `vergil.toml`; keep `versions:` /
     `container-tag:` inputs **accepted but optional** (supplied value still
     works; absence reads the toml). Cut a new tag.
  2. Fleet sweep: a `fleet_sweep` consumer bumps each repo's workflow ref and
     removes the hardcoded `versions:` / `container-tag:` from `ci.yml`.
  3. Cleanup: once no repo passes `versions:`, drop the deprecated inputs.

## 4. Components & interfaces

| Repo | Change |
|---|---|
| **vergil-actions** | `matrix` job reads `vergil.toml` (dynamic matrix); setup action outputs `versions`/`primary-version`; `evidence` gate runs `if: always()` + explicit result assertion; deprecate then drop `versions`/`container-tag` inputs. **No new gate jobs.** |
| **vergil-tooling** | `desired_ci_gates_ruleset` → `<kind> / evidence`; ongoing audit reconciliation + classic-protection read with **scoped** cleanup; `repo_init.render_ci_workflow` emits the thin caller; fleet-sweep consumer for the `ci.yml` simplification; `ops.yml`-missing audit signal. |
| **docs** | Higher-level docs on the CI model (dynamic versions, evidence gates). |
| **consuming repos** | `ci.yml` loses `versions:`/`container-tag:`; branch-protection required checks become the `<kind> / evidence` gates (applied by nightly `apply`). |

## 5. Testing & acceptance

- **Unit (vergil-tooling):** evidence-name ruleset generation; audit
  reconciliation branches; classic-protection read + **scoped** cleanup (asserts
  non-CI classic settings are preserved and reported, not removed); the
  thin-caller renderer; `primary-version` = max with the `[ci].primary-version`
  override. 100% coverage gate.
- **Workflow (vergil-actions):** the `evidence` gate fails red on a failed/skipped
  leg and passes when all legs pass; the `matrix` job reads `[ci].versions`
  correctly for single- and multi-version repos and for at least one non-Python
  language.
- **Acceptance (validation task):** a **matrix-reducing** change on a
  vergil-project repo merges through the **normal gate with no `--admin`**, and
  the same on a non-Python repo — reproducing in-org the real-world
  matrix-reduction case that first exposed the deadlock.

## 6. Out of scope / linked

- Independent fleet-sweep bugs are **linked, not folded**: `vergil-tooling#2979`
  (relative-path probe), `#2980` (stale-lock), and the `report-ready` stale-head
  nit.
- No change to *what* the checks assert — only how the matrix is sourced and
  which check branch protection requires.
- Cross-org linking is out of scope; the motivating other-org PR is referenced as
  context only, and the acceptance test is reproduced within vergil-project.
