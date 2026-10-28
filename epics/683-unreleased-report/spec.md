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
- **Guessing.** If a lookup fails (a transport error, an auth or permission
  error, a rate limit), the command fails with the error. It never prints a
  partial report as if it were whole (CLAUDE.md: no silent failures).
- **Aborting on bad data.** Bad data in a PR is not a failed lookup. A PR
  body with more than one tracking ref (`extract_tracking_ref` raises), a
  tracking ref to an issue that doesn't exist (404), or a commit subject
  naming a PR that doesn't exist (5.2), is listed under Unlinked with its
  reason, and the run goes on. An untrusted PR or epic author is
  not bad data: it aborts (5.5).

## 5. Design

### 5.1 The release boundary

Each release creates three things (vergil-actions
`actions/cd/release/tag-and-release/action.yml`): the annotated tag `vX.Y.Z`
on `main`, the lightweight tag `develop-vX.Y.Z` on the `develop` tip, and a
GitHub Release. The changelog tooling already uses `develop-v*` as its
boundary (`lib/changelog.py`, `configs/cliff.toml`).

- **Newest release:** the highest `vX.Y.Z` tag by semantic version (not by
  date). Rolling `vX.Y` tags are ignored.
- **Pending set:** the commits reachable from `develop` and not from `vX.Y.Z`
  (`compare/vX.Y.Z...develop`, paginated so that a large gap is never
  truncated). The merge base is the commit the release branch was cut from,
  so a PR merged to `develop` while that release was in flight is correctly
  reported as unreleased. This intentionally differs from the changelog's
  `develop-v*` boundary, which counts such a PR as released (a separate
  defect, vergil-project/vergil-tooling#3805).
- **Release date:** the `published_at` of the GitHub Release for `vX.Y.Z`. If
  that release doesn't exist, use the date of the `vX.Y.Z` tag's commit and
  say so in the header.
- **No release yet:** if the repo has no `vX.Y.Z` tag, the report says
  "no release yet" and lists nothing. Listing all of `develop` would be noise.
- **Release in flight.** The command reports a snapshot and assumes `develop`
  is quiet, which is when it is normally run (just before `vrg-release`). If a
  release is running, the output may look odd, and the report says why rather
  than trying to model it. It checks cheaply for an open release tracking
  issue (`release.is_release_tracking_issue`) or an open PR from a `release/*`
  branch, by a trusted author (5.5). If either exists, the header carries one line: `release in flight
  (<ref>): results may be inconsistent until it finishes`. This is
  informational only and changes nothing else.

### 5.2 Commits to PRs

- Feature PRs are squash-merged (subject ends `(#N)`). Dependency-update PRs
  use merge commits (`Merge pull request #N …`). `linkage.extract_merge_pr`
  already parses both forms.
- Commits brought in by a merge commit (the branch side of a dependency-update
  PR) belong to that PR and are not listed separately. Membership is decided
  from commit parents (the commits reachable from the merge commit's second
  parent and not from its first), never from commit subjects.
- **Release-machinery PRs are excluded.** After the boundary tag is placed,
  `vrg-release` merges its own `release/*` PRs into `develop` (for example
  the `release/post-<version>` bump). A PR whose head branch is a release
  branch (`release.is_release_branch`), and every commit it brings in, is the
  release boundary itself rather than unreleased work. It is never listed and
  never counts toward the PR count, the "nothing unreleased" check or the
  org-mode age. Straight after a release, the report must say "nothing
  unreleased".
- **`--show-release-prs`** lists them anyway, for diagnosis, in a
  `Release machinery` group after every other group. They still don't count
  toward the PR count, the "nothing unreleased" check or the age. The flag is
  separate from `--verbose`, which controls how much of each PR is shown, not
  which PRs are shown.
- A PR number parsed from a subject is accepted only if that PR's
  `merge_commit_sha` is this commit. Otherwise, as for a commit whose subject
  has no PR number, the commit is looked up with
  `repos/{repo}/commits/{sha}/pulls`. Only if that finds nothing is it listed
  as a **direct commit** (see 5.4). A parsed PR number that doesn't exist
  (404) is bad data, not a failed lookup: the commit goes on to the same
  fallback.

### 5.3 PRs to issues to epics

- **PR to issue:** the tracking ref in the PR body (`Ref` or `Closes`), parsed
  with `linkage.extract_tracking_ref`, which is the convention the fleet
  writes. `closingIssuesReferences` is not used, because the default `Ref`
  linkage never populates it.
- **Issue to epic:** `epics.parent_of`, the native parent first and then the
  `Parent:` body line.
- **Dependency-update PR:** a PR with the `dependency-update` label, which
  needs triage rights to apply. It is classified before any issue lookup,
  because these PRs carry no `Ref` line. A PR whose head branch starts with
  `deps/update-` but lacks the label goes to Unlinked
  (`— deps branch without label`).

### 5.4 Groups and order

| Group | Which PRs | Heading |
| --- | --- | --- |
| Epic | Issue's parent is a finite epic (labelled `epic`, not a standing kind) | `Epic <owner/repo#N>: <epic title>` |
| Ad-hoc | Parent is the repo's ad-hoc epic **or one of its quarterly archives** (closed ad-hoc tasks drain there) | `Ad-hoc` |
| Dependency updates | Recognised as in 5.3 | `Dependency updates` |
| Unlinked | **Everything else** (the catch-all), including no tracking ref, an issue with no parent, and any parent that is neither a finite epic nor this repo's ad-hoc epic or archive (other standing epics and their archives, another repo's ad-hoc epic, a plain issue); plus direct commits | `Unlinked` |

- Epic groups come first, ordered by their earliest merge. Ad-hoc,
  Dependency updates and Unlinked follow in that order. Empty groups are
  omitted.
- Within a group, PRs are in merge order.
- Each Unlinked entry gives its reason in a short suffix
  (`— no tracking ref`, `— multiple tracking refs`, `— tracking issue not
  found`, `— issue has no epic`, `— parent <ref> is not an epic`,
  `— deps branch without label`, `— direct commit`), because an unlinked change is worth a second
  look in an org that tracks everything.

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
- **Author trust.** Everything the report displays comes from a trusted
  author, or the run aborts:

  | Author | Rule |
  | --- | --- |
  | PR author (feature and dependency-update PRs; covers the PR title, squash message and merge-PR body) | Must be trusted. Otherwise the command fails, naming the PR and the login. |
  | Author of an epic shown as a group heading | Must be trusted. Otherwise the command fails, naming the epic and the login. |
  | Author of the issue a PR tracks | Not checked: only its parent link is used, and setting that needs write access. |
  | Git author of a commit | Not checked: unverified metadata. PR commits are covered by the PR author, and direct commits are already flagged under Unlinked. |
  | Author of a release tracking issue or `release/*` PR (5.1) | An untrusted one is ignored, and the in-flight line does not fire. |

  PRs come only from trusted authors, so an untrusted PR or epic author is a
  security anomaly, not bad data. It never goes to Unlinked.
- **The trust list is required.** It comes from `load_trusted_authors()`
  (the `trusted_authors` stanza in the identities file). If it is missing,
  the command fails with the library's `TrustedAuthorsNotConfiguredError`.
  There is no default: without the list, nothing is trusted and nothing is
  reported.
- **One shared comparison.** `lib/trusted_authors` has a loader but no
  comparison helper yet. The core task adds one there (for example
  `TrustedAuthors.is_trusted(login)`), so every consumer applies the same
  rule rather than reimplementing it. The rule: strip only a leading `app/`;
  a `<login>[bot]` login is trusted only if `<login>` is in the `agents`
  list, never by matching the `human` list; every match is exact and
  case-sensitive.

### 5.6 Org mode

- `--org [ORG]` sweeps every public, non-archived repo in the org that is
  managed (`epics.is_managed_repo`: it has a `vergil.toml`) and has a
  `develop` branch. `ORG` defaults to the current repo's owner. Repos are
  listed with `github.list_org_repos`.
- **Private repos are excluded by default.** `--include-private` adds them.
  The command reads through the GitHub App identity, so it can see what the
  caller may not be entitled to, and agents relay command output into public
  places (report-ready notes, issue comments, docs PRs).
- The output opens with a **summary table**: one row per repo with pending
  work (repo, PR count, newest release and its date, age of the oldest
  unreleased merge), sorted so that **the repo whose unreleased work has waited
  longest comes first**. Below the table, a line names the repos with nothing
  pending and the repos with no release yet.
- Then each repo with pending work gets its single-repo block (5.5), in the
  same order as the table. `--verbose` applies to every block.

### 5.7 Private content

- Whatever the mode, anything resolved from a private repo or a private
  epic home is marked `(private)` wherever it appears: a repo block, a
  summary table row, an epic heading, and a ref inside an Unlinked reason.
  This includes a public repo's PR whose tracked issue belongs to a private
  epic.
- `--repo` naming a private repo, or running inside a private repo's
  checkout, is allowed without `--include-private`, because the caller named
  that repo explicitly. Its output is still marked.
- `--help` and the CLI overview entry state the rule: output marked
  `(private)` must not be relayed into public artifacts.

### 5.8 Interface

```text
vrg-unreleased [--repo OWNER/NAME | --org [ORG] [--include-private]]
               [--verbose] [--show-release-prs]
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
  commits, an archive parent landing in Unlinked, multiple tracking refs and
  a missing tracking issue landing in Unlinked, a subject `(#N)` whose PR did
  not produce the commit, a deps branch without the label, no release,
  nothing pending, and a failed lookup that aborts. One test uses the real
  post-release shape: only the `release/post-*` merge commit and its bump
  commit past `vX.Y.Z`. The expected result is "nothing unreleased".
- The security controls have named tests:
  - private repos excluded without `--include-private`, and included and
    marked with it;
  - a public repo's PR under a private epic, marked;
  - an untrusted PR author and an untrusted epic author, each aborting;
  - a missing trust list, aborting;
  - an untrusted release tracking issue, ignored by the in-flight check;
  - the `is_trusted()` rules (`app/` prefix, `[bot]` against the `agents`
    list only, exact case-sensitive match).

## 7. Delivery

Two implementation tasks in vergil-tooling, then the bookends:

1. **Core:** the single-repo command (5.1 to 5.5, 5.7, and 5.8 without
   `--org`), with tests and the CLI overview entry.
2. **Org mode:** `--org` and `--include-private` (5.6), with tests and the
   overview entry updated.
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
