# Continuous ad-hoc epic archiving — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Continuously drain closed children out of each live ad-hoc epic into per-quarter archive epics bucketed by close date, so ad-hoc epics stay small without ever losing history.

**Architecture:** All logic lands in `vergil-tooling`. Shared functions in `lib/epics.py` do the draining; three call sites drive it — the `issues.closed` event path (`epics.rollup`, steady state), the daily `vrg-epic-audit --close` sweep (backstop + backlog + past-archive close), and a new `vrg-adhoc-epic archive` CLI (manual/preview/validation). No new workflows or crons: the event and sweep wiring already invoke these CLIs.

**Tech Stack:** Python 3.12+, `argparse`, `unittest.mock`, `pytest`. GitHub access via `lib/github.py` (`gh` CLI + GraphQL sub-issue API). Epic model in `lib/epics.py`.

## Global Constraints

- **Line length: 100** (`ruff`, `pyproject.toml`). Copy verbatim; wrap comments/code to fit.
- **100% test coverage required** (`vrg-validate` enforces it — the repo's `test` gate fails below 100%). Every new branch needs a test.
- **Validation is the only gate:** `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here). Never run individual linters.
- **All git via `vrg-git`, commits via `vrg-commit`.** Work inside the worktree assigned at execution time; never edit at the repo root.
- **Archive title format (exact):** `Epic (ad hoc): <bare> — <YYYY>-Q<n>` — the separator is a space, U+2014 em-dash, space (`" — "`).
- **Archive labels:** `["epic", "ad-hoc"]` (same as the live ad-hoc epic).
- **Quarters are computed in UTC.** `Q = (month - 1) // 3 + 1`.
- **The live ad-hoc epic is never renamed, recreated, or closed** — it only loses re-parented children.

---

## File structure

**Modified:**

- `src/vergil_tooling/lib/epics.py` — `ChildState.closed_at`, `closedAt` in the sub-issue query + reflink, quarter helpers, archive title constants/regex, `_find_epic_by_title`, `find_adhoc_epic`, `ensure_adhoc_archive`, `list_open_adhoc_archives`, `DrainPlan`, `plan_adhoc_drain`/`apply_adhoc_drain`/`drain_adhoc_repo`/`drain_adhoc_org`, `_issue_closed_at`, and the `rollup()` hook.
- `src/vergil_tooling/lib/github.py` — `list_org_repos`.
- `src/vergil_tooling/bin/vrg_adhoc_epic.py` — `archive` subcommand.
- `src/vergil_tooling/bin/vrg_epic_audit.py` — org-wide drain in the `--close` sweep.

**Tests (mirror existing `unittest.mock.patch`-by-dotted-path style):**

- `tests/vergil_tooling/test_epics.py` (extend), `tests/vergil_tooling/test_vrg_adhoc_epic.py` (extend), `tests/vergil_tooling/test_vrg_epic_audit.py` (extend), `tests/vergil_tooling/test_github.py` (extend for `list_org_repos`).

---

## Task 1: Thread `closedAt` through child enumeration

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`ChildState` ~37-48; `_SUBISSUES_QUERY` ~99-109; `_native_child_states` ~160-168; `_reflink_child_states` ~181-216)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Produces: `ChildState(ref: IssueRef, state: str, title: str = "", closed_at: str = "")` — `closed_at` is the ISO-8601 `closedAt` string (empty when open).

- [ ] **Step 1: Write the failing test**

```python
def test_child_states_native_includes_closed_at() -> None:
    data = {
        "node": {
            "subIssues": {
                "nodes": [
                    {"number": 101, "state": "CLOSED", "title": "t",
                     "closedAt": "2026-08-01T10:00:00Z", "repository": _repo_node("org", "repo-a")},
                    {"number": 102, "state": "OPEN", "title": "u",
                     "closedAt": None, "repository": _repo_node("org", "repo-b")},
                ]
            }
        }
    }
    with (
        patch("vergil_tooling.lib.epics._node_id", return_value="NODE"),
        patch("vergil_tooling.lib.github.graphql", return_value=data),
    ):
        result = epics.child_states(EPIC)
    assert result[0].closed_at == "2026-08-01T10:00:00Z"
    assert result[1].closed_at == ""
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_epics.py::test_child_states_native_includes_closed_at -v`
Expected: FAIL — `ChildState.__init__() got an unexpected keyword 'closed_at'` or attribute error.

- [ ] **Step 3: Implement**

Add the field to `ChildState`:

```python
@dataclass(frozen=True)
class ChildState:
    ref: IssueRef
    state: str  # "OPEN" | "CLOSED"
    title: str = ""
    closed_at: str = ""  # ISO-8601 closedAt; "" when open
```

Add `closedAt` to the query node selection (`_SUBISSUES_QUERY`):

```python
        nodes { number state title closedAt repository { name owner { login } } }
```

In `_native_child_states`, set it when building each `ChildState`:

```python
            closed_at=str(n.get("closedAt") or ""),
```

In `_reflink_child_states`, add `closedAt` to the `--json` field list and set `closed_at=str(row.get("closedAt") or "")` on the built `ChildState`.

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k closed_at -v`
Expected: PASS. Also run the full `test_epics.py` to confirm existing `ChildState(...)` equality assertions (which omit `closed_at`, defaulting to `""`) still pass.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "thread closedAt through child enumeration (#238)" \
  --body "Add ChildState.closed_at, fetched via the sub-issue query and reflink fallback, so archiving can bucket closed children by close-quarter."
```

---

## Task 2: Quarter helpers

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add near the top: `from datetime import datetime`; helpers after the dataclasses)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Produces:
  - `quarter_of(closed_at: str) -> str` — `"2026-08-01T10:00:00Z"` → `"2026-Q3"`; raises `ValueError` on empty/unparseable.
  - `current_quarter(now: datetime) -> str` — `datetime(2026, 8, 8, tzinfo=UTC)` → `"2026-Q3"`.

- [ ] **Step 1: Write the failing tests**

```python
import pytest
from datetime import datetime, timezone

@pytest.mark.parametrize("iso,expected", [
    ("2026-01-31T00:00:00Z", "2026-Q1"),
    ("2026-03-31T23:59:59Z", "2026-Q1"),
    ("2026-04-01T00:00:00Z", "2026-Q2"),
    ("2026-07-15T12:00:00Z", "2026-Q3"),
    ("2026-12-31T23:59:59Z", "2026-Q4"),
    ("2026-08-01T10:00:00+00:00", "2026-Q3"),
])
def test_quarter_of(iso: str, expected: str) -> None:
    assert epics.quarter_of(iso) == expected

def test_quarter_of_rejects_empty() -> None:
    with pytest.raises(ValueError):
        epics.quarter_of("")

def test_current_quarter() -> None:
    assert epics.current_quarter(datetime(2026, 8, 8, tzinfo=timezone.utc)) == "2026-Q3"
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k quarter -v`
Expected: FAIL — `module 'epics' has no attribute 'quarter_of'`.

- [ ] **Step 3: Implement**

```python
def _quarter_str(dt: datetime) -> str:
    return f"{dt.year}-Q{(dt.month - 1) // 3 + 1}"


def quarter_of(closed_at: str) -> str:
    """Return the ``YYYY-Qn`` quarter of an ISO-8601 timestamp (UTC)."""
    if not closed_at:
        raise ValueError("quarter_of: empty timestamp")
    dt = datetime.fromisoformat(closed_at.replace("Z", "+00:00"))
    return _quarter_str(dt)


def current_quarter(now: datetime) -> str:
    """Return the ``YYYY-Qn`` quarter containing *now*."""
    return _quarter_str(now)
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k quarter -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add close-quarter helpers (#238)" \
  --body "quarter_of() buckets an ISO closedAt into YYYY-Qn (UTC); current_quarter() for the past-archive close rule."
```

---

## Task 3: Archive/live epic finders

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add title constants/regex ~near `_ADHOC_EPIC_*`; `_find_epic_by_title`; `find_adhoc_epic`; refactor `ensure_adhoc_epic` to use it; `ensure_adhoc_archive`; `list_open_adhoc_archives`)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Produces:
  - `find_adhoc_epic(target_repo: str) -> IssueRef | None` — the live canonical ad-hoc epic, or None (does **not** create).
  - `ensure_adhoc_archive(target_repo: str, quarter: str) -> IssueRef` — the `— <quarter>` archive, create-if-missing.
  - `list_open_adhoc_archives(home: str) -> list[tuple[IssueRef, str]]` — open stamped archives in *home* with their quarter.
- Constants: `_ADHOC_ARCHIVE_TITLE = _ADHOC_EPIC_TITLE_PREFIX + "{bare} — {quarter}"`; `_ADHOC_ARCHIVE_RE = re.compile(r"^Epic \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$")`.

- [ ] **Step 1: Write the failing tests**

```python
def test_find_adhoc_epic_returns_none_when_absent() -> None:
    with (
        patch("vergil_tooling.lib.epics.resolve_epic_home", return_value="org/.github"),
        patch("vergil_tooling.lib.github.read_json", return_value=[]),
    ):
        assert epics.find_adhoc_epic("org/tooling") is None

def test_ensure_adhoc_archive_creates_stamped_title() -> None:
    created = "https://github.com/org/.github/issues/88"
    with (
        patch("vergil_tooling.lib.epics.resolve_epic_home", return_value="org/.github"),
        patch("vergil_tooling.lib.github.read_json", return_value=[]),
        patch("vergil_tooling.lib.github.create_issue", return_value=created) as mock_create,
    ):
        ref = epics.ensure_adhoc_archive("org/tooling", "2026-Q3")
    assert ref == IssueRef("org", ".github", 88)
    assert mock_create.call_args.kwargs["title"] == "Epic (ad hoc): tooling — 2026-Q3"
    assert mock_create.call_args.kwargs["labels"] == ["epic", "ad-hoc"]

def test_list_open_adhoc_archives_parses_quarter() -> None:
    rows = [
        {"number": 88, "title": "Epic (ad hoc): tooling — 2026-Q2"},
        {"number": 90, "title": "Epic (ad hoc): tooling"},          # live, not an archive
        {"number": 91, "title": "Epic (ad hoc): tooling — 2026-Q3"},
    ]
    with patch("vergil_tooling.lib.github.read_json", return_value=rows):
        got = epics.list_open_adhoc_archives("org/.github")
    assert (IssueRef("org", ".github", 88), "2026-Q2") in got
    assert (IssueRef("org", ".github", 91), "2026-Q3") in got
    assert all(q for _, q in got) and len(got) == 2
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "adhoc_archive or find_adhoc or open_adhoc" -v`
Expected: FAIL — attributes not defined.

- [ ] **Step 3: Implement**

Add constants beside `_ADHOC_EPIC_*`:

```python
_ADHOC_ARCHIVE_RE = re.compile(r"^Epic \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$")
```

Extract the search core and build the finders:

```python
def _find_epic_by_title(home: str, title: str) -> IssueRef | None:
    owner, home_repo = home.split("/", 1)
    raw: Any = github.read_json(
        "issue", "list", "--repo", home, "--label", "epic", "--label", "ad-hoc",
        "--state", "open", "--json", "number,title",
    )
    rows = [r for r in raw if isinstance(r, dict) and r.get("title") == title] if isinstance(raw, list) else []
    if len(rows) > 1:
        nums = ", ".join(f"#{r['number']}" for r in rows)
        raise ValueError(f"multiple epics titled {title!r} in {home} ({nums}) — pass an explicit --epic")
    if rows:
        return IssueRef(owner=owner, repo=home_repo, number=int(rows[0]["number"]))
    return None


def find_adhoc_epic(target_repo: str) -> IssueRef | None:
    """Return the live canonical ad-hoc epic for *target_repo*, or None. Does not create."""
    if "/" not in target_repo:
        raise ValueError(f"cannot resolve repo for ad-hoc epic (repo={target_repo!r})")
    owner, bare = target_repo.split("/", 1)
    home = resolve_epic_home(owner, bare)
    return _find_epic_by_title(home, f"{_ADHOC_EPIC_TITLE_PREFIX}{bare}")


def ensure_adhoc_archive(target_repo: str, quarter: str) -> IssueRef:
    """Return *target_repo*'s ``— <quarter>`` archive epic, creating it if absent."""
    owner, bare = target_repo.split("/", 1)
    home = resolve_epic_home(owner, bare)
    home_repo = home.split("/", 1)[1]
    title = f"{_ADHOC_EPIC_TITLE_PREFIX}{bare} — {quarter}"
    existing = _find_epic_by_title(home, title)
    if existing is not None:
        return existing
    url = github.create_issue(
        repo=home, title=title,
        body=f"Ad-hoc work in {target_repo} finished in {quarter}. Managed automatically.\n",
        labels=list(_ADHOC_EPIC_LABELS),
    )
    return IssueRef(owner=owner, repo=home_repo, number=int(url.rstrip("/").rsplit("/", 1)[-1]))


def list_open_adhoc_archives(home: str) -> list[tuple[IssueRef, str]]:
    """Open ``ad-hoc`` archive epics in *home* (title carries a ``— YYYY-Qn`` stamp) with their quarter."""
    owner, home_repo = home.split("/", 1)
    raw: Any = github.read_json(
        "issue", "list", "--repo", home, "--label", "epic", "--label", "ad-hoc",
        "--state", "open", "--json", "number,title",
    )
    out: list[tuple[IssueRef, str]] = []
    for r in raw if isinstance(raw, list) else []:
        m = _ADHOC_ARCHIVE_RE.match(str(r.get("title", ""))) if isinstance(r, dict) else None
        if m:
            out.append((IssueRef(owner, home_repo, int(r["number"])), m.group("quarter")))
    return out
```

Then refactor `ensure_adhoc_epic` to reuse `_find_epic_by_title` (replace its inline search with `found = _find_epic_by_title(home, title); if found: return found`), keeping its create path unchanged.

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "adhoc" -v`
Expected: PASS (including the pre-existing `ensure_adhoc_epic` tests, unaffected by the refactor).

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add live/archive ad-hoc epic finders (#238)" \
  --body "find_adhoc_epic (non-creating), ensure_adhoc_archive (create-if-missing stamped), list_open_adhoc_archives; refactor ensure_adhoc_epic onto the shared title search."
```

---

## Task 4: Per-repo drain (plan + apply)

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`DrainPlan` dataclass; `plan_adhoc_drain`; `apply_adhoc_drain`; `drain_adhoc_repo`)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: `find_adhoc_epic`, `child_states`, `quarter_of`, `current_quarter`, `ensure_adhoc_archive`, `list_open_adhoc_archives`, `add_child`, `remove_child`, `github.run`.
- Produces:
  - `@dataclass DrainPlan: live: IssueRef; moves: list[tuple[IssueRef, str]]; close: list[IssueRef]`
  - `plan_adhoc_drain(target_repo: str, *, now: datetime) -> DrainPlan | None` — None if no live epic.
  - `apply_adhoc_drain(target_repo: str, plan: DrainPlan) -> None`
  - `drain_adhoc_repo(target_repo: str, *, apply: bool, now: datetime) -> DrainPlan | None`

- [ ] **Step 1: Write the failing tests**

```python
from datetime import datetime, timezone
NOW = datetime(2026, 8, 8, tzinfo=timezone.utc)  # 2026-Q3

def _child(n, state, closed_at=""):
    return ChildState(IssueRef("org", ".github", n), state, "t", closed_at)

def test_plan_drain_moves_closed_buckets_by_quarter_and_closes_past() -> None:
    live = IssueRef("org", ".github", 40)
    kids = [_child(101, "CLOSED", "2026-05-10T00:00:00Z"),  # Q2
            _child(102, "CLOSED", "2026-07-02T00:00:00Z"),  # Q3
            _child(103, "OPEN")]                             # stays
    with (
        patch("vergil_tooling.lib.epics.find_adhoc_epic", return_value=live),
        patch("vergil_tooling.lib.epics.child_states", return_value=kids),
        patch("vergil_tooling.lib.epics.list_open_adhoc_archives",
              return_value=[(IssueRef("org", ".github", 88), "2026-Q2"),
                            (IssueRef("org", ".github", 91), "2026-Q3")]),
    ):
        plan = epics.plan_adhoc_drain("org/tooling", now=NOW)
    assert plan is not None
    assert (IssueRef("org", ".github", 101), "2026-Q2") in plan.moves
    assert (IssueRef("org", ".github", 102), "2026-Q3") in plan.moves
    assert all(ref.number != 103 for ref, _ in plan.moves)          # open child not moved
    assert plan.close == [IssueRef("org", ".github", 88)]           # Q2 < Q3 → close; Q3 stays open

def test_plan_drain_none_when_no_live_epic() -> None:
    with patch("vergil_tooling.lib.epics.find_adhoc_epic", return_value=None):
        assert epics.plan_adhoc_drain("org/tooling", now=NOW) is None

def test_apply_drain_ensures_archive_moves_then_closes() -> None:
    live = IssueRef("org", ".github", 40)
    plan = epics.DrainPlan(live=live,
        moves=[(IssueRef("org", ".github", 101), "2026-Q2")],
        close=[IssueRef("org", ".github", 88)])
    arch = IssueRef("org", ".github", 88)
    calls = []
    with (
        patch("vergil_tooling.lib.epics.ensure_adhoc_archive", return_value=arch) as mock_ens,
        patch("vergil_tooling.lib.epics.add_child", side_effect=lambda e, t: calls.append(("add", e, t))),
        patch("vergil_tooling.lib.epics.remove_child", side_effect=lambda e, t: calls.append(("rm", e, t))),
        patch("vergil_tooling.lib.github.run") as mock_run,
    ):
        epics.apply_adhoc_drain("org/tooling", plan)
    mock_ens.assert_called_once_with("org/tooling", "2026-Q2")
    # add_child before remove_child (never orphan-under-neither)
    assert calls == [("add", arch, IssueRef("org", ".github", 101)),
                     ("rm", live, IssueRef("org", ".github", 101))]
    assert mock_run.call_args.args[:2] == ("issue", "close")
    assert "88" in mock_run.call_args.args
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "drain" -v`
Expected: FAIL — `DrainPlan`/`plan_adhoc_drain`/`apply_adhoc_drain` undefined.

- [ ] **Step 3: Implement**

```python
@dataclass(frozen=True)
class DrainPlan:
    live: IssueRef
    moves: list[tuple[IssueRef, str]]  # (closed child, its close-quarter)
    close: list[IssueRef]              # open archives whose quarter is now past


def plan_adhoc_drain(target_repo: str, *, now: datetime) -> DrainPlan | None:
    live = find_adhoc_epic(target_repo)
    if live is None:
        return None
    moves = [
        (c.ref, quarter_of(c.closed_at))
        for c in child_states(live)
        if c.state == "CLOSED" and c.closed_at
    ]
    owner, bare = target_repo.split("/", 1)
    home = resolve_epic_home(owner, bare)
    cur = current_quarter(now)
    close = [ref for ref, q in list_open_adhoc_archives(home) if q < cur]
    return DrainPlan(live=live, moves=moves, close=close)


def apply_adhoc_drain(target_repo: str, plan: DrainPlan) -> None:
    for child, quarter in plan.moves:
        archive = ensure_adhoc_archive(target_repo, quarter)
        if archive == plan.live:
            continue  # defensive: never re-parent into the live epic
        add_child(archive, child)      # add before remove: never orphan-under-neither
        remove_child(plan.live, child)
    for archive in plan.close:
        github.run("issue", "close", str(archive.number),
                   "--repo", f"{archive.owner}/{archive.repo}")


def drain_adhoc_repo(target_repo: str, *, apply: bool, now: datetime) -> DrainPlan | None:
    plan = plan_adhoc_drain(target_repo, now=now)
    if plan is not None and apply:
        apply_adhoc_drain(target_repo, plan)
    return plan
```

Note the quarter-string comparison `q < cur` is a correct chronological test because `YYYY-Qn` is lexicographically ordered.

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "drain" -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add per-repo ad-hoc drain plan/apply (#238)" \
  --body "plan_adhoc_drain buckets closed children by close-quarter and lists past archives to close; apply_adhoc_drain ensures each archive, re-parents add-before-remove, and closes past archives. Idempotent and dry-run-capable."
```

---

## Task 5: Org-wide drain (visibility-aware)

**Files:**

- Modify: `src/vergil_tooling/lib/github.py` (`list_org_repos`); `src/vergil_tooling/lib/epics.py` (`drain_adhoc_org`)
- Test: `tests/vergil_tooling/test_github.py`, `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Produces:
  - `github.list_org_repos(org: str) -> list[str]` — bare repo names in *org*.
  - `epics.drain_adhoc_org(org: str, *, apply: bool, now: datetime) -> list[DrainPlan]` — one entry per repo with a live ad-hoc epic.

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_github.py
def test_list_org_repos() -> None:
    with patch("vergil_tooling.lib.github.read_json",
               return_value=[{"name": "tooling"}, {"name": ".github"}, {"name": "vm"}]):
        assert github.list_org_repos("org") == ["tooling", ".github", "vm"]

# tests/vergil_tooling/test_epics.py
def test_drain_adhoc_org_iterates_repos_visibility_aware() -> None:
    seen = []
    def fake_repo(target_repo, *, apply, now):
        seen.append((target_repo, apply))
        return None
    with (
        patch("vergil_tooling.lib.github.list_org_repos", return_value=["tooling", "priv"]),
        patch("vergil_tooling.lib.epics.drain_adhoc_repo", side_effect=fake_repo),
    ):
        epics.drain_adhoc_org("org", apply=True, now=NOW)
    assert ("org/tooling", True) in seen and ("org/priv", True) in seen

def test_drain_adhoc_org_skips_repo_that_raises() -> None:
    good = epics.DrainPlan(IssueRef("org", ".github", 40), moves=[], close=[])
    seen = []
    def fake(target_repo, *, apply, now):
        seen.append(target_repo)
        if target_repo == "org/bad":
            raise ValueError("multiple ad-hoc epics — corruption")
        return good
    with (
        patch("vergil_tooling.lib.github.list_org_repos", return_value=["bad", "good"]),
        patch("vergil_tooling.lib.epics.drain_adhoc_repo", side_effect=fake),
    ):
        plans = epics.drain_adhoc_org("org", apply=True, now=NOW)  # must NOT raise
    assert seen == ["org/bad", "org/good"]   # continued past the failure
    assert plans == [good]                    # the healthy repo still drained
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_github.py -k list_org_repos tests/vergil_tooling/test_epics.py -k drain_adhoc_org -v`
Expected: FAIL — undefined.

- [ ] **Step 3: Implement**

In `github.py`:

```python
def list_org_repos(org: str) -> list[str]:
    """Return the bare names of all repositories in *org*."""
    raw: Any = read_json("repo", "list", org, "--no-archived", "--limit", "1000", "--json", "name")
    return [str(r["name"]) for r in raw if isinstance(r, dict) and "name" in r] if isinstance(raw, list) else []
```

In `epics.py` (add `import sys` at the top if absent). Each repo resolves its own home via `drain_adhoc_repo` → `find_adhoc_epic` → `resolve_epic_home`, so public and private repos are both covered. **Per-repo failures are isolated** (spec §7 "report and skip"): a corrupted repo (e.g. `_find_epic_by_title` raising on a `>1` ad-hoc epic) is skipped with a message, never aborting the whole sweep:

```python
def drain_adhoc_org(org: str, *, apply: bool, now: datetime) -> list[DrainPlan]:
    plans: list[DrainPlan] = []
    for bare in github.list_org_repos(org):
        try:
            plan = drain_adhoc_repo(f"{org}/{bare}", apply=apply, now=now)
        except (ValueError, RuntimeError) as exc:
            print(f"skipped {org}/{bare}: {exc}", file=sys.stderr)
            continue
        if plan is not None:
            plans.append(plan)
    return plans
```

- [ ] **Step 4: Run to verify it passes**

Run the two selectors from Step 2. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add visibility-aware org-wide ad-hoc drain (#238)" \
  --body "drain_adhoc_org iterates every org repo and drains each via drain_adhoc_repo, which resolves each repo's own epic home — so public (.github-homed) and private (self-homed) ad-hoc epics are both covered."
```

---

## Task 6: Event-path hook in `rollup()`

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`rollup` ~460-474; add `_issue_closed_at`)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: `parent_of`, `_labels`, `quarter_of`, `ensure_adhoc_archive`, `add_child`, `remove_child`.
- Behavior change: when the just-closed *task*'s parent is a **live** ad-hoc epic (canonical title, i.e. **not** an archive), archive that one task into its close-quarter archive. Archive/finite parents behave as before.

- [ ] **Step 1: Write the failing tests**

```python
def test_rollup_archives_closed_child_under_live_adhoc() -> None:
    task = IssueRef("org", ".github", 101)
    live = IssueRef("org", ".github", 40)
    arch = IssueRef("org", ".github", 88)
    with (
        patch("vergil_tooling.lib.epics.parent_of", return_value=live),
        patch("vergil_tooling.lib.epics._labels", return_value={"epic", "ad-hoc"}),
        patch("vergil_tooling.lib.epics._issue_title", return_value="Epic (ad hoc): .github"),
        patch("vergil_tooling.lib.epics._issue_closed_at", return_value="2026-05-01T00:00:00Z"),
        patch("vergil_tooling.lib.epics.ensure_adhoc_archive", return_value=arch) as mock_ens,
        patch("vergil_tooling.lib.epics.add_child") as mock_add,
        patch("vergil_tooling.lib.epics.remove_child") as mock_rm,
    ):
        epics.rollup(task)
    mock_ens.assert_called_once_with("org/.github", "2026-Q2")
    mock_add.assert_called_once_with(arch, task)
    mock_rm.assert_called_once_with(live, task)

def test_rollup_noop_when_parent_is_adhoc_archive() -> None:
    task = IssueRef("org", ".github", 101)
    with (
        patch("vergil_tooling.lib.epics.parent_of", return_value=IssueRef("org", ".github", 88)),
        patch("vergil_tooling.lib.epics._labels", return_value={"epic", "ad-hoc"}),
        patch("vergil_tooling.lib.epics._issue_title", return_value="Epic (ad hoc): .github — 2026-Q2"),
        patch("vergil_tooling.lib.epics.add_child") as mock_add,
    ):
        epics.rollup(task)
    mock_add.assert_not_called()
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k "rollup_archives or rollup_noop_when_parent_is_adhoc_archive" -v`
Expected: FAIL — `_issue_title`/`_issue_closed_at` undefined and old rollup returns early on ad-hoc.

- [ ] **Step 3: Implement**

Add small readers mirroring `_issue_state`:

```python
def _issue_title(ref: IssueRef) -> str:
    return github.read_output("api", _issue_endpoint(ref), "--jq", ".title")


def _issue_closed_at(ref: IssueRef) -> str:
    return github.read_output("api", _issue_endpoint(ref), "--jq", ".closed_at // \"\"")
```

Replace the ad-hoc early return in `rollup()`:

```python
    if "ad-hoc" in _labels(parent):
        owner, home_repo = parent.owner, parent.repo
        title = _issue_title(parent)
        # Only the LIVE canonical ad-hoc epic drains; archives (stamped) are terminal.
        if _ADHOC_ARCHIVE_RE.match(title):
            return
        closed_at = _issue_closed_at(task)
        if not closed_at:
            return
        archive = ensure_adhoc_archive(f"{owner}/{home_repo}", quarter_of(closed_at))
        if archive != parent:
            add_child(archive, task)
            remove_child(parent, task)
        return
```

(Deriving `target_repo` as `parent.owner/parent.repo` is correct: an ad-hoc epic lives in its home, and `ensure_adhoc_archive` resolves the same home from that pair; the bare name in the archive title comes from the parent's own title prefix via `ensure_adhoc_archive`'s `target_repo.split`. For a `.github`-homed public-repo epic the home is `.github` and the archive is created alongside — matching where the live epic lives.)

> **Implementer note:** verify against `ensure_adhoc_archive` that passing `f"{parent.owner}/{parent.repo}"` yields the archive in the same home as `parent`. `parent.repo` is the home repo (e.g. `.github`), and `resolve_epic_home(owner, ".github")` returns `owner/.github` — consistent. If a future bare-name mismatch is possible, thread the bare name parsed from the parent title instead. This is covered by the Task 6 tests and the live validation (#2677).

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k rollup -v`
Expected: PASS (including existing finite-epic rollup tests — the non-ad-hoc path is unchanged).

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "drain closed child on close via rollup hook (#238)" \
  --body "rollup() now archives a just-closed child of a LIVE ad-hoc epic into its close-quarter archive (steady-state event path); archive and finite parents are unchanged."
```

---

## Task 7: `vrg-adhoc-epic archive` subcommand

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_adhoc_epic.py`
- Test: `tests/vergil_tooling/test_vrg_adhoc_epic.py`

**Interfaces:**

- Consumes: `epics.drain_adhoc_repo`, `epics.drain_adhoc_org`, `github.current_repo`.
- CLI: `vrg-adhoc-epic archive [--repo O/R | --all-in ORG] [--apply]` — dry-run default.

- [ ] **Step 1: Write the failing tests**

```python
_MOD = "vergil_tooling.bin.vrg_adhoc_epic"

def test_archive_repo_dry_run_default(capsys) -> None:
    plan = epics.DrainPlan(IssueRef("org", ".github", 40),
                           moves=[(IssueRef("org", ".github", 101), "2026-Q2")], close=[])
    with (
        patch(f"{_MOD}.github.current_repo", return_value="org/tooling"),
        patch(f"{_MOD}.github.target_org") as mock_scope,   # MagicMock is a ctx manager
        patch(f"{_MOD}.epics.drain_adhoc_repo", return_value=plan) as mock_drain,
    ):
        rc = vrg_adhoc_epic.main(["archive"])
    assert rc == 0
    assert mock_drain.call_args.kwargs["apply"] is False
    assert mock_scope.call_args.args[0] == "org"            # token scoped to the owner
    assert "org/.github#101" in capsys.readouterr().out

def test_archive_all_in_apply(capsys) -> None:
    with (
        patch(f"{_MOD}.github.target_org") as mock_scope,
        patch(f"{_MOD}.epics.drain_adhoc_org", return_value=[]) as mock_org,
    ):
        rc = vrg_adhoc_epic.main(["archive", "--all-in", "org", "--apply"])
    assert rc == 0
    assert mock_org.call_args.args[0] == "org"
    assert mock_org.call_args.kwargs["apply"] is True
    assert mock_scope.call_args.args[0] == "org"            # token scoped to the org
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_vrg_adhoc_epic.py -k archive -v`
Expected: FAIL — `invalid choice: 'archive'`.

- [ ] **Step 3: Implement**

Add imports `from datetime import datetime, timezone`, then:

```python
def _render_plan(plan: epics.DrainPlan) -> str:
    lines = [f"{plan.live.slug}: {len(plan.moves)} closed child(ren) to archive"]
    for child, quarter in plan.moves:
        lines.append(f"  {child.slug} -> {quarter}")
    for archive in plan.close:
        lines.append(f"  close past archive {archive.slug}")
    return "\n".join(lines)


def cmd_archive(args: argparse.Namespace) -> int:
    now = datetime.now(timezone.utc)
    verb = "APPLY" if args.apply else "DRY-RUN"
    if args.all_in:
        # Scope the App token to the org before any mutation, mirroring
        # vrg-epic-move / vrg-issue-create (spec §7).
        with github.target_org(args.all_in):
            plans = epics.drain_adhoc_org(args.all_in, apply=args.apply, now=now)
        print(f"[{verb}] {args.all_in}: {len(plans)} ad-hoc epic(s) with work to archive")
        for plan in plans:
            print(_render_plan(plan))
        return 0
    repo = args.repo or github.current_repo()
    if "/" not in repo:
        print(f"vrg-adhoc-epic: --repo must be 'owner/repo' (got {repo!r})", file=sys.stderr)
        return 1
    owner = repo.split("/", 1)[0]
    with github.target_org(owner):
        plan = epics.drain_adhoc_repo(repo, apply=args.apply, now=now)
    print(f"[{verb}] " + (_render_plan(plan) if plan else f"{repo}: no ad-hoc epic"))
    return 0
```

Wire the subparser in `parse_args` (mutually-exclusive scope):

```python
    p_arch = sub.add_parser("archive", help="Drain closed children into per-quarter archives (dry-run unless --apply).")
    scope = p_arch.add_mutually_exclusive_group()
    scope.add_argument("--repo", help="Target repo owner/name (defaults to the current repo)")
    scope.add_argument("--all-in", metavar="ORG", help="Sweep every repo in ORG")
    p_arch.add_argument("--apply", action="store_true", help="Execute (default: dry-run)")
    p_arch.set_defaults(func=cmd_archive)
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_vrg_adhoc_epic.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope adhoc-epic --message "add 'archive' subcommand (#238)" \
  --body "vrg-adhoc-epic archive drains closed ad-hoc children into per-quarter archives; dry-run by default, --apply to execute, --repo or --all-in <org> (visibility-aware)."
```

---

## Task 8: Backstop drain in the daily `vrg-epic-audit --close` sweep

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (the `--close` write path, ~154-168; report path)
- Test: `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `epics.drain_adhoc_org`. Runs under the same human/`VRG_EPIC_SWEEP` gate as the existing closes. Provides the backstop for missed close events, the one-time backlog distribution, and past-archive closing.

- [ ] **Step 1: Write the failing test**

```python
_AMOD = "vergil_tooling.bin.vrg_epic_audit"

def test_close_sweep_also_drains_adhoc(monkeypatch) -> None:
    monkeypatch.setenv("VRG_EPIC_SWEEP", "1")
    with (
        patch(f"{_AMOD}.github.detect_org", return_value="org"),
        patch(f"{_AMOD}.epic_audit.task_drift", return_value=[]),
        patch(f"{_AMOD}.epic_audit.epic_drift", return_value=[]),
        patch(f"{_AMOD}.epic_audit.closed_epic_open_child", return_value=[]),
        patch(f"{_AMOD}.epic_audit.close_drift", return_value=[]),
        patch(f"{_AMOD}.epic_audit.reopen_epics_with_open_children", return_value=[]),
        patch(f"{_AMOD}.epic_audit.render_closed", return_value=""),
        patch(f"{_AMOD}.epics.drain_adhoc_org", return_value=[]) as mock_drain,
    ):
        rc = vrg_epic_audit.main(["--close"])
    assert rc == 0
    assert mock_drain.call_args.args[0] == "org"
    assert mock_drain.call_args.kwargs["apply"] is True
```

- [ ] **Step 2: Run to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_vrg_epic_audit.py -k drains_adhoc -v`
Expected: FAIL — `drain_adhoc_org` not called.

- [ ] **Step 3: Implement**

Add `from datetime import datetime, timezone` (if absent) and `from vergil_tooling.lib import epics` (confirm import).

> **Token scoping (spec §7) — read before wiring.** First check how `vrg_epic_audit` establishes org scope today: if its `--close` path already runs inside `github.target_org(org)` (or the daily Action mints the token pre-scoped), the drain inherits that scope and needs no wrapper. If it does **not**, wrap the `drain_adhoc_org` call below in `with github.target_org(org):`, matching the standalone CLI (Task 7) and `vrg-epic-move`. Do not leave the mutation unscoped.

In the `if args.close:` block, after `reopen_epics_with_open_children`, add (inside the org scope per the note above):

```python
        drained = epics.drain_adhoc_org(org, apply=True, now=datetime.now(timezone.utc))
        moved = sum(len(p.moves) for p in drained)
        closed_archives = sum(len(p.close) for p in drained)
        print(f"Ad-hoc drain: {moved} child(ren) archived, {closed_archives} past archive(s) closed.")
```

In the dry-run/report path (no `--close`), add a preview:

```python
        preview = epics.drain_adhoc_org(org, apply=False, now=datetime.now(timezone.utc))
        pmoved = sum(len(p.moves) for p in preview)
        print(f"Ad-hoc drain (preview): {pmoved} child(ren) would be archived.")
```

- [ ] **Step 4: Run to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_vrg_epic_audit.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epic-audit --message "drain ad-hoc epics in the daily close sweep (#238)" \
  --body "The daily epic-sweep (vrg-epic-audit --close) now also runs the org-wide ad-hoc drain — the backstop for missed close events, the one-time backlog distribution, and past-archive closing — under the same human/VRG_EPIC_SWEEP gate. No new cron."
```

---

## Task 9: Full validation

**Files:** none (gate run)

- [ ] **Step 1: Run the full validation gate**

Run: `vrg-container-run -- vrg-validate`
Expected: all green — lint (100-col), typecheck, tests, **100% coverage**, audit. Fix any failure and re-run; never suppress a gate.

- [ ] **Step 2: Commit any fixes**

```bash
vrg-commit --type fix --scope epics --message "address validation findings (#238)"
```

---

## Self-review (author check against the spec)

- **§3.1 drain (list → bucket closed by `closedAt` → ensure archive → add-before-remove → close past):** Tasks 1, 3, 4 (+ event path Task 6). ✓
- **§3.2 close-quarter bucketing / backlog self-distribution:** Task 2 (`quarter_of`) + Task 4 (uses each child's `closed_at`). ✓
- **§3.3 archive lifecycle (create-on-demand, close past):** Task 3 (`ensure_adhoc_archive`) + Task 4 (`close`). ✓
- **§3.4 CLI (dry-run default, `--repo`/`--all-in`, visibility-aware):** Tasks 5, 7. ✓
- **§3.5 automation (event + daily sweep, no new cron):** Task 6 (`rollup`) + Task 8 (sweep). ✓
- **§3.6 naming/labels/resolver-safety:** Task 3 constants + regex; live epic untouched. ✓
- **§4 tests (quarter math, drain, idempotency, straggler reopen, visibility, event hook):** Tasks 1–8 tests; live validation is #2677 (dev-tree `uv run`). ✓
- **Type consistency:** `DrainPlan(live, moves: list[(IssueRef, str)], close: list[IssueRef])`, `ChildState.closed_at`, `quarter_of`/`current_quarter`, `find_adhoc_epic`/`ensure_adhoc_archive` used consistently across Tasks 4–8. ✓
- **Straggler-into-closed-archive reopen** relies on `add_child` reopening a closed epic (`epics.py:279`) — exercised by the live validation; the past-archive close in Task 4 re-closes it next run.
