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
  release, a failed dependency update, or a failed install.
- A release that cannot be finished, because the defect is in the tagged
  commit itself, is superseded by the next release, which carries its issues
  and their records (§4.7).
- The tracking issue holds the state `--resume` needs, and `vrg-release`
  trusts it only while every identity on the issue is the human's (§4.2).
- The tracking issue closes only after every record is posted.
- `vX.Y` means the release its rolling tag names, in every vergil-tooling
  install path, so demoting the tag with `vrg-promote` is honoured (§4.6).
- `issue-deploy`, `epic-implement` and the deployment-task scaffold take a
  human-authored `Released:` record as release evidence (§4.10).

### 2.2 Non-goals

- **A merge-to-`develop` record.** An issue closed by a merged PR already
  records its merge. Recording the rest of the PR workflow on its issues is
  the follow-on brainstorm, vergil-project/.github#1444.
- **Generalizing the install stage.** It stays an optional, Vergil-specific
  stage that is a no-op where a repo configures no install. Other orgs and
  repos deploy through their own tracked operations.
- **`vrg-tooling-install` taking refs other than release tags** (branches,
  commits). That is a separate change, outside this epic.
- **Releasing several versions of one repo at once.** One release at a time
  (§4.7), with supersession as the only way past an open one.
- **Records on issues that a PR only references.** Only GitHub's closing
  references count (§4.1).
- **Protection against a repository admin.** An admin can rewrite anything;
  the identity rule (§4.2) guards against identities that can only comment
  on and edit issues.

### 2.3 Anti-goals

- **No automatic re-run on a fresh run.** A failed publish job, index wait or
  dependency update is not retried by the run that saw it fail. Running
  `--resume` is the human's signal that the cause is fixed.
- **No hand-run recovery step.** No message tells the human to run
  `gh run rerun`, `gh workflow run`, `vrg-promote` or the install commands
  themselves to finish a release.
- **No opt-out that skips part of the release** (§4.8). `--without-deps` is
  the one recorded, explicit exception.
- **No partial record.** A record is never posted for a release that has not
  finished, and never states anything the run did not verify.

## 3. Current state (vergil-tooling at 3da65ccec8)

- **Stages** (`lib/release/orchestrator.py`): `audit`, `preflight`,
  `prepare`, `merge-release`, `confirm-main`, `back-merge-bump`,
  `teardown-worktree`, `confirm-develop`, `package-index`, `promote`,
  `close-finalize`, `update-deps`, `publish-status`, `consumer-refresh`.
- **Resume re-runs every stage.** `lib/release/resume.py` finds the open
  `release: X.Y.Z` issue and checks its checklist against the running stage
  list, and every stage then adopts what an earlier run did. Once
  `close-finalize` has pruned the release branch, `prepare` cannot adopt the
  merged release PR, which is why the deferral comment says not to resume.
- **Deferred publishes live in memory.** `confirm-main` and
  `confirm-develop` add failed CD job names to
  `ctx.deferred_publish_failures`, and `package-index` adds itself on a miss.
  The tracking issue gets a `publish:deferred` comment with hand-run advice.
  Nothing machine-readable records which CD run or job failed.
- **Promote waits on packaging.** `promote` skips while `package-index` is
  deferred and leaves `vrg-promote` to the human.
- **The install waits on everything.** `consumer-refresh` holds on any
  deferred publish and on any failed stage but `update-deps`; with
  `--no-install` it only prints the commands. A failed install tells the
  human to re-run the commands by hand.
- **`update-deps` is optional.** It is the only member of
  `OPTIONAL_STAGES` (`lib/release/handoff.py`): its failure ends the run
  non-zero but holds nothing back, and the tracking issue closes regardless.
- **A failed CD release job** prints numbered hand-run recovery steps
  (`_failed_release_recovery`): re-run the release PR's CI, re-run the CD
  run, then `--resume`.
- **Preflight** refuses a fresh start only when a tracking issue is open for
  the **same** version.
- **The pending-release table** (`lib/pending_release.py`) exists, but only
  `vrg-repo-status` shows it. It maps each first-parent commit of
  `main..develop` to its merged PR by SHA; it does not read closing
  references.
- **`vX.Y` is read as "newest patch".** `lib/tooling_install.resolve()`
  lists `refs/tags/vX.Y*` and installs the highest `vX.Y.Z`, never reading
  the `vX.Y` tag. Its callers are the dev-container cache
  (`tooling_release.resolve_tooling_tag`, for every repo's
  `[dependencies].vergil` pin) and `vrg-vm` (`vm_packages`), and
  `vm_packages.apt_pin` pins a line as apt `version X.Y.*`. A tag demoted
  with `vrg-promote` is therefore not followed. The plugin marketplace and
  GitHub Actions refs do follow the tag.
- **`vrg-tooling-install`** takes only an exact `vX.Y.Z` tag. vergil-tooling's
  `[publish].consumer-refresh` runs `vrg-tooling-install v<VERSION>` then
  `vrg-vm update --all`; vergil-claude-plugin's runs `vrg-host plugins` then
  `vrg-vm update --all`.
- **Opt-outs:** `vrg-release --no-promote` and `--no-install`;
  `vrg-finalize-pr --release --no-install`; `vrg-submit-pr --no-finalize`
  and `--release --no-install`.
- **Tracking issues are human-only in practice.** On the 15 most recent
  vergil-tooling release tracking issues, the issue and all 162 comments are
  the human's, and none carries a label.
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

### 4.2 The tracking issue holds the state, and only the human writes it

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
  issue's ref and closing PR); the release PR, the back-merge PR, the tag,
  and the CD run IDs of `main` and `develop`; each failed CD publish job as
  its run ID and job name; whether the package index has the release;
  whether `--without-deps` was given; the stages done; whether the install
  ran; which records are posted; and, for a superseding release, the release
  it supersedes (§4.7).
- **One writer.** Only `vrg-release` writes it, by a read-modify-write of the
  body at each stage boundary, as the checklist is written today. A
  malformed or missing block on resume, or one naming a version other than
  the issue's, is a refusal that names the block, never a fallback to
  in-memory defaults.
- **The identity rule.** Every time `vrg-release` reads the tracking issue,
  it checks every identity on it: the issue's author, its body's last editor
  and every editor in its edit history, every comment's author and last
  editor, and the actor of every timeline event (label, rename, close,
  reopen and the like). Each must be one of the `human` logins of
  `vrg-trusted-authors`. Any other identity stops the run at once, before
  any write or stage, with an error that names the identity and what it did,
  as a suspected security breach. The human inspects and cleans up the
  issue, then resumes.

### 4.3 Resume runs only what is not done

`--resume` reads the state block (under the identity rule), rebuilds the
context from it, and runs only the stages it does not record as done, from
the first undone stage on. A stage marks itself done in the block when it
succeeds, alongside the facts later stages need (the PRs, the tag, the run
IDs). So a resume after `finalize` never re-enters the branch work of
`prepare` through `teardown-worktree`, whose release branch is gone.

- **Facts are checked, not trusted.** Before a stage relies on a recorded
  fact, it confirms it against GitHub or git: the tag exists on the recorded
  commit, the PRs are merged, a recorded CD run belongs to the release SHA.
  A mismatch is a refusal naming the fact.
- **`ctx.deferred_publish_failures`** becomes a view of the block's
  recorded failures, so a resumed run knows exactly what to re-run.
- **`--without-deps`** given on a resume is recorded in the block and
  skips a failed `update-deps` (§4.5).

### 4.4 Stages, in the new order

```text
audit, preflight, prepare, merge-release, confirm-main, back-merge-bump,
teardown-worktree, confirm-develop, promote, cd-publish, package-index,
finalize, update-deps, install, release-records, close-tracking,
publish-status
```

- **`promote`** runs once the tag and the GitHub Release exist; it no longer
  checks `package-index` (§4.6).
- **`cd-publish`** (new) delivers the CD publish jobs that `confirm-main` or
  `confirm-develop` recorded as failed (§4.5). It is a no-op when none
  failed.
- **`package-index`** waits for the index as today, and on resume also
  dispatches the index workflow (§4.5).
- **`finalize`** is today's `close-finalize` without the close: it runs
  `vrg-finalize-pr --cleanup-only`.
- **`update-deps`** is a required stage like `cd-publish` and
  `package-index` (§4.5).
- **`install`** is today's `consumer-refresh`, renamed, with its
  `vergil.toml` key renamed to match (§4.6). It runs after `finalize` and
  `update-deps`.
- **`release-records`** (new) posts the records (§4.9).
- **`close-tracking`** (new) closes the tracking issue with today's summary.
- **`publish-status`** stays the terminal exit summary: it names what is
  left and the one command that finishes it, `vrg-release --resume` (or,
  for a release that cannot finish, `vrg-release --supersede X.Y.Z`).

**Gating.** `promote` and `install` need only the stages through
`confirm-develop`; `cd-publish`, `package-index` and `update-deps` never
hold them back. `release-records` and `close-tracking` need every stage
before them. A stage that fails leaves the issue open, and the run ends
non-zero. `OPTIONAL_STAGES` is removed: no stage is optional.

**After the install.** In vergil-tooling's own release, the install replaces
the vergil-tooling that `vrg-release` runs from. `vrg-release` imports
everything it needs when it starts, and after the install it starts no
vergil-tooling subprocess, only `gh` and `git`. The remaining risk, an
install that breaks the last two stages, is accepted: it is fixed forward by
hand, posting the records administratively if need be.

### 4.5 Publishing and the dependency update become resumable

On a fresh run, a failed CD publish job, an index miss or a failed
dependency update is recorded in the state block and deferred; the run
carries on through `promote`, `finalize` and `install`, then stops before
`release-records` with the issue open.

On `--resume`:

- **`cd-publish` on `main`** re-runs each recorded failed run with
  `gh run rerun <id> --failed`, watches it with `--exit-status`, and
  re-judges its jobs with the rules `confirm-main` uses. A job that now
  succeeds is cleared from the block; one that fails stays recorded and the
  stage fails.
- **`cd-publish` on `develop`** counts a recorded failed job as delivered
  when a later `develop` CD run's same job has succeeded, found by querying
  GitHub, and re-runs it only when none has, so newer `develop` docs are
  never overwritten by an older run.
- **`package-index`** dispatches the org packages repo's `publish-index`
  workflow, then waits for the index as today. A listed release is recorded.
- **`update-deps`** runs again. Given `--without-deps`, it is skipped and
  the skip recorded; the release then finishes without the update, and the
  next release's existing handling meets the leftover update PR.
- **A failed CD release job** is recovered inside `confirm-main` on resume:
  it re-runs the release PR's CI run and watches it, then re-runs and
  watches the CD run, then judges it. The numbered hand-run steps go.

Each re-run is idempotent: a job recorded as delivered is never re-run, and
a second `--resume` after a partial success re-runs only what is still
recorded as failed.

### 4.6 Promote, install and `vX.Y`

- **`promote`** moves `vX.Y` once the tag and the GitHub Release exist,
  whatever the package index or the CD publish jobs show. While the index is
  deferred, `vX.Y` may name a version whose packages are not yet listed.
  That is accepted, since nothing in the fleet installs from the packages.
  The `vrg-promote` hand-run advice goes.
- **`vX.Y` follows its tag.** `tooling_install.resolve()` resolves a line
  `vX.Y` by reading the tag `refs/tags/vX.Y` itself and peeling it to its
  commit, in one `git ls-remote`, never by picking the newest `vX.Y.Z`.
  The release it names is the `vX.Y.Z` tag on that commit; none is a
  refusal. Every caller (the dev-container cache, `vrg-vm`, and the CLI)
  gets that one meaning, and `vm_packages.apt_pin` pins the exact version of
  that release rather than `X.Y.*`. A tag demoted with `vrg-promote` is then
  followed everywhere.
- **`vrg-tooling-install` takes `vX.Y`** as well as `vX.Y.Z`, through the
  fixed resolver.
- **Installs follow the line.** The install template gains a `<MINOR>`
  placeholder (the `X.Y` of the release), and vergil-tooling's template
  becomes `vrg-tooling-install v<MINOR>`. vergil-claude-plugin's already
  follows `vX.Y`.
- **The rename** (vergil-project/vergil-tooling#4590). The stage
  `consumer-refresh` becomes `install`, and `[publish].consumer-refresh`
  becomes `[publish].install`. The old key is read for one minor version with
  a deprecation warning, then removed. The "no sequence is configured"
  warning goes: a repo with no install is the normal case, shown as
  `none configured` on the stage line.
- **A failed install** records the failure and ends the run non-zero;
  `--resume` re-runs it. A resume whose block records the install as done
  for this version skips it.

### 4.7 One release at a time, and supersession

A fresh start refuses while **any** `release: X.Y.Z` tracking issue is open
in the repo, naming it and the `vrg-release --resume` that finishes it. This
replaces the same-version check. `--resume X.Y.Z` still accepts a version,
and the "multiple in-flight releases" refusal stays for a repo left with
several by older tooling.

A release that `--resume` cannot finish, because the defect is in the tagged
commit, is superseded with **`vrg-release --supersede X.Y.Z`**:

- **Eligible only when** `X.Y.Z` is the open release, its tag and GitHub
  Release exist, and its only stages left are among `cd-publish`,
  `package-index`, `update-deps` and `install`. Otherwise the run refuses
  and names what is open. A bare `vrg-release` never supersedes.
- **It is a fresh release** with every fresh-start check, and it combines
  with a `minor` or `major` bump. It cannot be combined with `--resume`.
- **The new release absorbs** the superseded release's issue list (read from
  its state block under the identity rule) into its own, and records the
  superseded version in its state block and Contents section.
- **The superseded tracking issue** is closed, once the new one exists, with
  a comment naming the release that superseded it. The superseded release
  posts no records.
- **The records** on absorbed issues name the new version, and say that it
  supersedes `X.Y.Z`.

### 4.8 The opt-outs go

Removed: `vrg-release --no-promote` and `--no-install`;
`vrg-finalize-pr --no-install`; `vrg-submit-pr --no-finalize` and
`--no-install`. Each is still parsed, hidden from `--help`, only to fail
with a message naming the removal, as `--install` does today.

- **`vrg-submit-pr` always finishes.** It returns only once each PR has
  merged or cannot merge, which is a classed error. Auto-merge is used
  throughout, and opening a PR without merging it once green serves no
  purpose. `vrg-finalize-pr` stays, for settling `develop` and for a fix to a
  PR that did not merge.
- **Kept:** `--without-deps` (the recorded skip of the dependency update,
  §4.5) and `--skip-audit` (the recovery valve for an over-strict audit).

### 4.9 The `Released:` record

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
- **A superseding release's record** adds a `- Supersedes: vX.Y.Z` line on
  each absorbed issue.
- **Posted on any state.** An issue closed as completed, reopened, or closed
  another way gets the record all the same: the release does contain its PR.
- **Human-authored only.** The posting identity is the human running
  `vrg-release`. A record counts, for idempotency here and as evidence
  anywhere (§4.10), only when its comment's author and last editor are both
  `human` logins of `vrg-trusted-authors`. A record marker on a comment by
  any other identity is a refusal that names the comment: in `vrg-release`,
  the identity rule's stop (§4.2).
- **Idempotent.** Before posting, the stage reads the issue's comments for a
  counting record; one found is recorded as posted in the state block. A run
  interrupted partway posts only the rest.
- **Ordering.** The stage fails if any record could not be posted, naming
  each issue, and `close-tracking` does not run; `--resume` posts the rest.

### 4.10 Agents read the records

Every consumer of a record applies §4.9's human-authored rule: a record
whose author or last editor is not a `human` login is refused and named,
never counted.

- **The deployment-task scaffold** names the release precondition as: each
  blocked-by task carries a human-authored `Released:` record. Human
  attestation stays only for a release outside `vrg-release` (none today).
- **`issue-deploy`** reads each blocked-by task's comments for the record and
  takes it as the release evidence, then cross-checks it with the existing
  `merge-base --is-ancestor` check of the task's merge commit against the
  recorded tag. A record and an ancestry check that disagree are a blocked
  precondition, never a pass.
- **`epic-implement`**, where it walks a task's dependencies for release
  evidence, uses the record the same way.
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
- **The resolver change** moves every `vX.Y` pin from "newest patch" to "the
  tag". The two agree wherever `vX.Y` was promoted normally, so existing pins
  install the same release.
- **Old releases** get no records retroactively.

## 6. Error handling

- Every GitHub read the records depend on (closing references, issue
  comments, the state block, the identities on the tracking issue) fails
  loudly with the call that failed; none is treated as "no issues", "no
  record" or "no other identity".
- A comment that GitHub rejects fails `release-records` for that issue; the
  others are still attempted, and the stage names every failure.
- A `vX.Y` tag that does not exist, or names a commit with no `vX.Y.Z` tag,
  is a resolver error naming the ref, never a fallback to the newest patch.

## 7. Delivery outline

Implementation lands mainly in vergil-tooling, with the skill and docs
changes in vergil-claude-plugin. The plan sequences it; in outline:

1. The tracking-issue state block, the identity rule, and resume from the
   first undone stage.
2. One release at a time, and `--supersede`.
3. The issue list in `preflight` and the Contents section.
4. Resumable publishing and dependency update: `cd-publish`, `package-index`
   dispatch, CD release-job recovery, `update-deps` as a required stage,
   `OPTIONAL_STAGES` removed.
5. Promote and install decoupled from packaging, and the install rename.
6. `vX.Y` follows its tag in `resolve()` and `apt_pin`, and
   `vrg-tooling-install vX.Y`.
7. The opt-outs removed, with `vrg-submit-pr` always finishing.
8. `release-records` and `close-tracking`.
9. The deployment scaffold, `issue-deploy`, `epic-implement` and
   `epic-create`.
10. Docs: the release guide and references, and the `--no-finalize`
    workflows in `submit-pr.md`, `git-workflow.md` and
    vergil-claude-plugin's `repository-standards.md`, with the
    agent-permission-model spec's accepted-gap paragraph narrowed to
    "between finalize runs, or after a finalize run stops watching".
11. A deployment of the new vergil-tooling, a live-release validation, and
    fixture runs for the release-tooling change.

## 8. Acceptance

- A release whose CD publish job fails ends with the issue open and a
  `--resume` instruction only; `--resume` re-runs the job and finishes the
  release with records posted and the issue closed, without re-entering
  `prepare` through `teardown-worktree`.
- A release whose package index misses, or whose dependency update fails,
  finishes the same way; `--resume --without-deps` finishes one whose update
  cannot be fixed.
- `vX.Y` moves and the install runs on a release whose index is deferred.
- A release whose remaining stage cannot succeed is superseded with
  `--supersede X.Y.Z`; its issues get records naming the new version, and
  its tracking issue is closed with a pointer.
- Every issue closed by a `feature/` PR in the release carries exactly one
  human-authored record, after any number of resumes.
- Any non-human identity on a tracking issue stops `vrg-release` before it
  writes anything.
- A fresh `vrg-release` refuses while any tracking issue is open.
- After `vrg-promote` demotes `vX.Y`, the dev-container cache, `vrg-vm` and
  `vrg-tooling-install vX.Y` all install the release the tag names.
- The removed flags fail with their removal message.
- `issue-deploy` takes a deployment task's release precondition from
  human-authored records alone, with the ancestry cross-check passing, and
  refuses a record by any other identity.
