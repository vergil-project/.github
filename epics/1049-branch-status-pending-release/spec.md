# vrg-branch-status: pending release — design

- **Epic:** vergil-project/.github#1049
- **Planning issue:** vergil-project/.github#1047 (review history is in its comments)
- **Status:** draft

## 1. Intent

`vrg-worktree-status` (vergil-tooling) shows the state of a repository's
in-flight work: the main checkout on the integration branch (`develop`), the
development worktrees whose PRs will merge into `develop`, and any release or
dependency machinery worktrees. It says nothing about the next stage of the
pipeline: what has merged to `develop` but has not yet been released to `main`.

This epic adds that stage. At any point, the command should show what the next
`vrg-release` will carry to `main`. That is the list of PRs in `develop` but not
in `main`, which by definition is the state of `main`. With it, one command
covers the whole path a change takes: in a worktree, then merged to `develop`,
then released to `main`.

Because the command now reports on the repository's branches, not only its
worktrees, it is renamed `vrg-branch-status`.

## 2. Goals

1. Rename `vrg-worktree-status` to `vrg-branch-status`.
2. Add a **Pending release** section listing every PR merged to `develop` that
   `main` does not yet contain.
3. Show automated PRs as ordinary rows: dependency updates
   (`vrg-update-deps`) and the release back-merge (`release/post-<v>`). A
   dependency update normally comes first, because `vrg-release` runs one
   before it cuts the release, but there can be several in a cycle; each one
   gets its own row.
4. Keep the view compact and in the style of the existing worktree tables.

## 3. Non-goals

- **What a dependency update changed** (how many packages, which ones). The PR
  body names only the ecosystems today. This is captured separately as
  vergil-project/.github#1048. This epic shows only that a dependency-update PR
  is pending.
- Any change to `vrg-release`, `vrg-update-deps` or the release flow.
- Pending-release reporting for a repo that does not publish to `main` (§5.5).
- A machine-readable (JSON) output. The command is for humans only.

## 4. Anti-goals

- **A silently incomplete list.** Omitting a PR that is in `develop` but not
  `main` is the worst failure here: the view would claim the release is smaller
  than it is. Every first-parent commit in the range appears as a row, whether
  or not it maps to a PR, and every failed or truncated lookup is reported.
- **Hiding automated PRs.** The back-merge and dependency updates are real
  differences between `develop` and `main`. A list that shows one automated PR
  but suppresses the other would itself be a red flag.
- **A slow command.** It runs on every status check. The new section adds a
  constant number of GitHub calls, never one per pending PR.

## 5. Design

### 5.1 The rename

- The console script becomes `vrg-branch-status`. The module, its tests, and
  every reference that describes the current tool are renamed to match: code
  comments, docstrings, site docs (`docs/site/…`) and the worktree convention
  (`docs/specs/worktree-convention.md`).
- **No alias.** `vrg-worktree-status` stops existing. This is safe because the
  tool is only run by hand; no script or skill invokes it (a fleet-wide search
  found references only in vergil-tooling).
- Historical records keep the old name: release notes, the changelog, dated
  specs and plans under `docs/specs/` and `docs/plans/`, and earlier epics'
  documents in `.github`. They describe what was true at the time.
- The output's first section (main checkout) and the existing worktree
  sections are unchanged.

### 5.2 What "pending release" means

The pending set is the first-parent history of `develop` that `main` lacks:
`main..develop` with `--first-parent`, evaluated against GitHub's current
`main` and `develop`, not a possibly stale local fetch.

Each first-parent commit is one change that landed on `develop`:

- a squash or rebase merge, whose subject ends `(#N)`;
- a merge commit, whose subject is `Merge pull request #N from <owner>/<branch>`
  (dependency updates and release back-merges land this way today);
- or, rarely, a direct commit with no PR.

Each one becomes a row. A commit with no recognizable PR number still gets a
row (PR `-`, its commit subject as the title), so nothing in the range is
dropped.

### 5.3 The section

The section follows the worktree sections, after "Other / unknown" and before
the summary lines. It is always shown in a publishing repo (§5.5), including
when nothing is pending.

```text
Pending release (develop → main, since v2.1.303, released 4h ago)
PR     BRANCH                         TYPE     MERGED  TITLE
#4247  deps/update-2.1.303            deps     5h ago  chore(deps): dependency update for 2.1.303
#4251  release/post-2.1.303           release  4h ago  chore(release): back-merge 2.1.303 and bump to 2.1.304
#4252  feature/4249-refuse-ubuntu     feat     1h ago  feat(validate): refuse ubuntu-latest runner labels
#4255  feature/4253-batch-docs        docs     0h ago  docs(batch): document batched same-repo PRs…
4 PRs pending release: 1 dependency update, 1 release back-merge, 2 other.
```

(The branch names and timings above are illustrative.)

- **Header.** It names the newest release tag reachable from `main` and how
  long ago `main` last moved. If no tag is reachable, it names `main`'s tip
  commit instead.
- **Rows** come in merge order, oldest first, so a release-time dependency
  update sits at the top.
- **Columns:**
  - `PR`: `#N`, or `-` for a commit with no PR.
  - `BRANCH`: the PR's head branch, or `-`.
  - `TYPE`: the conventional-commit type from the PR title (`feat`, `fix`,
    `docs`, `chore`, …), with two overrides. A dependency update (a
    `deps/update-*` head branch or a `dependency-update` label) shows `deps`,
    and a release back-merge (a `release/post-*` head branch) shows `release`.
    A title with no conventional type shows `-`.
  - `MERGED`: relative age in the existing `_format_age` style.
  - `TITLE`: the PR title, or the commit subject when there is no PR,
    truncated to keep each row on one line.
- **Nothing pending** prints the header and `Nothing pending release: develop
  and main are level.` in place of the table.
- **Summary line.** The section ends with its own count, as in the mock-up
  above. The existing final summary line, which counts worktrees, is unchanged.

### 5.4 Data sources and failure handling

The pending set and the PR metadata come from GitHub, through the existing
`lib/github.py` helpers, in a constant number of calls:

1. A compare of `main...develop`, filtered to `develop`'s first-parent chain,
   gives the commits, their subjects and their commit dates. Alternatively,
   fetching `origin/main` and `origin/develop` then running a local `git log`
   is acceptable, provided the result reflects GitHub's current refs.
2. One batched GraphQL query for the PR numbers found in step 1 gives each PR's
   title, head branch, labels and merge time.

Failure rules (the no-silent-failures principle):

- A failed lookup, or one the API truncates (for example a compare capped at
  its commit limit), prints a `note:` line with the reason. The section never
  appears complete when it is not.
- If step 1 fails, the section prints its header and the note, with no table.
- If step 2 fails, the rows still print from the commit data alone (PR number,
  commit subject, commit date), with a note that PR details are unavailable.
- A PR number with no matching PR in the response prints its row from the
  commit data, with a note naming the number.
- The existing worktree sections are unaffected by any failure here. The
  command still exits 0, as it does today when a worktree's PR lookup fails.

### 5.5 Repos that don't publish

If the repository has no `main` branch on its remote (for example `.github`,
which has only `develop`), the section is skipped entirely and nothing is
printed for it. A failure to *determine* whether `main` exists is a failed
lookup (§5.4), not a skip.

## 6. Constraints

- The command stays read-only: it observes, it never changes branches.
- Python in vergil-tooling, using the existing `lib/github.py` and `lib/git.py`
  helpers and the existing table rendering (`_render_table`, `_format_age`).
- `main` is the release branch name; `vrg-release` hardcodes it, and this
  design does too. The integration branch comes from the existing
  `config.integration_branch` lookup.
- Tests follow the existing `test_vrg_worktree_status.py` patterns, renamed with
  the module.

## 7. Left to the builder

- The exact GitHub calls for §5.4 (compare plus GraphQL, or fetch plus local
  log), within the constant-call and no-stale-data constraints.
- The `TITLE` truncation width and the header's exact wording.
- Whether the rename and the new section are one PR or two (the plan decides).
