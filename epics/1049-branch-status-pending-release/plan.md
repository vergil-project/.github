# vrg-repo-status: pending release — implementation plan

- **Epic:** vergil-project/.github#1049
- **Planning issue:** vergil-project/.github#1047 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task below is one vergil-tooling issue, one
> worktree and one PR, driven by `epic-implement` / `issue-implement`. Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename `vrg-worktree-status` to `vrg-repo-status` and add a Pending
release section that lists every change in `develop` but not `main`, mapped to
its PR by commit SHA, with the current release tags.

**Architecture:** A new library module, `lib/pending_release.py`, reads the
remote state (one `ls-remote`, a paginated compare, batched GraphQL) and
returns a frozen `PendingRelease` value or raises `PendingReleaseError`. The
command, `bin/vrg_repo_status.py`, renders it with the existing table helpers,
gates it on `[publish].release`, and exits non-zero on an error.

**Tech Stack:** Python 3, vergil-tooling's `lib/git.py` and `lib/github.py`
wrappers (both retry transient network failures), pytest with
`unittest.mock`.

**Spec:** [`spec.md`](spec.md) in this directory.

## Global Constraints

- Validation is `vrg-container-run -- vrg-validate` only. No direct pytest,
  linter or formatter runs. "Run the tests" below always means this command.
- The command is read-only: it never fetches and never writes a ref
  (spec §6).
- Every git call goes through `git.read_output`, and every GitHub call
  through `github.read_json` / `github.graphql`. No new retry logic (spec
  §5.4).
- The release branch is `main` (hardcoded); the integration branch comes from
  `config.integration_branch` (spec §6).
- A lookup that fails or is incomplete is an error, never a partial list
  (spec §4, §5.4).
- `develop-v*` tags and rolling major `vX` tags are ignored (spec §3, §5.3).
- Historical records keep the old name: `releases/`, `CHANGELOG.md`, dated
  `docs/specs/2026-*` and `docs/plans/2026-*` files, and `epics/` (spec §5.1).

## Review Focus

1. **A batch PR rebase-merged as several commits** whose subjects carry issue
   numbers must render as one row with the PR's own number (Task 2,
   `test_batch_pr_commits_group_into_one_row`).
2. **Annotated tags**: `ls-remote` lists both `refs/tags/vX^{}` (peeled) and
   the tag object. The peeled commit is the one compared (Task 2,
   `test_parse_release_tags_uses_peeled_commit`).
3. **Numeric version order**: `v2.1.303` outranks `v2.1.99`, and `v2.10`
   outranks `v2.9` (Task 2, `test_parse_release_tags_orders_numerically`).
4. **A compare that returns fewer commits than `total_commits`** is an error,
   not a short list (Task 2, `test_incomplete_compare_raises`).
5. **A repo with a `main` branch but `[publish].release` false**, such as
   `docs`, shows no section (Task 3,
   `test_section_skipped_when_not_published`).

## Delivery

| Task | Repo | Delivers | Blocked by |
|---|---|---|---|
| 1 | vergil-tooling | Rename to `vrg-repo-status` | — |
| 2 | vergil-tooling | `lib/pending_release.py` (data layer) | — |
| 3 | vergil-tooling | The Pending release section in `vrg-repo-status`, with docs | 1, 2 |

The bookends are already filed: the documentation review
(vergil-project/vergil-tooling#4256) is blocked by Task 3, and the
retrospective (vergil-project/.github#1051) closes the epic. No operational
tasks are needed: the command is human-run and read-only, and the pipeline's
tests cover it.

---

### Task 1: Rename `vrg-worktree-status` to `vrg-repo-status`

**Files:**
- Modify: `pyproject.toml` (the console-script entry: `vrg-repo-status =
  "vergil_tooling.bin.vrg_repo_status:main"`, replacing the
  `vrg-worktree-status` line)
- Move: `src/vergil_tooling/bin/vrg_worktree_status.py` →
  `src/vergil_tooling/bin/vrg_repo_status.py` (`vrg-git mv`)
- Move: `tests/vergil_tooling/test_vrg_worktree_status.py` →
  `tests/vergil_tooling/test_vrg_repo_status.py`
- Modify (references only): `src/vergil_tooling/bin/vrg_finalize_pr.py`,
  `src/vergil_tooling/lib/git.py`, `src/vergil_tooling/lib/worktrees.py`,
  `tests/vergil_tooling/test_worktrees.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/fixtures/host/uv-tool-list.txt`,
  `tests/vergil_tooling/fixtures/update_deps/vrg-validate-typecheck-failure.stdout.txt`,
  `docs/specs/worktree-convention.md`,
  `docs/site/docs/guides/git-workflow.md`,
  `docs/site/docs/guides/permission-model.md` (the tool list; keep it
  alphabetical, so `vrg-repo-status` moves before `vrg-resolve-tracking-issue`),
  `docs/site/docs/guides/consuming-repo-setup.md`,
  `docs/site/docs/reference/cli-tools-overview.md`,
  `docs/site/docs/reference/index.md`

**Interfaces:**
- Produces: module `vergil_tooling.bin.vrg_repo_status` with `main(argv:
  list[str] | None = None) -> int` and `parse_args`, unchanged in behaviour.
  The test module's `_MOD = "vergil_tooling.bin.vrg_repo_status"`.
- Internal names (`gather_worktree_status`, `WorktreeStatus`, …) are **not**
  renamed: they describe worktrees, which they still are.

- [ ] **Step 1: Move the module and its tests**, and update the test module's
  import line and `_MOD` constant to `vrg_repo_status`.
- [ ] **Step 2: Update the entry point** in `pyproject.toml`.
- [ ] **Step 3: Update the module docstring's first line** to "Show the state
  of the repository clone: its worktrees, grouped by kind, and (Task 3) what
  is pending release." Update the argparse `description` to match.
- [ ] **Step 4: Replace every remaining `vrg-worktree-status` /
  `vrg_worktree_status`** in the files listed, outside the historical records
  in Global Constraints. Check: `grep -rn "worktree.status"` over `src`,
  `tests`, `docs/site` and `docs/specs/worktree-convention.md` prints
  nothing. The function name `gather_worktree_status` is expected to remain:
  run the grep with `-w vrg-worktree-status -w vrg_worktree_status`.
- [ ] **Step 5: Run the tests.** `vrg-container-run -- vrg-validate`.
  Expected: green; the moved tests pass unchanged.
- [ ] **Step 6: Commit.** `vrg-commit --type refactor --scope repo-status
  --message "rename vrg-worktree-status to vrg-repo-status"`

---

### Task 2: `lib/pending_release.py`, the data layer

**Files:**
- Create: `src/vergil_tooling/lib/pending_release.py`
- Test: `tests/vergil_tooling/test_pending_release.py`

**Interfaces:**
- Produces (all in `vergil_tooling.lib.pending_release`):

  ```python
  class PendingReleaseError(RuntimeError): ...   # message is the human reason

  class ChangeKind(enum.Enum):
      DEPS = "deps"; RELEASE = "release"; OTHER = "other"; NO_PR = "no-pr"

  @dataclass(frozen=True)
  class ReleaseTags:
      latest: str | None           # e.g. "v2.1.303"; None when no vX.Y.Z tag
      rolling: str | None          # e.g. "v2.1"; None when no vX.Y tag
      rolling_target: str | None   # the vX.Y.Z on the rolling tag's commit,
                                   # else its 7-char short SHA; None iff rolling is None

  @dataclass(frozen=True)
  class PendingChange:
      pr_number: int | None        # None only for ChangeKind.NO_PR
      branch: str | None
      title: str                   # PR title, or commit subject for NO_PR
      kind: ChangeKind
      type_label: str              # "deps", "release", the conventional type, or "-"
      merged_ts: float             # epoch seconds: PR mergedAt, or commit date for NO_PR
      shas: tuple[str, ...]        # first-parent commits of this change, oldest first

  @dataclass(frozen=True)
  class PendingRelease:
      tags: ReleaseTags
      main_moved_ts: float         # main tip's committer date
      changes: tuple[PendingChange, ...]   # merge order, oldest first

  def parse_release_tags(ls_remote: str) -> ReleaseTags: ...
  def first_parent_chain(commits: list[dict[str, object]], head: str) -> list[dict[str, object]]: ...
  def classify(title: str, branch: str | None, labels: Sequence[str]) -> tuple[ChangeKind, str]: ...
  def gather_pending_release(*, repo: str, integration: str, remote: str = "origin") -> PendingRelease: ...
  ```

- Consumes: `git.read_output`, `github.read_json`, `github.graphql`.

**Algorithm decisions** (the rest is the implementer's):

- **Refs, one call:** `git.read_output("ls-remote", remote, "refs/heads/main",
  f"refs/heads/{integration}", "refs/tags/v*")`. Either branch missing →
  `PendingReleaseError("main or <integration> not found on <remote>")`.
- **Tags:** three-part `^v(\d+)\.(\d+)\.(\d+)$` and two-part `^v(\d+)\.(\d+)$`,
  compared as int tuples. A `refs/tags/<t>^{}` line's SHA overrides the plain
  line's, since it is the peeled commit. Everything else (`develop-v*`, `vX`)
  is ignored. If several `vX.Y.Z` tags share the rolling tag's commit,
  `rolling_target` is the highest.
- **Commits:** the `repos/{repo}/compare/{main_sha}...{dev_sha}` endpoint,
  read with `github.read_json("api", f"{endpoint}?per_page=100&page={n}")`
  from `n = 1` until the collected `commits` count equals `total_commits`. A
  page that adds nothing before then → `PendingReleaseError("compare
  main...<integration> returned <k> of <total> commits")`. `main_moved_ts`
  comes from `base_commit.commit.committer.date` on page 1.
- **First-parent chain:** start at `dev_sha` and follow `parents[0].sha`
  while it is in the compare set. Return the chain oldest first.
- **PR mapping:** GraphQL in chunks of 50 SHAs, one aliased `object(oid:
  "<sha>") { ... on Commit { associatedPullRequests(first: 5) { nodes {
  number title headRefName mergedAt baseRefName merged labels(first: 20) {
  nodes { name } } } } } }` per SHA, under `repository(owner:, name:)`. Before
  a SHA is interpolated, assert it matches `^[0-9a-f]{40}$`. Keep only nodes
  with `merged` true and `baseRefName == integration`. A commit with more
  than one such PR → `PendingReleaseError`. Consecutive chain commits with the
  same PR form one `PendingChange`. A commit with none is a `NO_PR` change of
  its own.
- **classify:** a `release/post-` branch prefix → `(RELEASE, "release")`; a
  `deps/update-` prefix or a `dependency-update` label → `(DEPS, "deps")`;
  otherwise `(OTHER, m.group(1))` for `^([a-z]+)(\([^)]*\))?!?:`, or
  `(OTHER, "-")`.
- **Errors:** `gather_pending_release` catches `subprocess.CalledProcessError`
  (which covers `GitHubAPIError`), `RuntimeError` from `graphql`, and
  `KeyError` / `TypeError` / `ValueError` from parsing the response shape. It
  re-raises each as `PendingReleaseError(<reason>) from exc`, with the
  reason taken from the error's stderr or message. Nothing is swallowed.

- [ ] **Step 1: Write the failing tests** in `test_pending_release.py`, with
  `git.read_output`, `github.read_json` and `github.graphql` patched:
  - `test_parse_release_tags_latest_and_rolling`: tags `v2.1.302`,
    `v2.1.303` and `v2.1`, the last on 303's commit →
    `ReleaseTags("v2.1.303", "v2.1", "v2.1.303")`.
  - `test_parse_release_tags_rolling_lags`: `v2.1` on 302's commit →
    `rolling_target == "v2.1.302"`.
  - `test_parse_release_tags_uses_peeled_commit`: annotated `v2.1` whose
    `^{}` line points at 303's commit → `rolling_target == "v2.1.303"`.
  - `test_parse_release_tags_orders_numerically`: `v2.1.99` and `v2.1.303` →
    latest `v2.1.303`; `v2.9` and `v2.10` → rolling `v2.10`.
  - `test_parse_release_tags_ignores_develop_and_major`: `develop-v2.1.304`
    and `v2` present → neither is chosen.
  - `test_parse_release_tags_none`: no tags → `ReleaseTags(None, None,
    None)`. A rolling tag on an untagged commit → `rolling_target` is the
    7-char SHA.
  - `test_first_parent_chain_skips_side_commits`: a merge commit whose second
    parent's commits are in the compare set → only the first-parent commits,
    oldest first.
  - `test_classify`: `("chore(release): back-merge 2.1.303 and bump to
    2.1.304", "release/post-2.1.303", [])` → `(RELEASE, "release")`;
    `("chore(deps): dependency update for 2.1.304", "deps/update-2.1.304",
    ["dependency-update"])` → `(DEPS, "deps")`; `("feat(validate): refuse
    ubuntu-latest runner labels", "feature/4249-x", [])` → `(OTHER,
    "feat")`; `("Tidy things", "feature/1-x", [])` → `(OTHER, "-")`.
  - `test_batch_pr_commits_group_into_one_row`: three chain commits whose
    subjects end `(#11)`, `(#12)` and `(#13)`, all associated with PR #20 →
    one change, `pr_number == 20`, three `shas`.
  - `test_merge_order_oldest_first`: a back-merge, then a deps PR, then a feat
    PR → the changes in that order.
  - `test_commit_without_pr_is_no_pr_change`: a commit with no associated
    merged PR into `develop` → `kind == NO_PR`, `pr_number is None`, title =
    the commit subject.
  - `test_unmerged_or_other_base_pr_ignored`: an associated PR into `main`,
    or an unmerged one, does not count.
  - `test_incomplete_compare_raises`: `total_commits` 300, pages yielding 250
    and then 0 → `PendingReleaseError`, message containing `250 of 300`.
  - `test_missing_branch_raises`: `ls-remote` without `refs/heads/main` →
    `PendingReleaseError`.
  - `test_api_failure_raises_pending_release_error`: `graphql` raising
    `RuntimeError("GraphQL errors: …")` → `PendingReleaseError` whose
    `__cause__` is that error.
  - `test_multiple_prs_for_one_commit_raises`: two merged PRs into `develop`
    on one commit → `PendingReleaseError`.
- [ ] **Step 2: Run the tests.** `vrg-container-run -- vrg-validate`.
  Expected: FAIL, `ModuleNotFoundError: vergil_tooling.lib.pending_release`.
- [ ] **Step 3: Implement the module** to the Interfaces and Algorithm
  decisions above.
- [ ] **Step 4: Run the tests.** `vrg-container-run -- vrg-validate`.
  Expected: green, coverage gate included.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope repo-status
  --message "add the pending-release data layer"`

---

### Task 3: The Pending release section in `vrg-repo-status`

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_repo_status.py`
- Test: `tests/vergil_tooling/test_vrg_repo_status.py`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (the
  `vrg-repo-status` entry: describe the section, its release line, the
  publish gate and the non-zero exit on a lookup error)

**Interfaces:**
- Consumes: Task 1's module; Task 2's `gather_pending_release`,
  `PendingRelease`, `PendingChange`, `ReleaseTags`, `ChangeKind`,
  `PendingReleaseError`; `config.read_config(root).publish.release`;
  `github.current_repo()`; the module's existing `_render_table` and
  `_format_age`.
- Produces: `_PENDING_COLUMNS = ("PR", "BRANCH", "TYPE", "MERGED", "TITLE")`,
  `_TITLE_WIDTH = 72`, and these functions, each returning printable text:
  `_release_line(tags: ReleaseTags, main_moved_ts: float, now: float) -> str`,
  `_pending_rows(changes, now) -> list[tuple[str, ...]]`,
  `_pending_summary(changes) -> str`, and
  `_print_pending(root: Path, integration: str, now: float) -> bool`, which
  returns False only on an error.

**Exact copy** (spec §5.3, §5.5):

- Heading: `Pending release (develop → main)`, with the configured
  integration branch in place of `develop`.
- Release line: `  latest release: <latest | no release yet>, main moved
  <age>; rolling tag <rolling> → <target>`, plus ` (lags <latest>)` when
  `target != latest`, or `; no rolling tag`.
- Nothing pending: `Nothing pending release: <integration> and main are level.`
- Summary: `<n> changes pending release: <d> dependency update(s), <r>
  release back-merge(s), <o> other PR(s).`, with `, <p> PROBLEM: commits with
  no PR` before the period only when `p > 0`. Pluralise each bucket on its
  count (`1 dependency update`, `2 dependency updates`).
- A NO_PR row has `PR` set to `PROBLEM` and `BRANCH` set to `-`, and a note
  after the table: `  PROBLEM: <sha7> reached <integration> without a PR`.
- Error: `ERROR: pending release: <reason>`, printed under the heading with
  nothing else in the section.
- No `vergil.toml`: `  note: no vergil.toml; pending-release section skipped`.
- Titles longer than `_TITLE_WIDTH` are cut to `_TITLE_WIDTH - 1` characters
  plus `…`.

**Wiring:** in `main`, after the worktree-section loop and before the blank
line and `_summary`, call `_print_pending` only when `vergil.toml` exists and
`publish.release` is true. With `publish.release` false or absent, print
nothing. With no `vergil.toml`, print only the note. If `_print_pending`
returned False, `main` returns 1 after printing everything else; otherwise
it returns 0 as today.

- [ ] **Step 1: Write the failing tests** in `test_vrg_repo_status.py`,
  patching `gather_pending_release` and `read_config` in `_MOD`, alongside the
  existing worktree fixtures:
  - `test_pending_section_renders_rows_in_order`: the spec's §5.3 mock-up
    data (back-merge #4251, deps #4258, feat #4252, docs #4255) → the output
    contains the heading, the release line `latest release: v2.1.303, main
    moved 4h ago; rolling tag v2.1 → v2.1.303`, the four rows in that order
    with TYPE `release`, `deps`, `feat` and `docs`, and `4 changes pending
    release: 1 dependency update, 1 release back-merge, 2 other PRs.`
  - `test_release_line_lag`: target `v2.1.302`, latest `v2.1.303` → the line
    ends `rolling tag v2.1 → v2.1.302 (lags v2.1.303)`.
  - `test_release_line_no_tags`: `ReleaseTags(None, None, None)` →
    `latest release: no release yet, main moved …; no rolling tag`.
  - `test_nothing_pending`: no changes → `Nothing pending release: develop
    and main are level.` and no table header.
  - `test_no_pr_change_is_problem`: one NO_PR change → its row's PR cell is
    `PROBLEM`, the `PROBLEM: <sha7> reached develop without a PR` note is
    printed, and the summary ends `, 1 PROBLEM: commits with no PR.`
  - `test_pending_error_exits_nonzero`: `gather_pending_release` raising
    `PendingReleaseError("boom")` → `ERROR: pending release: boom` printed,
    the worktree summary lines still printed, and `main()` returns 1.
  - `test_section_skipped_when_not_published`: `publish.release` False → no
    `Pending release` text, `gather_pending_release` not called, and
    `main()` returns 0.
  - `test_section_skipped_without_config`: no `vergil.toml` → the skip note
    is printed and `gather_pending_release` is not called.
  - `test_long_title_truncated`: a 100-character title → the cell is 72
    characters and ends with `…`.
- [ ] **Step 2: Run the tests.** `vrg-container-run -- vrg-validate`.
  Expected: FAIL in the new tests.
- [ ] **Step 3: Implement** the functions, copy and wiring above.
- [ ] **Step 4: Update the `cli-tools-overview.md` entry** for
  `vrg-repo-status`.
- [ ] **Step 5: Run the tests.** `vrg-container-run -- vrg-validate`.
  Expected: green.
- [ ] **Step 6: Check against live data.** In the vergil-tooling main clone,
  run `vrg-repo-status`. Then check that its pending rows match the PRs
  behind `vrg-git log --first-parent --oneline origin/main..origin/develop`
  (after the human's normal fetch), and that its release line names the
  highest `v2.1.Z` tag. Put the comparison in the PR's notes.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope repo-status
  --message "show what develop holds that main does not, with the release tags"`
