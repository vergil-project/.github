# vrg-repo-status: pending release — design

- **Epic:** vergil-project/.github#1049
- **Planning issue:** vergil-project/.github#1047 (review history is in its comments)
- **Status:** approved

## 1. Intent

`vrg-worktree-status` (vergil-tooling) shows the state of a repository's
in-flight work: the main checkout on the integration branch (`develop`), the
development worktrees whose PRs will merge into `develop`, and any release or
dependency machinery worktrees. It says nothing about the next stage of the
pipeline: what has merged to `develop` but has not yet been released to `main`.

This epic adds that stage. At any point, the command should show what the next
`vrg-release` will carry to `main`. That is the list of PRs in `develop` but not
in `main`, which by definition is the state of `main`. It should also show the
current release: the latest release tag, and where the rolling minor tag
points. With it, one command covers the whole path a change takes: in a
worktree, then merged to `develop`, then released to `main`.

The command now reports on the whole repository clone (its worktrees, its
branches and its release tags), not only its worktrees, so it is renamed
`vrg-repo-status`.

## 2. Goals

1. Rename `vrg-worktree-status` to `vrg-repo-status`.
2. Add a **Pending release** section listing every change merged to `develop`
   that `main` does not yet contain.
3. Show the current release in that section: the highest `vX.Y.Z` tag, and
   the `vX.Y.Z` that the highest rolling `vX.Y` tag points to.
4. Show automated PRs as ordinary rows: the release back-merge
   (`release/post-<v>`) and dependency updates (`vrg-update-deps`). Rows are in
   plain merge order, so the back-merge is always the first pending row after a
   release, followed by the post-release dependency update if one ran. Further
   dependency updates during the cycle appear where they merged, each with its
   own row.
5. Keep the view compact and in the style of the existing worktree tables.

## 3. Non-goals

- **What a dependency update changed** (how many packages, which ones). The PR
  body names only the ecosystems today. This is captured separately as
  vergil-project/.github#1048. This epic shows only that a dependency-update PR
  is pending.
- Any change to `vrg-release`, `vrg-update-deps` or the release flow.
- Pending-release reporting for a repo that does not publish (§5.5).
- The `develop-v*` tags. They are being retired; this design ignores them.
- A machine-readable (JSON) output. The command is for humans only.

## 4. Anti-goals

- **A silently incomplete list.** Omitting a change that is in `develop` but
  not `main` is the worst failure here: the view would claim the release is
  smaller than it is. Every first-parent commit in the range is accounted for
  in a row, and a lookup that fails or cannot give a complete answer is an
  error, never a partial list.
- **Hiding automated PRs.** The back-merge and dependency updates are real
  differences between `develop` and `main`. A list that shows one automated PR
  but suppresses the other would itself be a red flag.
- **A slow command.** It runs on every status check. The new section adds a
  small number of GitHub calls, bounded by the size of the pending range (one
  per page of commits, one per batch of commits), never one per pending PR.

## 5. Design

### 5.1 The rename

- The console script becomes `vrg-repo-status`. The module, its tests, and
  every reference that describes the current tool are renamed to match: code
  comments, docstrings, site docs (`docs/site/…`), the worktree convention
  (`docs/specs/worktree-convention.md`), the permission-model tool list and
  test fixtures that name the tool.
- **No alias.** `vrg-worktree-status` stops existing. This is safe because the
  tool is only run by hand; no script or skill invokes it (a fleet-wide search
  found references only in vergil-tooling).
- Historical records keep the old name: release notes, the changelog, dated
  specs and plans under `docs/specs/` and `docs/plans/`, the dated evidence
  under vergil-tooling's `epics/`, and earlier epics' documents in `.github`.
  They describe what was true at the time.
- The output's first section (main checkout) and the existing worktree
  sections are unchanged.

### 5.2 What "pending release" means

The pending set is the first-parent history of `develop` that `main` lacks:
`main..develop` with `--first-parent`, evaluated at the current `main` and
`develop` on the remote (§5.4), not at possibly stale local refs.

Each first-parent commit is mapped to the PR that landed it **by commit SHA**,
through GitHub's associated-pull-request lookup (§5.4), restricted to merged
PRs whose base is `develop`. This one mechanism covers every way a PR lands:

- a squash merge (one commit, subject ending `(#<PR>)`);
- a merge commit (`Merge pull request #N from <owner>/<branch>`), the way
  dependency updates and release back-merges land today;
- a rebase merge, including a batch PR rebase-merged as one commit per issue,
  whose subjects carry issue numbers, not the PR number.

All the commits of one PR form **one row**.

`develop` is protected, so every change reaches it through a PR, the
automated back-merge and dependency updates included. A commit with no
associated PR means protection was bypassed, so it is shown as an exception:
a row of its own with `PROBLEM` in the `PR` column and its commit subject as
the title, plus a `PROBLEM:` note naming the commit's short SHA and saying it
reached `develop` without a PR. Nothing in the range is dropped.

### 5.3 The section

The section follows the worktree sections, after "Other / unknown" and before
the summary lines. It is always shown in a publishing repo (§5.5), including
when nothing is pending.

```text
Pending release (develop → main)
  latest release: v2.1.303, main moved 4h ago; rolling tag v2.1 → v2.1.303
PR     BRANCH                         TYPE     MERGED  TITLE
#4251  release/post-2.1.303           release  4h ago  chore(release): back-merge 2.1.303 and bump to 2.1.304
#4258  deps/update-2.1.304            deps     4h ago  chore(deps): dependency update for 2.1.304
#4252  feature/4249-refuse-ubuntu     feat     1h ago  feat(validate): refuse ubuntu-latest runner labels
#4255  feature/4253-batch-docs        docs     0h ago  docs(batch): document batched same-repo PRs…
4 changes pending release: 1 dependency update, 1 release back-merge, 2 other PRs.
```

(The PR numbers, branch names and timings above are illustrative.)

- **Release line.** It shows:
  - the **latest release**: the highest `vX.Y.Z` tag (exactly three numeric
    parts) on the remote, compared by version, and how long ago `main` last
    moved. A `vX.Y.Z` tag is only ever created by the release flow, so the
    highest one is the latest release; whether its commit is an ancestor of
    `main` is not checked (some repos, such as vergil-actions, tag a freeze
    commit made on top of `main`). If no `vX.Y.Z` tag exists, the line says
    `no release yet`;
  - the **rolling tag**: the highest `vX.Y` tag (exactly two numeric parts),
    compared by version, and the `vX.Y.Z` tag on the same commit. Normally that
    is the latest release. When it is not (for example, `v2.1` moved back to
    `v2.1.302` after `v2.1.303` broke something), the line says so plainly:
    `rolling tag v2.1 → v2.1.302 (lags v2.1.303)`. A lag is information, not a
    problem: it can be deliberate. If the rolling tag's commit carries no
    `vX.Y.Z` tag, the line shows its short SHA. If no `vX.Y` tag exists, it
    says `no rolling tag`.
  - Rolling major tags (`vX`) and `develop-v*` tags are ignored.
- **Rows** come in plain merge order, oldest first, with no special placement
  for any kind of PR.
- **Columns:**
  - `PR`: `#N`, or `PROBLEM` for a commit with no PR (§5.2).
  - `BRANCH`: the PR's head branch, or `-` for a commit with no PR.
  - `TYPE`: the conventional-commit type from the PR title (`feat`, `fix`,
    `docs`, `chore`, …), with two overrides. A dependency update (a
    `deps/update-*` head branch or a `dependency-update` label) shows `deps`,
    and a release back-merge (a `release/post-*` head branch) shows `release`.
    A title with no conventional type shows `-`, and so does a commit with no
    PR: a type is never inferred from a commit subject.
  - `MERGED`: relative age in the existing `_format_age` style: the PR's merge
    time, or the commit date for a commit with no PR.
  - `TITLE`: the PR title, or the commit subject when there is no PR,
    truncated to keep each row on one line.
- **Nothing pending** prints the section heading, the release line and
  `Nothing pending release: develop and main are level.` in place of the
  table.
- **Summary line.** The section ends with its own count, of rows as
  *changes*, broken down as in the mock-up above: dependency updates, release
  back-merges and other PRs, each bucket always listed. When any commit has no
  PR, the line adds `, <n> PROBLEM: commits with no PR`; otherwise it says
  nothing about them. The existing final summary line, which counts
  worktrees, is unchanged.

### 5.4 Data sources and failure handling

The command stays read-only (§6): it never fetches and never writes a ref.
Everything comes from the remote, in a number of calls bounded as §4 says,
through the existing `lib/git.py` and `lib/github.py` helpers:

1. **Current refs.** The current SHAs of `main` and `develop`, and the
   remote's `vX.Y.Z` and `vX.Y` tags with the commits they point to, are read
   without fetching (`ls-remote`, as the command already does for its
   branch-gone check, or the GitHub API).
2. **Pending commits.** The first-parent commits of `main..develop` at those
   SHAs, with their subjects and commit dates, come from the GitHub compare
   API, or from a local `git log` when both SHAs are already present locally.
   The answer must be complete: page through the API as needed.
3. **PR metadata.** Batched GraphQL queries, each covering a fixed number of
   the commit SHAs from step 2, return each commit's associated merged PRs
   into `develop`, with number, title, head branch, labels and merge time.

Failure rules (the no-silent-failures principle):

- Retrying transient failures is the helpers' job: every call goes through
  the existing `lib/git.py` and `lib/github.py` wrappers and their retry
  handling, with no new retry logic in the command.
- A lookup that still fails, or that cannot give a complete answer, is an
  **error**, not a degraded view. The command cannot say what is pending, so
  it says so: the section prints its heading and an `ERROR:` line with the
  reason, and no table, release line or summary. There are no fallback rows
  built from partial data, and a PR number is never guessed from a commit
  subject.
- An error's reason never contains credentials: any userinfo in a URL in the
  reason is removed before it is printed.
- The rest of the output (the main checkout and the worktree sections) still
  prints, and the command then **exits non-zero**. An existing worktree's PR
  lookup failure keeps its current handling (`unknown`, with a note).

### 5.5 Repos that don't publish

The section is shown only when the repository's `vergil.toml` sets
`[publish].release = true`, the same flag `vrg-release` uses to decide whether
a repo releases. The command already loads `vergil.toml` for the integration
branch, so the check costs no call.

- With the flag false or absent, the section is skipped and nothing is printed
  for it.
- With no `vergil.toml` at all, the section is skipped with a `note:` saying
  so, beside the existing integration-branch note.

## 6. Constraints

- The command stays read-only: it never changes a branch, never fetches, and
  never writes a ref (remote-tracking refs and `FETCH_HEAD` included). It reads
  remote state with `ls-remote` and the GitHub API, as it already does.
- Python in vergil-tooling, using the existing `lib/github.py` and `lib/git.py`
  helpers and the existing table rendering (`_render_table`, `_format_age`).
- `main` is the release branch name; `vrg-release` hardcodes it, and this
  design does too. The integration branch comes from the existing
  `config.integration_branch` lookup.
- Tests follow the existing `test_vrg_worktree_status.py` patterns, renamed with
  the module.

## 7. Left to the builder

- The exact calls for §5.4 within its constraints: `ls-remote` or the API for
  step 1, compare or local log for step 2.
- The `TITLE` truncation width and the exact wording of the heading and the
  release line.
- Whether the rename and the new section are one PR or two (the plan decides).
