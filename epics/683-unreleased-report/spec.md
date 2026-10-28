# vrg-unreleased: report unreleased work on develop

- **Epic:** vergil-project/.github#683
- **Planning issue:** vergil-project/.github#468 (review history is in its
  comments)
- **Status:** in design review
- **Repos:** vergil-tooling (the command and its site docs)
- **Related, out of scope:** vergil-project/.github#682 (show what a
  dependency-update PR changed)

## 1. Why

Several epics often merge into the same repos in parallel. An epic of epics,
for example, lands PRs from three child epics into `vergil-tooling` and the
plugin within the same week. Once a PR merges to `develop` it is staged for
the next release of that repo, whether or not the epics it came from are
ready to ship together. Before cutting a release, the human (or an agent)
needs to see exactly what has accumulated on `develop` since the last release
and which epic each change belongs to. Today that means reading `git log` and
following each PR to its issue and epic by hand.

## 2. Goals

1. A read-only command, `vrg-unreleased`, that prints a **release-notes
   preview** of everything merged on `develop` since the repo's newest
   release.
2. The pending PRs are **grouped by the epic** their tracked issue belongs to,
   so a glance answers "two PRs from epic A, one from epic B, plus a dependency
   update".
3. It runs against the current repo by default, and against **every managed
   repo in an org** with `--org`.
4. A terse default (one line per PR) and a `--verbose` mode that shows each
   PR's full commit message.
5. Usable by humans and agents alike: plain markdown on stdout, with no
   prompts.

## 3. Non-goals

- **No scheduled job.** Nothing in `ops.yml` or any workflow runs it. It is an
  on-demand tool, because `develop` changes too often for a daily snapshot to
  be useful.
- **No `vrg-release` integration.** `vrg-release` does not call it or gain an
  option for it. The command costs a few API calls per PR, and it has an
  org-wide mode that has no place inside a single-repo release.
- **Not a gate.** It never fails because work is pending, and nothing refuses
  to run because of its output.
- **No dependency diffs.** Dependency-update PRs are listed one line each, like
  any other PR. Showing which dependencies moved is #682.
- **Dropped from the original issue:** the per-author breakdown and the
  flagging of sensitive paths such as `.github/workflows/**`. The epic grouping
  replaced them as the useful view.

## 4. Anti-goals

- **Silently missing a change.** Every commit on `develop` past the release
  boundary must appear in the output, either under a PR or as a direct commit.
  A report that looks complete but isn't is worse than no report.
- **Guessing.** If a lookup fails (API error, permission, rate limit), the
  command fails with the error. It never prints a partial report as if it were
  whole (CLAUDE.md: no silent failures).

## 5. Design

### 5.1 The release boundary

Each release creates three things (vergil-actions
`actions/cd/release/tag-and-release/action.yml`): the annotated tag `vX.Y.Z`
on `main`, the lightweight tag `develop-vX.Y.Z` on the `develop` tip, and a
GitHub Release. The changelog tooling already uses `develop-v*` as its
boundary (`lib/changelog.py`, `configs/cliff.toml`).

- **Boundary:** the highest `develop-vX.Y.Z` tag by semantic version (not by
  date). The pending set is the commits reachable from `develop` and not from
  that tag (`compare/develop-vX.Y.Z...develop`, paginated so that a large gap
  is never truncated).
- **Release date:** the `published_at` of the GitHub Release for `vX.Y.Z`. If
  that release doesn't exist, use the date of the `develop-vX.Y.Z` tag's
  commit and say so in the header.
- **No release yet:** if the repo has no `develop-v*` tag, the report says
  "no release yet" and lists nothing. Listing all of `develop` would be noise.
- **Known limit:** the `develop-v` tag is taken at CD time, after the release
  PR merges, so a PR merged to `develop` while a release was in flight counts
  as released. This matches the changelog's own boundary and is accepted.

### 5.2 Commits to PRs

- Feature PRs are squash-merged (subject ends `(#N)`). Dependency-update PRs
  use merge commits (`Merge pull request #N …`). `linkage.extract_merge_pr`
  already parses both forms.
- Commits brought in by a merge commit (the branch side of a dependency-update
  PR) belong to that PR and are not listed separately.
- A commit whose PR can't be parsed from its subject is looked up with
  `repos/{repo}/commits/{sha}/pulls`. Only if that finds nothing is it listed
  as a **direct commit** (see 5.4).

### 5.3 PRs to issues to epics

- **PR to issue:** the tracking ref in the PR body (`Ref` or `Closes`), parsed
  with `linkage.extract_tracking_ref`, which is the convention the fleet
  writes. `closingIssuesReferences` is not used, because the default `Ref`
  linkage never populates it.
- **Issue to epic:** `epics.parent_of`, the native parent first and then the
  `Parent:` body line.
- **Dependency-update PR:** the `dependency-update` label, or a head branch
  starting with `deps/update-`. It is classified before any issue lookup,
  because these PRs carry no `Ref` line.

### 5.4 Groups and order

| Group | Which PRs | Heading |
| --- | --- | --- |
| Epic | Issue's parent is a finite epic (labelled `epic`, not a standing kind) | `Epic <owner/repo#N>: <epic title>` |
| Ad-hoc | Parent is the repo's ad-hoc epic **or one of its quarterly archives** (closed ad-hoc tasks drain there) | `Ad-hoc` |
| Dependency updates | Recognised as in 5.3 | `Dependency updates` |
| Unlinked | No tracking ref, the issue has no parent, or the parent is any other standing epic (intake, planning, deferred); plus direct commits | `Unlinked` |

- Epic groups come first, ordered by their earliest merge. Ad-hoc,
  Dependency updates and Unlinked follow in that order. Empty groups are
  omitted.
- Within a group, PRs are in merge order.
- Each Unlinked entry gives its reason in a short suffix
  (`— no tracking ref`, `— issue has no epic`, `— direct commit`), because an
  unlinked change is worth a second look in an org that tracks everything.

### 5.5 Output

Markdown on stdout, the same style as `vrg-roadmap` and `vrg-activity-log`.

Single repo, terse (the default):

```text
## vergil-tooling — 6 PRs unreleased since v2.1.278 (2026-10-28)

### Epic vergil-project/.github#573: Several advisories per issue
- #712 feat(security): retarget accepts several --to refs
- #715 fix(queue): held note names every advisory

### Epic vergil-project/.github#607: Package toolchain images
- #709 feat(container): build package toolchain image

### Ad-hoc
- #711 fix(branch-sweep): skip protected branches

### Dependency updates
- #718 chore(deps): dependency update for 2.1.279
```

- The PR line is `#<N> <squash commit subject without the (#N) suffix>`. For
  a merge-commit PR it is the PR title.
- **`--verbose`:** under each PR line, its full commit message body (the
  squash commit's message after the subject; the PR body for a merge-commit
  PR), indented. It is the same data with more of it shown.
- **Nothing pending:** `## vergil-tooling — nothing unreleased since v2.1.278
  (2026-10-28)`.

### 5.6 Org mode

- `--org [ORG]` sweeps every non-archived repo in the org that is managed
  (`epics.is_managed_repo`: it has a `vergil.toml`) and has a `develop`
  branch. `ORG` defaults to the current repo's owner. Repos are listed with
  `github.list_org_repos`, and private repos are included, because the report
  goes only to the caller's terminal.
- The output opens with a **summary table**: one row per repo with pending
  work (repo, PR count, newest release and its date, age of the oldest
  unreleased merge), sorted so that **the repo whose unreleased work has waited
  longest comes first**. Below the table, a line names the repos with nothing
  pending and the repos with no release yet.
- Then each repo with pending work gets its single-repo block (5.5), in the
  same order as the table. `--verbose` applies to every block.

### 5.7 Interface

```text
vrg-unreleased [--repo OWNER/NAME | --org [ORG]] [--verbose]
```

- No target: the current directory's repo (resolved like `vrg-branch-sweep`).
- `--repo` and `--org` are mutually exclusive.
- Exit 0 whenever the report was produced, whatever it contains. Non-zero, with
  the error on stderr, on any failure, including a single failed lookup in org
  mode.
- Read-only: it makes GET and GraphQL query calls only, through the existing
  `lib/github.py` wrappers and the GitHub App identity, like `vrg-roadmap`.

## 6. Constraints

- Follow the command conventions: a thin `bin/vrg_unreleased.py` with
  `@cli_main`, the logic in `lib/unreleased.py` split into gather and render,
  and an entry point in `pyproject.toml`.
- Required docs: a `### vrg-unreleased` section in
  `docs/site/docs/reference/cli-tools-overview.md` and complete `--help` text.
  Both are enforced by existing tests.
- Tests fake GitHub by patching `lib/github.py` and `epics` functions, as the
  existing report tests do. They cover each group, merge-commit PRs, direct
  commits, no release, nothing pending, and a failed lookup that aborts.

## 7. Delivery

Two implementation tasks in vergil-tooling, then the bookends:

1. **Core:** the single-repo command (5.1 to 5.5 and 5.7 without `--org`),
   with tests and the CLI overview entry.
2. **Org mode:** `--org` (5.6), with tests and the overview entry updated.
3. Bookends: documentation review (vergil-project/vergil-tooling#3803), then
   the retrospective (#685).

No validation or deployment task: the behaviour is covered by the pipeline's
tests against faked GitHub, and nothing downstream needs it deployed.

## 8. Left to the builder

- How to batch the PR and issue lookups (GraphQL batching versus per-item
  REST), as long as the result is complete and the failure rule in section 4
  holds.
- The exact wording of headers, Unlinked reasons and the summary table
  columns, within the shapes above.
- The column format for "age" (for example `3d`).
