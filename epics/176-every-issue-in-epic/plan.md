# Every Issue Belongs to an Epic — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enforce, across every managed repo, the invariant that every issue is either an epic or has an epic parent — by containing intake kinds in a standing intake epic, refusing PRs against un-groomed intake issues, and self-healing orphans with a scheduled per-repo sweep.

**Architecture:** All epic-model logic extends `vergil_tooling/lib/epics.py`, which already carries the ad-hoc standing-epic pattern (`ensure_adhoc_epic`), sub-issue linking (`add_child`), and label predicates (`is_epic`, `is_operational`). We generalize the ad-hoc epic into a shared `_ensure_standing_epic`, add an intake twin, add one orphan resolver + sweep-domain predicate, and reuse the existing operational-task PR-refusal pattern for intake kinds. A new `vrg-epic-sweep` command drives both the one-shot migration and a scheduled GitHub Action (in `vergil-actions`, rolled into each repo) that runs authenticated as the app.

**Tech Stack:** Python 3.12+ (`vergil-tooling`), `pytest` + 100% branch coverage, `gh`/GraphQL via `vergil_tooling.lib.github`, GitHub Actions reusable workflows (`vergil-actions`).

## Global Constraints

- Validation is exactly `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here). Never run linters/formatters standalone; for TDD red/green run focused tests via `vrg-container-run -- uv run pytest <path> -v`.
- **100% branch coverage is enforced.** Every new line and branch needs a test; avoid unreachable branches.
- Git/GitHub only via `vrg-git` / `vrg-gh`; commits via `vrg-commit --type <t> --scope <s> --message <m> [--body <b>]`.
- One PR per task; each task is filed and lands in the repo named in its heading (placement law). No PR `Closes` an issue across repos.
- Reuse, don't duplicate: `ensure_adhoc_epic`, `add_child`, `parent_of`, `is_epic`, `_labels`, `resolve_epic_home`, `parse_issue_ref`, `IssueRef`, `github.create_issue`, and the `is_operational` / `_reject_if_operational_task` refusal pattern already exist.

---

### Task 1: Intake epic, orphan resolver, and sweep-domain predicate [`vergil-tooling`]

**Files:**
- Modify: `src/vergil_tooling/lib/epics.py` (generalize `ensure_adhoc_epic`; add intake epic, labels, predicates, resolver, sweep predicate)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**
- Consumes: existing `resolve_epic_home`, `IssueRef`, `github.create_issue`, `github.read_json`, `add_child`, `parent_of`, `is_epic`, `_labels`.
- Produces:
  - `ensure_intake_epic(target_repo: str) -> IssueRef`
  - `_INTAKE_LABELS: set[str]` = `{"triage", "idea", "research"}`
  - `is_intake(ref: IssueRef) -> bool`
  - `resolve_orphan_epic(ref: IssueRef) -> IssueRef` (intake-labelled → intake epic; else → ad-hoc epic)
  - `is_sweepable(issue: dict) -> bool` (True iff open, not a PR, not `epic`-labelled — parent-emptiness is checked separately by the sweep)

- [ ] **Step 1: Failing test — generalized standing-epic helper reuses ad-hoc behavior**

```python
# tests/vergil_tooling/test_epics.py
from vergil_tooling.lib import epics
from vergil_tooling.lib.epics import IssueRef

def test_ensure_intake_epic_reuses_existing(monkeypatch):
    monkeypatch.setattr(epics, "resolve_epic_home", lambda org, bare: "vergil-project/.github")
    monkeypatch.setattr(
        epics.github, "read_json",
        lambda *a, **k: [{"number": 200, "title": "Epic (intake): vergil-tooling"}],
    )
    ref = epics.ensure_intake_epic("vergil-project/vergil-tooling")
    assert ref == IssueRef(owner="vergil-project", repo=".github", number=200)
```

- [ ] **Step 2: Run it — expect FAIL (no `ensure_intake_epic`)**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py::test_ensure_intake_epic_reuses_existing -v`
Expected: FAIL (AttributeError: module has no attribute 'ensure_intake_epic').

- [ ] **Step 3: Generalize the ad-hoc epic into `_ensure_standing_epic`, add the intake twin**

Refactor `ensure_adhoc_epic` to delegate to a private generic, then add the intake epic beside it:

```python
_ADHOC_EPIC_TITLE_PREFIX = "Epic (ad hoc): "
_ADHOC_EPIC_LABELS = ("epic", "ad-hoc")
_ADHOC_EPIC_BODY = (
    "Perpetual umbrella for ad-hoc work in {repo}. Created and reused "
    "idempotently; tasks routed to the ad-hoc epic are linked here.\n"
)
_INTAKE_EPIC_TITLE_PREFIX = "Epic (intake): "
_INTAKE_EPIC_LABELS = ("epic", "intake")
_INTAKE_EPIC_BODY = (
    "Perpetual umbrella for intake (triage/idea/research) in {repo}. Created and "
    "reused idempotently; triage-review grooms items out of it into finite epics.\n"
)

def _ensure_standing_epic(
    target_repo: str, *, title_prefix: str, labels: tuple[str, ...], body: str
) -> IssueRef:
    """Return target_repo's standing epic of this kind in its resolved home,
    creating it if absent. Idempotent; two sharing the title is an error."""
    if "/" not in target_repo:
        raise ValueError(f"cannot resolve repo for standing epic (repo={target_repo!r})")
    owner, bare = target_repo.split("/", 1)
    home = resolve_epic_home(owner, bare)
    home_repo = home.split("/", 1)[1]
    title = f"{title_prefix}{bare}"
    raw: Any = github.read_json(
        "issue", "list", "--repo", home,
        *[arg for label in labels for arg in ("--label", label)],
        "--state", "open", "--json", "number,title",
    )
    rows = (
        [r for r in raw if isinstance(r, dict) and r.get("title") == title]
        if isinstance(raw, list) else []
    )
    if len(rows) > 1:
        nums = ", ".join(f"#{r['number']}" for r in rows)
        raise ValueError(f"multiple standing epics titled {title!r} in {home} ({nums})")
    if rows:
        return IssueRef(owner=owner, repo=home_repo, number=int(rows[0]["number"]))
    url = github.create_issue(repo=home, title=title, body=body.format(repo=target_repo), labels=list(labels))
    number = int(url.rstrip("/").rsplit("/", 1)[-1])
    return IssueRef(owner=owner, repo=home_repo, number=number)

def ensure_adhoc_epic(target_repo: str) -> IssueRef:
    return _ensure_standing_epic(
        target_repo, title_prefix=_ADHOC_EPIC_TITLE_PREFIX,
        labels=_ADHOC_EPIC_LABELS, body=_ADHOC_EPIC_BODY,
    )

def ensure_intake_epic(target_repo: str) -> IssueRef:
    return _ensure_standing_epic(
        target_repo, title_prefix=_INTAKE_EPIC_TITLE_PREFIX,
        labels=_INTAKE_EPIC_LABELS, body=_INTAKE_EPIC_BODY,
    )
```

- [ ] **Step 4: Run — expect PASS.** Also run the existing `ensure_adhoc_epic` tests to prove the refactor is behavior-preserving: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "adhoc or intake" -v` → PASS.

- [ ] **Step 5: Failing tests — intake predicate + resolver + sweep predicate**

```python
def test_is_intake_true_for_triage(monkeypatch):
    monkeypatch.setattr(epics, "_labels", lambda ref: {"triage"})
    assert epics.is_intake(IssueRef("o", "r", 1)) is True

def test_resolve_orphan_epic_intake_to_intake(monkeypatch):
    monkeypatch.setattr(epics, "_labels", lambda ref: {"idea"})
    monkeypatch.setattr(epics, "ensure_intake_epic", lambda repo: IssueRef("o", ".github", 9))
    assert epics.resolve_orphan_epic(IssueRef("o", "r", 1)) == IssueRef("o", ".github", 9)

def test_resolve_orphan_epic_other_to_adhoc(monkeypatch):
    monkeypatch.setattr(epics, "_labels", lambda ref: {"bug"})
    monkeypatch.setattr(epics, "ensure_adhoc_epic", lambda repo: IssueRef("o", ".github", 8))
    assert epics.resolve_orphan_epic(IssueRef("o", "r", 1)) == IssueRef("o", ".github", 8)

def test_is_sweepable_excludes_epics_prs_closed():
    base = {"state": "OPEN", "labels": [], "isPullRequest": False}
    assert epics.is_sweepable(base) is True
    assert epics.is_sweepable({**base, "state": "CLOSED"}) is False
    assert epics.is_sweepable({**base, "isPullRequest": True}) is False
    assert epics.is_sweepable({**base, "labels": [{"name": "epic"}]}) is False
```

- [ ] **Step 6: Run — expect FAIL** (`is_intake`, `resolve_orphan_epic`, `is_sweepable` undefined).

- [ ] **Step 7: Implement the predicates and resolver**

```python
_INTAKE_LABELS: set[str] = {"triage", "idea", "research"}

def is_intake(ref: IssueRef) -> bool:
    """True if *ref* carries any intake label (triage/idea/research)."""
    return bool(_labels(ref) & _INTAKE_LABELS)

def resolve_orphan_epic(ref: IssueRef) -> IssueRef:
    """The standing epic an orphan belongs to: intake-labelled → the repo's
    intake epic; everything else → its ad-hoc epic. Shared by the sweep and the
    scheduled workflow so they cannot diverge."""
    target = f"{ref.owner}/{ref.repo}"
    if is_intake(ref):
        return ensure_intake_epic(target)
    return ensure_adhoc_epic(target)

def is_sweepable(issue: dict) -> bool:
    """True iff *issue* (a gh-list row) is in the sweep domain: open, not a pull
    request, not an epic. Parent-emptiness is checked separately (needs a graphql
    call), so the sweep filters with is_sweepable then skips any parent_of != None."""
    if issue.get("state", "").upper() != "OPEN":
        return False
    if issue.get("isPullRequest"):
        return False
    names = {str(label.get("name", "")) for label in (issue.get("labels") or [])}
    return "epic" not in names
```

- [ ] **Step 8: Run — expect PASS.** Then `vrg-container-run -- vrg-validate` (full gate, 100% coverage) → green.

- [ ] **Step 9: Commit**

```bash
vrg-commit --type feat --scope epics --message "add intake standing epic, orphan resolver, and sweep-domain predicate" --body "Generalize ensure_adhoc_epic into _ensure_standing_epic and add ensure_intake_epic (Epic (intake): <repo>, perpetual). Add is_intake, resolve_orphan_epic (intake->intake epic, else->ad-hoc), and is_sweepable (open, non-PR, non-epic). Shared by born-linking, the sweep, and the scheduled workflow. Ref <TASK-1-ISSUE>."
```

---

### Task 2: `vrg-triage-create` born-links intake issues [`vergil-tooling`]

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_triage_create.py`
- Test: `tests/vergil_tooling/test_vrg_triage_create.py`

**Interfaces:**
- Consumes: `epics.ensure_intake_epic`, `epics.add_child`, `epics.parse_issue_ref`, `github.create_issue`.
- Produces: no new public API; behavior change — the created intake issue is attached under the repo's intake epic before the command returns.

- [ ] **Step 1: Failing test — the created issue is attached to the intake epic**

```python
# tests/vergil_tooling/test_vrg_triage_create.py
from unittest.mock import patch
from vergil_tooling.bin import vrg_triage_create as tc
from vergil_tooling.lib.epics import IssueRef

def test_intake_issue_is_born_linked():
    with (
        patch.object(tc.github, "detect_org", return_value="vergil-project"),
        patch.object(tc.github, "create_issue",
                     return_value="https://github.com/vergil-project/.github/issues/300"),
        patch.object(tc.epics, "ensure_intake_epic",
                     return_value=IssueRef("vergil-project", ".github", 176)) as ensure,
        patch.object(tc.epics, "add_child") as add_child,
    ):
        assert tc.main(["--title", "x", "--kind", "idea"]) == 0
    ensure.assert_called_once_with("vergil-project/.github")
    (epic, task), _ = add_child.call_args
    assert epic == IssueRef("vergil-project", ".github", 176)
    assert task.number == 300
```

- [ ] **Step 2: Run — expect FAIL** (no `epics` import / no attach).

- [ ] **Step 3: Attach after create**

In `vrg_triage_create.py`, import `epics`, and after the `create_issue` call:

```python
from vergil_tooling.lib import epics, github
...
    url = github.create_issue(
        repo=repo, title=args.title, body=args.body, body_file=args.body_file,
        labels=labels, assignees=args.assignee,
    )
    issue = epics.parse_issue_ref(url, default_repo=repo)
    epic = epics.ensure_intake_epic(repo)
    epics.add_child(epic, issue)
    print(f"Created {url} ({args.kind}); linked under {epic.slug}.")
    return 0
```

Update the module docstring: intake issues are no longer "unlinked" — they are born-linked under the repo's perpetual intake epic and groomed out later.

- [ ] **Step 4: Run — expect PASS.** Then `vrg-container-run -- vrg-validate` → green (update any existing test asserting the old "unlinked"/print text).

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope triage --message "born-link intake issues under the repo's intake epic" --body "vrg-triage-create no longer creates unlinked issues: each triage/idea/research issue is attached under ensure_intake_epic(repo). Closes the last sanctioned orphan-creation path. Ref <TASK-2-ISSUE>."
```

---

### Task 3: Refuse PRs against un-groomed intake issues [`vergil-tooling`] — closes `vergil-project/.github#175`

**Files:**
- Modify: `src/vergil_tooling/lib/epics.py` (add `is_intake_issue(ref: str, *, default_repo)`)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (add `_reject_if_intake` beside `_reject_if_operational_task`, call it)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (mirror the refusal in `report-ready`, at the existing operational check ~line 101)
- Test: `tests/vergil_tooling/test_epics.py`, `test_vrg_submit_pr.py`, `test_vrg_pr_workflow.py`

**Interfaces:**
- Consumes: `epics.parse_issue_ref`, `epics.is_intake`, `github.current_repo`.
- Produces: `epics.is_intake_issue(ref: str, *, default_repo: str) -> bool` (str-ref twin of `is_intake`, self-scoping like `is_operational_task`).

- [ ] **Step 1: Failing test — `is_intake_issue` parses then checks; unparseable is False**

```python
def test_is_intake_issue_true(monkeypatch):
    monkeypatch.setattr(epics, "is_intake", lambda ref: True)
    assert epics.is_intake_issue("#5", default_repo="o/r") is True

def test_is_intake_issue_unparseable_is_false(monkeypatch):
    def boom(*a, **k): raise ValueError("nope")
    monkeypatch.setattr(epics, "parse_issue_ref", boom)
    assert epics.is_intake_issue("garbage", default_repo="o/r") is False
```

- [ ] **Step 2: Run — expect FAIL.**

- [ ] **Step 3: Implement `is_intake_issue`** (mirror `is_operational_task`)

```python
def is_intake_issue(ref: str, *, default_repo: str) -> bool:
    """True if *ref* is an intake issue (triage/idea/research) — not PR-workable.
    Self-scoping: an unparseable ref is never intake and returns False."""
    try:
        issue = parse_issue_ref(ref, default_repo=default_repo)
    except ValueError:
        return False
    return is_intake(issue)
```

- [ ] **Step 4: Failing test — `vrg-submit-pr` refuses an intake issue**

```python
def test_submit_pr_refuses_intake(monkeypatch):
    monkeypatch.setattr(submit.epics, "is_intake_issue", lambda ref, default_repo: True)
    with pytest.raises(SystemExit, match="intake"):
        submit._reject_if_intake("#5")
```

- [ ] **Step 5: Run — expect FAIL.**

- [ ] **Step 6: Add `_reject_if_intake` and call it** (beside `_reject_if_operational_task`)

```python
def _reject_if_intake(issue_ref: str) -> None:
    """Abort if the linkage is an un-groomed intake issue (triage/idea/research).
    It is a seed, not a task: groom it via triage-review into a real task first."""
    if epics.is_intake_issue(issue_ref, default_repo=github.current_repo()):
        raise SystemExit(
            "vrg-submit-pr: --issue is an intake issue (triage/idea/research), which "
            "is not PR-workable; groom it into a real task with triage-review first."
        )
```

Call `_reject_if_intake(issue_ref)` wherever `_reject_if_operational_task(issue_ref)` is called.

- [ ] **Step 7: Mirror in `report-ready`** — at `vrg_pr_workflow.py` ~line 101, beside the operational check, refuse when `epics.is_intake_issue(issue, default_repo=github.current_repo())` with the same message. Add a `test_vrg_pr_workflow.py` test asserting `report-ready` on an intake issue raises `WorkflowError`/exits non-zero.

- [ ] **Step 8: Run focused tests — expect PASS.** Then `vrg-container-run -- vrg-validate` → green.

- [ ] **Step 9: Commit**

```bash
vrg-commit --type feat --scope pr --message "refuse PRs against un-groomed intake issues" --body "vrg-submit-pr and vrg-pr-workflow report-ready reject a triage/idea/research issue as not PR-workable, mirroring the operational-task refusal; groom via triage-review first. Ref vergil-project/.github#175."
```

*(Note: this task's PR links `Ref vergil-project/.github#175` — cross-repo, so #175 is closed manually or by a same-repo note, per the placement law; do not use Closes across repos.)*

---

### Task 4: `vrg-epic-sweep` — the orphan sweep command [`vergil-tooling`]

**Files:**
- Create: `src/vergil_tooling/bin/vrg_epic_sweep.py`
- Modify: `pyproject.toml` (register the `vrg-epic-sweep` console script)
- Test: `tests/vergil_tooling/test_vrg_epic_sweep.py`

**Interfaces:**
- Consumes: `epics.is_sweepable`, `epics.parent_of`, `epics.resolve_orphan_epic`, `epics.add_child`, `epics.IssueRef`, `github.read_json`.
- Produces: `main(argv) -> int`; `--repo owner/name` (required, repeatable) sweeps each named repo; prints a per-repo tally; idempotent.

- [ ] **Step 1: Failing test — sweeps open orphans, skips already-parented, attaches via resolver**

```python
# tests/vergil_tooling/test_vrg_epic_sweep.py
from unittest.mock import patch
from vergil_tooling.bin import vrg_epic_sweep as sweep
from vergil_tooling.lib.epics import IssueRef

def test_sweep_attaches_open_orphans():
    rows = [
        {"number": 10, "state": "OPEN", "isPullRequest": False, "labels": []},  # orphan -> attach
        {"number": 11, "state": "OPEN", "isPullRequest": False, "labels": [{"name": "epic"}]},  # epic -> skip
    ]
    def fake_parent(ref):
        return None if ref.number == 10 else IssueRef("o", ".github", 1)
    with (
        patch.object(sweep.github, "read_json", return_value=rows),
        patch.object(sweep.epics, "parent_of", side_effect=fake_parent),
        patch.object(sweep.epics, "resolve_orphan_epic",
                     return_value=IssueRef("o", ".github", 99)),
        patch.object(sweep.epics, "add_child") as add_child,
    ):
        assert sweep.main(["--repo", "o/r"]) == 0
    assert add_child.call_count == 1
    (epic, task), _ = add_child.call_args
    assert (epic.number, task.number) == (99, 10)
```

- [ ] **Step 2: Run — expect FAIL** (module does not exist).

- [ ] **Step 3: Implement the sweep**

```python
"""Sweep a repo's open orphan issues under their standing epic (issue #176).

Enumerates open, non-PR, non-epic issues with no epic parent and attaches each
via epics.resolve_orphan_epic. Idempotent and re-runnable; the one-shot migration
and the scheduled workflow both call it. Never closes or relabels."""
from __future__ import annotations
import argparse, sys
from vergil_tooling.lib import epics, github

def _sweep_repo(repo: str) -> int:
    rows = github.read_json(
        "issue", "list", "--repo", repo, "--state", "open", "--limit", "1000",
        "--json", "number,state,labels,isPullRequest",
    ) or []
    attached = 0
    for row in rows:
        if not epics.is_sweepable(row):
            continue
        owner, bare = repo.split("/", 1)
        ref = epics.IssueRef(owner=owner, repo=bare, number=int(row["number"]))
        if epics.parent_of(ref) is not None:
            continue
        epic = epics.resolve_orphan_epic(ref)
        epics.add_child(epic, ref)
        print(f"  attached {ref.slug} -> {epic.slug}")
        attached += 1
    print(f"{repo}: attached {attached} orphan(s).")
    return attached

def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="vrg-epic-sweep",
        description="Attach a repo's open orphan issues under their standing epic.")
    parser.add_argument("--repo", action="append", required=True,
        help="Target repo owner/name (repeatable)")
    args = parser.parse_args(argv)
    for repo in args.repo:
        _sweep_repo(repo)
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

Register in `pyproject.toml` `[project.scripts]`: `vrg-epic-sweep = "vergil_tooling.bin.vrg_epic_sweep:main"`.

- [ ] **Step 4: Add tests for idempotent re-run (all rows already parented → 0 attached) and multi-repo tally.** Run focused tests — expect PASS. Then `vrg-container-run -- vrg-validate` → green.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add vrg-epic-sweep orphan sweep command" --body "vrg-epic-sweep attaches a repo's open, non-PR, non-epic, unparented issues under their standing epic via resolve_orphan_epic. Idempotent; drives both the one-shot migration and the scheduled workflow. Ref <TASK-4-ISSUE>."
```

---

### Task 5: Reusable scheduled sweep workflow + rollout [`vergil-actions`]

**Files:**
- Create: `.github/workflows/epic-sweep.yml` (reusable, `on: workflow_call` + a thin `on: schedule` caller pattern per `vergil-actions` conventions)
- Modify: the standard repo scaffold so each managed repo gains a caller workflow `.github/workflows/epic-sweep.yml` invoking the reusable one on a daily schedule
- Test: follow `vergil-actions` workflow-test conventions (lint via `actionlint`; a smoke job asserting `vrg-epic-sweep --repo ${{ github.repository }}` runs)

**Interfaces:**
- Consumes: `vrg-epic-sweep` (installed via the standard vergil-tooling install step used by other reusable workflows); the app credentials (same GitHub App installation the tooling uses) provided as workflow secrets, because attaching a member-repo issue to its epic in `.github` is a cross-repo write the default `GITHUB_TOKEN` cannot do.
- Produces: a scheduled job that runs `vrg-epic-sweep --repo <this-repo>` daily.

- [ ] **Step 1:** Author the reusable workflow: check out, install vergil-tooling (reuse the existing install action other `vergil-actions` workflows use), authenticate as the app (mint an installation token — reuse the org's existing app-token action/secret), then run `vrg-epic-sweep --repo "${{ github.repository }}"`.
- [ ] **Step 2:** Author the per-repo caller (`on: schedule: - cron: "17 6 * * *"` + `on: workflow_dispatch`) that calls the reusable workflow; add it to the scaffold so new repos inherit it.
- [ ] **Step 3:** `actionlint` clean (via that repo's validation) and a manual `workflow_dispatch` dry run against one repo confirms it attaches (or no-ops) correctly.
- [ ] **Step 4:** Commit in `vergil-actions`; roll the caller into each managed repo (own small PRs, or the scaffold-sync mechanism).

*(This task's runtime correctness — that the schedule actually heals a planted orphan — is proven by the operational task below, not by unit tests.)*

---

### Task 6: Operational — run the one-shot sweep across all managed repos [`vergil-project/.github`, `--kind validation`]

Filed via: `vrg-issue-create --epic vergil-project/.github#176 --repo vergil-project/.github --kind validation --title "Validate: no orphaned issues remain org-wide after the sweep" --blocked-by <TASK-1> --blocked-by <TASK-4> --blocked-by <DEPLOY-TASK>`

- [ ] **Step 1:** Precondition self-check: Tasks 1 and 4 merged and the updated `vergil-tooling` installed/deployed (attest or probe `vrg-epic-sweep --help`).
- [ ] **Step 2:** Enumerate managed repos across every org; run `vrg-epic-sweep --repo <each>` (one invocation may list many `--repo`).
- [ ] **Step 3:** Re-run once (idempotence check) and confirm 0 further attachments; confirm no open, non-PR, non-epic, unparented issue remains in any managed repo.
- [ ] **Step 4:** Record `Outcome: SUCCESS` with the per-repo tally as a comment (closes this operational task; on any failure, record FAILURE and leave open).

---

## Self-Review

- **Spec coverage:** invariant/precision → Task 1 (`is_sweepable`, epic exclusion) + restated in spec; intake standing epic → Task 1; shared resolver → Task 1; client born-link → Task 2; non-PR-workable intake / closes #175 → Task 3; sweep command → Task 4; scheduled self-sweep authenticated as app → Task 5; one-shot migration → Task 6. All spec sections map to a task.
- **Placeholders:** none — every code step shows real code grounded in existing signatures (`ensure_adhoc_epic`, `add_child`, `is_operational_task`).
- **Type consistency:** `IssueRef` used uniformly; `ensure_intake_epic`/`resolve_orphan_epic`/`is_sweepable`/`is_intake_issue` names match across producer (Task 1/3) and consumers (Tasks 2/4). `resolve_orphan_epic` takes an `IssueRef`; `is_intake_issue` takes a `str` ref (str twin), matching the existing `is_operational`/`is_operational_task` split.
- **Coverage note:** each new branch has a paired test; `is_sweepable`'s four exclusion branches are covered by the parametrized Task 1 Step 5 test.
