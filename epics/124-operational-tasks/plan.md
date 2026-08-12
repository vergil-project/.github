# Deployment Tasks & the Operational-Task Family — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generalize the not-PR-workable validation machinery (epic #115) into a shared **operational-task** family and add a **deployment** kind that gates on being *deployed*, not merely merged.

**Architecture:** A behavior-preserving atomic rename of the just-shipped `validation*` guard/audit code to `operational*` (matching a *set* of operational labels, initially just `validation`), then the `deployment` label + `--kind deployment` scaffold is added to that set. A new `issue-deploy` run skill and a shared operational-task lifecycle join `issue-validate`. The generic `Blocked-by:` plumbing is reused unchanged.

**Tech Stack:** Python 3.12+ (`vergil-tooling`, `pytest`, tests mock the `github` boundary); Markdown SKILL.md doctrine (`vergil-claude-plugin`).

## Global Constraints

- **Repos:** tooling → `vergil-tooling`; skills → `vergil-project/vergil-claude-plugin`; epic docs → `vergil-project/.github`. One PR per repo per task; cross-org out of scope.
- **The rename is behavior-preserving and atomic.** Tasks 1–2 rename `validation*` → `operational*` with the existing validation tests kept **green throughout** (updated for the new names only). **No back-compat aliases** — all callers are ours and update in the same change.
- **Operational tasks are never PR-workable and never auto-close.** The operational label set is the single signal.
- **`issue-deploy` runs only agent-safe deploy steps.** A required **release** is a human-gated **precondition**, never performed by the agent.
- **Result marker:** `Outcome: SUCCESS` / `Outcome: FAILURE`, unified across kinds; the audit also recognizes legacy `Outcome: PASS`.
- **Label descriptions ≤ 100 chars** (enforced by the existing `test_label_descriptions_within_github_limit`).
- **Tests mock `vergil_tooling.lib.github`**; git/GitHub via `vrg-*` wrappers; validation is `vrg-container-run -- vrg-validate`.

---

### Task 1: Rename the guard `validation` → `operational` (behavior-preserving) — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`is_validation_task` → `is_operational_task`; add `_OPERATIONAL_LABELS`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (`_reject_if_validation_task` → `_reject_if_operational_task`)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (`_reject_validation_issue` → `_reject_operational_issue`)
- Test: `tests/vergil_tooling/test_epics.py`, `test_vrg_submit_pr.py`, `pr_workflow/test_cli_e2e.py`

**Interfaces:**

- Produces: `epics.is_operational_task(ref, *, default_repo) -> bool` (True if the ref carries any label in `_OPERATIONAL_LABELS`); `epics._OPERATIONAL_LABELS: set[str]` (initially `{"validation"}`); `epics.is_validation(ref)` unchanged (per-label predicate).

- [ ] **Step 1: Update the library tests to the new names (still validation-only behavior)**

In `tests/vergil_tooling/test_epics.py`, rename the three `is_validation_task` tests to `is_operational_task`, keeping identical behavior (patch `epics.is_validation`):

```python
def test_is_operational_task_true_for_operational() -> None:
    with patch("vergil_tooling.lib.epics.is_operational", return_value=True) as mock:
        assert epics.is_operational_task("org/repo#7", default_repo="org/repo") is True
    mock.assert_called_once_with(IssueRef("org", "repo", 7))

def test_is_operational_task_false_for_plain_task() -> None:
    with patch("vergil_tooling.lib.epics.is_operational", return_value=False):
        assert epics.is_operational_task("#42", default_repo="org/repo") is False

def test_is_operational_task_false_for_unparseable_ref() -> None:
    with patch("vergil_tooling.lib.epics.is_operational") as mock:
        assert epics.is_operational_task("#42", default_repo="") is False
    mock.assert_not_called()
```

- [ ] **Step 2: Run, verify fail**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k is_operational_task -v`
Expected: FAIL (`is_operational_task` undefined).

- [ ] **Step 3: Implement the rename in `epics.py`**

Add **public** operational predicates (so `epic_audit` never pokes `_labels`),
and have `is_operational_task` delegate. Rename `is_validation` → `is_operational`
(behavior-preserving: `validation` is the only operational label at this step):

```python
# Labels that mark a not-PR-workable operational task (run, don't merge).
# Extended as new operational kinds are added (e.g. deployment).
_OPERATIONAL_LABELS: set[str] = {"validation"}

def is_operational(ref: IssueRef) -> bool:
    """True if *ref* carries any operational label (validation, deployment, …)."""
    return bool(_labels(ref) & _OPERATIONAL_LABELS)

def operational_kind(ref: IssueRef) -> str | None:
    """The operational label on *ref* ('validation' / 'deployment'), or None."""
    kinds = _labels(ref) & _OPERATIONAL_LABELS
    return next(iter(kinds)) if kinds else None

def is_operational_task(ref: str, *, default_repo: str) -> bool:
    """True if *ref* is an operational task, so PR tooling must refuse it.

    Single source of truth shared by ``vrg-submit-pr`` and
    ``vrg-pr-workflow report-ready``. An operational task carries a label in
    ``_OPERATIONAL_LABELS`` (validation, deployment, …); its acceptance is a
    recorded ``Outcome:`` comment, not a merged PR. Self-scoping: an unparseable
    ref is never operational and returns False.
    """
    try:
        issue = parse_issue_ref(ref, default_repo=default_repo)
    except ValueError:
        return False
    return is_operational(issue)
```

(Delete `is_validation_task` and `is_validation`; `is_operational` /
`operational_kind` subsume them. Update the two `is_validation` tests in
`test_epics.py` to `is_operational`.)

- [ ] **Step 4: Rename the two guard call sites**

In `vrg_submit_pr.py`, rename `_reject_if_validation_task` → `_reject_if_operational_task`, call `epics.is_operational_task`, and generalize the message ("--issue is an operational task (validation/deployment), which is not PR-workable; run it with its run skill, don't open a PR"). Update both call sites (~lines 297, 536+). In `vrg_pr_workflow.py`, rename `_reject_validation_issue` → `_reject_operational_issue`, call `epics.is_operational_task`, same message shape; update the call in `cmd_report_ready`.

- [ ] **Step 5: Update the guard tests to the new names**

`test_vrg_submit_pr.py`: rename `test_reject_if_validation_task_*` → `_operational_*`, patch `epics.is_validation` (still the underlying predicate); update the autouse fixture patch `_reject_if_validation_task` → `_reject_if_operational_task`; update the import. `pr_workflow/test_cli_e2e.py`: `test_report_ready_rejects_validation_task` → `_operational_task`, patch `epics.is_operational_task`.

- [ ] **Step 6: Run the affected suites, verify green (behavior unchanged)**

Run: `uv run pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_vrg_submit_pr.py tests/vergil_tooling/pr_workflow/ -q`
Expected: PASS — the guard still refuses validation, now via the operational set.

- [ ] **Step 7: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epics.py src/vergil_tooling/bin/vrg_submit_pr.py src/vergil_tooling/bin/vrg_pr_workflow.py tests/vergil_tooling/
vrg-commit --type refactor --scope pr-guard --message "generalize the PR-workability guard to operational tasks" --body "Behavior-preserving rename is_validation_task -> is_operational_task (label-set based). Ref #<TASK1>."
```

---

### Task 2: Rename the audit `validation` → `operational` (behavior-preserving) — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`ValidationStatus` → `OperationalStatus`; `validation_status`/`validation_pending`/`closed_validation_without_pass` → `operational_*`; `_VALIDATION_PASS_RE` → `_OPERATIONAL_SUCCESS_RE`; render section title)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (call sites, param names)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `test_vrg_epic_audit.py`

**Interfaces:**

- Produces: `epic_audit.OperationalStatus` (fields `epic`, `runnable`, `blocked`, prop `pending`); `operational_status(epic) -> OperationalStatus`; `operational_pending(org) -> list[OperationalStatus]`; `closed_operational_without_success(org) -> list[str]`. `is_operational` child check via `epics._labels(ref) & epics._OPERATIONAL_LABELS`.

- [ ] **Step 1: Update the audit tests to the new names (behavior identical, still matches `PASS`)**

In `test_epic_audit.py`, rename the six validation-aware tests to `operational_*` and update the assertions to the new symbol names; keep the `Outcome: PASS` fixtures (regex still matches PASS at this step). E.g.:

```python
def test_operational_status_classifies_runnable_vs_blocked() -> None:
    ...  # identical body, epic_audit.operational_status(epic)
def test_closed_operational_without_success_flags_missing_marker() -> None:
    ...  # fixture comment "- Outcome: PASS" still counts as success
```

- [ ] **Step 2: Run, verify fail**

Run: `uv run pytest tests/vergil_tooling/test_epic_audit.py -k operational -v`
Expected: FAIL (new symbols undefined).

- [ ] **Step 3: Rename in `epic_audit.py` (behavior-preserving)**

Rename the dataclass and functions; the classifier still keys off `"validation" in _labels(...)` via the operational set:

```python
@dataclass(frozen=True)
class OperationalStatus:
    epic: epics.IssueRef
    runnable: tuple[epics.IssueRef, ...]
    blocked: tuple[epics.IssueRef, ...]
    @property
    def pending(self) -> tuple[epics.IssueRef, ...]:
        return self.runnable + self.blocked

def operational_status(epic: epics.IssueRef) -> OperationalStatus:
    runnable, blocked = [], []
    for child in epics.child_states(epic):
        if child.state != "OPEN" or not epics.is_operational(child.ref):
            continue
        (runnable if epics.all_blockers_closed(child.ref) else blocked).append(child.ref)
    return OperationalStatus(epic=epic, runnable=tuple(runnable), blocked=tuple(blocked))
```

Use the **public** `epics.is_operational` (added in Task 1) — do not reach into
`epics._labels` / `epics._OPERATIONAL_LABELS`. Rename `validation_pending` →
`operational_pending` (calls `operational_status`); `closed_validation_without_pass`
→ `closed_operational_without_success`, searching every label in
`epics._OPERATIONAL_LABELS` (loop the set) and keeping `_OPERATIONAL_SUCCESS_RE =
re.compile(r"^\s*[-*]?\s*Outcome:\s*PASS\s*$", re.MULTILINE | re.IGNORECASE)` for
now (SUCCESS added in Task 3). Rename the render section to "Operational tasks
pending".

- [ ] **Step 4: Update `vrg_epic_audit.py` call sites + render params**

Rename `validation_pending`/`closed_validation_without_pass` calls to the new names and the render kwargs (`pending_validation` → `pending_operational`, `closed_validation_no_pass` → `closed_operational_no_success`). Update `test_vrg_epic_audit.py` mocks accordingly.

- [ ] **Step 5: Run, verify green (behavior unchanged)**

Run: `uv run pytest tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py -q`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epic_audit.py src/vergil_tooling/bin/vrg_epic_audit.py tests/vergil_tooling/
vrg-commit --type refactor --scope epic-audit --message "generalize validation-aware audit to operational tasks" --body "Behavior-preserving rename; still matches Outcome: PASS. Ref #<TASK2>."
```

---

### Task 3: Unified `Outcome: SUCCESS` marker (tooling side) — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_OPERATIONAL_SUCCESS_RE`)
- Modify: `src/vergil_tooling/data/validation_task_body.md` (results template)
- Test: `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Produces: the success invariant recognizes `Outcome: SUCCESS` **and** legacy `Outcome: PASS`.

- [ ] **Step 1: Write the failing test**

```python
def test_success_marker_accepts_success_and_legacy_pass() -> None:
    search = [{"number": n, "repository": {"nameWithOwner": "org/repo"}} for n in (1, 2, 3)]
    bodies = {"1": "- Outcome: SUCCESS", "2": "- Outcome: PASS", "3": "closed early"}
    def fake(*args):
        return search if args[0] == "search" else {"comments": [{"body": bodies[args[2]]}]}
    with patch("vergil_tooling.lib.github.read_json", side_effect=fake):
        # #1 SUCCESS ok, #2 legacy PASS ok, #3 flagged
        assert epic_audit.closed_operational_without_success("org") == ["org/repo#3"]
```

- [ ] **Step 2: Run, verify fail** (`SUCCESS` not yet accepted → #1 wrongly flagged).

- [ ] **Step 3: Broaden the regex**

```python
_OPERATIONAL_SUCCESS_RE = re.compile(
    r"^\s*[-*]?\s*Outcome:\s*(?:SUCCESS|PASS)\s*$", re.MULTILINE | re.IGNORECASE
)
```

- [ ] **Step 4: Update the validation scaffold results template**

In `data/validation_task_body.md`, change the results line to `- Outcome: SUCCESS / FAILURE` (was `PASS / FAIL`). Keep the rest of the scaffold.

- [ ] **Step 5: Run, verify pass** (`uv run pytest tests/vergil_tooling/test_epic_audit.py -k success -v`).

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epic_audit.py src/vergil_tooling/data/validation_task_body.md tests/vergil_tooling/test_epic_audit.py
vrg-commit --type feat --scope epic-audit --message "unify operational success marker (SUCCESS, legacy PASS)" --body "Ref #<TASK3>."
```

---

### Task 4: Add the `deployment` label to the operational set — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/data/labels.json`
- Modify: `src/vergil_tooling/lib/epics.py` (`_OPERATIONAL_LABELS`)
- Test: `tests/vergil_tooling/test_labels.py`, `test_epics.py`

- [ ] **Step 1: Write failing tests**

```python
# test_labels.py
def test_registry_includes_deployment_label() -> None:
    entry = next((l for l in load_labels()["labels"] if l["name"] == "deployment"), None)
    assert entry is not None and entry["description"]

# test_epics.py
def test_is_operational_task_true_for_deployment() -> None:
    with patch.object(epics, "_labels", return_value={"deployment"}):
        assert epics.is_operational_task("o/r#7", default_repo="o/r") is True
```

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Add the label + extend the set**

`labels.json` (description ≤ 100 chars — the existing limit test enforces it):

```json
{"name": "deployment", "color": "1d76db", "description": "Deployment task: install/sync merged changes so they are usable; never auto-closed"}
```

`epics.py`: `_OPERATIONAL_LABELS = {"validation", "deployment"}`.

- [ ] **Step 4: Bump the `vrg-ensure-label` sync count**

`test_vrg_ensure_label.py::test_main_sync_provisions_all_labels`: `17` → `18`.

- [ ] **Step 5: Run, verify green** (`uv run pytest tests/vergil_tooling/test_labels.py tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_vrg_ensure_label.py -q`).

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/data/labels.json src/vergil_tooling/lib/epics.py tests/vergil_tooling/
vrg-commit --type feat --scope labels --message "add deployment label to the operational set" --body "Ref #<TASK4>."
```

---

### Task 5: `vrg-issue-create --kind deployment` + scaffold — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_issue_create.py` (`--kind` choices; `_render_validation_body` → `_render_operational_body(kind)`)
- Create: `src/vergil_tooling/data/deployment_task_body.md`
- Test: `tests/vergil_tooling/test_vrg_issue_create.py`

**Interfaces:**

- Consumes: `epics.render_blocked_by`, the `deployment` label (Task 4).
- Produces: `vrg-issue-create --kind {task,validation,deployment}`; deployment kind stamps the `deployment` label + `deployment_task_body.md` + `Blocked-by:` reflinks.

- [ ] **Step 1: Write the failing test**

```python
def test_kind_deployment_applies_label_and_scaffold() -> None:
    with (
        patch(f"{_MOD}.github.current_repo", return_value="org/repo"),
        patch(f"{_MOD}.epics.resolve_epic_ref", return_value=EPIC),
        patch(f"{_MOD}.github.create_issue", return_value=_URL) as mock_create,
        patch(f"{_MOD}.epics.add_child"),
    ):
        rc = main(["--epic", "adhoc", "--kind", "deployment", "--title", "Deploy X",
                   "--blocked-by", "org/repo#5"])
    assert rc == 0
    kwargs = mock_create.call_args.kwargs
    assert "deployment" in kwargs["labels"]
    body = kwargs["body"]
    assert "Blocked-by: org/repo#5" in body
    assert "## Preconditions" in body and "release" in body.lower()  # release-as-precondition
    assert "## Results" in body
```

- [ ] **Step 2: Run, verify fail** (`--kind deployment` unrecognized).

- [ ] **Step 3: Create the deployment scaffold**

`data/deployment_task_body.md` — mirror the validation scaffold but for deploying, with the release boundary and idempotency baked in:

```markdown
{intro}

This is a **deployment task**: run the agent-safe deploy steps below so the
merged change is usable, then record the result as a comment. It is not
PR-workable and never auto-closes. Close it only on SUCCESS; on FAILURE, retry
(the deploy is idempotent), and file a fix task only if it cannot succeed without
a code change — leaving this task (and its epic) open.

## Preconditions (self-check — run first, do not fabricate)

Declare this task's preconditions. If deploying requires a **release**
(bump/tag/publish), that release is a human-gated precondition — attest it here;
`issue-deploy` never cuts a release.

- \<precondition — e.g. "release vX.Y.Z is published", plus reachability\>

{blocked_by}## Deploy steps (agent-safe; idempotent)

- \<install / sync / restart command(s)\>

## Acceptance criteria

- \<how you know the change is deployed and usable\>

## Results

- Outcome: SUCCESS / FAILURE
- Evidence: \<command output / observations\>
- On FAILURE: retry; file a fix task only for a genuine defect; leave open.
```

- [ ] **Step 4: Generalize the body builder**

Rename `_render_validation_body(*, intro, deps)` → `_render_operational_body(*, kind, intro, deps)` selecting `f"{kind}_task_body.md"`. Extend `--kind` choices to `("task", "validation", "deployment")`. In `main`, the operational branch fires for `kind in ("validation", "deployment")`: append the kind's label, render the kind's scaffold, reject `--body-file`, parse `--blocked-by`.

- [ ] **Step 5: Run, verify green** — add a `test_kind_validation_*` regression check that validation still works via the renamed builder.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_issue_create.py src/vergil_tooling/data/deployment_task_body.md tests/vergil_tooling/test_vrg_issue_create.py
vrg-commit --type feat --scope issue-create --message "add --kind deployment path + scaffold" --body "Ref #<TASK5>."
```

---

### Task 6: Audit kind-awareness (validation vs deployment) — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`OperationalStatus` carries per-child kind; render shows kind)
- Test: `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Produces: `operational_status` tags each pending child with its kind so the report distinguishes deployment-pending from validation-pending.

- [ ] **Step 1: Write the failing test**

```python
def test_operational_status_tags_kind() -> None:
    epic = epics.IssueRef("org", ".github", 124)
    val = epics.IssueRef("org", "repo", 7); dep = epics.IssueRef("org", "repo", 8)
    children = [epics.ChildState(val, "OPEN"), epics.ChildState(dep, "OPEN")]
    def kind(ref): return "validation" if ref.number == 7 else "deployment"
    with (
        patch.object(epic_audit.epics, "child_states", return_value=children),
        patch.object(epic_audit.epics, "is_operational", return_value=True),
        patch.object(epic_audit.epics, "operational_kind", side_effect=kind),
        patch.object(epic_audit.epics, "all_blockers_closed", return_value=True),
    ):
        status = epic_audit.operational_status(epic)
    # keyed by IssueRef (cross-repo safe), not bare number
    assert status.by_kind[val] == "validation" and status.by_kind[dep] == "deployment"
```

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement**

Add `by_kind: dict[IssueRef, str]` (**keyed by ref** — cross-repo safe, since
`IssueRef` is a frozen/hashable dataclass) to `OperationalStatus`, populated in
`operational_status` via the public `epics.operational_kind(child.ref)`. Update
`render` to annotate each pending item with its kind, e.g. `- {ref.slug}
(deployment) — runnable`.

- [ ] **Step 4: Run, verify green.**

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epic_audit.py tests/vergil_tooling/test_epic_audit.py
vrg-commit --type feat --scope epic-audit --message "tag operational-pending items by kind" --body "Ref #<TASK6>."
```

---

### Task 7: Shared operational-task lifecycle + `issue-validate` marker update — `vergil-claude-plugin`

**Files:**

- Create: `skills/issue-validate/references/operational-task-lifecycle.md` (shared lifecycle)
- Modify: `skills/issue-validate/SKILL.md` (record `Outcome: SUCCESS`; point to the shared lifecycle)

- [ ] **Step 1:** Write the shared lifecycle reference: the common contract both run skills follow — preflight (USER + operational label) → preconditions gate (block, don't fabricate) → run the recorded procedure → record `Outcome: SUCCESS/FAILURE` comment → **close on SUCCESS / hold-open on FAILURE**, plus the "not-PR-workable / gates-the-epic" invariants.
- [ ] **Step 2:** Update `issue-validate` to record `Outcome: SUCCESS`/`FAILURE` (was PASS/FAIL) and to reference the shared lifecycle instead of restating it; keep validation-specific "verify, don't change" framing.
- [ ] **Step 3:** Self-review; commit.

```bash
vrg-git add skills/issue-validate/
vrg-commit --type docs --scope issue-validate --message "extract shared operational lifecycle; record SUCCESS" --body "Ref #<TASK7>."
```

---

### Task 8: `issue-deploy` skill (new) — `vergil-claude-plugin`

**Files:**

- Create: `skills/issue-deploy/SKILL.md`

- [ ] **Step 1:** Frontmatter (`name: issue-deploy`; triggers: "deploy #N", "run the deployment", picking up a `deployment`-labelled issue).
- [ ] **Step 2:** Preflight (USER + `deployment` label) → **preconditions gate**, calling out that a required **release is a human-gated precondition** (attested; `issue-deploy` never cuts a release).
- [ ] **Step 3:** Run the **agent-safe** deploy steps; record `Outcome: SUCCESS/FAILURE`.
- [ ] **Step 4:** **Close semantics** — SUCCESS → close (epic can roll up). FAILURE → **retry first** (idempotent); file a fix task only for a genuine defect; leave the task and epic open.
- [ ] **Step 5:** Reference the shared operational-task lifecycle (Task 7); contrast with `issue-validate` (deploy *changes* state) and `issue-implement` (no worktree/PR).
- [ ] **Step 6:** Self-review; commit.

```bash
vrg-git add skills/issue-deploy/
vrg-commit --type feat --scope issue-deploy --message "add issue-deploy skill" --body "Ref #<TASK8>."
```

---

### Task 9: `epic-create` doctrine — Operational tasks — `vergil-claude-plugin`

**Files:**

- Modify: `skills/epic-create/SKILL.md`

- [ ] **Step 1:** Generalize the "Validation tasks" section to **"Operational tasks"** covering both kinds and the invariants.
- [ ] **Step 2:** Document the **impl → deploy → validate** ordering and *when to add a deployment task* (the next step needs the thing **deployed/usable**, not merely merged), including the merged-vs-deployed `Blocked-by` structure.
- [ ] **Step 3:** Document creation via `vrg-issue-create --kind deployment` and the release-as-precondition boundary.
- [ ] **Step 4:** Self-review; commit.

```bash
vrg-git add skills/epic-create/SKILL.md
vrg-commit --type docs --scope epic-create --message "generalize doctrine to operational tasks (+ deployment)" --body "Ref #<TASK9>."
```

---

### Task 10: `issue-implement` redirect — any operational kind — `vergil-claude-plugin`

**Files:**

- Modify: `skills/issue-implement/SKILL.md`

- [ ] **Step 1:** Broaden the wrong-skill redirect: a `validation`-labelled issue → `issue-validate`; a `deployment`-labelled issue → `issue-deploy`; both are operational tasks, not code-implementation.
- [ ] **Step 2:** Extend discover-and-create: when a step must be **deployed** before the next can run or be validated, mint a **deployment task** (`--kind deployment`, blocked-by the impl task) and don't declare done while it's open.
- [ ] **Step 3:** Self-review; commit.

```bash
vrg-git add skills/issue-implement/SKILL.md
vrg-commit --type docs --scope issue-implement --message "redirect deployment issues to issue-deploy; discover deploy needs" --body "Ref #<TASK10>."
```

---

### Task 11: Dogfood — a real deployment task, impl→deploy→validate — `vergil-project/.github`

**Deliverable:** a `deployment`-labelled task created via the new tooling that exercises the chain end-to-end with **agent-safe** deploy steps (the release is the human precondition). Blocked-by the tooling tasks.

- [ ] **Step 1:** After Tasks 4–6 merge and deploy, create it:

```bash
vrg-issue-create --epic vergil-project/.github#124 --repo vergil-project/.github \
  --kind deployment --title "Deploy: the deployment-task tooling into the org" \
  --blocked-by vergil-project/vergil-tooling#<TASK4> \
  --blocked-by vergil-project/vergil-tooling#<TASK5>
```

- [ ] **Step 2:** Preconditions: attest the release is published (human). Deploy
  steps (agent-safe, **non-circular** — the `.github` label already exists to
  create this task, so target the **member repos**): confirm `deployment` is
  *absent* in a member repo (e.g. `vergil-tooling`), run `vrg-ensure-label --sync
  --repo vergil-project/vergil-tooling` to provision it, confirm it is now
  *present* (observable before/after), then smoke-check that `vrg-issue-create
  --kind deployment --repo vergil-project/vergil-tooling …` succeeds against the
  now-provisioned repo. That is a real idempotent deploy with an observable
  effect — not a no-op.
- [ ] **Step 3:** Run via `issue-deploy`; on SUCCESS record `Outcome: SUCCESS` and close; confirm `vrg-epic-audit` showed it **runnable** (blockers closed) beforehand and stops listing it after close.

---

## Documentation

Human-facing **site docs** — generalizing the GitHub Issue Standards "Validation tasks" section to **"Operational tasks"** (both kinds, impl→deploy→validate) — are **authored-and-verified by the doc-review closing bookend (`.github#127`)**, not a separate task (a reminder/sanity gate, per the #115 precedent).

## Task ordering / dependencies

- **Tasks 1–2** (behavior-preserving renames) come first and land green against the existing validation tests.
- **Task 3** (marker) and **Task 4** (label) build on the renamed code; **Task 4** extends `_OPERATIONAL_LABELS`.
- **Task 5** (`--kind deployment`) needs Task 4; **Task 6** (kind-aware audit) needs Tasks 2 + 4.
- **Tasks 7–10** (skills) reference the tooling surface (esp. Task 5) and each other (Task 8 references Task 7).
- **Task 11** (dogfood) is `blocked-by` Tasks 4/5/6 and run via Task 8's skill.

## Self-review — spec coverage

- Operational-task family + shared invariants → Tasks 1–2 (rename), 7 (lifecycle) ✅
- Deployment kind (label, scaffold, creation) → Tasks 4, 5 ✅
- Merged-vs-deployed graph structure → reuses `Blocked-by`/`all_blockers_closed` (unchanged), exercised in Task 11 ✅
- Unified `Outcome: SUCCESS` + legacy `PASS` → Task 3 (tooling), Task 7 (issue-validate) ✅
- Deployment retry-first + release-as-precondition boundary → Task 5 scaffold, Task 8 skill ✅
- Guard + audit generalization → Tasks 1, 2, 6 ✅
- Skills (issue-deploy, epic-create, issue-implement) → Tasks 8, 9, 10 ✅
- Behavior-preserving atomic rename (no aliases) → Tasks 1–2 ✅
- Docs → doc-review bookend #127 ✅
