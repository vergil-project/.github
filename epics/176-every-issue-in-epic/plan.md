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
- Reuse, don't duplicate: `ensure_adhoc_epic`, `add_child`, `parent_of`, `is_epic`, `_labels`, `resolve_epic_home`, `parse_issue_ref`, `IssueRef`, `github.create_issue`, and the `is_operational` / `_reject_if_operational_task` refusal pattern already exist. The **org-wide reconciliation sweep already exists too**: `vrg-epic-audit` (org-scanning; automated writes gated by `VRG_EPIC_SWEEP`) driven by the reusable `ops-epic-sweep` workflow with an org-wide app token — extend these, do not build a parallel sweep.

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

### Task 2: `vrg-intake-create` born-links intake issues [`vergil-tooling`]

> Sequencing: Task 7 renames the module to `vrg_intake_create.py` first; this born-link edit applies there (the logic is filename-agnostic).

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_intake_create.py` (post-Task-7 name)
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

### Task 4: Extend `vrg-epic-audit` with an org-wide orphan-attach [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (add `orphan_drift(org)` + `attach_orphans(...)`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (run the orphan-attach in the sweep, gated by `VRG_EPIC_SWEEP`)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `tests/vergil_tooling/test_vrg_epic_audit.py`

**Rationale (reuse, not rebuild):** `vrg-epic-audit` already scans the whole org (`task_drift(since, org=org)`, `epic_drift(org, home=home)`, `close_drift(...)`) and already carries the automated-write authorization `VRG_EPIC_SWEEP` that `ops-epic-sweep` sets. Folding orphan-attach here means the existing scheduled sweep heals orphans in the same pass it closes drift — no new command, no new workflow.

**Interfaces:**

- Consumes: the existing org-issue listing the audit uses, `epics.is_sweepable`, `epics.parent_of`, `epics.resolve_orphan_epic`, `epics.add_child`, `epics.IssueRef`.
- Produces: `epic_audit.orphan_drift(org: str) -> list[IssueRef]` (open, non-PR, non-epic, unparented issues org-wide); `epic_audit.attach_orphans(orphans: list[IssueRef]) -> int` (attaches each via the resolver, returns count).

- [ ] **Step 1: Failing test — `orphan_drift` returns only sweepable, unparented issues**

```python
# tests/vergil_tooling/test_epic_audit.py
from vergil_tooling.lib import epic_audit
from vergil_tooling.lib.epics import IssueRef

def test_orphan_drift_filters(monkeypatch):
    rows = [
        {"number": 10, "state": "OPEN", "isPullRequest": False, "labels": [],
         "repository": {"nameWithOwner": "o/r"}},
        {"number": 11, "state": "OPEN", "isPullRequest": False,
         "labels": [{"name": "epic"}], "repository": {"nameWithOwner": "o/r"}},
    ]
    monkeypatch.setattr(epic_audit, "_org_open_issues", lambda org: rows)
    monkeypatch.setattr(epic_audit.epics, "parent_of", lambda ref: None)
    got = epic_audit.orphan_drift("o")
    assert [r.number for r in got] == [10]  # epic row excluded by is_sweepable
```

- [ ] **Step 2: Run — expect FAIL** (`orphan_drift` undefined).

- [ ] **Step 3: Implement `orphan_drift` + `attach_orphans`**

```python
def orphan_drift(org: str) -> list[IssueRef]:
    """Open, non-PR, non-epic issues org-wide with no epic parent."""
    out: list[IssueRef] = []
    for row in _org_open_issues(org):
        if not epics.is_sweepable(row):
            continue
        owner, bare = row["repository"]["nameWithOwner"].split("/", 1)
        ref = epics.IssueRef(owner=owner, repo=bare, number=int(row["number"]))
        if epics.parent_of(ref) is None:
            out.append(ref)
    return out

def attach_orphans(orphans: list[IssueRef]) -> int:
    for ref in orphans:
        epics.add_child(epics.resolve_orphan_epic(ref), ref)
    return len(orphans)
```

(`_org_open_issues(org)` reuses the org-wide `gh issue list --owner <org> --state open --json number,state,labels,isPullRequest,repository` pattern the audit already uses for drift; if no such helper exists yet, extract one so drift and orphan scans share it.)

- [ ] **Step 4: Wire into the sweep.** In `vrg_epic_audit.py` `main`, after `close_drift(...)` runs under the sweep authorization, call `attach_orphans(orphan_drift(org))` and add the count to the Markdown output. In read-only preview (no `VRG_EPIC_SWEEP`, no `--close`), *list* orphans without attaching — mirroring how `--close` previews drift.

- [ ] **Step 5: Tests** — preview lists but does not attach; authorized sweep attaches; idempotent re-run attaches 0 (all parented). Then `vrg-container-run -- vrg-validate` → green.

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope epics --message "attach orphaned issues in the epic-audit sweep" --body "vrg-epic-audit gains an org-wide orphan_drift + attach_orphans pass: open, non-PR, non-epic, unparented issues are attached under their standing epic via resolve_orphan_epic, in the same authorized sweep that closes drift (VRG_EPIC_SWEEP). Preview lists; sweep attaches; idempotent. Ref <TASK-4-ISSUE>."
```

---

### Task 5: Extend `ops-epic-sweep` to run the orphan-attach [`vergil-actions`]

The existing reusable `ops-epic-sweep.yml` already runs `vrg-epic-audit --close --window-days` with an org-wide app token and `VRG_EPIC_SWEEP=1`. Because Task 4 folds orphan-attach into that same authorized sweep, **the workflow needs no structural change** — the orphan-attach runs whenever the scheduled `vrg-epic-audit` sweep runs. There is **no new workflow and no per-repo rollout**; the sweep is already centralized and org-wide.

- [ ] **Step 1:** Update the run step's name/comment in `ops-epic-sweep.yml` to "reconcile drift + attach orphans" so the workflow's intent matches its new behavior.
- [ ] **Step 2:** If Task 4 gates orphan-attach behind its own flag (optional), add that flag to the `vrg-epic-audit` run line; otherwise the YAML change is comment-only.
- [ ] **Step 3:** `actionlint` clean; a `workflow_dispatch` dry run confirms it attaches a planted orphan and no-ops on a second run.
- [ ] **Step 4:** Commit in `vergil-actions`.

(Runtime correctness — the schedule actually heals a planted orphan — is proven by the operational task below, not by unit tests.)

---

### Task 6: Operational — run the one-shot org-wide sweep [`vergil-project/.github`, `--kind validation`]

Filed via: `vrg-issue-create --epic vergil-project/.github#176 --repo vergil-project/.github --kind validation --title "Validate: no orphaned issues remain org-wide after the sweep" --blocked-by <TASK-1> --blocked-by <TASK-4> --blocked-by <DEPLOY-TASK>`

- [ ] **Step 1:** Precondition self-check: Tasks 1 and 4 merged and the updated `vergil-tooling` deployed (probe `vrg-epic-audit --help`).
- [ ] **Step 2:** Run the org-wide sweep once with the sweep authorization (`VRG_EPIC_SWEEP=1 vrg-epic-audit --close` from inside a repo in each org, or trigger `ops-epic-sweep` via `workflow_dispatch`).
- [ ] **Step 3:** Re-run (idempotence) → 0 further attachments; confirm no open, non-PR, non-epic, unparented issue remains org-wide.
- [ ] **Step 4:** Record `Outcome: SUCCESS` with the tally as a comment (closes the task; on failure, record FAILURE and leave open).

---

### Task 7: Rename the intake tool family + generalize `intake-review` [`vergil-tooling` + plugin/skills repo]

**Files:**

- Rename: `src/vergil_tooling/bin/vrg_triage_create.py → vrg_intake_create.py`; in `pyproject.toml` register `vrg-intake-create` and keep a deprecated `vrg-triage-create` alias entry point for one release
- Modify: the `vrg-gh` denial message that names `vrg-triage-create`; docs/CLAUDE.md references
- Rename (plugin/skills repo): `triage-review → intake-review`, `triage-capture → intake-capture` skill dirs + SKILL.md; generalize `intake-review`'s listing query and promote-in-place label removal to all three kinds
- Test: `tests/vergil_tooling/test_vrg_intake_create.py` (+ alias test)

**Interfaces:**

- Produces: console scripts `vrg-intake-create` (canonical) and `vrg-triage-create` (deprecated alias → warns on stderr, forwards to `vrg_intake_create.main`).

- [ ] **Step 1: Failing test — the deprecated alias warns and forwards**

```python
from unittest.mock import patch
from vergil_tooling.bin import vrg_intake_create as ic

def test_deprecated_alias_warns_and_forwards(capsys):
    with patch.object(ic, "main", return_value=0) as m:
        rc = ic.deprecated_triage_create_main(["--title", "x"])
    assert rc == 0
    assert "deprecated" in capsys.readouterr().err.lower()
    m.assert_called_once()
```

- [ ] **Step 2: Run — expect FAIL.**

- [ ] **Step 3:** Move the module to `vrg_intake_create.py`; add `deprecated_triage_create_main(argv)` that prints a deprecation warning to stderr and returns `main(argv)`. Register both scripts in `pyproject.toml` (`vrg-intake-create = "…vrg_intake_create:main"`, `vrg-triage-create = "…vrg_intake_create:deprecated_triage_create_main"`). Update the `vrg-gh` denial string to name `vrg-intake-create`.

- [ ] **Step 4:** Generalize `intake-review` (its SKILL.md): the listing query unions all three intake labels (`--label triage --label idea --label research`), and the promote-in-place step removes *whichever* intake label the issue carries (`vrg-gh issue edit <N> --remove-label <kind>`). Rename `triage-capture → intake-capture` and `triage-review → intake-review` dirs; update cross-references (CLAUDE.md skill listing, any skill naming them).

- [ ] **Step 5:** Run tests — expect PASS; `vrg-container-run -- vrg-validate` → green.

- [ ] **Step 6: Commit**

```bash
vrg-commit --type refactor --scope intake --message "rename the triage-* tool family to intake-* and generalize the grooming exit to all three kinds" --body "vrg-triage-create -> vrg-intake-create (one-release deprecated alias), triage-review -> intake-review (generalized to triage/idea/research), triage-capture -> intake-capture. The three kind labels are unchanged. Fixes the naming mistake and makes the grooming exit strip whichever intake label is present. Ref <TASK-7-ISSUE>."
```

*(Sequencing: Task 7's console-script rename lands before Task 2's born-link edit and Task 3's refusal, so those reference the command by its new name `vrg-intake-create`; Task 4's audit/resolver work is independent of the rename.)*

---

## Self-Review

- **Spec coverage:** invariant/precision → Task 1 (`is_sweepable`, epic exclusion) + restated in spec; intake standing epic → Task 1; shared resolver → Task 1; client born-link → Task 2; non-PR-workable intake / closes #175 → Task 3; org-wide orphan sweep → Task 4 (extend `vrg-epic-audit`); scheduled sweep → Task 5 (extend `ops-epic-sweep`); one-shot migration → Task 6; rename family + generalize `intake-review` (grooming exit for all three kinds) → Task 7. All spec sections map to a task.
- **Reuse check (alignment finding [2]):** Tasks 4/5 extend the existing `vrg-epic-audit` / `ops-epic-sweep` rather than building a parallel command + per-repo workflow; the "per-repo self-sweep" earlier draft is dropped in favor of the existing org-wide-token central sweep.
- **Placeholders:** none — every code step shows real code grounded in existing signatures (`ensure_adhoc_epic`, `add_child`, `is_operational_task`, `task_drift`/`epic_drift`/`close_drift`).
- **Type consistency:** `IssueRef` used uniformly; `ensure_intake_epic`/`resolve_orphan_epic`/`is_sweepable`/`is_intake_issue`/`orphan_drift`/`attach_orphans` names match across producer (Tasks 1/3/4) and consumers (Tasks 2/4). `resolve_orphan_epic` takes an `IssueRef`; `is_intake_issue` takes a `str` ref (str twin), matching the existing `is_operational`/`is_operational_task` split.
- **Coverage note:** each new branch has a paired test; `is_sweepable`'s four exclusion branches are covered by the parametrized Task 1 Step 5 test.
