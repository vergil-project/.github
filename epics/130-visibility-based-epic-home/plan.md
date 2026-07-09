# Visibility-based Epic Home — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route a private member repo's epics into the repo itself (not the public `<org>/.github`), derived automatically from repository visibility, so private work never leaks onto a public surface.

**Architecture:** One config-less resolver, `resolve_epic_home(org, target_repo)` in `lib/epics.py`, maps an explicitly-named target repo to the repo where its epic *issue* lives, using a memoized, fail-loud visibility probe (`repo_visibility`/`is_public`) in `lib/github.py`. The four call sites that hard-code `".github"` (`vrg-epic-create`, `ensure_adhoc_epic`, the audit invariant, roadmap sourcing) consume the resolver; the creating/reporting entry points gain an explicit `--repo` target (cwd as default). Ref-driven tools (`vrg-epic-link/move/rollup`) are untouched.

**Tech Stack:** Python 3.12+ (`argparse`, `functools.lru_cache`), `gh` via `vergil_tooling.lib.github` (`read_json`/`create_issue`), pytest with `unittest.mock.patch`.

## Global Constraints

- **Fail loud, never guess.** A visibility lookup that errors (missing repo, auth, network, empty result) MUST raise — never fall back to `.github`. A silent default is a disclosure bug. (`read_json` already raises `GitHubAPIError` on `gh` failure.)
- **Visibility is binary: `PUBLIC` vs. not-`PUBLIC`.** `INTERNAL` and `PRIVATE` both count as private for routing (`is_public` is `visibility == "PUBLIC"`).
- **No config key.** Routing is automatic; `vergil.toml` gains no epic section.
- **Backward-compatible.** Public target → `<org>/.github`; private-org (`.github` itself private) → `<org>/.github`. Only *public `.github` + private target → self* is new.
- **Tasks never move.** Only the epic *home* varies. Tasks stay in their member repo (1:1 with their PR).
- **Single org.** Cross-org is out of scope; the resolver operates within one owner.
- **Memoize per run.** Each repo's visibility is probed at most once per process.
- **Wrappers only.** All git/gh via `vrg-git`/`vrg-gh`. Validation is exactly `vrg-container-run -- vrg-validate`. Portable macOS/Linux; any bash is shellcheck-clean.

---

## File Structure

- `src/vergil_tooling/lib/github.py` — add `repo_visibility(name_with_owner)` + `is_public(name_with_owner)` (memoized, fail-loud). *(Task 1)*
- `src/vergil_tooling/lib/epics.py` — add `resolve_epic_home(org, target_repo)`; rewire `ensure_adhoc_epic`. *(Tasks 1, 3)*
- `src/vergil_tooling/bin/vrg_epic_create.py` — `--repo` target, resolver, print home. *(Task 2)*
- `src/vergil_tooling/bin/vrg_adhoc_epic.py` — `--repo` target. *(Task 3)*
- `src/vergil_tooling/lib/epic_audit.py` — generalize `epic_outside_dotgithub`; `--repo` targeting. *(Task 4)*
- `src/vergil_tooling/lib/roadmap.py` — source from resolved home; `--repo` targeting. *(Task 5)*
- `src/vergil_tooling/bin/vrg_epic_link.py` — visibility-boundary guard. *(Task 10)*
- Tests alongside each in `tests/vergil_tooling/`.
- Docs/skills/convention edits land in `vergil-project/.github` and the marketplace repo. *(Tasks 6–9)*

---

### Task 1: Resolver + visibility probe

**Files:**
- Modify: `src/vergil_tooling/lib/github.py` (add `repo_visibility`, `is_public` near `detect_org`, ~L106)
- Modify: `src/vergil_tooling/lib/epics.py` (add `resolve_epic_home` near `resolve_epic_ref`, ~L282)
- Test: `tests/vergil_tooling/test_github.py`, `tests/vergil_tooling/test_epics.py`

**Interfaces:**
- Produces: `github.repo_visibility(name_with_owner: str) -> str` (memoized; raises `GitHubAPIError` on failure); `github.is_public(name_with_owner: str) -> bool`; `epics.resolve_epic_home(org: str, target_repo: str) -> str` returning `"owner/repo"`.
- Consumes: `github.read_json`, `github.GitHubAPIError`.

- [ ] **Step 1: Write the failing test for the visibility probe**

```python
# tests/vergil_tooling/test_github.py
from unittest.mock import patch
import pytest
from vergil_tooling.lib import github

def test_is_public_true_for_public_repo() -> None:
    github.repo_visibility.cache_clear()
    with patch("vergil_tooling.lib.github.read_json", return_value={"visibility": "PUBLIC"}):
        assert github.is_public("org/repo") is True

def test_is_public_false_for_private_and_internal() -> None:
    github.repo_visibility.cache_clear()
    with patch("vergil_tooling.lib.github.read_json", return_value={"visibility": "PRIVATE"}):
        assert github.is_public("org/priv") is False
    github.repo_visibility.cache_clear()
    with patch("vergil_tooling.lib.github.read_json", return_value={"visibility": "INTERNAL"}):
        assert github.is_public("org/intern") is False

def test_repo_visibility_memoizes_one_probe_per_repo() -> None:
    github.repo_visibility.cache_clear()
    with patch("vergil_tooling.lib.github.read_json", return_value={"visibility": "PUBLIC"}) as rj:
        github.repo_visibility("org/repo")
        github.repo_visibility("org/repo")
    assert rj.call_count == 1

def test_repo_visibility_fails_loud_on_empty() -> None:
    github.repo_visibility.cache_clear()
    with patch("vergil_tooling.lib.github.read_json", return_value={}):
        with pytest.raises(github.GitHubAPIError):
            github.repo_visibility("org/repo")
```

- [ ] **Step 2: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_github.py -k visibility -v`
Expected: FAIL (`AttributeError: module ... has no attribute 'repo_visibility'`).

- [ ] **Step 3: Implement the probe**

```python
# src/vergil_tooling/lib/github.py  (add after detect_org, and `import functools` at top)
@functools.lru_cache(maxsize=None)
def repo_visibility(name_with_owner: str) -> str:
    """Return *name_with_owner*'s GitHub visibility ("PUBLIC"/"PRIVATE"/"INTERNAL").

    Fail-loud: a gh error (missing repo, auth, network) propagates as
    GitHubAPIError from read_output; an empty result is also an error. Memoized
    per process so audit/roadmap sweeps probe each repo once.
    """
    data = read_json("repo", "view", name_with_owner, "--json", "visibility")
    visibility = data.get("visibility") if isinstance(data, dict) else None
    if not isinstance(visibility, str) or not visibility:
        raise GitHubAPIError(1, f"gh repo view {name_with_owner} --json visibility",
                             "empty or missing visibility")
    return visibility


def is_public(name_with_owner: str) -> bool:
    """True only when *name_with_owner* is PUBLIC (INTERNAL and PRIVATE are not)."""
    return repo_visibility(name_with_owner) == "PUBLIC"
```

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_github.py -k visibility -v`
Expected: PASS.

- [ ] **Step 5: Write the failing test for the resolver**

```python
# tests/vergil_tooling/test_epics.py
def test_resolve_epic_home_dotgithub_short_circuits() -> None:
    # ".github" target never probes visibility
    with patch("vergil_tooling.lib.epics.github.is_public") as pub:
        assert epics.resolve_epic_home("org", ".github") == "org/.github"
        pub.assert_not_called()

def test_resolve_epic_home_public_target_is_central() -> None:
    with patch("vergil_tooling.lib.epics.github.is_public", return_value=True):
        assert epics.resolve_epic_home("org", "tooling") == "org/.github"

def test_resolve_epic_home_private_target_public_dotgithub_is_self() -> None:
    def pub(nwo: str) -> bool:
        return {"org/lab": False, "org/.github": True}[nwo]
    with patch("vergil_tooling.lib.epics.github.is_public", side_effect=pub):
        assert epics.resolve_epic_home("org", "lab") == "org/lab"

def test_resolve_epic_home_private_org_is_central() -> None:
    def pub(nwo: str) -> bool:
        return {"org/lab": False, "org/.github": False}[nwo]
    with patch("vergil_tooling.lib.epics.github.is_public", side_effect=pub):
        assert epics.resolve_epic_home("org", "lab") == "org/.github"

def test_resolve_epic_home_fails_loud() -> None:
    with patch("vergil_tooling.lib.epics.github.is_public",
               side_effect=github.GitHubAPIError(1, "cmd", "boom")):
        with pytest.raises(github.GitHubAPIError):
            epics.resolve_epic_home("org", "missing")
```

- [ ] **Step 6: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k resolve_epic_home -v`
Expected: FAIL (`AttributeError: ... 'resolve_epic_home'`).

- [ ] **Step 7: Implement the resolver**

```python
# src/vergil_tooling/lib/epics.py  (add near resolve_epic_ref)
def resolve_epic_home(org: str, target_repo: str) -> str:
    """Map an explicit *target_repo* (bare name) to its epic home ``"owner/repo"``.

    Public target -> central ``<org>/.github`` (today's behavior). A private
    target with a public ``.github`` homes its epics in itself (self-contained).
    A private ``.github`` means the whole org is private, so everything routes to
    ``.github``. Fail-loud: visibility errors propagate (see github.repo_visibility).
    """
    if target_repo == ".github":
        return f"{org}/.github"
    if github.is_public(f"{org}/{target_repo}"):
        return f"{org}/.github"
    if github.is_public(f"{org}/.github"):
        return f"{org}/{target_repo}"
    return f"{org}/.github"
```

- [ ] **Step 8: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k resolve_epic_home -v`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
cd <worktree> && vrg-git add src/vergil_tooling/lib/github.py src/vergil_tooling/lib/epics.py tests/vergil_tooling/test_github.py tests/vergil_tooling/test_epics.py
vrg-git commit -m "feat(epics): add visibility-based epic-home resolver"
```

---

### Task 2: `vrg-epic-create --repo` target + resolved home

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_epic_create.py` (L44-66 `main`, add `--repo`)
- Test: `tests/vergil_tooling/test_vrg_epic_create.py`

**Interfaces:**
- Consumes: `epics.resolve_epic_home`, `github.repo_visibility`, `github.current_repo`.
- Produces: unchanged `main(argv) -> int`.

- [ ] **Step 1: Update the failing tests**

Replace the `.github`-pinned assertions with target-and-home behavior:

```python
# tests/vergil_tooling/test_vrg_epic_create.py
_MOD = "vergil_tooling.bin.vrg_epic_create"

def test_default_target_is_current_repo_public_goes_to_dotgithub() -> None:
    with (
        patch(f"{_MOD}.github.current_repo", return_value="org/tooling"),
        patch(f"{_MOD}.epics.resolve_epic_home", return_value="org/.github") as home,
        patch(f"{_MOD}.github.repo_visibility", return_value="PUBLIC"),
        patch(f"{_MOD}.github.create_issue", return_value=_URL) as mock_create,
    ):
        rc = main(["--title", "Epic: X", "--body", "B"])
    assert rc == 0
    home.assert_called_once_with("org", "tooling")
    assert mock_create.call_args.kwargs["repo"] == "org/.github"

def test_explicit_private_target_homes_in_self() -> None:
    with (
        patch(f"{_MOD}.epics.resolve_epic_home", return_value="org/lab") as home,
        patch(f"{_MOD}.github.repo_visibility", return_value="PRIVATE"),
        patch(f"{_MOD}.github.create_issue", return_value=_URL) as mock_create,
    ):
        rc = main(["--repo", "org/lab", "--title", "T"])
    assert rc == 0
    home.assert_called_once_with("org", "lab")
    assert mock_create.call_args.kwargs["repo"] == "org/lab"

def test_prints_resolved_home(capsys: pytest.CaptureFixture[str]) -> None:
    with (
        patch(f"{_MOD}.epics.resolve_epic_home", return_value="org/lab"),
        patch(f"{_MOD}.github.repo_visibility", return_value="PRIVATE"),
        patch(f"{_MOD}.github.create_issue", return_value=_URL),
    ):
        main(["--repo", "org/lab", "--title", "T"])
    assert "epic home: org/lab [PRIVATE]" in capsys.readouterr().out
```

(Keep `test_parse_args_requires_title`, `test_main_adds_extra_labels`, `test_main_dedups_epic_label` — update them to patch `current_repo`/`resolve_epic_home`/`repo_visibility` like above. Delete `test_main_creates_epic_in_org_dot_github` and `test_main_errors_when_org_undetectable`; replace the latter with a `current_repo`-error propagation test.)

- [ ] **Step 2: Run to verify failures**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_epic_create.py -v`
Expected: FAIL (`--repo` unknown / `resolve_epic_home` not called).

- [ ] **Step 3: Implement**

```python
# src/vergil_tooling/bin/vrg_epic_create.py
from vergil_tooling.lib import epics, github  # add epics

# in parse_args, add:
parser.add_argument(
    "--repo",
    help="Target repo 'owner/repo' the epic is for (default: current repo). "
         "The epic home is derived from the target's visibility.",
)

def main(argv: list[str] | None = None) -> int:
    args = parse_args(argv)
    target = args.repo or github.current_repo()  # "owner/repo"; raises if undeterminable
    owner, bare = target.split("/", 1)
    home = epics.resolve_epic_home(owner, bare)
    print(f"-> epic home: {home} [{github.repo_visibility(home)}]")
    labels = list(dict.fromkeys(["epic", *args.label]))
    url = github.create_issue(
        repo=home, title=args.title, body=args.body,
        body_file=args.body_file, labels=labels, assignees=args.assignee,
    )
    print(f"Created {url} (epic).")
    return 0
```

Update the module docstring: epics live in their *resolved* home, not always `.github`.

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_epic_create.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_epic_create.py tests/vergil_tooling/test_vrg_epic_create.py
vrg-git commit -m "feat(epic-create): explicit --repo target, resolver-derived home"
```

---

### Task 3: `vrg-adhoc-epic --repo` + resolver-homed ad-hoc epic

**Files:**
- Modify: `src/vergil_tooling/lib/epics.py:293-342` (`ensure_adhoc_epic`)
- Modify: `src/vergil_tooling/bin/vrg_adhoc_epic.py` (add `--repo`)
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/test_vrg_adhoc_epic.py`

**Interfaces:**
- Consumes: `resolve_epic_home`.
- Produces: `ensure_adhoc_epic(target_repo: str) -> IssueRef` (unchanged signature; `IssueRef.repo` is now the resolved home's repo, e.g. `".github"` **or** the private repo's own name).

- [ ] **Step 1: Update the failing tests**

```python
# tests/vergil_tooling/test_epics.py — replace the .github-pinned adhoc tests
def test_ensure_adhoc_epic_public_repo_creates_in_dotgithub() -> None:
    with (
        patch("vergil_tooling.lib.epics.resolve_epic_home", return_value="org/.github"),
        patch("vergil_tooling.lib.epics.github.read_json", return_value=[]),
        patch("vergil_tooling.lib.epics.github.create_issue",
              return_value="https://github.com/org/.github/issues/7"),
    ):
        ref = epics.ensure_adhoc_epic("org/tooling")
    assert (ref.owner, ref.repo, ref.number) == ("org", ".github", 7)

def test_ensure_adhoc_epic_private_repo_homes_in_self() -> None:
    with (
        patch("vergil_tooling.lib.epics.resolve_epic_home", return_value="org/lab"),
        patch("vergil_tooling.lib.epics.github.read_json", return_value=[]),
        patch("vergil_tooling.lib.epics.github.create_issue",
              return_value="https://github.com/org/lab/issues/3"),
    ):
        ref = epics.ensure_adhoc_epic("org/lab")
    assert (ref.owner, ref.repo, ref.number) == ("org", "lab", 3)
```

- [ ] **Step 2: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py -k ensure_adhoc_epic -v`
Expected: FAIL (still homes in `.github`).

- [ ] **Step 3: Implement — home via resolver**

```python
# src/vergil_tooling/lib/epics.py  ensure_adhoc_epic body
    owner, bare = target_repo.split("/", 1)
    home = resolve_epic_home(owner, bare)            # was: dotgithub = f"{owner}/.github"
    home_repo = home.split("/", 1)[1]
    title = f"{_ADHOC_EPIC_TITLE_PREFIX}{bare}"
    raw: Any = github.read_json(
        "issue", "list", "--repo", home, "--label", "epic", "--label", "ad-hoc",
        "--state", "open", "--json", "number,title",
    )
    # ... unchanged filtering ...
    if len(rows) > 1:
        nums = ", ".join(f"#{r['number']}" for r in rows)
        raise ValueError(f"multiple ad-hoc epics titled {title!r} in {home} ({nums}) — pass an explicit --epic")
    if rows:
        return IssueRef(owner=owner, repo=home_repo, number=int(rows[0]["number"]))
    url = github.create_issue(repo=home, title=title,
                              body=_ADHOC_EPIC_BODY.format(repo=target_repo),
                              labels=list(_ADHOC_EPIC_LABELS))
    number = int(url.rstrip("/").rsplit("/", 1)[-1])
    return IssueRef(owner=owner, repo=home_repo, number=number)
```

Update the `ensure_adhoc_epic` and `resolve_epic_ref` docstrings (drop "in `<org>/.github`"; say "in the repo's resolved epic home"). Add `--repo` to `vrg_adhoc_epic.py` mirroring Task 2's default-to-`current_repo` logic, and — like `vrg-epic-create` — echo the resolved home before creating:

```python
# src/vergil_tooling/bin/vrg_adhoc_epic.py  (in main, before ensure_adhoc_epic)
home = epics.resolve_epic_home(owner, bare)
print(f"-> epic home: {home} [{github.repo_visibility(home)}]")
```

Add a test asserting the `-> epic home: ... [<VIS>]` line appears in stdout (mirrors `test_prints_resolved_home` in Task 2).

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_vrg_adhoc_epic.py -k adhoc -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epics.py src/vergil_tooling/bin/vrg_adhoc_epic.py tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_vrg_adhoc_epic.py
vrg-git commit -m "feat(epics): home ad-hoc epics via the resolver (self for private repos)"
```

---

### Task 4: Generalize the audit invariant (public-only flagging, fail-loud)

**Files:**
- Modify: `src/vergil_tooling/lib/epic_audit.py:208-234` (`epic_outside_dotgithub`), `render` text, and `--repo` targeting on `bin/vrg_epic_audit.py`
- Test: `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**
- Consumes: `github.is_public`.
- Produces: `epic_outside_dotgithub(org: str) -> list[str]` (same signature; now flags only *public*-repo epics outside `.github`).

- [ ] **Step 1: Write the failing tests**

```python
# tests/vergil_tooling/test_epic_audit.py
def test_epic_outside_dotgithub_flags_public_repo_epic() -> None:
    rows = [{"number": 9, "repository": {"nameWithOwner": "org/tooling"}}]
    with (
        patch("vergil_tooling.lib.epic_audit.github.read_json", return_value=rows),
        patch("vergil_tooling.lib.epic_audit.github.is_public", return_value=True),
    ):
        assert epic_audit.epic_outside_dotgithub("org") == ["org/tooling#9"]

def test_epic_outside_dotgithub_ignores_private_repo_epic() -> None:
    rows = [{"number": 4, "repository": {"nameWithOwner": "org/lab"}}]
    with (
        patch("vergil_tooling.lib.epic_audit.github.read_json", return_value=rows),
        patch("vergil_tooling.lib.epic_audit.github.is_public", return_value=False),
    ):
        assert epic_audit.epic_outside_dotgithub("org") == []

def test_epic_outside_dotgithub_fails_loud_on_probe_error() -> None:
    rows = [{"number": 4, "repository": {"nameWithOwner": "org/lab"}}]
    with (
        patch("vergil_tooling.lib.epic_audit.github.read_json", return_value=rows),
        patch("vergil_tooling.lib.epic_audit.github.is_public",
              side_effect=github.GitHubAPIError(1, "cmd", "boom")),
    ):
        with pytest.raises(github.GitHubAPIError):
            epic_audit.epic_outside_dotgithub("org")
```

- [ ] **Step 2: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epic_audit.py -k outside_dotgithub -v`
Expected: FAIL (private-repo epic still flagged).

- [ ] **Step 3: Implement the per-epic rule**

```python
# src/vergil_tooling/lib/epic_audit.py  epic_outside_dotgithub loop
    dotgithub = f"{org}/.github"
    violations: list[str] = []
    for item in raw if isinstance(raw, list) else []:
        name_with_owner = str((item.get("repository") or {}).get("nameWithOwner", ""))
        if not name_with_owner or name_with_owner == dotgithub:
            continue
        if not github.is_public(name_with_owner):   # private repo legitimately self-homes; fail-loud on probe error
            continue
        violations.append(f"{name_with_owner}#{item['number']}")
    return violations
```

Update the docstring (invariant 1 now: "public repos' epics live in `.github`; private repos self-home"). Update `render` (L330-340) text: "Epics outside `.github` (public repos)". Add `--repo` targeting to `vrg-epic-audit` so it sources from `resolve_epic_home` (org audit for a public target, self-audit for a private target).

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epic_audit.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/epic_audit.py src/vergil_tooling/bin/vrg_epic_audit.py tests/vergil_tooling/test_epic_audit.py
vrg-git commit -m "feat(epic-audit): flag epics outside .github only for public repos"
```

---

### Task 5: Roadmap sourcing from the resolved home

**Files:**
- Modify: `src/vergil_tooling/lib/roadmap.py:31-93` (`_open_epics`, `gather`, `render`), `bin/vrg_roadmap.py` (add `--repo`)
- Test: `tests/vergil_tooling/test_roadmap.py`

**Interfaces:**
- Consumes: `epics.resolve_epic_home`.
- Produces: `gather(org: str | None = None, *, home: str | None = None) -> list[EpicSummary]` — reads epics from *home* (default: `resolve_epic_home(org, ".github")` → `<org>/.github`).

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_roadmap.py
def test_gather_reads_from_explicit_private_home() -> None:
    epic_rows = [{"number": 2, "title": "Epic: lab", "createdAt": "2026-07-09T00:00:00Z",
                  "milestone": None, "labels": [{"name": "epic"}], "url": "u", "body": ""}]
    with (
        patch("vergil_tooling.lib.roadmap.github.read_json", return_value=epic_rows),
        patch("vergil_tooling.lib.roadmap.epics.child_states", return_value=[]),
    ) as _:
        out = roadmap.gather("org", home="org/lab")
    assert out[0].number == 2
    # the child rollup must key off the private home, not .github:
    # assert the IssueRef passed to child_states used repo "lab" (see Step 3)
```

- [ ] **Step 2: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_roadmap.py -k private_home -v`
Expected: FAIL (`gather` has no `home` kwarg / reads `.github`).

- [ ] **Step 3: Implement**

```python
# src/vergil_tooling/lib/roadmap.py
def _open_epics(home: str) -> list[Any]:
    raw: Any = github.read_json(
        "issue", "list", "--repo", home, "--label", "epic", "--state", "open",
        "--limit", "200", "--json", "number,title,createdAt,milestone,labels,url,body",
    )
    return raw if isinstance(raw, list) else []

def gather(org: str | None = None, *, home: str | None = None) -> list[EpicSummary]:
    if org is None:
        org = github.current_org()
    if home is None:
        home = epics.resolve_epic_home(org, ".github")   # -> f"{org}/.github"
    home_owner, home_repo = home.split("/", 1)
    summaries: list[EpicSummary] = []
    for epic in _open_epics(home):
        # ... unchanged perpetual/release skips ...
        number = int(epic["number"])
        children = epics.child_states(epics.IssueRef(home_owner, home_repo, number))
        # ... unchanged rollup ...
    return summaries
```

Update `render`'s `source` label to use `home`. Add `--repo` to `vrg-roadmap`: resolve `home = resolve_epic_home(owner, bare)` from the target (default cwd) and pass it to `gather`.

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_roadmap.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/roadmap.py src/vergil_tooling/bin/vrg_roadmap.py tests/vergil_tooling/test_roadmap.py
vrg-git commit -m "feat(roadmap): source epics from the resolved home (self for private repos)"
```

---

### Task 6: Bootstrap a private repo's epic machinery *(precondition; lands in the private repo's own plan)*

**Deliverable:** a private repo can self-host epics only once it has the roll-up automation, labels, and config. This is not vergil-tooling code; it is a checklist/runbook, executed in the private repo, paired with migrating its epic.

- [ ] Confirm/establish the "install standard workflows into a repo" path during execution (is there an existing `repo_init`/`vrg-github-repo-config` path, or is it manual?). Record the finding.
- [ ] Ensure `.github/workflows/epic-rollup.yml` (calling the reusable `ops-epic-rollup.yml@v2.1`) exists in the private repo, so closing a task rolls up its parent epic.
- [ ] Ensure the `epic` label and operational labels (`validation`, `deployment`, …) exist via `vrg-ensure-label` / `labels.json`.
- [ ] Ensure the private repo has a `vergil.toml`.
- [ ] Verify: create a throwaway epic + linked task in the private repo, close the task, confirm the roll-up fires and the epic closes. Then delete the throwaway.

---

### Task 7: Amend convention `vergil-project/.github#40`

**Files:** `epics/40-epic-task-convention/spec.md` in `vergil-project/.github`; the issue body of #40.

- [ ] Replace "epics live in `.github`" (§3.2) with the **explicit-target / derived-home** principle: an epic is created for a named target repo; its home is `<org>/.github` for a public target and the target repo itself for a private target (public `.github`); a private `.github` routes everything to `.github`.
- [ ] Add the case table and the **non-goal**: no private-`.github`-fronting-public-repos topology.
- [ ] State that tasks never move (only the epic home varies) and that the org roadmap omits private epics by design.
- [ ] Validate (`vrg-container-run -- vrg-validate`) in the `.github` worktree; hand off the PR (report-ready → human runs `vrg-submit-pr`).

---

### Task 8: Amend the `epic-create` and `migrate-repo` skills

**Files:** `epic-create/SKILL.md`, `migrate-repo/SKILL.md` in the marketplace repo (source of truth). Scope is limited to the **routing** edits; the separate epic-create *pipeline* gap (brainstorm → pushback → writing-plans → alignment → docs-PR + task expansion) that this session hit is tracked as its **own plugin/skill issue**, not this epic (see Follow-on).

- [ ] `epic-create`: preflight "confirm the org's epic home (`<owner>/.github`)" → "confirm the resolved epic home for the target"; teach `vrg-epic-create --repo`; document the resolver's case table.
- [ ] `migrate-repo`: teach the resolved home for a private member repo; keep cross-org out of scope.
- [ ] Validate and hand off the PR in the marketplace repo.

---

### Task 9: Document the visibility-flip procedure

**Files:** a runbook section (in `#40`'s spec or a `docs/` runbook in the appropriate repo).

- [ ] Document: on a public↔private flip, for each of the repo's epics run `vrg-epic-move` from the old home to the newly-resolved home (which re-points linked sub-issues). No new tool.
- [ ] Note the `--to-home` convenience wrapper is explicitly deferred unless the manual procedure proves painful.

---

### Task 10: `vrg-epic-link` visibility-boundary guard

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_epic_link.py:31-48` (`main`, add guard after `single_target_org`)
- Test: `tests/vergil_tooling/test_vrg_epic_link.py`

**Interfaces:**
- Consumes: `github.is_public`, `epics.parse_issue_ref`, `epics.single_target_org`.
- Produces: unchanged `main(argv) -> int` (returns 1 + stderr on a boundary violation).

- [ ] **Step 1: Write the failing test**

```python
# tests/vergil_tooling/test_vrg_epic_link.py
_MOD = "vergil_tooling.bin.vrg_epic_link"

def test_refuses_public_task_under_private_epic(capsys: pytest.CaptureFixture[str]) -> None:
    def pub(nwo: str) -> bool:
        return {"org/tooling": True, "org/lab": False}[nwo]  # task public, epic home private
    with (
        patch(f"{_MOD}.github.current_repo", return_value="org/tooling"),
        patch(f"{_MOD}.github.is_public", side_effect=pub),
        patch(f"{_MOD}.epics.add_child") as add_child,
    ):
        rc = main(["--epic", "org/lab#5", "--task", "org/tooling#9"])
    assert rc == 1
    err = capsys.readouterr().err
    assert "more public" in err and "Blocked-by" in err
    add_child.assert_not_called()

def test_allows_private_task_under_public_epic() -> None:
    def pub(nwo: str) -> bool:
        return {"org/.github": True, "org/lab": False}[nwo]  # task private, epic home public
    with (
        patch(f"{_MOD}.github.current_repo", return_value="org/lab"),
        patch(f"{_MOD}.github.is_public", side_effect=pub),
        patch(f"{_MOD}.github.target_org"),
        patch(f"{_MOD}.epics.add_child") as add_child,
    ):
        rc = main(["--epic", "org/.github#5", "--task", "org/lab#9"])
    assert rc == 0
    add_child.assert_called_once()
```

- [ ] **Step 2: Run to verify it fails**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_epic_link.py -k public -v`
Expected: FAIL (link created regardless of visibility).

- [ ] **Step 3: Implement the guard**

```python
# src/vergil_tooling/bin/vrg_epic_link.py  (in main, after single_target_org, before target_org block)
    task_repo = f"{task.owner}/{task.repo}"
    epic_home = f"{epic.owner}/{epic.repo}"
    if github.is_public(task_repo) and not github.is_public(epic_home):
        print(
            f"vrg-epic-link: refusing to link public task {task.slug} under "
            f"less-visible epic {epic.slug} — a public issue must not name a "
            f"private epic (leak) and cross-boundary roll-up cannot fire. File "
            f"the public work as its own task and reference it from the private "
            f"epic's body with 'Blocked-by: {task_repo}#{task.number}'.",
            file=sys.stderr,
        )
        return 1
```

Refresh the module docstring (epics live in their *resolved* home; note the boundary rule).

- [ ] **Step 4: Run to verify it passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_epic_link.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_epic_link.py tests/vergil_tooling/test_vrg_epic_link.py
vrg-git commit -m "feat(epic-link): refuse public task under a private epic (visibility boundary)"
```

> **Note:** Task 7 (convention #40) also documents this asymmetric rule and the `Blocked-by:`-from-the-private-side pattern.

---

## Follow-on (tail of epic) / separate issues

- **Docs build isolation:** ensure the public docs site never renders a private repo's docs — the self-contained principle extended to the published documentation surface. Separate spec/plan when picked up.
- **epic-create pipeline gap** *(separate plugin/skill issue, not this epic)*: the loaded `epic-create` skill omits the full brainstorm → pushback → writing-plans → alignment → docs-PR + task-expansion pipeline. File against the marketplace/standard-tooling suite.

---

## Self-Review

- **Spec coverage:** resolver + probe (T1) ✓; four consumers (T2 create, T3 ad-hoc, T4 audit, T5 roadmap) ✓; bootstrap precondition (T6, Issue 1) ✓; convention #40 (T7) ✓; skills (T8) ✓; flip procedure (T9) ✓; cross-visibility linkage guard (T10) ✓; docs-build follow-on ✓; explicit-target/derived-home (T2/T3/T4/T5 `--repo`) ✓; fail-loud (T1/T4/T10) ✓; INTERNAL-as-private (T1) ✓; org-roadmap-omits-private-by-design (T5/T7) ✓; asymmetric-linkage non-goal (T10/T7) ✓.
- **Placeholder scan:** none — every code step shows real code against real signatures; doc tasks list concrete edits, not "TBD".
- **Type consistency:** `resolve_epic_home(org, target_repo) -> str` ("owner/repo") used identically in T2/T3/T5; `is_public(name_with_owner) -> bool` used in T1/T4; `repo_visibility(name_with_owner) -> str` used in T1/T2; `ensure_adhoc_epic(target_repo) -> IssueRef` signature unchanged; `gather(org, *, home)` matches its callers.
