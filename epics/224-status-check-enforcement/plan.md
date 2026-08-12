# Status-check-enforcement Implementation Plan

> **For agentic workers:** implement each task via `/vergil:issue-implement` in a
> worktree, TDD, validate green (`vrg-container-run -- vrg-validate`), hand off
> with `vrg-pr-workflow report-ready`. Steps use checkbox (`- [ ]`) tracking.

**Goal:** Make an optional PR gate impossible (enforced by `vrg-github-repo-config`)
and make `vrg-finalize-pr` resilient to GitHub-orphaned check-runs.

**Architecture:** Two independent code changes in `vergil-tooling`, plus a
fleet-wide re-apply. Task A completes the desired required-check set (contract
layer, hard-failed by the existing `audit`) so no universal PR check can be
optional. Task B replaces the unbounded `gh pr checks --watch` with a bounded
watch that cross-checks a still-pending check against its backing workflow run
via `gh run view` and raises on an orphan. (The empirical repo-local reconciler
and daily ops job are deferred to follow-on #227.)

**Tech Stack:** Python 3.12+, `pytest`, `gh` CLI wrappers in
`vergil_tooling.lib.github`, GitHub rulesets API.

## Global Constraints

- All work lands in `vergil-project/vergil-tooling` (each task's PR is same-repo).
- Portability: macOS + Linux. shellcheck-clean for any shell.
- No new runtime deps; use existing `github.py` `gh` wrappers and `_run_with_retry`.
- Validation is the single entry point: `vrg-container-run -- vrg-validate`.
- Scope of I1 is `pull_request` checks only; post-merge workflows are excluded.
- Reusable-CI PR checks are assumed unconditional (no path filters) — see spec §2.

---

## File structure

- `src/vergil_tooling/lib/github_config.py` — `desired_ci_gates_ruleset()` gains
  `docs / docs`; completeness test pins the universal contract set.
- `src/vergil_tooling/lib/github.py` — orphan-aware check waiting
  (`wait_for_checks` / helpers).
- `src/vergil_tooling/lib/pr_merge.py` — consume the orphan error (message only;
  logic lives in `github.py`).
- Tests under `tests/vergil_tooling/…` mirroring each module.

---

## Task A1: `docs / docs` universal required check + completeness test

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` (`desired_ci_gates_ruleset`, the
  "Always present" block near line 279)
- Test: `tests/vergil_tooling/test_github_config.py`

**Interfaces:**

- Consumes: existing `_make_check(context)` → `{"context", "integration_id"}`.
- Produces: `desired_ci_gates_ruleset(project, ci, ghas=…)` now includes a
  `docs / docs` context in `required_status_checks`.

- [ ] **Step 1: Write the failing test** — the universal set includes `docs / docs`.

```python
def test_desired_ci_gates_requires_docs_check():
    project = ProjectConfig(primary_language="python", release_model="calver")
    ci = CiConfig(versions=["3.12"], integration_tests=False)
    ruleset = desired_ci_gates_ruleset(project, ci, ghas=False)
    contexts = {
        c["context"]
        for rule in ruleset.rules
        for c in rule["parameters"]["required_status_checks"]
    }
    assert "docs / docs" in contexts
```

- [ ] **Step 2: Run it to verify it fails**

Run: `pytest tests/vergil_tooling/test_github_config.py::test_desired_ci_gates_requires_docs_check -v`
Expected: FAIL (`docs / docs` not in contexts).

- [ ] **Step 3: Add the check** in the "Always present" block:

```python
    checks.append(_make_check("docs / docs"))
```

- [ ] **Step 4: Run to verify it passes.**

- [ ] **Step 5: Write the completeness test** pinning the universal contract set,
  so removing any universal check (or CI adding one without updating this
  function) fails CI:

```python
UNIVERSAL_REUSABLE_CI_CHECKS = frozenset({
    "quality / common", "security / trivy", "security / semgrep",
    "docs / docs", "version / version-bump",
})

def test_universal_reusable_ci_checks_are_all_required():
    """Pins the vergil-actions reusable-CI universal checks. Update this set
    deliberately when the reusable CI adds/removes a universal PR job."""
    project = ProjectConfig(primary_language="python", release_model="calver")
    ci = CiConfig(versions=["3.12"], integration_tests=False)
    ruleset = desired_ci_gates_ruleset(project, ci, ghas=False)
    contexts = {
        c["context"]
        for rule in ruleset.rules
        for c in rule["parameters"]["required_status_checks"]
    }
    assert UNIVERSAL_REUSABLE_CI_CHECKS <= contexts
```

- [ ] **Step 6: Run the full module tests; confirm green.**

Run: `pytest tests/vergil_tooling/test_github_config.py -v`

- [ ] **Step 7: Commit.**

```bash
vrg-commit --type fix --scope pr-gates \
  --message "require docs / docs check + pin universal reusable-CI set" \
  --body "docs / docs is emitted unconditionally by the reusable CI (no if: guard) and must be a required PR gate. Add it to desired_ci_gates_ruleset and pin the universal contract set so a future universal check cannot silently become optional."
```

---

## Task B: Orphaned-check resilience in `vrg-finalize-pr`

**Files:**

- Modify: `src/vergil_tooling/lib/github.py` (`wait_for_checks`,
  `_poll_and_watch_checks`; new orphan helpers)
- Modify: `src/vergil_tooling/lib/pr_merge.py` (surface the orphan error message)
- Test: `tests/vergil_tooling/test_github.py`, `tests/vergil_tooling/test_pr_merge.py`

**Interfaces:**

- Consumes: `github.pr_checks(pr)` (extended to include `link`), `gh run view`.
- Produces:
  - `OrphanedCheckError(Exception)` with an actionable message.
  - `github.run_completed(run_id: str) -> bool` (via `gh run view <id> --json status`).
  - `github.all_checks_terminal(pr: str) -> bool` — True when every check on the
    PR has a terminal `bucket` (`pass`/`fail`/`skipping`/`cancel`, not `pending`).
  - `github.orphaned_check_names(pr: str) -> list[str]` — non-terminal checks
    whose backing Actions run is completed. App-posted statuses (`link` not
    `/actions/runs/.../job/...`) are excluded (no job to probe).
  - `wait_for_checks(pr, …)` returns normally once all checks are terminal
    (letting `pr_merge`'s existing `failed_check_names` gate catch failures), and
    raises `OrphanedCheckError` (or a plain timeout error) instead of hanging.

- [ ] **Step 1: Write the failing test** for `run_completed`:

```python
def test_run_completed_true_when_status_completed(monkeypatch):
    monkeypatch.setattr(github, "read_json",
        lambda *a, **k: {"status": "completed", "conclusion": "success"})
    assert github.run_completed("31101639453") is True

def test_run_completed_false_when_in_progress(monkeypatch):
    monkeypatch.setattr(github, "read_json",
        lambda *a, **k: {"status": "in_progress"})
    assert github.run_completed("31101639453") is False
```

- [ ] **Step 2: Run to verify it fails.**

- [ ] **Step 3: Implement `run_completed`:**

```python
def run_completed(run_id: str) -> bool:
    """True if the workflow run has reached a terminal status."""
    data = read_json("run", "view", run_id, "--json", "status")
    return isinstance(data, dict) and data.get("status") == "completed"
```

- [ ] **Step 4: Run to verify it passes.**

- [ ] **Step 5: Write the failing test** for `orphaned_check_names` — a
  non-terminal check over a completed run is an orphan; over a running run it is
  not; an app-posted status is never an orphan.

```python
def test_orphaned_check_names_flags_completed_run(monkeypatch):
    monkeypatch.setattr(github, "pr_checks", lambda pr: [
        {"name": "docs / docs", "bucket": "pending", "state": "IN_PROGRESS",
         "link": "https://github.com/o/r/actions/runs/999/job/1"},
        {"name": "quality / common", "bucket": "pass", "state": "SUCCESS",
         "link": "https://github.com/o/r/actions/runs/999/job/2"},
    ])
    monkeypatch.setattr(github, "run_completed", lambda rid: True)
    assert github.orphaned_check_names("934") == ["docs / docs"]

def test_orphaned_check_names_ignores_running_and_app_status(monkeypatch):
    monkeypatch.setattr(github, "pr_checks", lambda pr: [
        {"name": "docs / docs", "bucket": "pending", "state": "IN_PROGRESS",
         "link": "https://github.com/o/r/actions/runs/999/job/1"},
        {"name": "CodeQL", "bucket": "pending", "state": "IN_PROGRESS",
         "link": "https://github.com/o/r/runs/5"},  # app-posted: no /actions/runs
    ])
    monkeypatch.setattr(github, "run_completed", lambda rid: False)
    assert github.orphaned_check_names("934") == []
```

- [ ] **Step 6: Implement `orphaned_check_names`** — parse the run id from a
  check `link` matching `/actions/runs/<id>/job/`, skip non-terminal checks whose
  link is not an Actions job link, and return names where `run_completed(run_id)`
  is True. (Extend `pr_checks` to request `link` in its `--json` fields.)

- [ ] **Step 7: Run to verify green.**

- [ ] **Step 8: Write the failing test** for `all_checks_terminal`:

```python
def test_all_checks_terminal_true_when_no_pending(monkeypatch):
    monkeypatch.setattr(github, "pr_checks", lambda pr: [
        {"name": "a", "bucket": "pass", "state": "SUCCESS", "link": ""},
        {"name": "b", "bucket": "fail", "state": "FAILURE", "link": ""},
    ])
    assert github.all_checks_terminal("934") is True

def test_all_checks_terminal_false_when_pending(monkeypatch):
    monkeypatch.setattr(github, "pr_checks", lambda pr: [
        {"name": "a", "bucket": "pending", "state": "IN_PROGRESS", "link": ""},
    ])
    assert github.all_checks_terminal("934") is False
```

- [ ] **Step 9: Implement `all_checks_terminal`:**

```python
def all_checks_terminal(pr: str) -> bool:
    return all(c.get("bucket") != "pending" for c in pr_checks(pr))
```

- [ ] **Step 10: Run to verify green.**

- [ ] **Step 11: Write the failing test** for the bounded wait raising on orphan:

```python
def test_wait_for_checks_raises_on_orphan(monkeypatch):
    # checks never all-terminal; timeout elapses; orphan detected
    monkeypatch.setattr(github, "all_checks_terminal", lambda pr: False)
    monkeypatch.setattr(github, "orphaned_check_names", lambda pr: ["docs / docs"])
    with pytest.raises(github.OrphanedCheckError, match="docs / docs"):
        github.wait_for_checks("934", poll_timeout=0, poll_interval=0)
```

- [ ] **Step 12: Write the failing passthrough test** — when all checks reach a
  terminal state (including a failure), `wait_for_checks` **returns normally** so
  `pr_merge`'s `failed_check_names` gate still catches the failure (spec §4.4):

```python
def test_wait_for_checks_returns_when_all_terminal(monkeypatch):
    monkeypatch.setattr(github, "all_checks_terminal", lambda pr: True)
    # must not consult orphan logic or raise when everything is terminal
    monkeypatch.setattr(github, "orphaned_check_names",
        lambda pr: pytest.fail("orphan check must not run when all terminal"))
    github.wait_for_checks("934", poll_timeout=0, poll_interval=0)  # returns None
```

- [ ] **Step 13: Restructure `wait_for_checks`** to a bounded loop, preserving the
  head-moved re-registration handling from `_poll_and_watch_checks`:

```python
def wait_for_checks(pr, *, poll_interval=_POLL_INTERVAL_SECS,
                    poll_timeout=_POLL_TIMEOUT_SECS):
    deadline = time.monotonic() + poll_timeout
    while True:
        if all_checks_terminal(pr):
            return  # let pr_merge.failed_check_names catch any failure
        if time.monotonic() >= deadline:
            orphans = orphaned_check_names(pr)
            if orphans:
                raise OrphanedCheckError(
                    f"GitHub left {', '.join(orphans)} non-terminal after its run "
                    "completed (orphaned check). Close/reopen the PR to re-run the "
                    "gate, then re-run vrg-finalize-pr."
                )
            raise GitHubAPIError(1, ("gh", "pr", "checks", pr),
                stderr=f"checks still pending after {poll_timeout}s")
        time.sleep(poll_interval)
```

- [ ] **Step 14: Run to verify green** (orphan + passthrough + terminal tests).

- [ ] **Step 15: Write the failing test** in `pr_merge` — an `OrphanedCheckError`
  from `wait_for_checks` surfaces as a `MergeAbortError` with an actionable
  message (close/reopen the PR to re-run the gate), and the PR is **not** merged.

```python
def test_wait_and_merge_aborts_on_orphaned_check(monkeypatch):
    monkeypatch.setattr(pr_merge.github, "wait_for_checks",
        lambda pr: (_ for _ in ()).throw(github.OrphanedCheckError("docs / docs")))
    ...
    with pytest.raises(pr_merge.MergeAbortError, match="orphan"):
        pr_merge.wait_and_merge("934", strategy="squash")
    assert not merged_called
```

- [ ] **Step 16: Handle the error** in `wait_and_merge` — catch
  `OrphanedCheckError`, raise `MergeAbortError` with guidance ("GitHub left a
  check-run non-terminal after its run completed; close/reopen the PR to re-run
  the gate, then re-run vrg-finalize-pr"). Never call `github.merge`.

- [ ] **Step 17: Run both modules; confirm green.**

- [ ] **Step 18: Commit.**

```bash
vrg-commit --type fix --scope finalize \
  --message "raise on GitHub-orphaned check-run instead of hanging" \
  --body "Bound the check watch and cross-check a still-pending check against its backing workflow run via gh run view; a non-terminal check over a completed run is a GitHub orphan and now aborts with guidance rather than hanging vrg-finalize-pr forever. App-posted statuses fall back to a timeout error."
```

---

## Deployment task (operational — filed at step 9, not implemented here)

**Kind:** `deployment`, **blocked-by** Task A1 (needs the complete desired set).

`vrg-github-repo-config apply --repo <each managed repo>` across the fleet to
bring every repo's required-check set current (adds `docs / docs` where missing).
Precondition self-check: Task A1 merged and the host tool re-installed.
Procedure: enumerate managed repos, `apply` each, record per-repo result.
Acceptance / `Outcome: SUCCESS`: every managed repo's `audit` exits 0.

Seed with:

```bash
vrg-issue-create --epic vergil-project/.github#224 --repo vergil-project/vergil-tooling \
  --kind deployment --title "Apply canonical config fleet-wide (require docs / docs)" \
  --blocked-by vergil-project/vergil-tooling#<A1>
```

---

## Sequencing

1. **A1** (docs/docs + completeness) — unblocks the deployment re-apply.
2. **B** (orphan resilience) — independent; can run in parallel with A1.
3. **Deployment** — after A1 merges + tool re-installed.
4. Documentation-review (#2596), then Retrospective (#226) — closing bookends.

## Self-review notes

- **Spec coverage:** §3.1 → A1; §3.2 (contract completeness) → A1; §3.4
  (hard-fail via existing audit) → A1 relies on it, no new code; §4 → B; §6
  deployment → deployment task; §5 empirical reconciler + daily job → deferred to
  follow-on #227 (out of scope). All in-scope requirements covered.
- **Types:** `run_completed(str)->bool`, `all_checks_terminal(str)->bool`,
  `orphaned_check_names(str)->list[str]`, `OrphanedCheckError` — consistent across
  Task B steps.
- **Passthrough preserved:** Task B step 12 asserts `wait_for_checks` returns
  normally when all checks are terminal, so `pr_merge.failed_check_names` still
  catches real failures (spec §4.4).
