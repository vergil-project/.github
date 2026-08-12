# Post-Merge Validation Follow-On — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make post-merge validation a first-class, gating part of the epic/task framework — a `validation` task type that is not PR-workable, gates epic closure by staying open, and is executed by a dedicated `issue-validate` skill.

**Architecture:** New tooling in `vergil-tooling` (a `validation` label, a `vrg-issue-create --kind validation` path that stamps an executable body + dependency reflink, a PR-workability guard mirroring `epics.is_epic_linkage`, and validation-aware rollup/audit); new doctrine + a new skill in `vergil-project/vergil-claude-plugin`. Dependency storage is decided by a spike (Task 1); a portable `Blocked-by:` body reflink is the guaranteed baseline, mirroring the existing `Parent:` sub-issue reflink in `lib/epics.py`.

**Tech Stack:** Python 3.12+ (`vergil-tooling`, `argparse` CLIs, `pytest`, tests mock the `github` boundary); Markdown SKILL.md doctrine (`vergil-claude-plugin`).

## Global Constraints

- **Repos:** tooling → `vergil-tooling`; skills → `vergil-project/vergil-claude-plugin`; epic docs → `vergil-project/.github`. One PR per repo per task; cross-org is out of scope.
- **Validation is never auto-closed and never PR-workable.** The `validation` label is the single signal enforced everywhere an issue can close automatically.
- **Tests mock the `github` boundary** (`vergil_tooling.lib.github`) — no live network in unit tests; match existing patterns in `tests/vergil_tooling/`.
- **Validation command:** `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here). This is the only validation command.
- **Git/GitHub via wrappers only:** `vrg-git`, `vrg-gh`, `vrg-commit`, `vrg-issue-create`. Raw `git`/`gh` are denied.
- **Dependency reflink format (baseline):** `Blocked-by: <owner>/<repo>#<N>` (one per line), parsed case-sensitively at line start — the mirror of `Parent: <owner>/<repo>#<N>`.

---

### Task 1: `blocked-by` feasibility spike (storage decision) — `vergil-tooling`

**Deliverable:** a short written finding (committed as `docs/specs/notes/` or the task's PR description) answering: can our App token write and read GitHub's native issue-dependency (blocked-by/blocking) relation? Decide native-vs-reflink. This gates Tasks 4 and 6.

**Files:**

- Create: `docs/specs/2026-07-07-blocked-by-storage-spike.md` (the finding)

**Interfaces:**

- Produces: a decision — `STORAGE = "native"` or `STORAGE = "reflink"`. Tasks 4/6 implement the reflink baseline unconditionally; the native path is added only if the spike says `native`.

- [ ] **Step 1: Probe the GraphQL schema for the dependency mutation**

Run (as a human or USER agent with a token):

```bash
vrg-gh api graphql -f query='query { __type(name: "Mutation") { fields { name } } }' \
  --jq '.data.__type.fields[].name' | grep -i -E "depend|block"
```

Expected: either a mutation name (e.g. an `addIssueDependency`-style field) or no output.

- [ ] **Step 2: If a mutation exists, attempt a real write on two throwaway issues**

Create two scratch issues under a scratch epic, attempt to set one blocked-by the other via the discovered mutation, then read it back via a query. Record whether write and read both succeed with the App token (not just a PAT).

- [ ] **Step 3: Write the finding and decide**

Document the result. Decision rule: use `native` only if **both** write and read succeed under the App-installation token used by `vrg-gh`; otherwise `reflink`. State the decision explicitly.

- [ ] **Step 4: Commit**

```bash
vrg-git add docs/specs/2026-07-07-blocked-by-storage-spike.md
vrg-commit --type docs --scope validation --message "record blocked-by storage spike finding" --body "Decides native issue-dependency vs Blocked-by reflink for validation task deps. Ref #<TASK1>."
```

---

### Task 2: Add the `validation` label to the registry — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/data/labels.json` (add the `validation` entry)
- Test: `tests/vergil_tooling/test_labels.py` (create if absent, else extend)

**Interfaces:**

- Produces: a canonical `validation` label provisioned by `vrg-ensure-label --sync`.

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_labels.py
from vergil_tooling.lib.labels import load_labels

def test_validation_label_is_registered():
    labels = load_labels()
    names = {entry["name"] for entry in labels["labels"]}
    assert "validation" in names
    entry = next(e for e in labels["labels"] if e["name"] == "validation")
    assert entry["description"]  # non-empty, explains the gate
    assert entry["color"]
```

- [ ] **Step 2: Run it, verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_labels.py::test_validation_label_is_registered -v`
Expected: FAIL (`validation` not in names).

- [ ] **Step 3: Add the label entry**

Add to the `labels` array in `src/vergil_tooling/data/labels.json`:

```json
{
  "name": "validation",
  "color": "5319e7",
  "description": "Post-merge validation task: run the checklist, record PASS/FAIL as a comment; never PR-workable, never auto-closed."
}
```

- [ ] **Step 4: Run test, verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_labels.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/data/labels.json tests/vergil_tooling/test_labels.py
vrg-commit --type feat --scope labels --message "add validation label to the registry" --body "Marks post-merge validation tasks. Ref #<TASK2>."
```

---

### Task 3: PR-workability guard — refuse `validation` tasks in submit/report-ready — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add `is_validation_task`, near `is_epic_linkage:311`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py:241` (refuse alongside the epic-linkage refusal)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py:45` (refuse in report-ready)
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/test_vrg_submit_pr.py`, `tests/vergil_tooling/pr_workflow/test_submission.py`

**Interfaces:**

- Consumes: `epics.parse_issue_ref`, `epics._labels` (Task uses existing helpers).
- Produces: `epics.is_validation_task(ref: str, *, default_repo: str) -> bool` — the single source of truth, mirroring `is_epic_linkage`.

- [ ] **Step 1: Write the failing test for the library predicate**

```python
# tests/vergil_tooling/test_epics.py
from unittest.mock import patch
from vergil_tooling.lib import epics

def test_is_validation_task_true_when_labeled():
    with patch.object(epics, "_labels", return_value={"validation", "task"}):
        assert epics.is_validation_task("owner/repo#7", default_repo="owner/repo") is True

def test_is_validation_task_false_when_unlabeled():
    with patch.object(epics, "_labels", return_value={"task"}):
        assert epics.is_validation_task("owner/repo#7", default_repo="owner/repo") is False

def test_is_validation_task_false_on_unparseable_ref():
    assert epics.is_validation_task("not-a-ref", default_repo="owner/repo") is False
```

- [ ] **Step 2: Run, verify fail**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k is_validation_task -v`
Expected: FAIL (`AttributeError: module 'vergil_tooling.lib.epics' has no attribute 'is_validation_task'`).

- [ ] **Step 3: Implement the predicate**

In `src/vergil_tooling/lib/epics.py`, directly after `is_epic_linkage`:

```python
def is_validation_task(ref: str, *, default_repo: str) -> bool:
    """True if *ref* is a validation task, so PR tooling must refuse it.

    Single source of truth for "is this a validation task?", shared by
    ``vrg-submit-pr`` and ``vrg-pr-workflow report-ready``. Self-scoping: an
    unparseable ref is never a validation task and returns False. A validation
    task is proven by post-merge execution recorded as a comment — it has no
    code PR — so the PR path is refused before any work begins.
    """
    try:
        issue = parse_issue_ref(ref, default_repo=default_repo)
    except ValueError:
        return False
    return "validation" in _labels(issue)
```

- [ ] **Step 4: Run, verify pass**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k is_validation_task -v`
Expected: PASS.

- [ ] **Step 5: Write the failing guard test for `vrg-submit-pr`**

```python
# tests/vergil_tooling/test_vrg_submit_pr.py
from unittest.mock import patch
from vergil_tooling.bin import vrg_submit_pr

def test_submit_pr_refuses_validation_task(capsys):
    with patch("vergil_tooling.bin.vrg_submit_pr.epics.is_validation_task", return_value=True), \
         patch("vergil_tooling.bin.vrg_submit_pr.epics.is_epic_linkage", return_value=False):
        rc = vrg_submit_pr.main(["--issue", "42"])  # adapt to real arg surface
    assert rc == 1
    assert "validation task" in capsys.readouterr().err.lower()
```

(Adapt the invocation to `vrg_submit_pr`'s real entry — read `main`/`parse_args` around line 241 first; the assertion is the contract.)

- [ ] **Step 6: Run, verify fail; then add the refusal**

In `src/vergil_tooling/bin/vrg_submit_pr.py`, beside the existing `if epics.is_epic_linkage(issue_ref, ...):` block (~line 241):

```python
    if epics.is_validation_task(issue_ref, default_repo=github.current_repo()):
        print(
            "vrg-submit-pr: refusing a validation task — a validation task is "
            "not PR-workable. Run the validation and record PASS/FAIL as a "
            "comment (see the issue-validate skill); do not open a PR against it.",
            file=sys.stderr,
        )
        return 1
```

- [ ] **Step 7: Write + implement the same refusal for `vrg-pr-workflow report-ready`**

In `src/vergil_tooling/bin/vrg_pr_workflow.py` (~line 45, beside `links_epic = epics.is_epic_linkage(...)`):

```python
        if epics.is_validation_task(issue, default_repo=github.current_repo()):
            print(
                "vrg-pr-workflow: refusing report-ready for a validation task — "
                "it is not PR-workable. Record PASS/FAIL as an issue comment "
                "(issue-validate skill) instead of preparing a PR.",
                file=sys.stderr,
            )
            return 1
```

Add a mirror test in `tests/vergil_tooling/pr_workflow/test_submission.py` asserting `report-ready` returns non-zero and mentions "validation task".

- [ ] **Step 8: Run the guard tests, verify pass**

Run: `uv run pytest tests/vergil_tooling/test_vrg_submit_pr.py tests/vergil_tooling/pr_workflow/test_submission.py -k validation -v`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epics.py src/vergil_tooling/bin/vrg_submit_pr.py src/vergil_tooling/bin/vrg_pr_workflow.py tests/vergil_tooling/
vrg-commit --type feat --scope pr-guard --message "refuse PR development against validation tasks" --body "A validation task is not PR-workable: vrg-submit-pr and vrg-pr-workflow report-ready both refuse it, making both auto-close vectors moot. Ref #<TASK3>."
```

---

### Task 4: Dependency link plumbing (`Blocked-by:` reflink baseline) — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add `_BLOCKED_BY_RE`, `blockers_of`, `all_blockers_closed`, `render_blocked_by`)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: Task 1's `STORAGE` decision (reflink baseline always; native additive if `native`).
- Produces:
  - `epics.render_blocked_by(deps: list[IssueRef]) -> str` — the `Blocked-by:` lines to embed in a validation task body.
  - `epics.blockers_of(task: IssueRef) -> list[IssueRef]` — the deps a task is blocked by (native preferred, reflink fallback), mirroring `child_states`.
  - `epics.all_blockers_closed(task: IssueRef) -> bool` — True iff every blocker is CLOSED (used by rollup runnable/blocked).

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_epics.py
from unittest.mock import patch
from vergil_tooling.lib import epics
from vergil_tooling.lib.epics import IssueRef, ChildState

def test_render_blocked_by_emits_one_line_per_dep():
    out = epics.render_blocked_by([IssueRef("o", "r", 5), IssueRef("o", "r", 8)])
    assert "Blocked-by: o/r#5" in out
    assert "Blocked-by: o/r#8" in out

def test_blockers_of_parses_reflink_body():
    body = "Do the thing.\nBlocked-by: o/r#5\nBlocked-by: o/r#8\n"
    with patch.object(epics, "_native_blockers", return_value=[]), \
         patch.object(epics.github, "read_output", return_value=body):
        refs = epics.blockers_of(IssueRef("o", "r", 42))
    assert IssueRef("o", "r", 5) in refs and IssueRef("o", "r", 8) in refs

def test_all_blockers_closed_true_when_all_closed():
    with patch.object(epics, "blockers_of", return_value=[IssueRef("o","r",5)]), \
         patch.object(epics, "_issue_state", return_value="CLOSED"):
        assert epics.all_blockers_closed(IssueRef("o","r",42)) is True

def test_all_blockers_closed_false_when_any_open():
    with patch.object(epics, "blockers_of", return_value=[IssueRef("o","r",5)]), \
         patch.object(epics, "_issue_state", return_value="OPEN"):
        assert epics.all_blockers_closed(IssueRef("o","r",42)) is False
```

- [ ] **Step 2: Run, verify fail**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "blocked_by or blockers" -v`
Expected: FAIL (attributes undefined).

- [ ] **Step 3: Implement (reflink baseline)**

In `src/vergil_tooling/lib/epics.py`:

```python
_BLOCKED_BY_RE = re.compile(
    r"^\s*Blocked-by:\s*([A-Za-z0-9._-]+)/([A-Za-z0-9._-]+)#(\d+)\s*$",
    re.MULTILINE,
)

def render_blocked_by(deps: list[IssueRef]) -> str:
    """Render the ``Blocked-by:`` reflink lines for a validation task body."""
    return "".join(f"Blocked-by: {d.slug}\n" for d in deps)

def _native_blockers(task: IssueRef) -> list[IssueRef]:
    """Native issue-dependency blockers, or [] when unsupported (per Task 1)."""
    return []  # replaced with the real query only if the spike selected native

def _reflink_blockers(task: IssueRef) -> list[IssueRef]:
    body = github.read_output("api", _issue_endpoint(task), "--jq", ".body") or ""
    return [
        IssueRef(owner=m.group(1), repo=m.group(2), number=int(m.group(3)))
        for m in _BLOCKED_BY_RE.finditer(body)
    ]

def blockers_of(task: IssueRef) -> list[IssueRef]:
    """Deps *task* is blocked by: native preferred, reflink fallback."""
    native = _native_blockers(task)
    return native if native else _reflink_blockers(task)

def all_blockers_closed(task: IssueRef) -> bool:
    """True iff every blocker of *task* is CLOSED (empty ⇒ True, i.e. runnable)."""
    return all(_issue_state(dep) == "CLOSED" for dep in blockers_of(task))
```

If Task 1 selected `native`, additionally implement the real mutation (to set) and query (in `_native_blockers`); the reflink stays as the fallback.

- [ ] **Step 4: Run, verify pass**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "blocked_by or blockers" -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epics.py tests/vergil_tooling/test_epics.py
vrg-commit --type feat --scope epics --message "add Blocked-by dependency reflink plumbing" --body "render/parse/evaluate validation-task blockers; reflink baseline mirrors the Parent: sub-issue fallback. Ref #<TASK4>."
```

---

### Task 5: `vrg-issue-create --kind validation` — label + scaffold + blocked-by — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_issue_create.py` (add `--kind`, `--blocked-by`; stamp scaffold; apply label)
- Create: `src/vergil_tooling/data/validation_task_body.md` (the scaffold template)
- Test: `tests/vergil_tooling/test_vrg_issue_create.py`

**Interfaces:**

- Consumes: `epics.render_blocked_by` (Task 4), the `validation` label (Task 2).
- Produces: `vrg-issue-create --epic <E> --kind validation --title ... --blocked-by owner/repo#N [--blocked-by ...]` → an issue with the `validation` label, the scaffold body (with rendered `Blocked-by:` lines and a commit-floor precondition placeholder resolved from the deps), linked under the epic.

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_vrg_issue_create.py
from unittest.mock import patch
from vergil_tooling.bin import vrg_issue_create

def test_validation_kind_applies_label_and_scaffold():
    captured = {}
    def fake_create(**kwargs):
        captured.update(kwargs)
        return "https://github.com/o/r/issues/9"
    with patch("vergil_tooling.bin.vrg_issue_create.github.create_issue", side_effect=fake_create), \
         patch("vergil_tooling.bin.vrg_issue_create.epics.resolve_epic_ref"), \
         patch("vergil_tooling.bin.vrg_issue_create.epics.add_child"):
        rc = vrg_issue_create.main([
            "--epic", "o/.github#1", "--repo", "o/r", "--kind", "validation",
            "--title", "Validate cold rebuild", "--blocked-by", "o/r#5",
        ])
    assert rc == 0
    assert "validation" in captured["labels"]
    body = captured["body"]
    assert "Blocked-by: o/r#5" in body
    assert "## Preconditions" in body  # generic, author-defined self-check
    assert "## Results" in body  # PASS/FAIL template present
```

- [ ] **Step 2: Run, verify fail**

Run: `uv run pytest tests/vergil_tooling/test_vrg_issue_create.py -k validation -v`
Expected: FAIL (`--kind` unrecognized).

- [ ] **Step 3: Create the scaffold template**

`src/vergil_tooling/data/validation_task_body.md`:

```markdown
{intro}

## Preconditions (self-check — run first, do not fabricate)

Declare this task's preconditions here. They are **author-defined and generic** —
a machine-checkable probe (a health/status command, a check that the dependency
change is deployed) *or* a human-attested statement (e.g. "the target has been
rebuilt to include the dependency below"). The framework prescribes no mechanism.

- <precondition 1 — probe or human-attested>
- <precondition 2 — optional>

{blocked_by}

If any precondition fails: comment "blocked: preconditions not met — <which>"
and stop. Do not run the checklist. Never fabricate a result.

## Commands to run

<concrete checklist — fill in at creation/refinement time>

## Acceptance criteria

<explicit pass/fail conditions>

## Results (post as a comment, then close on PASS only)

- Outcome: PASS | FAIL
- Evidence: <command output / observations>
- On FAIL: file follow-on fix task(s); leave this task and the epic OPEN.
```

- [ ] **Step 4: Implement the CLI path**

In `parse_args`, add:

```python
    parser.add_argument("--kind", choices=("task", "validation"), default="task",
                        help="Issue kind; 'validation' stamps the validation scaffold + label")
    parser.add_argument("--blocked-by", action="append", default=[], metavar="REF",
                        help="Dependency this validation task is blocked by (repeatable)")
```

In `main`, when `args.kind == "validation"`:

```python
    labels = [*args.label, "validation"]
    deps = [epics.parse_issue_ref(r, default_repo=repo) for r in args.blocked_by]
    template = _read_data("validation_task_body.md")
    body = template.format(
        intro=(args.body or f"Post-merge validation for {args.title}."),
        blocked_by=epics.render_blocked_by(deps),
    )
```

then proceed through the existing create+link path with `labels`/`body`. (Non-validation kind is unchanged.)

- [ ] **Step 5: Run, verify pass**

Run: `uv run pytest tests/vergil_tooling/test_vrg_issue_create.py -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_issue_create.py src/vergil_tooling/data/validation_task_body.md tests/vergil_tooling/test_vrg_issue_create.py
vrg-commit --type feat --scope issue-create --message "add --kind validation path" --body "Stamps the validation label, executable scaffold, and Blocked-by reflinks. Ref #<TASK5>."
```

---

### Task 6: Validation-aware rollup/audit — `vergil-tooling`

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (validation-pending state; runnable/blocked; report-only invariant)
- Modify: `src/vergil_tooling/lib/roadmap.py` (surface a `validation_pending` signal on `EpicSummary` if needed for rendering)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `tests/vergil_tooling/test_vrg_epic_rollup.py`

**Interfaces:**

- Consumes: `epics.child_states`, `epics.blockers_of`, `epics.all_blockers_closed` (Task 4), the `validation` label (Task 2).
- Produces:
  - `epic_audit.validation_status(epic) -> {"pending": [...], "runnable": [...], "blocked": [...]}`
  - a report-only invariant: `epic_audit.closed_validation_without_pass(org) -> list[str]` (validation tasks CLOSED with no PASS comment).

- [ ] **Step 1: Write the failing test for rollup-holds-epic-open**

```python
# tests/vergil_tooling/test_vrg_epic_rollup.py
from unittest.mock import patch
from vergil_tooling.lib import epics
from vergil_tooling.lib.epics import IssueRef, ChildState

def test_rollup_holds_epic_open_while_validation_child_open():
    epic = IssueRef("o", ".github", 1)
    children = [ChildState(IssueRef("o","r",5), "CLOSED"),
                ChildState(IssueRef("o",".github",7), "OPEN")]  # the validation task
    with patch.object(epics, "parent_of", return_value=epic), \
         patch.object(epics, "is_epic", return_value=True), \
         patch.object(epics, "_labels", return_value=set()), \
         patch.object(epics, "child_states", return_value=children), \
         patch.object(epics.github, "run") as run:
        epics.rollup(IssueRef("o","r",5))
    run.assert_not_called()  # epic NOT closed: a validation child is still open
```

- [ ] **Step 2: Run, verify it PASSES already (regression guard)**

Run: `uv run pytest tests/vergil_tooling/test_vrg_epic_rollup.py -k validation_child -v`
Expected: PASS — `all_children_closed` is already False with an open child. This test *locks in* the safe-by-construction behavior; if it fails, rollup logic regressed. Keep it.

- [ ] **Step 3: Write the failing test for validation_status**

```python
# tests/vergil_tooling/test_epic_audit.py
from unittest.mock import patch
from vergil_tooling.lib import epic_audit, epics
from vergil_tooling.lib.epics import IssueRef, ChildState

def test_validation_status_classifies_runnable_vs_blocked():
    epic = IssueRef("o", ".github", 1)
    val_runnable = IssueRef("o", ".github", 7)
    val_blocked = IssueRef("o", ".github", 8)
    children = [ChildState(val_runnable, "OPEN"), ChildState(val_blocked, "OPEN"),
                ChildState(IssueRef("o","r",5), "CLOSED")]
    def labels(ref):
        return {"validation"} if ref.number in (7, 8) else {"task"}
    def runnable(ref):
        return ref.number == 7  # 7's blockers closed, 8's not
    with patch.object(epic_audit.epics, "child_states", return_value=children), \
         patch.object(epic_audit.epics, "_labels", side_effect=labels), \
         patch.object(epic_audit.epics, "all_blockers_closed", side_effect=runnable):
        status = epic_audit.validation_status(epic)
    assert [r.number for r in status["runnable"]] == [7]
    assert [r.number for r in status["blocked"]] == [8]
    assert {r.number for r in status["pending"]} == {7, 8}
```

- [ ] **Step 4: Run, verify fail; then implement**

In `src/vergil_tooling/lib/epic_audit.py`:

```python
def validation_status(epic: epics.IssueRef) -> dict[str, list[epics.IssueRef]]:
    """Outstanding validation children of *epic*, split runnable vs blocked.

    A validation child is any OPEN child carrying the ``validation`` label.
    Runnable ⇔ all its blockers are closed; otherwise blocked. The union is the
    "validation-pending" set that keeps the epic honest even when all code tasks
    are closed.
    """
    pending, runnable, blocked = [], [], []
    for child in epics.child_states(epic):
        if child.state != "OPEN" or "validation" not in epics._labels(child.ref):
            continue
        pending.append(child.ref)
        (runnable if epics.all_blockers_closed(child.ref) else blocked).append(child.ref)
    return {"pending": pending, "runnable": runnable, "blocked": blocked}
```

- [ ] **Step 5: Add the report-only invariant + wire into `render`**

Add `closed_validation_without_pass(org)` (search closed `validation`-labelled issues; flag any whose comments contain no `Outcome: PASS`), and extend `render` (and `vrg_epic_audit.main`) to print a **"Validation pending"** section (runnable/blocked) and a report-only **invariant** section. Add tests asserting both render blocks appear. Keep it report-only — never auto-close.

- [ ] **Step 6: Run the audit tests, verify pass**

Run: `uv run pytest tests/vergil_tooling/test_epic_audit.py -v`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epic_audit.py src/vergil_tooling/lib/roadmap.py tests/vergil_tooling/
vrg-commit --type feat --scope epic-audit --message "make rollup/audit validation-aware" --body "Report code-complete/validation-pending, classify runnable vs blocked, flag validation tasks closed without a PASS comment (report-only). Ref #<TASK6>."
```

---

### Task 7: `epic-create` skill — validation doctrine + default seeding — `vergil-claude-plugin`

**Files:**

- Modify: `skills/epic-create/SKILL.md`

**Deliverable:** doctrine prose (no code). "Tests" = self-review against the spec + the dogfood in Task 10.

- [ ] **Step 1:** Add the **validation task type** to the bookend architecture section: what it is (a PR that can't merge), that it is not PR-workable, and that it gates rollup by staying open.
- [ ] **Step 2:** Add the **judgment doctrine** — when to add one (cold rebuild / live-lab / deploy smoke test; infra/provisioning ⇒ cold-rebuild validation by default), when not (docs, pipeline-covered code), and the granularity choices (1:1 / epoch / epic bookend).
- [ ] **Step 3:** Document **creation** via `vrg-issue-create --kind validation --blocked-by …`, and the **anatomy** (preconditions self-check with the deployed-version commit-floor, commands, acceptance, results template).
- [ ] **Step 4:** State the **default seeding** rule for infra epics at step 2 of the workflow.
- [ ] **Step 5:** Self-review against `spec.md`; commit.

```bash
vrg-git add skills/epic-create/SKILL.md
vrg-commit --type docs --scope epic-create --message "add post-merge validation doctrine" --body "Validation task type, judgment doctrine, default infra seeding, creation + anatomy. Ref #<TASK7>."
```

---

### Task 8: `issue-implement` skill — discover-and-create + wrong-skill redirect — `vergil-claude-plugin`

**Files:**

- Modify: `skills/issue-implement/SKILL.md`

- [ ] **Step 1:** Add the **wrong-skill redirect** at the top of the workflow: on picking up a `validation`-labelled issue, stop and redirect to `issue-validate` — this issue is not code-implementation.
- [ ] **Step 2:** Add the **discover-and-create** rule: when pipeline unit/integration tests cannot prove acceptance (the change must be merged + deployed to validate), mint a validation follow-on with `vrg-issue-create --kind validation --blocked-by <this task>` (1:1 or attach to an existing epoch validation).
- [ ] **Step 3:** Add the **don't-declare-done** rule: never treat the task/epic as complete while a paired or epoch validation is outstanding.
- [ ] **Step 4:** Self-review against `spec.md`; commit.

```bash
vrg-git add skills/issue-implement/SKILL.md
vrg-commit --type docs --scope issue-implement --message "discover/create validation follow-ons + redirect" --body "Redirect validation-labelled issues to issue-validate; mint follow-ons on discovery; don't declare done while a validation is open. Ref #<TASK8>."
```

---

### Task 9: `issue-validate` skill (new) — the execution lifecycle — `vergil-claude-plugin`

**Files:**

- Create: `skills/issue-validate/SKILL.md`
- Modify: the plugin's skill manifest/registry if one enumerates skills (check `.claude-plugin/` and any marketplace index)

**Deliverable:** a complete new skill with its own design. This is the heaviest skill task.

- [ ] **Step 1:** Frontmatter (`name: issue-validate`, description with triggers: "validate issue", "run the validation task", picking up a `validation`-labelled issue).
- [ ] **Step 2:** **Preflight / reachability gate** — run the body's *declared* preconditions first, whatever form they take (a machine probe or a human-attested statement — the framework prescribes no mechanism). If any is unmet: comment `blocked: preconditions not met — <which>` and STOP. Never fabricate, never partial-fake.
- [ ] **Step 3:** **Run** the recorded commands; capture evidence.
- [ ] **Step 4:** **Record** PASS/FAIL as an issue comment from the results template.
- [ ] **Step 5:** **Close semantics** — PASS ⇒ comment then close (`vrg-gh issue close`). FAIL ⇒ comment evidence, file follow-on fix task(s) via `vrg-issue-create`, and **leave the task and epic OPEN**. Triage-discovered problems are out-of-band new issues, not edits here.
- [ ] **Step 6:** **Boundaries** — no worktree, no commits, no PR; explicitly contrast with `issue-implement`.
- [ ] **Step 7:** Self-review against `spec.md`; commit.

```bash
vrg-git add skills/issue-validate/
vrg-commit --type feat --scope issue-validate --message "add issue-validate skill" --body "Execute a validation task: reachability gate, run, record PASS/FAIL comment, close on PASS / hold open on FAIL. No code, no PR. Ref #<TASK9>."
```

---

### Task 10: Dogfood validation (the epic's own validation follow-on) — `vergil-project/.github`

**Deliverable:** a `validation`-labelled task (created *by the new tooling*) that exercises the framework end-to-end without a live lab — proving the pattern eats its own dogfood. This is the epic's epoch validation, blocked-by Tasks 3, 5, 6.

- [ ] **Step 1:** After Tasks 2–6 merge, create it with the new path:

```bash
vrg-issue-create --epic vergil-project/.github#115 --repo vergil-project/.github \
  --kind validation --title "Validate: post-merge validation framework end-to-end" \
  --blocked-by vergil-project/vergil-tooling#<TASK3> \
  --blocked-by vergil-project/vergil-tooling#<TASK5> \
  --blocked-by vergil-project/vergil-tooling#<TASK6>
```

- [ ] **Step 2:** In its body, record the checklist: (a) `vrg-submit-pr` and `report-ready` both refuse a `validation`-labelled issue; (b) `vrg-issue-create --kind validation` stamps label + scaffold + `Blocked-by:`; (c) `vrg-epic-audit` reports the validation as runnable once its blockers close, blocked before; (d) `vrg-epic-rollup` holds a test epic open while a validation child is open.
- [ ] **Step 3:** Run it via `issue-validate` (Task 9); post PASS/FAIL; close on PASS only.

---

## Documentation

Human-facing **site docs** (`vergil-tooling/docs/site/…`) for the validation
task type are **authored-and-verified by the doc-review closing bookend
(`.github#118`)**, not a separate task. That bookend is a reminder/sanity gate —
"before you're done, make sure the docs reflect what changed, authoring what's
missing" — a judgment call between agent and human, not a hard separate
deliverable.

## Task ordering / dependencies

- **Task 1** (spike) gates Tasks 4 and 6's storage.
- **Task 2** (label) gates Tasks 3, 5, 6.
- **Task 4** (deps) gates Tasks 5 and 6.
- **Tasks 3, 5, 6** are the tooling core; **Task 10** (dogfood) is `blocked-by` them.
- **Tasks 7, 8, 9** (skills) can proceed in parallel with tooling but reference the `vrg-issue-create --kind validation` surface (Task 5) and the scaffold anatomy.

## Self-review — spec coverage

- Concept / invariants → Tasks 3 (not PR-workable), 4 (blocked-by), 6 (gates rollup) ✅
- Validation task anatomy (preconditions, commit-floor, results) → Task 5 scaffold + Task 9 execution ✅
- FAIL semantics → Task 9 close semantics ✅
- Provisioning boundary (human-driven; self-check only) → Task 5 scaffold + Task 9 preflight ✅
- Tooling (issue-create / guard / rollup-audit) → Tasks 2–6 ✅
- Skills (epic-create / issue-implement / issue-validate) → Tasks 7–9 ✅
- `blocked-by` feasibility + fallback → Task 1 + Task 4 reflink baseline ✅
- Testing (guard both paths, rollup-holds-open, invariant flag) → Tasks 3, 6 ✅
- Automation trajectory (runnable/blocked signal) → Task 6 `validation_status` ✅ (runner itself is a non-goal)
