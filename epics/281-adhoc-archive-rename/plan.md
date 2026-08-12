# Reclassify per-quarter ad-hoc buckets as archives — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reclassify the terminal per-quarter ad-hoc buckets so they are titled `Archive (ad hoc): <repo> — YYYY-Qn` and labelled `archive` + `ad-hoc` instead of `epic` + `ad-hoc`, with a self-healing creation path, a one-time org-wide normalize sweep, and the one audit fix this requires.

**Architecture:** All logic lives in `src/vergil_tooling/lib/epics.py` (constants, discovery, creation self-heal, normalize sweep), its CLI driver `src/vergil_tooling/bin/vrg_adhoc_epic.py` (a new `normalize` subcommand), the label set in `src/vergil_tooling/data/labels.json`, and one skip-set fix in `src/vergil_tooling/lib/epic_audit.py`. The live ad-hoc epic and all drain/rollup/re-parenting mechanics are unchanged.

**Tech Stack:** Python 3.12+, pytest, `unittest.mock.patch`. GitHub mutations go through `vergil_tooling.lib.github` wrappers (`create_issue`, `run`, `read_json`, `list_org_repos`, `target_org`).

## Global Constraints

- **Spec:** `epics/281-adhoc-archive-rename/spec.md` (this epic, `vergil-project/.github#281`).
- **Live ad-hoc epic is untouched:** title `Epic (ad hoc): <repo>`, labels `epic` + `ad-hoc`. Only `— YYYY-Qn`-stamped buckets change. `_ADHOC_EPIC_TITLE_PREFIX = "Epic (ad hoc): "` stays as-is.
- **New archive form (exact):** title prefix `"Archive (ad hoc): "`; labels tuple `("archive", "ad-hoc")`; the em-dash separator is a space + U+2014 + space, identical to today.
- **`ad-hoc` label is retained** on archives (keeps `roadmap.py::_is_perpetual` filtering them for free).
- **Commit wrapper:** use `vrg-commit` (raw `git commit` is denied repo-wide). Git ops via `vrg-git`.
- **Test invocation:** run inside the dev container, e.g. `vrg-container-run -- uv run pytest <path>::<test> -v`. Full gate: `vrg-container-run -- vrg-validate`.
- **No silent failures:** no swallowed exceptions or error-hiding fallbacks (repo policy).

---

## File Structure

- `src/vergil_tooling/data/labels.json` — **modify**: add the `archive` label definition.
- `src/vergil_tooling/lib/epics.py` — **modify**: archive constants + new-form regex + legacy recognizer; parameterize `_find_epic_by_title` label set; switch `list_open_adhoc_archives` to the archive label; rewrite `ensure_adhoc_archive` with the self-heal branch; add `_normalize_archive_in_place`, `ArchiveConversion`, `plan_normalize_adhoc`, `apply_normalize`, `normalize_adhoc_archives`.
- `src/vergil_tooling/bin/vrg_adhoc_epic.py` — **modify**: add the `normalize` subcommand + `cmd_normalize`; update the prog description.
- `src/vergil_tooling/lib/epic_audit.py` — **modify**: add `archive` to the skip set in `stray_dotgithub_issue`.
- `tests/vergil_tooling/test_epics.py` — **modify**: update archive tests to new form; add self-heal + normalize tests.
- `tests/vergil_tooling/test_vrg_adhoc_epic.py` — **modify**: add `normalize` CLI tests.
- `tests/vergil_tooling/test_epic_audit.py` — **modify**: add the open-archive-not-stray regression test.

---

## Task 1: Add the `archive` label

**Files:**

- Modify: `src/vergil_tooling/data/labels.json`
- Test: `tests/vergil_tooling/test_data_labels.py` (create if absent)

**Interfaces:**

- Consumes: nothing.
- Produces: an `archive` label available to `vrg-ensure-label` and to issue creation.

- [ ] **Step 1: Write the failing test**

Create `tests/vergil_tooling/test_data_labels.py`:

```python
from __future__ import annotations

import json
from importlib import resources


def _labels() -> list[dict[str, str]]:
    data = json.loads(
        resources.files("vergil_tooling.data").joinpath("labels.json").read_text()
    )
    return data["labels"]


def test_archive_label_present_and_wellformed() -> None:
    by_name = {entry["name"]: entry for entry in _labels()}
    assert "archive" in by_name, "labels.json must define the 'archive' label"
    archive = by_name["archive"]
    assert set(archive) == {"name", "color", "description"}
    assert archive["color"]  # non-empty hex
    assert "archive" in archive["description"].lower()


def test_label_names_are_unique() -> None:
    names = [entry["name"] for entry in _labels()]
    assert len(names) == len(set(names)), "duplicate label names in labels.json"
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_data_labels.py -v`
Expected: `test_archive_label_present_and_wellformed` FAILS ("archive" not in by_name); the uniqueness test passes.

- [ ] **Step 3: Add the label**

In `src/vergil_tooling/data/labels.json`, add this entry to the `labels` array immediately after the `ad-hoc` entry:

```json
    {"name": "archive", "color": "6a737d", "description": "Terminal per-quarter archive of closed ad-hoc work; historical record"},
```

(Colour `6a737d` is a neutral grey, visually distinct from `epic` `6f42c1` and `ad-hoc` `5319e7`.)

- [ ] **Step 4: Run the test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_data_labels.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add 'archive' label for per-quarter ad-hoc archives"
```

---

## Task 2: Archive constants, new-form regex, legacy recognizer, parameterized title search

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py:399-452` (constants block + `_find_epic_by_title`)
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Produces:
  - `_ADHOC_ARCHIVE_TITLE_PREFIX: str = "Archive (ad hoc): "`
  - `_ADHOC_ARCHIVE_LABELS: tuple[str, str] = ("archive", "ad-hoc")`
  - `_ADHOC_ARCHIVE_RE` — matches `^Archive \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$`
  - `_LEGACY_ADHOC_ARCHIVE_RE` — matches `^Epic \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$`
  - `_find_epic_by_title(home, title, *, prefer_oldest=False, labels=("epic", "ad-hoc"))` — now takes a `labels` sequence used in the `gh issue list` query.

- [ ] **Step 1: Write the failing tests**

Add to `tests/vergil_tooling/test_epics.py`:

```python
def test_archive_regex_matches_new_form_only() -> None:
    assert epics._ADHOC_ARCHIVE_RE.match("Archive (ad hoc): tooling — 2026-Q3")
    # Live epic and legacy archive titles must NOT match the new-form regex.
    assert epics._ADHOC_ARCHIVE_RE.match("Epic (ad hoc): tooling") is None
    assert epics._ADHOC_ARCHIVE_RE.match("Epic (ad hoc): tooling — 2026-Q3") is None


def test_legacy_archive_regex_matches_old_form_only() -> None:
    m = epics._LEGACY_ADHOC_ARCHIVE_RE.match("Epic (ad hoc): tooling — 2026-Q3")
    assert m and m.group("bare") == "tooling" and m.group("quarter") == "2026-Q3"
    assert epics._LEGACY_ADHOC_ARCHIVE_RE.match("Archive (ad hoc): tooling — 2026-Q3") is None
    assert epics._LEGACY_ADHOC_ARCHIVE_RE.match("Epic (ad hoc): tooling") is None


def test_find_epic_by_title_uses_supplied_labels() -> None:
    title = "Archive (ad hoc): tooling — 2026-Q3"
    rows = [{"number": 42, "title": title}]
    with patch("vergil_tooling.lib.github.read_json", return_value=rows) as mock_list:
        got = epics._find_epic_by_title(
            "org/.github", title, labels=("archive", "ad-hoc")
        )
    assert got == IssueRef("org", ".github", 42)
    # The label filters passed to gh reflect the supplied labels, not epic+ad-hoc.
    args = list(mock_list.call_args.args)
    assert "archive" in args and "ad-hoc" in args and "epic" not in args
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "archive_regex or legacy_archive_regex or supplied_labels" -v`
Expected: FAIL (`_ADHOC_ARCHIVE_RE` still matches old form; `_LEGACY_ADHOC_ARCHIVE_RE` undefined; `_find_epic_by_title` has no `labels` kwarg).

- [ ] **Step 3: Update the constants block**

Replace `epics.py:399-408` with:

```python
_ADHOC_EPIC_TITLE_PREFIX = "Epic (ad hoc): "
_ADHOC_EPIC_LABELS = ("epic", "ad-hoc")
_ADHOC_ARCHIVE_TITLE_PREFIX = "Archive (ad hoc): "
_ADHOC_ARCHIVE_LABELS = ("archive", "ad-hoc")
_ADHOC_EPIC_BODY = (
    "Perpetual umbrella for ad-hoc work in {repo}. Created and reused "
    "idempotently; tasks routed to the ad-hoc epic are linked here.\n"
)
# A stamped per-quarter archive: "Archive (ad hoc): <bare> — <YYYY>-Qn". The
# separator is a space, U+2014 em-dash, space. Matching this distinguishes a
# terminal archive from the live canonical ad-hoc epic (which has no stamp).
_ADHOC_ARCHIVE_RE = re.compile(
    r"^Archive \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$"
)
# Legacy pre-rename archive title ("Epic (ad hoc): <bare> — <YYYY>-Qn"), used
# ONLY by the self-healing creation path and the normalize sweep to find
# archives still in the old form. Steady-state code keys off _ADHOC_ARCHIVE_RE.
_LEGACY_ADHOC_ARCHIVE_RE = re.compile(
    r"^Epic \(ad hoc\): (?P<bare>.+) — (?P<quarter>\d{4}-Q[1-4])$"
)
```

- [ ] **Step 4: Parameterize `_find_epic_by_title`**

In `epics.py` change the signature and the two label lines in the `gh issue list` query. New signature:

```python
def _find_epic_by_title(
    home: str,
    title: str,
    *,
    prefer_oldest: bool = False,
    labels: Sequence[str] = ("epic", "ad-hoc"),
) -> IssueRef | None:
```

Replace the hardcoded label args in its `github.read_json("issue", "list", ...)` call — the two lines that today read `"--label", "epic", "--label", "ad-hoc"` — with a comprehension that expands `labels`:

```python
    raw: Any = github.read_json(
        "issue",
        "list",
        "--repo",
        home,
        *[arg for label in labels for arg in ("--label", label)],
        "--state",
        "open",
        "--json",
        "number,title",
    )
```

If `Sequence` is not already imported, add `from collections.abc import Sequence` to the imports.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "archive_regex or legacy_archive_regex or supplied_labels" -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-commit --type refactor --scope epics --message "add archive constants, new/legacy recognizers, parameterize title search"
```

---

## Task 3: Switch archive creation and discovery to the new form

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` — `ensure_adhoc_archive` (`499-520`) and `list_open_adhoc_archives` (`523-550`)
- Test: `tests/vergil_tooling/test_epics.py:934-970` (existing archive tests → new form)

**Interfaces:**

- Produces: `ensure_adhoc_archive(target_repo, quarter)` creates `Archive (ad hoc): <bare> — <quarter>` labelled `("archive", "ad-hoc")` and looks up existing archives with those labels. `list_open_adhoc_archives(home)` finds archives via `_ADHOC_ARCHIVE_RE` under the archive labels.

- [ ] **Step 1: Update the existing tests to the new form (they will fail)**

In `tests/vergil_tooling/test_epics.py`:

Change `test_ensure_adhoc_archive_creates_stamped_title` assertions to:

```python
    assert mock_create.call_args.kwargs["title"] == "Archive (ad hoc): tooling — 2026-Q3"
    assert mock_create.call_args.kwargs["labels"] == ["archive", "ad-hoc"]
```

Change the `title`/`rows` literals in `test_ensure_adhoc_archive_reuses_oldest_duplicate` and `test_ensure_adhoc_archive_reuses_existing_stamped` from `"Epic (ad hoc): tooling — 2026-Q3"` to `"Archive (ad hoc): tooling — 2026-Q3"` (these represent an already-migrated archive).

Change `test_list_open_adhoc_archives_parses_quarter` rows to new-form titles, keeping the live epic unstamped:

```python
    rows = [
        {"number": 88, "title": "Archive (ad hoc): tooling — 2026-Q2"},
        {"number": 90, "title": "Epic (ad hoc): tooling"},  # live, not an archive
        {"number": 91, "title": "Archive (ad hoc): tooling — 2026-Q3"},
    ]
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "ensure_adhoc_archive or list_open_adhoc_archives" -v`
Expected: FAIL (code still builds/queries old form).

- [ ] **Step 3: Update `ensure_adhoc_archive`**

In `epics.py`, replace the title/lookup/create lines in `ensure_adhoc_archive` so the title uses the archive prefix, the lookup uses the archive labels, and creation applies the archive labels:

```python
    title = f"{_ADHOC_ARCHIVE_TITLE_PREFIX}{bare} — {quarter}"
    existing = _find_epic_by_title(
        home, title, prefer_oldest=True, labels=_ADHOC_ARCHIVE_LABELS
    )
    if existing is not None:
        return existing
    url = github.create_issue(
        repo=home,
        title=title,
        body=f"Ad-hoc work in {target_repo} finished in {quarter}. Managed automatically.\n",
        labels=list(_ADHOC_ARCHIVE_LABELS),
    )
    return IssueRef(owner=owner, repo=home_repo, number=int(url.rstrip("/").rsplit("/", 1)[-1]))
```

Also update the docstring's "same `epic` + `ad-hoc` labels" wording to "`archive` + `ad-hoc` labels".

- [ ] **Step 4: Update `list_open_adhoc_archives`**

In `epics.py`, in `list_open_adhoc_archives`, replace the two hardcoded label lines (`"--label", "epic", "--label", "ad-hoc"`) with the archive labels, matching the Task 2 expansion style:

```python
        *[arg for label in _ADHOC_ARCHIVE_LABELS for arg in ("--label", label)],
```

(The `_ADHOC_ARCHIVE_RE.match` filter already selects only stamped archives; it now matches the new prefix.)

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "ensure_adhoc_archive or list_open_adhoc_archives" -v`
Expected: PASS.

- [ ] **Step 6: Add a rollup regression test (child under a non-epic archive does not drain)**

Add to `tests/vergil_tooling/test_epics.py`:

```python
def test_rollup_noop_for_child_under_archive_parent() -> None:
    # A converted archive is not epic-labelled, so rollup returns without draining.
    task = IssueRef("org", ".github", 200)
    archive = IssueRef("org", ".github", 88)
    with (
        patch("vergil_tooling.lib.epics.parent_of", return_value=archive),
        patch("vergil_tooling.lib.epics._labels", return_value={"archive", "ad-hoc"}),
        patch("vergil_tooling.lib.epics.ensure_adhoc_archive") as mock_ensure,
        patch("vergil_tooling.lib.epics.reparent_child") as mock_reparent,
    ):
        epics.rollup(task)
    mock_ensure.assert_not_called()
    mock_reparent.assert_not_called()
```

- [ ] **Step 7: Run the rollup test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py::test_rollup_noop_for_child_under_archive_parent -v`
Expected: PASS (`is_epic` returns False for `{"archive", "ad-hoc"}`, so `rollup` returns at the epic gate).

- [ ] **Step 8: Commit**

```bash
vrg-commit --type feat --scope epics --message "create and discover per-quarter buckets as archives, not epics"
```

---

## Task 4: Self-healing creation — heal a legacy archive in place

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` — add `_normalize_archive_in_place`; extend `ensure_adhoc_archive` with the legacy-heal branch
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: `_ADHOC_EPIC_TITLE_PREFIX`, `_ADHOC_ARCHIVE_TITLE_PREFIX`, `_find_epic_by_title` (both label sets).
- Produces:
  - `_normalize_archive_in_place(ref: IssueRef, new_title: str) -> None` — retitle + add `archive` + remove `epic` (keeps `ad-hoc`) via `github.run("issue", "edit", ...)`.
  - `ensure_adhoc_archive` now resolves in order: new-form lookup → legacy heal → create.

- [ ] **Step 1: Write the failing tests**

Add to `tests/vergil_tooling/test_epics.py`:

```python
def test_normalize_archive_in_place_edits_title_and_labels() -> None:
    ref = IssueRef("org", ".github", 88)
    with patch("vergil_tooling.lib.github.run") as mock_run:
        epics._normalize_archive_in_place(ref, "Archive (ad hoc): tooling — 2026-Q3")
    args = list(mock_run.call_args.args)
    assert args[:2] == ["issue", "edit"] and "88" in args
    assert "--title" in args and "Archive (ad hoc): tooling — 2026-Q3" in args
    assert args[args.index("--add-label") + 1] == "archive"
    assert args[args.index("--remove-label") + 1] == "epic"


def test_ensure_adhoc_archive_heals_legacy_in_place() -> None:
    # No new-form archive exists, but a legacy one does: heal it, don't create.
    def fake_find(home, title, *, prefer_oldest=False, labels=("epic", "ad-hoc")):
        if tuple(labels) == ("archive", "ad-hoc"):
            return None  # no new-form archive yet
        if title == "Epic (ad hoc): tooling — 2026-Q3":
            return IssueRef("org", ".github", 88)  # legacy archive found
        return None

    with (
        patch("vergil_tooling.lib.epics.resolve_epic_home", return_value="org/.github"),
        patch("vergil_tooling.lib.epics._find_epic_by_title", side_effect=fake_find),
        patch("vergil_tooling.lib.epics._normalize_archive_in_place") as mock_heal,
        patch("vergil_tooling.lib.github.create_issue") as mock_create,
    ):
        got = epics.ensure_adhoc_archive("org/tooling", "2026-Q3")
    assert got == IssueRef("org", ".github", 88)
    mock_heal.assert_called_once_with(
        IssueRef("org", ".github", 88), "Archive (ad hoc): tooling — 2026-Q3"
    )
    mock_create.assert_not_called()
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "normalize_archive_in_place or heals_legacy" -v`
Expected: FAIL (`_normalize_archive_in_place` undefined; no legacy-heal branch).

- [ ] **Step 3: Add `_normalize_archive_in_place`**

Add near `ensure_adhoc_archive` in `epics.py`:

```python
def _normalize_archive_in_place(ref: IssueRef, new_title: str) -> None:
    """Convert a legacy-form archive to the new form: retitle, +archive, -epic.

    Keeps ``ad-hoc``. Works on open or closed issues (``gh issue edit`` permits
    editing a closed issue's title and labels).
    """
    github.run(
        "issue",
        "edit",
        str(ref.number),
        "--repo",
        f"{ref.owner}/{ref.repo}",
        "--title",
        new_title,
        "--add-label",
        "archive",
        "--remove-label",
        "epic",
    )
```

- [ ] **Step 4: Add the legacy-heal branch to `ensure_adhoc_archive`**

In `ensure_adhoc_archive`, after the new-form lookup returns `None` and before `create_issue`, insert:

```python
    legacy_title = f"{_ADHOC_EPIC_TITLE_PREFIX}{bare} — {quarter}"
    legacy = _find_epic_by_title(home, legacy_title, prefer_oldest=True)
    if legacy is not None:
        _normalize_archive_in_place(legacy, title)
        return legacy
```

(`title` here is the new-form title computed at the top of the function; `_find_epic_by_title` with default labels finds the `epic`+`ad-hoc` legacy archive.)

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "normalize_archive_in_place or heals_legacy or ensure_adhoc_archive" -v`
Expected: PASS (including the three unchanged `ensure_adhoc_archive` tests from Task 3 — new-form-exists and create-when-absent still hold, because `fake`/`read_json` return no legacy match in those).

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope epics --message "self-heal legacy archives in place during creation (no duplicate bucket)"
```

---

## Task 5: One-time idempotent normalize sweep (per-repo home traversal)

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` — add `ArchiveConversion`, `plan_normalize_adhoc`, `apply_normalize`, `normalize_adhoc_archives`
- Test: `tests/vergil_tooling/test_epics.py`

**Interfaces:**

- Consumes: `resolve_epic_home`, `github.list_org_repos`, `github.read_json`, `_LEGACY_ADHOC_ARCHIVE_RE`, `_normalize_archive_in_place`, `_ADHOC_ARCHIVE_TITLE_PREFIX`.
- Produces:
  - `@dataclass(frozen=True) class ArchiveConversion: ref: IssueRef; old_title: str; new_title: str`
  - `plan_normalize_adhoc(org: str) -> list[ArchiveConversion]` — pure/read-only; enumerates every legacy-form archive (open **and** closed) across all resolved homes.
  - `apply_normalize(conversions: list[ArchiveConversion]) -> None`
  - `normalize_adhoc_archives(org: str, *, apply: bool) -> list[ArchiveConversion]`

- [ ] **Step 1: Write the failing tests**

Add to `tests/vergil_tooling/test_epics.py`:

```python
def test_plan_normalize_dedupes_homes_and_finds_legacy_all_states() -> None:
    # Two public repos share the .github home; one private repo self-homes.
    def fake_home(org, bare):
        return "org/.github" if bare in {"tooling", "actions"} else f"org/{bare}"

    def fake_list(*args, **kwargs):
        home = args[args.index("--repo") + 1]
        if home == "org/.github":
            return [
                {"number": 88, "title": "Epic (ad hoc): tooling — 2026-Q2", "state": "CLOSED"},
                {"number": 90, "title": "Epic (ad hoc): tooling", "state": "OPEN"},  # live epic
                {"number": 91, "title": "Archive (ad hoc): actions — 2026-Q3", "state": "OPEN"},  # already migrated
            ]
        if home == "org/private":
            return [
                {"number": 5, "title": "Epic (ad hoc): private — 2026-Q1", "state": "CLOSED"},
            ]
        return []

    with (
        patch("vergil_tooling.lib.github.list_org_repos", return_value=["tooling", "actions", "private"]),
        patch("vergil_tooling.lib.epics.resolve_epic_home", side_effect=fake_home),
        patch("vergil_tooling.lib.github.read_json", side_effect=fake_list),
    ):
        plan = epics.plan_normalize_adhoc("org")
    refs = {(c.ref.repo, c.ref.number): c for c in plan}
    # Legacy archives (public-homed + private self-homed) are converted; the live
    # epic and the already-migrated archive are not.
    assert (".github", 88) in refs and ("private", 5) in refs
    assert refs[(".github", 88)].new_title == "Archive (ad hoc): tooling — 2026-Q2"
    assert (".github", 90) not in refs and (".github", 91) not in refs
    assert len(plan) == 2


def test_apply_normalize_calls_in_place_for_each() -> None:
    conv = [
        epics.ArchiveConversion(IssueRef("org", ".github", 88), "old", "Archive (ad hoc): tooling — 2026-Q2"),
    ]
    with patch("vergil_tooling.lib.epics._normalize_archive_in_place") as mock_heal:
        epics.apply_normalize(conv)
    mock_heal.assert_called_once_with(IssueRef("org", ".github", 88), "Archive (ad hoc): tooling — 2026-Q2")


def test_normalize_adhoc_archives_dry_run_does_not_mutate() -> None:
    with (
        patch("vergil_tooling.lib.epics.plan_normalize_adhoc", return_value=[]) as mock_plan,
        patch("vergil_tooling.lib.epics.apply_normalize") as mock_apply,
    ):
        out = epics.normalize_adhoc_archives("org", apply=False)
    assert out == []
    mock_plan.assert_called_once_with("org")
    mock_apply.assert_not_called()
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "normalize" -v`
Expected: FAIL (`ArchiveConversion`, `plan_normalize_adhoc`, `apply_normalize`, `normalize_adhoc_archives` undefined).

- [ ] **Step 3: Implement the sweep**

Add to `epics.py` (near the drain functions):

```python
@dataclass(frozen=True)
class ArchiveConversion:
    """A single legacy archive to convert: where it is and its new title."""

    ref: IssueRef
    old_title: str
    new_title: str


def plan_normalize_adhoc(org: str) -> list[ArchiveConversion]:
    """Every legacy-form archive across *org*, open or closed, to convert.

    Pure/read-only. Resolves each repo's own epic home (public → ``<org>/.github``,
    private → itself) the way :func:`drain_adhoc_org` does, deduping homes so a
    shared ``.github`` is scanned once. A legacy archive is an issue whose title
    matches :data:`_LEGACY_ADHOC_ARCHIVE_RE`; the unstamped live epic never matches.
    """
    homes: dict[str, tuple[str, str]] = {}
    for bare in github.list_org_repos(org):
        home = resolve_epic_home(org, bare)
        homes[home] = tuple(home.split("/", 1))  # type: ignore[assignment]
    homes.setdefault(f"{org}/.github", (org, ".github"))
    conversions: list[ArchiveConversion] = []
    for home, (owner, home_repo) in homes.items():
        raw: Any = github.read_json(
            "issue",
            "list",
            "--repo",
            home,
            "--label",
            "epic",
            "--label",
            "ad-hoc",
            "--state",
            "all",
            "--limit",
            "500",
            "--json",
            "number,title",
        )
        for item in raw if isinstance(raw, list) else []:
            if not isinstance(item, dict):
                continue
            old_title = str(item.get("title", ""))
            if not _LEGACY_ADHOC_ARCHIVE_RE.match(old_title):
                continue
            new_title = _ADHOC_ARCHIVE_TITLE_PREFIX + old_title.removeprefix(
                _ADHOC_EPIC_TITLE_PREFIX
            )
            conversions.append(
                ArchiveConversion(
                    IssueRef(owner, home_repo, int(item["number"])), old_title, new_title
                )
            )
    return conversions


def apply_normalize(conversions: list[ArchiveConversion]) -> None:
    """Convert each planned archive in place (retitle, +archive, -epic)."""
    for conv in conversions:
        _normalize_archive_in_place(conv.ref, conv.new_title)


def normalize_adhoc_archives(org: str, *, apply: bool) -> list[ArchiveConversion]:
    """Plan (and, when *apply*, execute) the org-wide legacy-archive conversion.

    Idempotent: an already-migrated archive is new-form, does not match the legacy
    recognizer, and is skipped — so re-running is a no-op.
    """
    conversions = plan_normalize_adhoc(org)
    if apply:
        apply_normalize(conversions)
    return conversions
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k "normalize" -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type feat --scope epics --message "add org-wide idempotent normalize sweep for legacy archives"
```

---

## Task 6: `vrg-adhoc-epic normalize` subcommand

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_adhoc_epic.py` — add `cmd_normalize` + the `normalize` subparser; update the prog description
- Test: `tests/vergil_tooling/test_vrg_adhoc_epic.py`

**Interfaces:**

- Consumes: `epics.normalize_adhoc_archives(org, apply=…)`, `github.target_org`.
- Produces: CLI `vrg-adhoc-epic normalize --all-in ORG [--apply]` (dry-run default), printing one line per planned conversion.

- [ ] **Step 1: Write the failing tests**

Add to `tests/vergil_tooling/test_vrg_adhoc_epic.py`:

```python
def test_normalize_dry_run_lists_conversions(capsys: pytest.CaptureFixture[str]) -> None:
    conv = [
        epics.ArchiveConversion(
            epics.IssueRef("org", ".github", 88),
            "Epic (ad hoc): tooling — 2026-Q2",
            "Archive (ad hoc): tooling — 2026-Q2",
        )
    ]
    with (
        patch(f"{_MOD}.github.target_org"),
        patch(f"{_MOD}.epics.normalize_adhoc_archives", return_value=conv) as mock_norm,
    ):
        rc = main(["normalize", "--all-in", "org"])
    assert rc == 0
    mock_norm.assert_called_once_with("org", apply=False)
    out = capsys.readouterr().out
    assert "DRY-RUN" in out and "1 archive(s)" in out
    assert "org/.github#88" in out and "Archive (ad hoc): tooling — 2026-Q2" in out


def test_normalize_apply_passes_apply_true() -> None:
    with (
        patch(f"{_MOD}.github.target_org"),
        patch(f"{_MOD}.epics.normalize_adhoc_archives", return_value=[]) as mock_norm,
    ):
        rc = main(["normalize", "--all-in", "org", "--apply"])
    assert rc == 0
    mock_norm.assert_called_once_with("org", apply=True)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_adhoc_epic.py -k "normalize" -v`
Expected: FAIL (no `normalize` subcommand).

- [ ] **Step 3: Add `cmd_normalize`**

Add to `src/vergil_tooling/bin/vrg_adhoc_epic.py`:

```python
def cmd_normalize(args: argparse.Namespace) -> int:
    verb = "APPLY" if args.apply else "DRY-RUN"
    with github.target_org(args.all_in):
        conversions = epics.normalize_adhoc_archives(args.all_in, apply=args.apply)
    print(f"[{verb}] {args.all_in}: {len(conversions)} archive(s) to normalize")
    for conv in conversions:
        print(f"  {conv.ref.slug}: {conv.old_title!r} -> {conv.new_title!r}")
    return 0
```

- [ ] **Step 4: Register the subparser**

In `parse_args`, after the `archive` subparser block, add:

```python
    p_norm = sub.add_parser(
        "normalize",
        help="Convert legacy 'Epic (ad hoc): … — Qn' archives to the 'Archive (ad hoc): …' form (dry-run unless --apply).",
    )
    p_norm.add_argument("--all-in", metavar="ORG", required=True, help="Sweep every repo in ORG")
    p_norm.add_argument("--apply", action="store_true", help="Execute (default: dry-run)")
    p_norm.set_defaults(func=cmd_normalize)
```

Update the parser `description` string to mention that stamped quarter buckets are `Archive (ad hoc): … — Qn` labelled `archive` + `ad-hoc`, while the live epic stays `Epic (ad hoc): <repo>` labelled `epic` + `ad-hoc`.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_adhoc_epic.py -k "normalize" -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
vrg-commit --type feat --scope adhoc-epic --message "add 'normalize' subcommand to migrate legacy archives"
```

---

## Task 7: Fix `stray_dotgithub_issue` so archives are not flagged as strays

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py:332` (the skip-set line in `stray_dotgithub_issue`)
- Test: `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Consumes: nothing new.
- Produces: an open `archive`-labelled home issue is treated as legitimately non-stray.

- [ ] **Step 1: Write the failing test**

Add to `tests/vergil_tooling/test_epic_audit.py` (match the module's existing patch style — mirror an existing `stray_dotgithub_issue` test for the exact `patch` targets if they differ):

```python
def test_stray_skips_open_archive() -> None:
    from vergil_tooling.lib import epic_audit

    rows = [
        {"number": 88, "labels": [{"name": "archive"}, {"name": "ad-hoc"}]},  # open archive
    ]
    with patch("vergil_tooling.lib.github.read_json", return_value=rows):
        strays = epic_audit.stray_dotgithub_issue("org", home="org/.github")
    assert strays == []  # an archive is not a stray
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epic_audit.py::test_stray_skips_open_archive -v`
Expected: FAIL — the archive is currently reported as `org/.github#88` because it lacks `epic` and has no parent.

- [ ] **Step 3: Add `archive` to the skip set**

In `epic_audit.py`, change the skip condition (currently `if "epic" in labels or (labels & _INTAKE_LABELS): continue`) to also skip archives:

```python
        if "epic" in labels or "archive" in labels or (labels & _INTAKE_LABELS):
            continue
```

Update the function's docstring to note that `archive`-labelled per-quarter buckets are a legitimate top-level home issue, skipped like epics and intake.

- [ ] **Step 4: Run the test to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epic_audit.py::test_stray_skips_open_archive -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-commit --type fix --scope epic-audit --message "do not flag 'archive'-labelled per-quarter buckets as strays"
```

---

## Task 8: Full validation gate

**Files:** none (verification only).

- [ ] **Step 1: Run the full validation pipeline**

Run: `vrg-container-run -- vrg-validate`
Expected: all checks pass (lint, typecheck, the full test suite, common checks). Fix any regression surfaced in the touched modules before proceeding.

- [ ] **Step 2: Confirm no stale old-form references remain in code**

Run: `grep -rn "Epic (ad hoc):.*—\|_ADHOC_EPIC_TITLE_PREFIX.*— " src/ ; grep -rn "label.*epic.*ad-hoc" src/vergil_tooling/lib/epics.py`
Expected: the only remaining old-form (`Epic (ad hoc):`) construction is the **live** epic (`ensure_adhoc_epic`) and the **legacy recognizer** used by the sweep/heal. No archive creation or discovery path builds or queries the old form.

- [ ] **Step 3: Commit (only if Step 1/2 required fixes)**

```bash
vrg-commit --type test --scope epics --message "fix regressions surfaced by full validation"
```

---

## Rollout (operational — a deployment task, not a code PR)

After this plan's code merges to develop, the migration is run as the epic's **deployment task** (seeded under epic #281, `Blocked-by` the implementation), run via `issue-deploy`:

1. **Label sync** — `vrg-ensure-label` provisions the new `archive` label into `vergil-project/.github` and every managed repo.
2. **Dry-run** — `vrg-adhoc-epic normalize --all-in vergil-project` and review the printed conversions.
3. **Apply** — `vrg-adhoc-epic normalize --all-in vergil-project --apply` to migrate all existing archives (open and closed).
4. **Spot-check** — `label:epic state:open` in `.github` no longer lists per-quarter buckets; `label:archive` lists them; the live `Epic (ad hoc): <repo>` epics are unchanged.

These steps are agent-safe (no release), so `issue-deploy` runs them; success is recorded as the deployment task's `Outcome: SUCCESS` comment.

---

## Self-Review

**Spec coverage:**

- §3.1 new `archive` label → Task 1. ✓
- §3.2 constants, new-form regex, legacy recognizer → Task 2. ✓
- §3.3 creation/discovery switch + parameterized label set → Tasks 2 (param) + 3 (switch). ✓
- §3.4 self-healing creation (new-form → legacy heal → create) → Task 4 (heal branch) atop Task 3 (new-form lookup) + create. ✓
- §3.5 idempotent normalize sweep, per-repo home traversal, open+closed → Task 5. ✓
- §3.6 `stray_dotgithub_issue` required fix → Task 7; other audits "no change" verified in Task 8 Step 2 + covered by the existing suite. ✓
- CLI surface (`normalize` subcommand) → Task 6. ✓
- Rollout (label sync + sweep) → Rollout section (deployment task). ✓
- Docs (site + tooling) → handled by the epic's documentation-review bookend (#2737) and the `normalize` help text in Task 6; not a code task here.

**Placeholder scan:** none — every step carries concrete code or an exact command. ✓

**Type consistency:** `_ADHOC_ARCHIVE_LABELS` is a `tuple[str, str]`; `_find_epic_by_title(..., labels=Sequence[str])`; `ArchiveConversion(ref, old_title, new_title)` used identically in Tasks 5 and 6; `normalize_adhoc_archives(org, *, apply)` signature matches its CLI caller and tests. `_normalize_archive_in_place(ref, new_title)` signature matches all call sites (self-heal in Task 4, `apply_normalize` in Task 5). ✓
