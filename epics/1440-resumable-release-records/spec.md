# resumable-release-records: release records and a resumable release — design

- **Epic:** vergil-project/.github#1440
- **Planning issue:** vergil-project/.github#1437 (review history is in its comments)
- **Status:** draft

## 1. Intent

When an epic has a deployment step, the human cuts the release and installs
it, and the agent then asks the human to post a comment attesting that the
change is released and installed. Writing that comment is friction, and the
facts it states are ones `vrg-release` already holds: the version, the date,
the release, and whether the install ran.

A second problem sits underneath it. A release that hits a failed CD publish
job or a package index that never lists it ends "complete with deferral":
the tag and GitHub Release exist, but the tracking issue stays open with
advice to re-run workflows by hand, then to run `vrg-promote` by hand, and
explicitly **not** to run `vrg-release --resume`. A process that leaves its
last steps to hand-run commands cannot write a reliable record of its own
completion.

**The why:** make `vrg-release` one workflow that is always finished by
`vrg-release --resume`, with no step left to run by hand, and have it write,
as its last act, a `Released:` record on every issue the release contains.
Agents then read that record as release evidence instead of asking the human
for one.

## 2. Goals, non-goals, anti-goals

### 2.1 Goals

- Every issue closed by a `feature/` PR in a release gets one `Released:`
  record from `vrg-release`, posted once the release is fully complete.
- `vrg-release --resume` finishes any release that did not finish: a halted
  stage, a failed CD publish job, a package index that has not listed the
  release, or a failed install.
- The tracking issue holds all the state `--resume` needs; nothing lives
  only in the memory of the run that stopped.
- The tracking issue closes only after every record is posted.
- `issue-deploy` and the deployment-task scaffold take the trusted
  `Released:` records as release evidence.

### 2.2 Non-goals

- **A merge-to-`develop` record.** An issue closed by a merged PR already
  records its merge; the original proposal's `vrg-submit-pr` comment adds
  nothing.
- **Generalizing the install stage.** It stays an optional, Vergil-specific
  stage that is a no-op where a repo configures no install. Other orgs and
  repos deploy through their own tracked operations.
- **Releasing several versions of one repo at once.** One release at a time
  (§4.6).
- **Records on issues that a PR only references.** Only GitHub's closing
  references count (§4.1).

### 2.3 Anti-goals

- **No automatic re-run on a fresh run.** A failed publish job or index wait
  is not retried by the run that saw it fail. Running `--resume` is the
  human's signal that the cause is fixed.
- **No hand-run recovery step.** No message tells the human to run
  `gh run rerun`, `gh workflow run`, `vrg-promote` or the install commands
  themselves to finish a release.
- **No opt-out that skips part of the release.** A release does all of it or
  stays open (§4.7).
- **No partial record.** A record is never posted for a release that has not
  finished, and never states anything the run did not verify.

## 3. Current state (vergil-tooling at 3da65ccec8)

- **Stages** (`lib/release/orchestrator.py`): `audit`, `preflight`,
  `prepare`, `merge-release`, `confirm-main`, `back-merge-bump`,
  `teardown-worktree`, `confirm-develop`, `package-index`, `promote`,
  `close-finalize`, `update-deps`, `publish-status`, `consumer-refresh`.
- **Deferred publishes live in memory.** `confirm-main` and
  `confirm-develop` add failed CD job names to
  `ctx.deferred_publish_failures`, and `package-index` adds itself on a miss.
  The tracking issue gets a `publish:deferred` comment with hand-run advice
  and "do not run `--resume`". Nothing machine-readable records which CD run
  or job failed.
- **Promote waits on packaging.** `promote` skips while `package-index` is
  deferred and leaves `vrg-promote` to the human (#4293).
- **The install waits on everything.** `consumer-refresh` holds on any
  deferred publish and on any failed stage but `update-deps`; with
  `--no-install` it only prints the commands. A failed install tells the
  human to re-run the commands by hand.
- **A failed CD release job** prints numbered hand-run recovery steps
  (`_failed_release_recovery`, #4405): re-run the release PR's CI, re-run the
  CD run, then `--resume`.
- **Resume** (`lib/release/resume.py`) finds the open `release: X.Y.Z`
  issue, checks its checklist against the running stage list, and re-runs
  every stage; each stage adopts what an earlier run did.
- **Preflight** refuses a fresh start only when a tracking issue is open for
  the **same** version.
- **The pending-release table** (`lib/pending_release.py`) exists, but only
  `vrg-repo-status` shows it. It maps each first-parent commit of
  `main..develop` to its merged PR by SHA; it does not read closing
  references.
- **Opt-outs:** `vrg-release --no-promote` and `--no-install`;
  `vrg-finalize-pr --release --no-install`; `vrg-submit-pr --no-finalize`
  and `--release --no-install`.
- **`vrg-tooling-install`** takes only an exact `vX.Y.Z` tag. vergil-tooling's
  `[publish].consumer-refresh` runs `vrg-tooling-install v<VERSION>` then
  `vrg-vm update --all`; vergil-claude-plugin's runs `vrg-host plugins` then
  `vrg-vm update --all`.
- **Deployment evidence.** The deployment scaffold
  (`data/deployment_task_body.md`) asks for the release to be "attested";
  `issue-deploy` checks it with `vrg-git merge-base --is-ancestor`.

## 4. Design

### 4.1 The release's contents and issue list

At a fresh start, `preflight` gathers the pending release, pinned to the
`develop` commit the release branch is cut from (the commit preflight synced
the root checkout to), not to whatever `origin/develop` holds later. It
reuses `pending_release`'s commit-to-PR mapping, extended to read each merged
PR's `closingIssuesReferences` in the same GraphQL query (bounded and
truncation-checked as the existing connections are).

- **The issue list** is the union of the closing references of every PR in
  the range whose head branch starts `feature/`, a batch PR included. Release
  back-merges, dependency updates, automation PRs and commits with no PR
  contribute none. A closing reference to an issue in another repo is
  ignored: the placement law means it should not exist, and the record is
  posted only in the released repo.
- **Shown up front.** `preflight` prints the pending-release table and, under
  it, the issue list with each issue's PR, before any branch or issue is
  created.
- **An incomplete lookup refuses the release**, as `pending_release` already
  refuses a partial list: there is never a release whose issue list is
  missing an issue.
- **An empty list is valid.** A release whose PRs close no issues posts no
  records.

### 4.2 The tracking issue holds the state

`prepare` creates the tracking issue with, in its creation body, the stage
checklist (as today), a human-readable **Contents** section (the
pending-release table and the issue list), and a **state block**:

````text
<!-- vrg-release:state:begin -->
```json
{ "version": "2.1.340", "cut": "<sha>", "issues": [...], ... }
```
<!-- vrg-release:state:end -->
````

- **What it holds:** the version and the cut SHA; the issue list (each
  issue's ref and closing PR); each failed CD publish job as its run ID and
  job name, for `main` and for `develop`; whether the package index has the
  release; whether the install ran; and which records are posted.
- **One writer.** Only `vrg-release` writes it, by a read-modify-write of the
  body at each stage boundary, as the checklist is written today. A
  malformed or missing block on resume is a refusal that names the block, not
  a fallback to in-memory defaults.
- **Resume reads it.** `--resume` rebuilds the context from the state block,
  not from the stage that stopped. `ctx.deferred_publish_failures` becomes a
  view of the block's recorded failures, so a resumed run knows exactly what
  to re-run.
- **Who can change it.** Only users with write access to the repo can edit an
  issue body; that is the same trust as the repo itself.

### 4.3 Stages, in the new order

```text
audit, preflight, prepare, merge-release, confirm-main, back-merge-bump,
teardown-worktree, confirm-develop, promote, cd-publish, package-index,
finalize, update-deps, install, release-records, close-tracking,
publish-status
```

- **`promote`** runs once the tag and the GitHub Release exist; it no longer
  checks `package-index` (§4.5).
- **`cd-publish`** (new) delivers the CD publish jobs that `confirm-main` or
  `confirm-develop` recorded as failed (§4.4). It is a no-op when none failed.
- **`package-index`** waits for the index as today, and on resume also
  dispatches the index workflow (§4.4).
- **`finalize`** is today's `close-finalize` without the close: it runs
  `vrg-finalize-pr --cleanup-only`.
- **`update-deps`** is unchanged and stays optional: its failure is reported
  and ends the run non-zero, but holds back neither the install, the records
  nor the close.
- **`install`** is today's `consumer-refresh`, renamed, with its `vergil.toml`
  key renamed to match (§4.5). It runs after `finalize` and `update-deps`, so
  the rest of the run uses one installed vergil-tooling throughout.
- **`release-records`** (new) posts the records (§4.8).
- **`close-tracking`** (new) closes the tracking issue with today's summary.
- **`publish-status`** stays the terminal exit summary: it names what is
  left and the one command that finishes it, `vrg-release --resume`.

Every stage from `confirm-main` on gates the records on success. A stage
that fails leaves the issue open, and the run ends non-zero with the
`--resume` instruction.

### 4.4 Publishing becomes resumable

On a fresh run, a failed CD publish job or an index miss is recorded in the
state block and deferred, as today; the run carries on through `promote`,
`finalize`, `update-deps` and `install`, then stops before
`release-records` with the issue open.

On `--resume`:

- **`cd-publish`** re-runs each recorded failed CD run with
  `gh run rerun <id> --failed`, watches it with `--exit-status`, and re-judges
  its jobs with the same rules `confirm-main` and `confirm-develop` use. A
  job that now succeeds is cleared from the state block; one that fails stays
  recorded and the stage fails.
- **`package-index`** dispatches the org packages repo's `publish-index`
  workflow, then waits for the index as today. A listed release is recorded.
- **A failed CD release job** (#4405) is recovered the same way, inside
  `confirm-main` on resume: it re-runs the release PR's CI run and watches
  it, then re-runs and watches the CD run, then judges it. The numbered
  hand-run steps go.

Each re-run is idempotent: a job already recorded as delivered is never
re-run, and a second `--resume` after a partial success re-runs only what
is still recorded as failed.

### 4.5 Promote and install no longer wait on packaging

- **`promote`** moves `vX.Y` once the tag and the GitHub Release exist,
  whatever the package index or the CD publish jobs show. While the index is
  deferred, `vX.Y` may name a version whose packages are not yet listed.
  That is accepted, since nothing in the fleet installs from the packages.
  The `vrg-promote` hand-run advice goes.
- **`install`** runs once `promote` has succeeded, whatever the CD publish
  jobs or the index show. It still never runs after a failed stage other
  than a deferred publish or `update-deps`.
- **Installs follow the rolling tag.** The install template gains a
  `<MINOR>` placeholder (the `X.Y` of the release), and vergil-tooling's
  template becomes `vrg-tooling-install v<MINOR>`. vergil-claude-plugin's
  already follows `vX.Y`.
- **`vrg-tooling-install` takes any ref.** Instead of only an exact
  `vX.Y.Z`, it takes any git ref of the vergil-tooling repo: a release tag, a
  rolling tag, a branch such as `develop` (to work around a tooling bug), or
  a commit SHA. It resolves the ref against the fixed vergil-tooling remote
  and installs the commit it names, locked to that commit's `uv.lock` as
  today, and prints the resolved commit. A ref that does not resolve (for
  example `v3.2` before any 3.2 release) is a fatal error, from the
  resolution itself, not from a pattern check up front. The argument is only
  a ref name, never a URL or another repo. It is checked as a valid git ref
  name, and anything starting with `-` is refused, so it can never be read
  as an option.
- **The rename** (vergil-tooling#4590). `[publish].consumer-refresh` becomes
  `[publish].install`. The old key is read for one minor version with a
  deprecation warning, then removed. The "no sequence is configured" warning
  goes: a repo with no install is the normal case, shown as `none configured`
  on the stage line.
- **A failed install** records the failure and ends the run non-zero with
  the `--resume` instruction; `--resume` re-runs the install. On a resume
  whose state block records the install as done for this version, the stage
  is skipped.

### 4.6 One release at a time

A fresh start refuses while **any** `release: X.Y.Z` tracking issue is open
in the repo, naming it and the `vrg-release --resume` that finishes it. This
replaces the same-version check. `--resume X.Y.Z` still accepts a version,
and the "multiple in-flight releases" refusal stays for a repo left with
several by older tooling.

### 4.7 The opt-outs go

Removed: `vrg-release --no-promote` and `--no-install`;
`vrg-finalize-pr --no-install`; `vrg-submit-pr --no-finalize` and
`--no-install`. Each is still parsed, hidden from `--help`, only to fail
with a message naming the removal, as `--install` does today (#1184).

Kept: `--without-deps` (the dependency update is optional) and
`--skip-audit` (the recovery valve for an over-strict audit).

### 4.8 The `Released:` record

`release-records` posts one comment on each issue in the state block's
issue list that has no record yet:

```text
<!-- vrg-release:released:<owner>/<repo>:v2.1.340 -->
Released: vergil-project/vergil-tooling v2.1.340 on 2026-11-20

- Release: https://github.com/vergil-project/vergil-tooling/releases/tag/v2.1.340
- Closed by: #4840 (merge commit 1a2b3c4)
- Install: ran (`[publish].install`)
```

- **The install line** reads `ran` where the repo configures an install and
  `none configured` where it does not. In a repo with no install, the record
  attests release and publish only; deploying is a separate tracked
  operation.
- **Posted on any state.** An issue closed as completed, reopened, or closed
  another way gets the record all the same: the release does contain its PR.
- **Idempotent.** Before posting, the stage reads the issue's comments for
  the marker by a trusted author (`vrg-trusted-authors`); one found is
  counted as posted and recorded in the state block. A run interrupted
  partway posts only the rest.
- **The posting identity** is the human running `vrg-release`, which is a
  trusted author.
- **Ordering.** The stage fails if any record could not be posted, naming
  each issue, and `close-tracking` does not run; `--resume` posts the rest.

### 4.9 Agents read the records

- **The deployment-task scaffold** names the release precondition as: each
  blocked-by task carries a trusted `Released:` record for a version at least
  the one the deployment needs. Human attestation stays only for a release
  outside `vrg-release` (none today).
- **`issue-deploy`** reads each blocked-by task's comments for the trusted
  record and takes it as the release evidence; it then cross-checks it with
  the existing `merge-base --is-ancestor` check of the task's merge commit
  against the recorded tag. A record and an ancestry check that disagree are
  a blocked precondition, never a pass.
- **`epic-create`'s** "Checking that a release carries the changes"
  paragraph names the record first and the ancestry check as its cross-check.

## 5. Migration

- **No release in flight at the switch.** The stage list changes, so the
  checklist check refuses to resume a release started by the older tooling,
  and its tracking issue has no state block. The deployment of the new
  vergil-tooling first finishes every open release tracking issue with the
  older tooling.
- **`vergil.toml` keys.** vergil-tooling and vergil-claude-plugin move to
  `[publish].install` in the same release window; the deprecated key covers
  any repo that lags.
- **Old releases** get no records retroactively.

## 6. Error handling

- Every GitHub read the records depend on (closing references, issue
  comments, the state block) fails loudly with the call that failed; none is
  treated as "no issues" or "no record".
- A state block that does not parse, or names a version other than the
  tracking issue's, refuses the resume.
- A comment that GitHub rejects fails `release-records` for that issue; the
  others are still attempted, and the stage names every failure.

## 7. Delivery outline

Implementation lands mainly in vergil-tooling, with the skill and docs
changes in vergil-claude-plugin. The plan sequences it; in outline:

1. The tracking-issue state block and resume from it.
2. One release at a time.
3. The issue list in `preflight` and the Contents section.
4. Resumable publishing: `cd-publish`, `package-index` dispatch, CD release
   job recovery.
5. Promote and install decoupled from packaging; `vrg-tooling-install <ref>`;
   the install rename.
6. The opt-outs removed.
7. `release-records` and `close-tracking`.
8. The deployment scaffold, `issue-deploy` and `epic-create`.
9. A deployment of the new vergil-tooling, a live-release validation, and
   fixture runs for the release-tooling change.

## 8. Acceptance

- A release whose CD publish job fails ends with the issue open and a
  `--resume` instruction only; `--resume` re-runs the job and finishes the
  release with records posted and the issue closed.
- A release whose package index misses finishes the same way.
- `vX.Y` moves and the install runs on a release whose index is deferred.
- Every issue closed by a `feature/` PR in the release carries exactly one
  record, after any number of resumes.
- A fresh `vrg-release` refuses while any tracking issue is open.
- The removed flags fail with their removal message.
- `issue-deploy` takes a deployment task's release precondition from the
  records alone, with the ancestry cross-check passing.
