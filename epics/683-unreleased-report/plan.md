# vrg-unreleased Implementation Plan

- **Epic:** vergil-project/.github#683
- **Planning issue:** vergil-project/.github#468 (review history is in its
  comments)
- **Status:** approved

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `vrg-unreleased`, a read-only command that prints a
release-notes preview of everything merged on `develop` since a repo's newest
release, grouped by epic, for one repo or (with `--org`) every managed repo in
an org.

**Architecture:** A thin `bin/vrg_unreleased.py` over `lib/unreleased.py`,
which gathers from GitHub through the existing `lib/github.py` wrappers into
frozen dataclasses, then renders markdown. Gathering has four stages: the
release boundary, partitioning the compare range into PRs, classifying each PR
into a group, and assembling a `RepoReport`. Trust checks go through a new
shared `TrustedAuthors.is_trusted`.

**Tech Stack:** Python 3 (vergil-tooling), argparse, `gh` via `lib/github.py`,
pytest with `unittest.mock.patch`.

**Spec:** [`epics/683-unreleased-report/spec.md`](spec.md)

## Global Constraints

- All code lands in `vergil-project/vergil-tooling`. Paths below are relative
  to its root.
- `vrg-container-run -- vrg-validate` is the only validation command. PR CI
  gates 100% branch coverage, so run
  `vrg-container-run -- vrg-validate --all-versions` before handing off.
- Read-only: GET and GraphQL queries only, through `lib/github.py` (`read_json`,
  `read_json_paginated`) and `epics` helpers. No writes of any kind.
- No silent failures. A transport, auth or rate-limit error propagates. Only
  the bad-data cases in spec §4 become Unlinked entries.
- Test fixtures use invented logins (`example-*`), never real ones.
- Commits use `vrg-commit`. Agents hand off with `vrg-pr-workflow
  report-ready`, never `vrg-submit-pr`.
- Group order: epics (by earliest merge), `Ad-hoc`, `Dependency updates`,
  `Unlinked`, then `Release machinery` (only with `--show-release-prs`).
- Unlinked reasons, verbatim: `— no tracking ref`, `— multiple tracking refs`,
  `— tracking issue not found`, `— tracking issue author not trusted`,
  `— issue has no epic`, `— parent <ref> not found`,
  `— parent <ref> is not an epic`, `— deps branch without label`,
  `— direct commit`.
- In-flight line, verbatim: `release in flight (<ref>): results may be
  inconsistent until it finishes`.
- Hidden-count line, verbatim: `<k> release-machinery PR(s) hidden;
  --show-release-prs to list`, shown whenever `<k>` > 0 and the flag is off.

## Review Focus

1. **A squash subject naming an unrelated PR** (a cherry-pick keeping
   `(#N)`): the commit must not be attributed to PR N. Pinned in Task 3
   (`test_subject_pr_with_other_merge_sha_falls_back`).
2. **A dependency PR merged with a merge commit whose branch side holds
   several commits**: one entry, not one per inner commit. Pinned in Task 3
   (`test_merge_commit_owns_branch_side`).
3. **A tracking ref to another repo's issue** (`owner/repo#N`): resolved in
   that repo, not the current one. Pinned in Task 4
   (`test_cross_repo_tracking_ref`).
4. **A repo with tags but no GitHub Release for the newest one**: falls back
   to the tag commit's date and says so. Pinned in Task 2
   (`test_newest_release_without_github_release`).
5. **Version ordering past a single digit** (`v2.1.10` vs `v2.1.9`): numeric,
   not lexical. Pinned in Task 2 (`test_newest_release_orders_numerically`).

---

## Delivery

The tasks group into the two implementation issues of spec §7, each one PR:

- **Issue A (core):** Tasks 1–6.
- **Issue B (org mode):** Task 7, blocked by Issue A.

## File map

| File | Responsibility |
| --- | --- |
| `src/vergil_tooling/lib/trusted_authors.py` | Add `TrustedAuthors.is_trusted` (Task 1) |
| `src/vergil_tooling/lib/unreleased.py` | Data types, gather and render (Tasks 2–5, 7) |
| `src/vergil_tooling/bin/vrg_unreleased.py` | argparse, trust loading, error boundary (Tasks 6, 7) |
| `pyproject.toml` | `[project.scripts]` entry (Task 6) |
| `docs/site/docs/reference/cli-tools-overview.md` | `### vrg-unreleased` section (Tasks 6, 7) |
| `tests/vergil_tooling/test_trusted_authors.py` | `is_trusted` tests (Task 1) |
| `tests/vergil_tooling/test_unreleased.py` | lib tests (Tasks 2–5, 7) |
| `tests/vergil_tooling/test_vrg_unreleased.py` | CLI tests (Tasks 6, 7) |

---

### Task 1: Shared trust comparison

**Files:**

- Modify: `src/vergil_tooling/lib/trusted_authors.py` (the `TrustedAuthors`
  dataclass)
- Test: `tests/vergil_tooling/test_trusted_authors.py`

**Interfaces:**

- Produces: `TrustedAuthors.is_trusted(self, login: str) -> bool`

- [ ] **Step 1: Write the failing tests**

```python
_TA = TrustedAuthors(human=("example-human",), agents=("example-agent",))

def test_is_trusted_exact_human_and_agent() -> None:
    assert _TA.is_trusted("example-human")
    assert _TA.is_trusted("example-agent")
    assert not _TA.is_trusted("Example-Human")      # case-sensitive
    assert not _TA.is_trusted("example-humanx")
    assert not _TA.is_trusted("")

def test_is_trusted_strips_only_leading_app_prefix() -> None:
    assert _TA.is_trusted("app/example-agent")
    assert not _TA.is_trusted("app/app/example-agent")
    assert not _TA.is_trusted("x/app/example-agent")

def test_is_trusted_bot_suffix_matches_agents_only() -> None:
    assert _TA.is_trusted("example-agent[bot]")
    assert not _TA.is_trusted("example-human[bot]")  # never via the human list
    assert not _TA.is_trusted("example-agent[bot][bot]")
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: the three tests fail with `AttributeError: ... 'is_trusted'`.

- [ ] **Step 3: Implement `is_trusted`**

Strip one leading `_APP_PREFIX` if present. If what remains ends with
`[bot]`, strip that suffix once and return membership in `self.agents` only.
Otherwise return membership in `self.human + self.agents`. Exact string
comparison throughout. Update the module docstring's "a caller strips a
leading ``app/``" sentence to point at `is_trusted`.

- [ ] **Step 4: Run to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope trusted-authors --message "add a shared is_trusted comparison"`

---

### Task 2: Release boundary and in-flight check

**Files:**

- Create: `src/vergil_tooling/lib/unreleased.py`
- Create: `tests/vergil_tooling/test_unreleased.py`

**Interfaces:**

- Consumes: `github.read_json(*args)`, `github.read_json_paginated(endpoint,
  key=None)`, `github.GitHubAPIError`, `remote_config.is_not_found(exc)`,
  `release.is_release_tracking_issue(title, body)`,
  `release.is_release_branch(branch)`, `TrustedAuthors.is_trusted`.
- Produces:
  - `@dataclass(frozen=True) class Release: tag: str; version: tuple[int, int, int]; date: str; from_github_release: bool`
  - `newest_release(repo: str) -> Release | None`
  - `pending_commits(repo: str, tag: str) -> list[dict[str, Any]]` (the raw
    compare `commits` items)
  - `in_flight(repo: str, trust: TrustedAuthors) -> str | None` (a ref such
    as `#3799`, or None)
  - `class UnreleasedError(Exception)`, the base of every error this module
    raises on purpose.

Endpoint calls, fixed so tests can fake them:

| Purpose | Call |
| --- | --- |
| tags | `read_json_paginated(f"repos/{repo}/tags")` |
| release | `read_json("api", f"repos/{repo}/releases/tags/{tag}")` → `published_at` |
| tag commit (fallback) | `read_json("api", f"repos/{repo}/commits/{tag}")` → `commit.committer.date` |
| compare total | `read_json("api", f"repos/{repo}/compare/{tag}...develop?per_page=1")` → `total_commits` |
| compare | `read_json_paginated(f"repos/{repo}/compare/{tag}...develop", key="commits")` |
| open issues | `read_json_paginated(f"repos/{repo}/issues?state=open")` (skip items with `pull_request`) |
| open PRs | `read_json_paginated(f"repos/{repo}/pulls?state=open")` |

- [ ] **Step 1: Write the test helper and failing tests**

A `_fake_github(routes: dict[str, object])` context manager in the test
module patches `vergil_tooling.lib.github.read_json` and
`read_json_paginated`. It looks the call up by endpoint string (the
`read_json` endpoint is `args[1]`), and returns the routed value. It raises
`github.GitHubAPIError(1, ["gh", "api"], stderr="gh: Not Found (HTTP 404)")`
when the value is the sentinel `NOT_FOUND`, raises the value itself when it
is a `BaseException` instance, and fails the test on any unrouted call. An
autouse fixture in the module calls `github.repo_visibility.cache_clear()`
before each test, and every test that touches visibility patches
`github.is_public` explicitly. The helper also patches `github.graphql` and
`github.read_output` to fail the test on any call not routed (or patched
away, as `epics.parent_of` and `github.repo_exists` are in Task 4's tests), so
no test can reach real `gh`. Every later task reuses both.

```python
def test_newest_release_orders_numerically() -> None:
    routes = {"repos/o/r/tags": [{"name": "v2.1.9"}, {"name": "v2.1.10"},
                                 {"name": "v2.1"}, {"name": "develop-v2.1.10"}],
              "repos/o/r/releases/tags/v2.1.10": {"published_at": "2026-10-28T16:29:38Z"}}
    with _fake_github(routes):
        rel = unreleased.newest_release("o/r")
    assert rel == unreleased.Release("v2.1.10", (2, 1, 10), "2026-10-28T16:29:38Z", True)

def test_newest_release_without_github_release() -> None:
    routes = {"repos/o/r/tags": [{"name": "v1.0.0"}],
              "repos/o/r/releases/tags/v1.0.0": NOT_FOUND,
              "repos/o/r/commits/v1.0.0": {"commit": {"committer": {"date": "2026-01-02T00:00:00Z"}}}}
    with _fake_github(routes):
        rel = unreleased.newest_release("o/r")
    assert rel is not None and rel.from_github_release is False
    assert rel.date == "2026-01-02T00:00:00Z"

def test_newest_release_none_when_no_semver_tag() -> None:
    with _fake_github({"repos/o/r/tags": [{"name": "v2"}, {"name": "develop-v1.0.0"}]}):
        assert unreleased.newest_release("o/r") is None

def test_newest_release_propagates_non_404() -> None:
    routes = {"repos/o/r/tags": [{"name": "v1.0.0"}],
              "repos/o/r/releases/tags/v1.0.0": github.GitHubAPIError(1, ["gh"], stderr="HTTP 502")}
    with _fake_github(routes), pytest.raises(github.GitHubAPIError):
        unreleased.newest_release("o/r")

def test_pending_commits_complete() -> None:
    routes = {"repos/o/r/compare/v1.0.0...develop?per_page=1": {"total_commits": 2},
              "repos/o/r/compare/v1.0.0...develop": [{"sha": "a"}, {"sha": "b"}]}
    with _fake_github(routes):
        assert [c["sha"] for c in unreleased.pending_commits("o/r", "v1.0.0")] == ["a", "b"]

def test_pending_commits_truncated_raises() -> None:
    routes = {"repos/o/r/compare/v1.0.0...develop?per_page=1": {"total_commits": 3},
              "repos/o/r/compare/v1.0.0...develop": [{"sha": "a"}, {"sha": "b"}]}
    with _fake_github(routes), pytest.raises(unreleased.UnreleasedError, match="2 of 3"):
        unreleased.pending_commits("o/r", "v1.0.0")

def test_in_flight_trusted_tracking_issue() -> None:
    routes = {"repos/o/r/issues?state=open": [
                  {"number": 3799, "title": "release: 2.1.279", "body": "", "user": {"login": "example-agent[bot]"}}],
              "repos/o/r/pulls?state=open": []}
    with _fake_github(routes):
        assert unreleased.in_flight("o/r", _TRUST) == "#3799"

def test_in_flight_ignores_untrusted_and_reports_release_pr() -> None:
    routes = {"repos/o/r/issues?state=open": [
                  {"number": 9, "title": "release: 9.9.9", "body": "", "user": {"login": "example-stranger"}}],
              "repos/o/r/pulls?state=open": [
                  {"number": 3800, "head": {"ref": "release/2.1.279"}, "user": {"login": "example-agent[bot]"}}]}
    with _fake_github(routes):
        assert unreleased.in_flight("o/r", _TRUST) == "#3800"

def test_in_flight_none() -> None:
    with _fake_github({"repos/o/r/issues?state=open": [], "repos/o/r/pulls?state=open": []}):
        assert unreleased.in_flight("o/r", _TRUST) is None
```

`_TRUST` is `TrustedAuthors(human=("example-human",), agents=("example-agent",))`,
defined once at the top of the module.

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `ModuleNotFoundError: vergil_tooling.lib.unreleased`.

- [ ] **Step 3: Implement**

`newest_release` matches tag names against `^v(\d+)\.(\d+)\.(\d+)$` and takes
the max by integer tuple. `pending_commits` reads `total_commits`, fetches
every page, and raises `UnreleasedError("compare returned <got> of <total>
commits")` when the counts differ. `in_flight` returns the first trusted
open release tracking issue, else the first trusted open PR whose head is a
release branch, else None. An issue takes precedence over a PR.

- [ ] **Step 4: Run to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "find the release boundary and detect an in-flight release"`

---

### Task 3: Partition the compare range into PRs

**Files:**

- Modify: `src/vergil_tooling/lib/unreleased.py`
- Test: `tests/vergil_tooling/test_unreleased.py`

**Interfaces:**

- Consumes: `linkage.extract_merge_pr(subject)`, `release.is_release_branch`,
  `TrustedAuthors.is_trusted`, Task 2's `UnreleasedError`.
- Produces:
  - `@dataclass(frozen=True) class Change: sha: str; merged_at: str; subject: str; message_body: str; is_merge: bool; rebased: bool; pr: dict[str, Any] | None`
    (`is_merge` is `len(parents) >= 2`; `rebased` marks a collapsed
    rebase-merged run; `pr` is the REST pull object, or None for a direct
    commit)
  - `mainline(commits: list[dict[str, Any]]) -> list[tuple[dict[str, Any], frozenset[str]]]`:
    first-parent commits oldest first, each with the SHAs it brings in
  - `resolve_pr(repo: str, commit: dict[str, Any], later: frozenset[str]) -> dict[str, Any] | None`
    (`later` is the mainline SHAs after this commit)
  - `class UntrustedAuthorError(UnreleasedError)`, whose message names the
    item and the login
  - `changes(repo: str, commits: list[dict[str, Any]], trust: TrustedAuthors) -> tuple[list[Change], list[Change]]`:
    `(counted, release_machinery)`, both oldest first

Endpoint calls: `read_json("api", f"repos/{repo}/pulls/{n}")` and
`read_json("api", f"repos/{repo}/commits/{sha}/pulls")`.

**The mainline algorithm** (not determined by the signature):

```text
if commits is empty: return []        (nothing past the merge base is valid)
S   = {c.sha for c in commits}
tip = the single sha in S that is no commit's parent within S
      (zero or several → raise UnreleasedError("compare range has no single tip"))
main = walk parents[0] from tip while the sha is in S; reverse to oldest first
reach(x) = shas in S reachable from x (inclusive), walking all parents, within S
for each m in main with len(parents) >= 2:
    owned(m) = (union of reach(p) for p in parents[1:]) - reach(parents[0]) - main
every sha in S must be in main or exactly one owned set,
      else raise UnreleasedError naming the stray sha
```

- [ ] **Step 1: Write the failing tests**

Build commits with a `_commit(sha, parents, message, date="2026-10-28T00:00:00Z")`
helper that returns the compare item shape (`sha`, `parents[].sha`,
`commit.message`, `commit.committer.date`).

```python
def test_merge_commit_owns_branch_side() -> None:
    commits = [_commit("a", ["base"], "feat: x (#10)"),
               _commit("b1", ["a"], "deps 1"), _commit("b2", ["b1"], "deps 2"),
               _commit("m", ["a", "b2"], "Merge pull request #11 from o/deps/update-2.1.279")]
    main = unreleased.mainline(commits)
    assert [(c["sha"], owned) for c, owned in main] == [("a", frozenset()), ("m", frozenset({"b1", "b2"}))]

def test_stacked_branches_share_commits() -> None:
    # deps branch B was cut from deps branch A before A merged
    commits = [_commit("a1", ["base"], "A 1"),
               _commit("m1", ["base", "a1"], "Merge pull request #11 from o/deps/a"),
               _commit("b1", ["a1"], "B 1"),
               _commit("m2", ["m1", "b1"], "Merge pull request #12 from o/deps/b")]
    main = unreleased.mainline(commits)
    assert [(c["sha"], owned) for c, owned in main] == [
        ("m1", frozenset({"a1"})), ("m2", frozenset({"b1"}))]

def test_empty_range() -> None:
    assert unreleased.mainline([]) == []
    assert unreleased.changes("o/r", [], _TRUST) == ([], [])

def test_mainline_rejects_two_tips() -> None:
    with pytest.raises(unreleased.UnreleasedError, match="single tip"):
        unreleased.mainline([_commit("a", ["base"], "x"), _commit("b", ["base"], "y")])

def test_subject_pr_with_matching_merge_sha() -> None:
    routes = {"repos/o/r/pulls/10": _pr(10, merge_commit_sha="a")}
    with _fake_github(routes):
        assert unreleased.resolve_pr("o/r", _commit("a", ["base"], "feat: x (#10)"), frozenset())["number"] == 10

def test_subject_pr_with_other_merge_sha_falls_back() -> None:
    routes = {"repos/o/r/pulls/10": _pr(10, merge_commit_sha="zzz"),
              "repos/o/r/commits/a/pulls": [_pr(12, merge_commit_sha="a")]}
    with _fake_github(routes):
        assert unreleased.resolve_pr("o/r", _commit("a", ["base"], "fix: y (#10)"), frozenset())["number"] == 12

def test_subject_pr_404_falls_back_to_direct_commit() -> None:
    routes = {"repos/o/r/pulls/10": NOT_FOUND, "repos/o/r/commits/a/pulls": []}
    with _fake_github(routes):
        assert unreleased.resolve_pr("o/r", _commit("a", ["base"], "fix: y (#10)"), frozenset()) is None

def test_post_release_shape_is_nothing_unreleased() -> None:
    commits = [_commit("bump", ["v-tag"], "chore: bump version"),
               _commit("post", ["v-tag", "bump"], "Merge pull request #3801 from o/release/post-2.1.278")]
    routes = {"repos/o/r/pulls/3801": _pr(3801, merge_commit_sha="post", head="release/post-2.1.278")}
    with _fake_github(routes):
        counted, machinery = unreleased.changes("o/r", commits, _TRUST)
    assert counted == []
    assert [c.pr["number"] for c in machinery] == [3801]

def test_direct_commit_has_no_pr() -> None:
    routes = {"repos/o/r/commits/a/pulls": []}
    with _fake_github(routes):
        counted, _ = unreleased.changes("o/r", [_commit("a", ["base"], "hotfix")], _TRUST)
    assert counted[0].pr is None and counted[0].subject == "hotfix"

def test_prless_merge_lists_owned_commits() -> None:
    commits = [_commit("b1", ["base"], "side 1"),
               _commit("m", ["base", "b1"], "Merge branch 'x' into develop")]
    routes = {"repos/o/r/commits/m/pulls": []}
    with _fake_github(routes):
        counted, _ = unreleased.changes("o/r", commits, _TRUST)
    assert [(c.sha, c.pr) for c in counted] == [("m", None), ("b1", None)]

def test_rebase_merged_pr_is_one_entry() -> None:
    commits = [_commit("r1", ["base"], "feat: part 1"),
               _commit("r2", ["r1"], "feat: part 2"),
               _commit("r3", ["r2"], "feat: part 3")]
    pr = _pr(20, merge_commit_sha="r3", title="feat: three-part change")
    routes = {"repos/o/r/commits/r1/pulls": [pr]}
    with _fake_github(routes):
        counted, _ = unreleased.changes("o/r", commits, _TRUST)
    assert [(c.sha, c.rebased, c.pr["number"]) for c in counted] == [("r3", True, 20)]

def test_untrusted_pr_author_aborts() -> None:
    routes = {"repos/o/r/pulls/10": _pr(10, merge_commit_sha="a", author="example-stranger")}
    with _fake_github(routes), pytest.raises(unreleased.UntrustedAuthorError, match="#10.*example-stranger"):
        unreleased.changes("o/r", [_commit("a", ["base"], "feat: x (#10)")], _TRUST)
```

`_pr(number, *, merge_commit_sha, head="feature/x", author="example-agent[bot]",
labels=(), body="Ref #1", title=None, merged_at="2026-10-28T00:00:00Z")`
builds the REST pull shape, with `base.ref` set to `develop`.

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `AttributeError` for `mainline`, `resolve_pr` and
`changes`.

- [ ] **Step 3: Implement**

`resolve_pr` follows spec §5.2: accept the subject's PR only when
`merge_commit_sha == commit sha`, then try `commits/{sha}/pulls`. From that
list it takes the first entry with that `merge_commit_sha`, or failing that a
merged PR with base `develop` whose `merge_commit_sha` is in `later` (a
rebase merge), else None. A 404 on `pulls/{n}` falls through, and any other
error propagates. `changes` runs `mainline` and resolves each mainline commit.
When the PR's `merge_commit_sha` is a later mainline commit, every mainline
commit from this one through that one collapses into a single `Change` with
`sha` set to the PR's `merge_commit_sha` and `rebased=True`, and the walk
resumes after it. The renderer uses PR text when `is_merge or rebased`.
`changes` also checks every resolved PR's `user.login` with
`is_trusted` (raising `UntrustedAuthorError`), and routes PRs whose
`head.ref` is a release branch to `release_machinery`. A mainline merge commit
with no PR is a direct commit, and so is each commit it owns (oldest first,
straight after it), so no commit is hidden. `merged_at` is the PR's
`merged_at`, or the commit's committer date for a direct commit.
`subject`/`message_body` split `commit.message` at the first newline. The PR
object stored on `Change` is a copy with `body` normalised to `body or ""`
(REST returns `null` for an empty body), so later code never sees `None`.

- [ ] **Step 4: Run to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "partition the unreleased range into PRs"`

---

### Task 4: Classify each change into a group

**Files:**

- Modify: `src/vergil_tooling/lib/unreleased.py`
- Test: `tests/vergil_tooling/test_unreleased.py`

**Interfaces:**

- Consumes: `linkage.extract_tracking_ref(text)` (raises `ValueError` on
  several refs), `epics.parent_of(IssueRef) -> IssueRef | None`,
  `epics.IssueRef`, `epics.standing_kind_of(labels)`, `epics.ADHOC`,
  `epics.archive_re(epics.ADHOC)`, `github.is_public(name_with_owner)`,
  `github.repo_exists(owner, name)`, Task 3's
  `Change` and `UntrustedAuthorError`.
- Produces:
  - `GroupKind = Literal["epic", "adhoc", "deps", "unlinked"]`
  - `@dataclass(frozen=True) class Placement: kind: GroupKind; epic: epics.IssueRef | None = None; epic_title: str = ""; epic_private: bool = False; reason: str = ""`
  - `class Visibility` with `is_private(self, name_with_owner: str) -> bool`,
    caching `not github.is_public(...)` per run (one instance per gather, never
    module-level, so tests don't leak)
  - `classify(repo: str, change: Change, trust: TrustedAuthors, vis: Visibility) -> Placement`

Endpoint call: `read_json("api", f"repos/{owner}/{name}/issues/{n}")` for the
tracked issue (existence) and the parent epic (`title`, `labels`,
`user.login`).

Rules, in order (spec §5.3, §5.4):

1. `change.pr is None` → unlinked, `— direct commit`.
2. `dependency-update` label → deps.
3. head ref starts `deps/update-` → unlinked, `— deps branch without label`.
4. `extract_tracking_ref(body)`: `ValueError` → `— multiple tracking refs`;
   None → `— no tracking ref`. A bare `#N` resolves in `repo`.
"Gone" below means a `GitHubAPIError` that is a 404
(`remote_config.is_not_found`) or a 410 (`"410"` in its stderr or stdout),
tested by a private `_is_gone(exc)`. Any other error propagates.

5. Tracked issue gone: probe its repo with `github.repo_exists(owner, name)`.
   If the repo is visible → `— tracking issue not found`. If it isn't, raise
   `UnreleasedError("cannot read <ref>: its repo is not visible to the App")`,
   because a permission problem must fail the run (spec §4).
5a. Tracked issue author (`user.login`) fails `is_trusted` →
   `— tracking issue author not trusted`. `parent_of` is never called for it,
   so neither its native parent nor its `Parent:` body line is followed.
6. `parent_of` None → `— issue has no epic`.
6a. Parent issue fetch gone (for example a mistyped `Parent:` line): first
   probe the parent's repo with `github.repo_exists(owner, name)`, as rule 5
   does. If the repo is not visible, raise
   `UnreleasedError("cannot read <ref>: its repo is not visible to the App")`.
   If it is visible → `— parent <ref> not found`, with `<ref>` marked
   the `(private)` marker as in rule 9. `vis.is_private` is never called on a repo the
   probe hasn't confirmed.
7. Parent labels have `epic` and `standing_kind_of` is None → epic. Check the
   epic author with `is_trusted` (raising `UntrustedAuthorError`).
   `epic_private = vis.is_private(<parent owner/repo>)`.
8. Parent is this repo's live ad-hoc epic (`standing_kind_of` is `ADHOC` and
   title is `ADHOC.title_prefix + <bare repo>`), or one of its archives
   (`archive` and `ad-hoc` labels, and `archive_re(ADHOC)` matches with
   `name == <bare repo>`), and in both cases the parent's owner equals the
   repo's owner → adhoc.
9. Otherwise → unlinked, `— parent <ref> is not an epic`, where `<ref>` is the
   parent's `slug` followed by the `(private)` marker when
   `vis.is_private(<parent owner/repo>)`.

- [ ] **Step 1: Write the failing tests**

One test per rule, each asserting the full `Placement`, named:

- `test_classify_direct_commit`
- `test_classify_deps_label`
- `test_classify_deps_branch_without_label`
- `test_classify_multiple_tracking_refs`
- `test_classify_no_tracking_ref`
- `test_classify_null_body`: a PR built with `body=None` and passed through
  `changes` lands in Unlinked with `— no tracking ref`.
Every Task 4 test patches `github.repo_exists` and `epics.parent_of`
explicitly.

- `test_classify_tracking_issue_not_found`: 404, and `repo_exists` is True.
- `test_classify_tracking_issue_gone_410`: 410, and `repo_exists` is True.
  Same reason.
- `test_classify_parent_not_found`: `parent_of` returns `o/r#999`, whose
  fetch 404s, and `repo_exists` is True. Expect `— parent o/r#999 not found`.
- `test_classify_parent_repo_invisible_aborts`: `parent_of` returns
  `o/.githb#5`, whose fetch 404s, and `repo_exists("o", ".githb")` is False.
  Raises `UnreleasedError`, and `is_public` is never called for `o/.githb`.
- `test_classify_tracking_issue_repo_invisible_aborts`: 404, and
  `repo_exists` is False. Raises `UnreleasedError`.
- `test_classify_untrusted_tracking_issue_author`: the tracked issue's
  author is `example-stranger`. Expect Unlinked with
  `— tracking issue author not trusted`, and assert that `parent_of` was not
  called.
- `test_classify_issue_without_epic`
- `test_classify_finite_epic`
- `test_classify_untrusted_epic_author_aborts`
- `test_classify_adhoc_live`
- `test_classify_adhoc_archive`
- `test_classify_intake_archive_is_unlinked`
- `test_classify_other_repos_adhoc_is_unlinked`
- `test_classify_same_name_adhoc_other_owner_is_unlinked`: the parent is
  `Epic (ad hoc): r` in `other-org/.github`, while the repo is `o/r`.
- `test_classify_private_parent_ref_marked`

Plus these two:

```python
def test_cross_repo_tracking_ref() -> None:
    change = _change(pr=_pr(5, merge_commit_sha="a", body="Ref o/other#7"))
    routes = {"repos/o/other/issues/7": {"title": "t", "labels": [], "user": {"login": "example-human"}},
              "repos/o/.github/issues/683": {"title": "Epic: vrg-unreleased", "labels": [{"name": "epic"}],
                                             "user": {"login": "example-agent[bot]"}}}
    with _fake_github(routes), patch("vergil_tooling.lib.epics.parent_of",
                                     return_value=epics.IssueRef("o", ".github", 683)) as parent, \
         patch("vergil_tooling.lib.github.is_public", return_value=True):
        p = unreleased.classify("o/r", change, _TRUST, unreleased.Visibility())
    parent.assert_called_once_with(epics.IssueRef("o", "other", 7))
    assert p == unreleased.Placement("epic", epics.IssueRef("o", ".github", 683), "Epic: vrg-unreleased", False)
```

And `test_classify_public_pr_under_private_epic_marked`: the parent epic is
`o/private-repo#5`, and `github.is_public` returns False for
`o/private-repo`. Assert `p.kind == "epic" and p.epic_private is True`.

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `AttributeError` for `classify`, `Placement` and
`Visibility`.

- [ ] **Step 3: Implement `classify` and `Visibility` following the rules above**

- [ ] **Step 4: Run to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "group unreleased PRs by epic"`

---

### Task 5: Assemble and render a repo report

**Files:**

- Modify: `src/vergil_tooling/lib/unreleased.py`
- Test: `tests/vergil_tooling/test_unreleased.py`

**Interfaces:**

- Consumes: Tasks 2–4.
- Produces:
  - `@dataclass(frozen=True) class Entry: change: Change; placement: Placement`
  - `@dataclass(frozen=True) class Group: heading: str; entries: tuple[Entry, ...]`
  - `@dataclass(frozen=True) class RepoReport: repo: str; private: bool; release: Release | None; in_flight: str | None; groups: tuple[Group, ...]; release_machinery: tuple[Change, ...]`,
    with properties `pr_count: int`, `direct_count: int` and
    `oldest_merge: str | None` (all over `groups` only, never
    `release_machinery`)
  - `gather_repo(repo: str, trust: TrustedAuthors, vis: Visibility) -> RepoReport`
    (no release → empty groups, and no compare call)
  - `render_repo(report: RepoReport, *, verbose: bool, show_release_prs: bool) -> str`

Group headings: `Epic <slug>: <title>` (with the `(private)` marker appended when
`epic_private`), `Ad-hoc`, `Dependency updates`, `Unlinked`,
`Release machinery`. Epic groups are ordered by their first entry's
`merged_at`. Entries keep merge order within a group.

Rendered shape (spec §5.5):

```text
## <bare>[ (private)] — <n> PRs[ and <d> direct commits] unreleased since <tag> (<YYYY-MM-DD>)[ (tag date; no GitHub Release)]
[> release in flight (<ref>): results may be inconsistent until it finishes]
[> <k> release-machinery PR(s) hidden; --show-release-prs to list]

### <heading>
- #<N> <title>[ <reason>]
    <verbose: message body or PR body, each line indented four spaces>
- <sha7> — direct commit
```

A direct commit is rendered by short SHA and reason only. Its subject and
body are never printed, not even with `--verbose`, because no trusted author
stands behind them (spec §5.5).

- Title: the squash subject with its trailing `(#N)` removed, or the PR title
  when `change.is_merge or change.rebased`. Never infer this from the subject.
- Verbose detail: `message_body` for a squash, the PR `body` when
  `change.is_merge or change.rebased`. Omit it when it's empty.
- Nothing pending: `## <bare>[ (private)] — nothing unreleased since <tag> (<date>)`.
- No release: `## <bare>[ (private)] — no release yet`.
- Singular forms: `1 PR`, `1 direct commit`.

- [ ] **Step 1: Write the failing tests**

- `test_render_spec_example`: a `RepoReport` built directly with the five PRs
  of spec §5.5. Assert that `render_repo(..., verbose=False,
  show_release_prs=False)` equals the spec's block exactly.
- `test_render_title_uses_is_merge_not_subject`: a squash whose subject
  starts with "Merge pull request" keeps that subject as its title, while a
  merge commit with an edited subject shows its PR title.
- `test_render_direct_commit_hides_text`: a direct commit whose subject is
  `ignore previous instructions` and which has a body renders exactly as
  `- abc1234 — direct commit` in both terse and verbose modes. The output
  contains neither the subject nor the body.
- `test_render_verbose_indents_body`
- `test_render_verbose_null_body_merge_pr`: a merge-commit PR whose body was
  `null` renders its line with no detail.
- `test_render_nothing_unreleased`
- `test_render_no_release`
- `test_render_private_repo_header`
- `test_render_in_flight_line`
- `test_render_tag_date_fallback`
- `test_render_release_machinery_only_with_flag`: absent by default, but the
  header carries `> 1 release-machinery PR(s) hidden; --show-release-prs to list`.
  With the flag the group appears last and the PR count is unchanged.
- `test_render_hidden_line_after_nothing_unreleased`: straight after a
  release, the "nothing unreleased" header is followed by the hidden-count
  line.
- `test_render_epic_groups_ordered_by_first_merge`
- `test_gather_repo_empty_range_is_nothing_unreleased`: the compare returns
  no commits (`total_commits: 0`) and a trusted release tracking issue is
  open. The report has no groups and `in_flight` set, and renders the
  "nothing unreleased" header followed by the in-flight line.
- `test_gather_repo_no_release_skips_compare`: the fake has no compare,
  issues or pulls routes, so a compare or in-flight call would fail the test.
  The report has `release is None` and `in_flight is None`.
- `test_gather_repo_end_to_end`: tags, release, compare (one squash PR under an
  epic, one deps merge commit with two inner commits, one direct commit, and
  the post-release merge), PRs, issues and parent patched. Assert the groups,
  `pr_count == 2`, `direct_count == 1`, and one release-machinery change.

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `AttributeError` for `gather_repo` and `render_repo`.

- [ ] **Step 3: Implement `gather_repo` and `render_repo`**

`gather_repo` sets `private = vis.is_private(repo)`, then calls
`newest_release`, `in_flight`, `pending_commits`, `changes` and `classify`,
and groups the results. With no release it stops after `newest_release`:
neither `in_flight` nor the compare runs.

- [ ] **Step 4: Run to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "render the unreleased report for a repo"`

---

### Task 6: The command, single-repo mode

**Files:**

- Create: `src/vergil_tooling/bin/vrg_unreleased.py`
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-unreleased = "vergil_tooling.bin.vrg_unreleased:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a new
  `### vrg-unreleased` section after `### vrg-activity-log`)
- Test: `tests/vergil_tooling/test_vrg_unreleased.py`

**Interfaces:**

- Consumes: `unreleased.gather_repo`, `unreleased.render_repo`,
  `unreleased.Visibility`, `unreleased.UnreleasedError`,
  `trusted_authors.load_trusted_authors()`,
  `trusted_authors.TrustedAuthorsNotConfiguredError`,
  `github.current_repo()`, `github.target_org(org)`.
- Produces: `main(argv: list[str] | None = None) -> int`, decorated with
  `@cli_main` and the module's first function, modelled on
  `bin/vrg_roadmap.py`.

Arguments in this task: `--repo OWNER/NAME`, `--verbose`,
`--show-release-prs`. The `--help` epilog states: "Output marked (private)
must not be relayed into public artifacts (issue comments, PR bodies,
report-ready notes)."

Flow: load the trust list first, so a missing one fails before any GitHub
call. Resolve the repo (`--repo`, or `github.current_repo()`). Then, inside
`github.target_org(owner)`, gather and print. Catch
`TrustedAuthorsNotConfiguredError`, a `ValueError` from the
`load_trusted_authors()` call alone (malformed list), and `UnreleasedError` (which includes
`UntrustedAuthorError`), print `vrg-unreleased: <message>` to stderr and
return 1. Handle `MissingGitHubTokenError` and `NoInstallationError` as
`vrg_roadmap.py` does. Let everything else propagate.

- [ ] **Step 1: Write the failing tests**

- `test_missing_trust_list_fails_before_github`: patch `load_trusted_authors`
  to raise and `gather_repo` to fail if called. Expect exit 1, with stderr
  containing `vrg-unreleased:`.
- `test_invalid_trust_list_exits_1`: `load_trusted_authors` raises
  `ValueError("trusted_authors: bad entry")`. Expect exit 1, a clean
  `vrg-unreleased:` line, and no traceback.
- `test_untrusted_author_exits_1`: `gather_repo` raises
  `UntrustedAuthorError("PR #10 by example-stranger")`. Expect exit 1, with
  stderr naming both.
- `test_repo_flag_renders`: patch `gather_repo` and `render_repo`. Assert
  they're called with `"o/r"` and `verbose=True, show_release_prs=True`, and
  that stdout is the rendered text.
- `test_default_repo_is_current`: patch `github.current_repo` to `"o/cwd"`.
- `test_repo_flag_requires_owner`: `--repo r` exits 1.
- `test_private_repo_explicit_allowed`: a private `--repo` renders without
  `--include-private`.

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `ModuleNotFoundError: vergil_tooling.bin.vrg_unreleased`.
The entry-point test and the overview test fail as well until Step 3.

- [ ] **Step 3: Implement `main`, add the entry point and the overview section**

Use the attribute table of `vrg-roadmap`:

- Source: `vergil_tooling.bin.vrg_unreleased`.
- Args: `--repo OWNER/NAME`, `--verbose`, `--show-release-prs`.
- Preconditions: GitHub App credentials for the target org, and the
  `[trusted_authors]` stanza.
- Failure mode: error on stderr.
- Exit codes: 0 report produced, 1 error.
- Status: Active.

Put one paragraph above the table describing the release-notes preview, and
the `(private)` relay rule as its last sentence.

- [ ] **Step 4: Run to verify they pass, on every interpreter**

Run: `vrg-container-run -- vrg-validate --all-versions`
Expected: PASS, with 100% branch coverage.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "add the vrg-unreleased command"`

Issue A ends here: hand off with `vrg-pr-workflow report-ready`.

---

### Task 7: Org mode (Issue B)

**Files:**

- Modify: `src/vergil_tooling/lib/unreleased.py`
- Modify: `src/vergil_tooling/bin/vrg_unreleased.py`
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_unreleased.py`, `tests/vergil_tooling/test_vrg_unreleased.py`

**Interfaces:**

- Consumes: `github.list_org_repos(org) -> list[str]` (bare names,
  non-archived), `epics.is_managed_repo(repo)`,
  `read_json("api", f"repos/{repo}/branches/develop")` (404 means no
  `develop`), and Task 5's `gather_repo` and `RepoReport`.
- Produces:
  - `@dataclass(frozen=True) class OrgReport: org: str; reports: tuple[RepoReport, ...]`
  - `gather_org(org: str, trust: TrustedAuthors, vis: Visibility, *, include_private: bool) -> OrgReport`
  - `render_org(report: OrgReport, *, verbose: bool, show_release_prs: bool, now: datetime) -> str`

`list_org_repos` returns bare names. Qualify each as `f"{org}/{name}"`
before any other call: `is_managed_repo`, `vis.is_private`, the `develop`
branch probe and `gather_repo` all take `owner/name`.

Repo filter, in this order to keep calls low: skip a private repo unless
`include_private` (`vis.is_private`), then an unmanaged one, then one with no
`develop`. One failed lookup in any repo propagates (spec §5.8).

Rendered shape (spec §5.6):

```text
# Unreleased work in <org>

| Repo | PRs | Newest release | Released | Oldest unreleased |
|---|---|---|---|---|
| <bare>[ (private)] | <n> | <tag> | <YYYY-MM-DD> | <days>d |

Nothing unreleased: <bare>[ (private)], <bare>[ (private)].
No release yet: <bare>[ (private)].

<render_repo block for each table row, same order>
```

Rows cover repos with `pr_count + direct_count > 0`, sorted by `oldest_merge`
ascending (longest waiting first), ties broken by repo name. `<days>` is the
whole days between `oldest_merge` and `now`. The `PRs` cell is `<n>`, or
`<n> + <d> direct` when `direct_count > 0`. Leave out either trailing line
when its list is empty.

The command runs the whole org sweep (`list_org_repos` included) inside
`github.target_org(org)`, so the App token is minted for the target org
rather than the current checkout's org.

- [ ] **Step 1: Write the failing tests**

- `test_gather_org_excludes_private_by_default`: three repos, one private.
  The private one is never gathered.
- `test_gather_org_include_private_marks`: with `include_private=True`, the
  private repo's row and block carry the `(private)` marker.
- `test_gather_org_skips_unmanaged_and_no_develop`: assert that the calls
  made use the qualified strings (`is_managed_repo("o/a")` and
  `repos/o/b/branches/develop`), never a bare name.
- `test_render_org_sorts_longest_waiting_first`
- `test_render_org_trailing_lines`
- `test_render_org_trailing_lines_mark_private`: one private repo with
  nothing pending and one private repo with no release both show
  the `(private)` marker in their trailing line.
- `test_render_org_days`: `now` is fixed, giving `3d`.
- `test_render_org_direct_only_row`: a repo with 0 PRs and 2 direct commits
  shows `0 + 2 direct`.
- CLI `test_org_scopes_token_to_target_org`: with `github.target_org`
  patched, `--org other` enters it with `"other"`, and `gather_org` runs
  inside it.
- `test_org_failed_lookup_aborts`: one repo's tags call raises a 502, and
  `gather_org` raises.
- CLI: `test_org_flag_defaults_to_current_owner` (`--org` with no value uses
  the owner of `github.current_repo()`), `test_include_private_requires_org`
  (exit 1, `--include-private` alone), `test_include_private_with_bare_org`
  (`--include-private --org` runs org mode), and
  `test_org_and_repo_mutually_exclusive` (exit 1, pinned to the bare form
  `--org --repo o/r` as well as `--org o --repo o/r`).

- [ ] **Step 2: Run to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL with `AttributeError` for `gather_org` and `render_org`, and
the CLI tests fail on the unknown `--org`.

- [ ] **Step 3: Implement `gather_org` and `render_org`, and add `--org [ORG]` (`nargs="?"`, `const=""`, `default=None`) and `--include-private` to the command, then update the overview's Args row and paragraph**

Every org-mode check uses `args.org is not None`, never truthiness, because
a bare `--org` gives the falsy `""`.

- [ ] **Step 4: Run to verify they pass, on every interpreter**

Run: `vrg-container-run -- vrg-validate --all-versions`
Expected: PASS, with 100% branch coverage.

- [ ] **Step 5: Refactor with the tests green**

Tidy names, duplication and structure without changing behaviour, then re-run `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope unreleased --message "add org mode to vrg-unreleased"`

Hand off Issue B with `vrg-pr-workflow report-ready`.
