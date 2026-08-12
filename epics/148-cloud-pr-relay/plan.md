# Cloud↔Mac PR relay (GitHubTransport) — Implementation Plan

> **For agentic workers:** Each Task below maps to one GitHub issue/PR under epic
> `vergil-project/.github#148` and is implemented via `vergil:issue-implement`
> (TDD, `vrg-container-run -- uv run vrg-validate` green, `report-ready`). Steps
> use checkbox (`- [ ]`) syntax.

**Goal:** Carry the PR-ready metadata from a cloud VM to the Mac through a reserved
git ref so the human opens the PR on the Mac with no local checkout and no
metadata regeneration, and retire `issue-localize`.

**Architecture:** A new `GitHubTransport` (implementing the existing `Transport`
ABC) writes/reads `pr-workflow.json` via `refs/vergil/pr-workflow/<branch>`, built
out-of-band via git plumbing. `report-ready` always pushes it; `vrg-submit-pr`
gains a worktree-free, push-free relay path selected by explicit branch args;
`vrg-finalize-pr` deletes the ref on cleanup.

**Tech Stack:** Python 3.12, `vergil_tooling` package; git plumbing
(`hash-object`, `commit-tree`); `vrg-git`/`vrg-gh`/`vrg-commit`.

## Global Constraints

- Python ≥ 3.12; `ruff` + `mypy` clean; 100% coverage. Validate **only** via
  `vrg-container-run -- uv run vrg-validate` (this repo's `[validation]` override).
- All git ops via `vrg-git`, GitHub via `vrg-gh`, commits via `vrg-commit`.
- Submission and merge stay **human-on-Mac**; agents stop at `report-ready`.
- Freeze (#146) arms inside `report-ready`; the relay push MUST be freeze-neutral
  (out-of-band, never advances the branch / touches HEAD/index/worktree).
- One branch per issue; **branches** (not issue numbers) identify relay submits.
- Reserved ref: `refs/vergil/pr-workflow/<branch>`; content is a single file
  `pr-workflow.json`.

---

## Sequence / dependency graph

```text
Task 1 (spike, validation) ─▶ Task 2 (GitHubTransport)
                                   ├─▶ Task 3 (report-ready always-push)
                                   ├─▶ Task 4 (vrg-submit-pr relay path)
                                   └─▶ Task 5 (vrg-finalize-pr ref cleanup)
Task 6 (plugin: retire issue-localize) ── after Task 4's behavior is fixed
Tasks 3+4+5 merged ─▶ human release ─▶ Task 7 (deploy) ─▶ Task 8 (live validation)
Bookends: #149 (this spec+plan PR), #2364 (doc-review), #150 (follow-on brainstorm)
```

**Bookend ownership — doc split (per alignment).** Task 6 owns the **plugin-side**
docs (remove `issue-localize`, plugin `CLAUDE.md` cloud contract). The
**`vergil-tooling`-side** doc/contract rewrite — the CLAUDE.md cloud-session
contract that today reads "cloud is triage-only, no PR-development" (Deliverable A)
and the site docs — is owned by the **doc-review bookend #2364**, which
*rewrites* those to match the relay (cloud can do PR-dev; submission stays
human-on-Mac), not merely verifies them. Naming it here so the tooling-side
contract change cannot be dropped between Task 6 and the bookend.

---

## Task 1: Feasibility spike — prove `refs/vergil/*` is pushable cloud→Mac

**Kind:** `validation` (operational — no PR). **Sequenced first; gates Task 2.**

**Precondition self-check:** a cloud VM session and Mac session are both available.

- [ ] **Step 1 — push a dummy ref from the cloud VM:**
  `vrg-git push origin HEAD:refs/vergil/probe/spike`
- [ ] **Step 2 — fetch it on the Mac:**
  `vrg-git fetch origin 'refs/vergil/probe/spike:refs/vergil/probe/spike'` and
  confirm `vrg-git rev-parse refs/vergil/probe/spike` resolves.
- [ ] **Step 3 — delete it from either side:** `vrg-git push origin :refs/vergil/probe/spike`
- [ ] **Step 4 — record outcome.** `Outcome: SUCCESS` if the ref pushed, fetched,
  and deleted with no protection/ruleset rejection; `FAILURE` with the exact
  rejection message otherwise.

**Acceptance:** the round-trip works with the cloud agent's normal token. On
FAILURE the epic's design is revisited before Task 2.

---

## Task 2: `GitHubTransport`

**Files:**

- Create: `src/vergil_tooling/lib/pr_workflow/github_transport.py`
- Test: `tests/vergil_tooling/lib/pr_workflow/test_github_transport.py`

**Interfaces — Produces:**

```python
class GitHubTransport(Transport):
    def __init__(self, repo_root: Path, branch: str, *, remote: str = "origin") -> None: ...
    def read(self) -> WorkflowState | None: ...   # fetch ref → parse pr-workflow.json; None if absent
    def write(self, state: WorkflowState) -> None: ...  # out-of-band build + force-push ref
    def delete(self) -> None: ...                  # push :ref ; no-op if absent
    def head_sha(self) -> str: ...
    def merge_base(self) -> str: ...
```

Ref path: `refs/vergil/pr-workflow/{branch}`.

**Consumes:** `Transport` ABC (`lib/pr_workflow/transport.py`), `WorkflowState`
(`lib/pr_workflow/state.py`).

**Implementation notes (production code via TDD):** `write()` must build the
commit off to the side — write the blob (`git hash-object -w --stdin`), build a
tree containing just `pr-workflow.json` (a temp `GIT_INDEX_FILE` +
`update-index --add --cacheinfo` + `write-tree`, or `mktree`), `commit-tree` the
tree, then `git push --force origin <commit>:refs/vergil/pr-workflow/<branch>`.
It must NOT run `git commit`.

- [ ] **Step 1 — failing test: round-trip.**

```python
def test_write_then_read_round_trips(tmp_git_remote):
    root, branch = tmp_git_remote
    state = make_state(issue="42", branch=branch, status="ready")
    GitHubTransport(root, branch).write(state)
    got = GitHubTransport(root, branch).read()
    assert got is not None and got.issue == "42" and got.status == "ready"
```

- [ ] **Step 2 — failing test: freeze-neutral invariant** (the load-bearing one):

```python
def test_write_does_not_touch_head_index_or_worktree(tmp_git_remote):
    root, branch = tmp_git_remote
    before_head = git_rev_parse(root, "HEAD")
    before_status = git_status_porcelain(root)
    GitHubTransport(root, branch).write(make_state(branch=branch))
    assert git_rev_parse(root, "HEAD") == before_head
    assert git_status_porcelain(root) == before_status
```

- [ ] **Step 3 — failing tests: `read()` on missing ref → `None`; `write()` twice
  force-updates; `delete()` removes it and is a no-op when absent.**
- [ ] **Step 4 — run tests, verify they fail** (`GitHubTransport` undefined).
- [ ] **Step 5 — implement `github_transport.py`** per the notes above.
- [ ] **Step 6 — run `vrg-container-run -- uv run vrg-validate`; green.**
- [ ] **Step 7 — commit** (`feat(pr-workflow): GitHubTransport relay over refs/vergil/pr-workflow`).

---

## Task 3: `report-ready` always-pushes the relay ref

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (the `report-ready` handler)
- Test: `tests/vergil_tooling/bin/test_vrg_pr_workflow.py`

**Consumes:** `GitHubTransport` (Task 2).

**Behavior:** after the existing `LocalFileTransport` write, always call
`GitHubTransport(repo_root, branch).write(state)`. A push failure surfaces loudly
(non-zero/warning) but does not undo the durable local write.

- [ ] **Step 1 — failing test: `report-ready` writes the ref.**

```python
def test_report_ready_pushes_relay_ref(monkeypatch, worktree):
    calls = record_calls(monkeypatch, "GitHubTransport.write")
    run_report_ready(worktree, issue="42", title="t", summary="s", notes="n")
    assert calls.count == 1 and calls.last_arg.status == "ready"
```

- [ ] **Step 2 — failing test: push failure surfaces, local write persists.**
- [ ] **Step 3 — run, verify fail. Step 4 — implement. Step 5 — validate green. Step 6 — commit.**

---

## Task 4: `vrg-submit-pr` — shared PR-open core + relay branch-list path

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py`
- Test: `tests/vergil_tooling/bin/test_vrg_submit_pr.py`

**Consumes:** `GitHubTransport.read` (Task 2), `LocalFileTransport`.

**Behavior:** extract a shared core `_open_pr(branch, base, metadata, *, push:
bool)` from today's flow. Add positional `branches: nargs="*"`.

- **No branches** → today's local-worktree batch (`push=True`, unchanged).
- **Branches** → relay path: per branch, resolve state (local worktree file if
  present, else `GitHubTransport.read()`); **verify origin tip ==
  `state.git["head_sha"]`** (else error); call `_open_pr(..., push=False)`; batch.
  No `current_branch`, no worktree lookup, no branch push. Provenance/standards run
  against GitHub data.

- [ ] **Step 1 — failing test: no-arg path still submits local ready worktrees** (regression guard).
- [ ] **Step 2 — failing test: relay path opens a PR from the ref without pushing.**

```python
def test_relay_submit_opens_pr_without_push(monkeypatch, origin_with_ref):
    branch = "feature/42-x"
    pushed = spy_no_push(monkeypatch)
    opened = spy_open_pr(monkeypatch)
    main(["feature/42-x"])
    assert opened.branch == branch and pushed.count == 0
```

- [ ] **Step 3 — failing tests: local-file preferred over ref; head_sha mismatch → error; missing ready-state → clear error; batch of two branches.**
- [ ] **Step 4 — run, verify fail. Step 5 — refactor + implement. Step 6 — validate green. Step 7 — commit.**

---

## Task 5: `vrg-finalize-pr` — delete the relay ref on cleanup

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (and the straggler sweep it
  shares with `lib/worktrees.py` if needed)
- Test: `tests/vergil_tooling/bin/test_vrg_finalize_pr.py`

**Consumes:** `GitHubTransport.delete` (Task 2).

**Behavior:** when a branch is cleaned up (merged **and** closed-PR paths in
`_delete_branch_and_worktree`), also `GitHubTransport(root, branch).delete()`. In
the straggler sweep, prune a relay ref whose branch no longer exists.

- [ ] **Step 1 — failing test: merged-branch cleanup deletes its relay ref.**
- [ ] **Step 2 — failing test: closed-PR cleanup deletes its relay ref; swept net prunes an orphan ref; absent ref is a no-op.**
- [ ] **Step 3 — run, verify fail. Step 4 — implement. Step 5 — validate green. Step 6 — commit.**

---

## Task 6: Plugin — retire `issue-localize`, reconcile the cloud-boundary contract

**Repo:** `vergil-project/vergil-claude-plugin`. **After Task 4's behavior is fixed.**

**Files:**

- Delete: `skills/issue-localize/` (the whole skill)
- Modify: plugin `CLAUDE.md` cloud-session contract; any docs referencing
  `issue-localize` or "cloud is triage-only, no PR-dev".

**Behavior:** rewrite the cloud contract — with the relay, the cloud **can** do
PR-development (implement → `report-ready` → relay); only submission/merge stay
human-on-Mac. Replace the "`/issue-localize` then submit" flow with
"`vrg-submit-pr <branch> …>` on the Mac". Note that `--notes` must not carry
secrets (public relay ref).

- [ ] **Step 1 — remove the skill; grep the repo for `issue-localize` and fix every reference.**
- [ ] **Step 2 — rewrite the cloud-boundary contract in `CLAUDE.md`.**
- [ ] **Step 3 — validate (`vrg-container-run -- vrg-validate`, plain form) green; commit.**

---

## Task 7: Deploy the relay tooling (deployment, operational)

**Kind:** `deployment`. **Blocked-by Tasks 3, 4, 5.**

**Precondition (human-gated):** a `vergil-tooling` release including Tasks 3–5 is
cut (agents never release). Then install the released version on the cloud VM and
the Mac.

- [ ] **Step 1 — self-check:** the release tag including Tasks 3–5 exists.
- [ ] **Step 2 — install/sync the released tooling on the cloud VM and Mac** (`issue-deploy`).
- [ ] **Step 3 — record `Outcome: SUCCESS`** with the installed version on both sides.

---

## Task 8: Live cloud→Mac relay round-trip (validation, operational)

**Kind:** `validation`. **Blocked-by Task 7.**

- [ ] **Step 1 — on a cloud VM:** implement a throwaway change on `feature/<N>-relay-smoke`, `report-ready` (pushes the relay ref).
- [ ] **Step 2 — on the Mac:** `vrg-submit-pr feature/<N>-relay-smoke` — confirm it fetches the ref, verifies head_sha, and opens the PR with the relayed title/summary/notes, **no local checkout**.
- [ ] **Step 3 — close the smoke PR; confirm `vrg-finalize-pr` cleanup deletes the relay ref.**
- [ ] **Step 4 — record `Outcome: SUCCESS`** (or FAILURE with the failing stage).

---

## Self-review

- **Spec coverage:** §3.1 GitHubTransport → Task 2; §3.2 always-push → Task 3;
  §3.3 submit-pr relay → Task 4; §3.4 ref cleanup → Task 5; §3.5 plugin retire →
  Task 6; §3.1 feasibility → Task 1; §6 live validation → Tasks 7–8. All covered.
- **Interfaces consistent:** `GitHubTransport.{read,write,delete}` names used
  identically in Tasks 3–5 as defined in Task 2; `state.git["head_sha"]` matches
  the engine's recorded field.
- **Deploy/validate gating:** Task 7 (deploy) blocks Task 8 (validate); both
  blocked on the impl tasks + a human release, per the impl→deploy→validate shape.
