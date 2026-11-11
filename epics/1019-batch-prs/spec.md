# Batched same-repo PRs — design

- **Epic:** vergil-project/.github#1019
- **Planning issue:** vergil-project/.github#1017 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

One agent should be able to implement a small batch of issues from one
repository on one branch and hand them off as one PR. The batch then pays
GitHub's per-PR gate cost once, not once per issue.

Each issue stays a standalone unit. It is implemented, validated and
merge-reviewed on its own, and lands on `develop` as exactly one commit with
its own changelog entry. The PR is what gets batched; the issues are not.

## 2. Why

### 2.1 The cost is per PR, not per change

Every task lands through its own PR today, and each PR pays the same fixed
gates however small its change is: PR CI, the merge, and the post-merge CD
run on `develop`. The existing serial batch in `vrg-submit-pr` (`--select`,
`--all`) already defers the CD *check* and the release to the end of the
batch, but each item is still rebased onto the new `develop`, gets its own
PR CI, and each merge pushes `develop` and starts its own CD run.

**Data** (GitHub Actions, the last 60 workflow runs per repo, pulled
2026-11-10 with `vrg-gh run list --limit 60`):

| Repo | PR CI median (max) | CD on `develop` push median (max) |
| --- | --- | --- |
| vergil-containers | 4 min (18) | 6 min (16) |
| vergil-tooling | 4 min (4) | 2 min (2) |

**Judgment:** a PR's CI and CD cost is roughly independent of how many small
changes it carries, so a batch of N pays close to one PR's cost instead of
N. The long tails in vergil-containers are where a batch pays most: N items
take one chance on an 18-minute run instead of N.

### 2.2 Small work arrives in bursts

**Data** (the last 150 merged PRs per repo, pulled 2026-11-10 with
`vrg-gh pr list --state merged --limit 150`; "small" is ≤150 changed lines
and ≤6 files, "tiny" is ≤30 lines and ≤3 files, both counting tests, and
excluding PRs that touch `.github/workflows/`):

| Repo (window) | Tiny or small PRs | Runs of ≥2 small PRs merged ≤6 h apart |
| --- | --- | --- |
| vergil-tooling (Nov 2–10) | 64 of 150 (43%) | 7 runs: 2, 3, 2, 18, 15, 8, 15 |
| vergil-containers (last 60) | 25 of 60 (42%) | not measured |
| .github (Jul–Nov) | 18 of 150; 123 are epic spec/plan PRs | 3 runs of 2 |
| docs (Jul–Nov) | 36 of 48 (75%) | 5 runs, mostly pairs |

**Judgment:** the main use is bursts of small vergil-tooling fixes. The
largest saving per PR is in vergil-containers.

## 3. Goals, non-goals, anti-goals

**Goals**

- `issue-implement` accepts a list of issues from one repository and delivers
  them as one PR that closes all of them.
- `epic-implement` can choose to batch small ready tasks, under written
  guidelines.
- `develop` gets one commit per issue, and the changelog one entry per issue,
  each under its own conventional-commit type.
- When a batch item proves bigger than expected, the human decides what
  happens to it.

**Non-goals**

- Combining branches that were already implemented and handed off separately.
  The batch is formed before implementation starts.
- Merging several issues into one issue. An issue has relationships (its
  epic, Blocked-by links, references from docs and other issues) that merging
  would break.
- Renaming merge review (a possible follow-on: "pre-submission review" names
  what it does).
- Timing instrumentation for the submit and finalize phases.
- Batching across repositories. A PR closes only issues in its own repo, and
  the CI being saved is per repo.

**Anti-goals** (this fails even if it technically works)

- An agent decides by itself to drop an item from a batch.
- A batch PR lands as one squashed commit, collapsing N changelog entries
  into one.
- A batch silently falls back to a squash merge, or opens with a commit count
  that does not match its issues.
- Single-issue PRs pay any new cost or follow any new rule.

## 4. Starting point (verified 2026-11-10, vergil-tooling `b970d09`)

- **One issue throughout the handoff.** The workflow state
  (`lib/pr_workflow/state.py`, `SCHEMA_VERSION = 2`) has a single `issue`
  field. `report-ready` takes one `--issue`. The relay record is that state,
  pushed at `refs/vergil/pr-workflow/<branch>`
  (`lib/pr_workflow/github_transport.py`). `build_pr_body` writes one
  `## Issue Linkage` line (`lib/pr_body.py`). The same-repo, task-only,
  operational and queue refusals in `report-ready` and `vrg-submit-pr`, and
  the `Closes` upgrade (`_task_linkage`), each take one ref.
- **GitHub closes the issues; finalize only backstops it.** A PR body's
  `Closes` lines make GitHub auto-close those issues on merge, and that works
  for any number of them. GitHub occasionally skips an auto-close (#4002:
  #3921 stayed open after PR #3928 merged). So `vrg-finalize-pr` waits
  briefly, re-reads the merged PR's closing references, and closes only those
  GitHub left open, with a comment saying so. Neither step assumes a single
  issue.
- **The daily epic sweep does assume a single issue.** `vrg-epic-audit
  --close` (run by `epic-sweep.yml`) is the backstop for a task its merged PR
  did not close. `task_drift` (`lib/epic_audit.py`) finds each PR's task
  with `linkage.extract_tracking_ref`, which raises `ValueError` on more
  than one linkage line, and it skips that PR silently
  (`except ValueError: continue`).
- **Feature PRs squash.** `_resolve_strategy` in `bin/vrg_finalize_pr.py`
  merges `release/*` branches with a merge commit and squashes everything
  else, unless `--strategy` overrides it. The org's branch ruleset allows
  merge, squash and rebase (`lib/github_config.py`, `allowed_merge_methods`),
  and has no signed-commit rule.
- **Cleanup is PR-state based.** Finalize's branch and worktree cleanup uses
  the merged PR's branch and the worktree's PR state, not ancestry, so it
  does not depend on the merge strategy.
- **The changelog comes from commit subjects.** git-cliff
  (`configs/cliff.toml`) writes the first line of each commit on `develop`,
  grouped by conventional-commit type. Today each line is a squashed PR title
  ending in `(#PR)`.
- **Merge review is per issue.** `vrg-merge-review-request` takes `--issue`,
  `--base` (any ref), `--step` and `--fix-base` (the HEAD the previous step
  judged). The skill reads that issue's comments and its epic's design
  sources, posts rulings on that issue, and ends with one eligibility line
  for report-ready's notes. It requires a green `vrg-validate` on the
  committed branch first.
- **pr-watch adds commits.** After submit, pr-watch fixes CI and review
  feedback by committing and pushing new commits, and rebases onto `develop`
  with a force-push when the PR conflicts.
- **A subagent's stop travels up.** An `issue-implement` run under
  `epic-implement` ends with a status (`stopped: request <id>`, `waiting:
  non-review items`). `epic-implement` presents every stop at its gate, and
  once the human has responded, dispatches a new run told only that a
  response is available.
- **Agents cannot rewrite or discard local work.** `vrg-git` denies `reset`,
  `clean`, `checkout -- .` / `*` and `rebase -i` / `--interactive`
  (`bin/vrg_git.py`, `_DENIED` and `_FLAG_DENY`). These have been there
  since the wrapper was first written, with no recorded incident. The one
  sanctioned rewrite is `vrg-reword` (`bin/vrg_reword.py`), which rewords a
  branch-local commit through a scripted rebase. It is bounded to the
  branch's own commits and the agent's own identity, and it refuses a frozen
  branch with the same freeze check as `vrg-commit` and the `vrg-git` push
  path. Raw `git commit` is refused, and `vrg-commit` builds only new
  `type(scope): message` commits, with no `--fixup` or `--amend`. What
  protects origin is separate: `push --force` is denied, `push
  --force-with-lease` is denied on protected branches, pushes to a frozen
  branch are refused, and GitHub's rulesets block force-pushes and deletion
  on `develop` and `main`.
- **One worktree per issue.** The worktree convention
  (`docs/specs/worktree-convention.md`) names `.worktrees/issue-<N>-<slug>/`
  on `feature/<N>-<slug>`, and its Agent prompt contract gives each agent one
  issue.

## 5. Design

### 5.1 What may be batched

**Hard rules.** The tooling refuses a batch that breaks one: first when the
batch starts, before any work (§5.2), and again at report-ready and at
submit as backstops:

1. Every issue is in the PR's own repository.
2. Every issue has the **same parent epic**, and that epic is either a
   planned (finite) epic or the repo's ad-hoc epic. Nothing else qualifies:
   no unparented issue, no queue item (intake, planning or deferred), and no
   mix of epics. Same-repo (rule 1) is still checked separately, since a
   finite epic's tasks can span repositories. Each issue must also not be an
   operational (validation or deployment) task. The existing per-issue
   refusals run on every issue in the list. Every batch issue is a task
   under an epic, so every one is linked with `Closes`.
3. No issue is directly Blocked-by an **open** deployment task, since the
   batch lands all at once. A closed deployment task means its change is
   already deployed, so it does not block. This is read from the issues'
   `Blocked-by:` reflinks and the operational kind labels.
4. A batch has at least 2 and at most 8 issues.

**Guidelines.** These are judgment for `epic-implement` and the human, written
into the `epic-implement` and `issue-implement` skills:

- **Batch** issues that look small from the issue body and plan entry, before
  any work: a contained bug fix, an error-message or output fix, a narrow
  refactor, a docs fix, or a config value change, in one area of the repo,
  adding no command, flag or schema. As a size cue, about ≤150 changed lines
  and ≤6 files each, tests included (§2.2).
- **Never batch** a security-labelled issue, an issue whose plan entry calls
  for design work, or a change to a contract other repositories consume (a
  CLI interface, a config schema, a reusable workflow).
- **Cap.** 4 issues by default. Up to 8 only when every item is tiny (a typo,
  a one-line config value, a wording fix; about ≤30 lines and ≤3 files).
- **Order.** Respect any Blocked-by links between batch items, then the
  plan's order. `epic-implement` batches only ready tasks that are
  independent of each other, so this applies to lists a human hands to
  `issue-implement` directly.

Substantial issues keep today's shape: one agent and one PR each.

### 5.2 Implementing a batch

`issue-implement` given a list first runs the hard-rule check (§5.1), the
same code report-ready runs, and starts nothing if the list breaks a rule.
It then writes the batch marker (below) and works on one worktree and one
branch for the whole batch. For each issue, in order:

1. Implement it. The issue's first commit is an ordinary `vrg-commit` with a
   conventional subject ending in the issue's ref, for example
   `fix(release): raise the package-index timeout to one hour (#4148)`.
   Every later commit for the issue, including any merge-review fix, is a
   **fixup of that first commit** (§5.10).
2. Validate (`vrg-container-run -- vrg-validate`) on the committed branch.
   The stack is cumulative, so validating issue *k* validates the batch up to
   *k*.
3. Merge-review it (§5.3).
4. Fold its fixups into its first commit with a non-interactive autosquash
   rebase onto the previous issue's commit (§5.10), leaving **exactly one
   commit** for the issue.
5. Move on to the next issue, on top of this commit.

Validation and merge review run once per issue, which is what N separate
agents would run today. The saving is in GitHub's gates, not local work.

**The batch marker.** When a batch starts, `issue-implement` writes the
batch's ordered issue list to `build/.vergil-state/issue-implement/<N>/batch`
in the main clone, for every member N. That is the per-issue directory
`epic-implement` already scans for stopped runs. `issue-implement` writes it,
not `epic-implement`, because a human can call `issue-implement` with a list
directly. A resumed run reads the list there and continues at the first issue
without its one commit. The marker is removed when report-ready succeeds,
together with the `merge-review/` directory. When an item is extracted, its
state directory is removed (§5.4) and every remaining member's list is
rewritten without it.

### 5.3 Merge review per issue

Each issue is reviewed on its own commit range, with `--issue` set to that
issue and `--base` set to the previous issue's collapsed commit (the batch's
base for the first issue). Its rulings, its closing comment and its
eligibility line stay on that issue, judged against its own acceptance
criteria and the design sources of the batch's one epic (§5.1).

A ruling pause is not a batch stop. A run that ends `stopped: request <id>`
resumes the batch at the same issue once the human has replied, as a
single-issue run does today.

### 5.4 Stopping mid-batch

The agent stops at an issue boundary and asks the human when an item:

- proves bigger than it looked (no longer fits the guidelines, needs design,
  or would need splitting);
- surfaces a security finding (it is filed through the normal security path);
- turns out to depend on another task's work being merged or deployed first;
- collides with an earlier batch item in a way that would need that item's
  commit reworked.

Commits already finished are never reopened. A colliding item is resolved
by extracting it: it lands later as its own PR, on top of the batch. Its
extraction adds a `Blocked-by:` link from it to the batch issue it collided
with, so `epic-implement` does not dispatch it until the batch has merged.

The run ends with a new stop status naming the item and the reason. Under
`epic-implement` it is presented at the gate with the other stops; when the
human ran `issue-implement` directly, it is reported in the terminal. The
agent never removes an item from the batch itself.

The human decides. When the human extracts an item:

- the agent discards the item's work, committed or not, with
  `vrg-git reset --hard <previous issue's commit>` and `vrg-git clean -fd`
  (§5.10), so the branch holds only the finished issues, and removes the
  item's whole `build/.vergil-state/issue-implement/<N>/` directory (its
  batch marker and any `merge-review/` state), so nothing resumes the
  discarded work;
- the item gets a comment, approved by the human, saying why it was pulled;
- it returns to its epic as an ordinary open task, to be picked up later on
  its own;
- the human decides whether the batch continues with its remaining items or
  is handed off with the issues already done.

A batch left with one issue is handed off as an ordinary single-issue PR,
under today's rules, squash merge included. A batch branch keeps working
when the issue it is named for has been extracted: the tooling reads the
batch's issues from the workflow state, never from the branch name.

The resumed run learns of the decision the way it learns of rulings today:
it is told a response is available, never given the decision text in its
prompt.

### 5.5 Handoff

`vrg-pr-workflow report-ready` accepts the batch's issues in order:

- The workflow state and its relay record carry the ordered list of issues.
  A single-issue state reads exactly as it does today.
- report-ready refuses unless the hard rules (§5.1) hold and the branch,
  counting from its base, has exactly one commit per issue, in the listed
  order, each subject ending in its issue's ref, with no merge commits.
- It records one title, summary and notes for the PR. The notes carry each
  issue's merge-review eligibility line, labelled by issue.
- The freeze, the relay push and the `.github/workflows/**` push refusal
  behave as they do for a single issue.

### 5.6 Submission

`vrg-submit-pr` runs its per-issue checks on every issue and writes a PR body
with one `Closes` line per issue (every batch issue is a task under the
batch's epic, §5.1). It re-checks the hard rules and the commit-per-issue invariant before
`gh pr create`, along with its existing reported-head checks. A batch branch is one branch, so the template, named-branch and
serial batch modes carry it unchanged.

The daily epic sweep's `task_drift` reads every linkage line of a PR body
and checks each issue, so a batch PR is never skipped.

### 5.7 After the PR is open

pr-watch keeps one commit per issue on a multi-issue PR. A CI or review fix
is committed as a fixup of the issue commit it belongs to and folded in with
a non-interactive autosquash rebase (§5.10), then force-pushed with a lease. It is never added as a new commit.
Rebasing onto `develop` when the PR conflicts is unchanged. Single-issue PRs
keep today's add-a-commit behaviour.

### 5.8 Finalize

A PR whose body closes more than one issue is a batch PR. For a batch PR,
`vrg-finalize-pr`:

- re-checks the commit-per-issue invariant on the PR's commits immediately
  before merging, and refuses with the reason if it fails, never falling back
  to squash;
- merges with `rebase`, so `develop` receives the N issue commits and the
  changelog gets N entries.

Single-issue PRs keep squash. An explicit `--strategy` still overrides, except
that `squash` on a multi-issue PR is refused (the anti-goal in §3).

### 5.9 Conventions and skills

- **Worktree convention.** A batch is one worktree and one branch, named for
  its first issue: `.worktrees/issue-<N>-batch-<slug>/` on
  `feature/<N>-batch-<slug>`. The Agent prompt contract gains a batch form
  that names every issue in order.
- **`issue-implement`** accepts a list and follows §5.2–§5.5.
- **`epic-implement`** may batch ready tasks on its own judgment, under the
  §5.1 guidelines. There is no batch-specific approval step: a batch is
  simply part of the plan it already shows after rebuilding the epic's
  state, named as a batch with its issues in order. It presents a batch stop
  (§5.4) at its gate. A batch stop means something found while working the
  batch shows that the decision to batch these issues would not have been
  made had it been known up front. Revisiting that decision is the human's
  call, never the agent's. The human extracts the one item, or **unbatches**:
  extraction (§5.4) applied to every item, each then redone as its own PR.
  While the batching guidelines are new, each such stop is also the
  evidence for refining them: the human asks why the issues were batched
  together, and the answer sharpens §5.1. Before dispatching any task, it
  checks the task's batch marker (§5.2). A task that has one is resumed as
  its whole batch, never dispatched on its own.
- **`merge-review`** reviews one issue of a batch against the previous issue's
  commit (§5.3). In a batch, both of its `--base` uses (the review and each
  check) take the previous issue's commit, not `origin/develop`, and its fix
  is committed as `vrg-commit --fixup <the issue's first commit>`, not a
  plain `vrg-commit`.
- **`pr-watch`** follows §5.7.

### 5.10 History rewriting for agents

**Principle: vrg-git protects origin, not the local checkout.** Rewriting or
discarding local work on a feature branch cannot harm origin. The guards
that protect origin (§4) stay exactly as they are. The local-only denials
that a batch runs into are lifted:

- **`vrg-git reset`** (every mode, `--hard` included) and **`vrg-git
  clean`** are allowed on any branch that is not protected, and refused on
  `develop`, `main` and release branches by the same `_is_protected_branch`
  check `push` uses. So is **`vrg-git checkout -- .`** / `*`.
- **The freeze still holds.** `reset`, `clean`, `checkout -- .` / `*` and
  `rebase -i --autosquash` are refused on a frozen (reported-ready) branch,
  with the same freeze check and wording as `vrg-commit`, the `vrg-git` push
  path and `vrg-reword`. The freeze lifts at submit, so pr-watch's folds
  (§5.7) are unaffected.
- **`vrg-commit --fixup <commit>`** creates a fixup commit of an earlier
  commit on the current branch. It is the only way an agent adds a commit to
  an issue that already has its first commit.
- **`vrg-git rebase -i --autosquash <base>`** folds the fixups into their
  targets. vrg-git allows `-i` / `--interactive` only together with
  `--autosquash`, and refuses it otherwise. Its existing no-op editors
  (`GIT_SEQUENCE_EDITOR=true`, `GIT_EDITOR=true`) keep the rebase
  non-interactive, and this works on every git in the fleet, including
  2.43. Plain `--autosquash` without `-i` needs git 2.44, which the Ubuntu
  24.04 VMs and images do not have. This allowance is technical debt, to be
  removed once git 2.44 or later is guaranteed everywhere
  (vergil-project/.github#1023).
- **Dropping an extracted item** (§5.4) is `vrg-git reset --hard` to the
  previous issue's commit, then `vrg-git clean -fd` for any untracked files
  the item created.

The one-commit-per-issue check runs inside the tooling, in report-ready,
submit and finalize.

## 6. Constraints

- Single-issue PRs behave exactly as they do today.
- **Delivery order.** The tooling, `issue-implement`, `merge-review` and
  `pr-watch` changes land first and are proved on a live batch (§8). The
  `epic-implement` change lands last.
- No silent fallback. Every check that fails refuses with its reason: the
  hard rules and the commit-per-issue invariant.
- A batch's commit subjects carry `(#issue)` rather than `(#PR)`. GitHub's
  rebase merge does not append the PR number. **Judgment:** acceptable, since
  both render as links.

## 7. Left to the builder

- The state schema change (a new version, or a list field beside `issue`), and
  how `report-ready` and `vrg-submit-pr` take the list on the command line.
- The batch marker's file format.
- The name of the batch stop status.
- How the commit-per-issue check is shared between report-ready, submit and
  finalize.
- The batch PR's title and summary format.
- Which repo carries each change. Most tooling changes are in vergil-tooling
  and the skill changes in vergil-claude-plugin.

## 8. Success

A live batch of at least two small vergil-tooling issues, implemented by one
agent from one `issue-implement` call:

- lands as one PR with one PR CI run and one post-merge CD run;
- puts one commit per issue on `develop`, each with its own changelog entry
  under its own type;
- closes every issue in the batch.

A single-issue PR in the same period still squash-merges unchanged.
