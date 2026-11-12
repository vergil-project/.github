# Batched same-repo PRs — retrospective

- **Epic:** vergil-project/.github#1019
- **Planning issue:** vergil-project/.github#1017
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md) · **paad feedback:** [paad-feedback.md](paad-feedback.md)

## 0. At a glance

Every task used to land through its own PR, so a run of small fixes paid the full cost each time: PR CI, the merge, and the post-merge CD run. We set out to let one agent implement a small batch of same-epic, same-repo issues on one branch and hand them off as one PR. That PR pays the gate cost once, keeps one commit per issue on `develop` and closes every issue.

That shipped, and it has run live three times:

- **The V1 test batch.** vergil-tooling#4186 and #4103 went in as PR #4248: one PR, one CI run and one CD run. Two commits were rebase-merged in order, each issue got its own changelog entry under its own type in v2.1.303, and both issues closed.
- **The collapsed-batch test.** In vergil-claude-plugin#1024, one issue was extracted from a two-issue batch. A fresh session then resumed the other from the batch worktree, with no worktree named in its prompt, and it squash-merged as a single-issue PR.
- **The first batch epic-implement chose on its own judgment.** vergil-tooling#4253 and #4254 went in as PR #4258.

Single-issue PRs still squash-merge (vergil-claude-plugin#1021, #1029).

| PR | Repo | What it did | Merged |
|---|---|---|---|
| [.github#1036](https://github.com/vergil-project/.github/pull/1036) | .github | Spec and plan | 2026-11-11 |
| [#4233](https://github.com/vergil-project/vergil-tooling/pull/4233) | vergil-tooling | `vrg-commit --fixup` | 2026-11-11 |
| [#4234](https://github.com/vergil-project/vergil-tooling/pull/4234) | vergil-tooling | Batch rules module: the hard rules and the one-commit-per-issue check | 2026-11-11 |
| [#4235](https://github.com/vergil-project/vergil-tooling/pull/4235) | vergil-tooling | Epic sweep checks every `Closes` line of a PR body | 2026-11-11 |
| [#4236](https://github.com/vergil-project/vergil-tooling/pull/4236) | vergil-tooling | Worktree convention: the batch form | 2026-11-11 |
| [#4232](https://github.com/vergil-project/vergil-tooling/pull/4232) | vergil-tooling | vrg-git allows local rewrites on unfrozen feature branches, and `rebase -i` only with `--autosquash` | 2026-11-11 |
| [.github#1037](https://github.com/vergil-project/.github/pull/1037) | .github | Feedback note for the paad author | 2026-11-11 |
| [plugin#1013](https://github.com/vergil-project/vergil-claude-plugin/pull/1013) | vergil-claude-plugin | merge-review reviews one issue of a batch | 2026-11-12 |
| [plugin#1014](https://github.com/vergil-project/vergil-claude-plugin/pull/1014) | vergil-claude-plugin | pr-watch folds fixups into the owning issue's commit | 2026-11-12 |
| [#4239](https://github.com/vergil-project/vergil-tooling/pull/4239) | vergil-tooling | `vrg-pr-workflow batch-start`, `batch-extract`, `batch-show` (the batch marker) | 2026-11-12 |
| [#4240](https://github.com/vergil-project/vergil-tooling/pull/4240) | vergil-tooling | `vrg-finalize-pr` detects a batch, updates it by rebase, re-checks it and rebase-merges it | 2026-11-12 |
| [#4242](https://github.com/vergil-project/vergil-tooling/pull/4242) | vergil-tooling | Workflow state and `report-ready` carry an ordered issue list | 2026-11-12 |
| [#4243](https://github.com/vergil-project/vergil-tooling/pull/4243) | vergil-tooling | `vrg-submit-pr`: per-issue checks and one `Closes` line per issue | 2026-11-12 |
| [plugin#1015](https://github.com/vergil-project/vergil-claude-plugin/pull/1015) | vergil-claude-plugin | issue-implement batch mode | 2026-11-12 |
| [plugin#1017](https://github.com/vergil-project/vergil-claude-plugin/pull/1017) | vergil-claude-plugin | `[publish].release = true`, so the plugin can be released at all | 2026-11-12 |
| [plugin#1021](https://github.com/vergil-project/vergil-claude-plugin/pull/1021) | vergil-claude-plugin | epic-implement batches ready tasks and resumes batches | 2026-11-12 |
| [plugin#1023](https://github.com/vergil-project/vergil-claude-plugin/pull/1023) | vergil-claude-plugin | Resume the last issue of a collapsed batch from the batch worktree | 2026-11-12 |
| [#4255](https://github.com/vergil-project/vergil-tooling/pull/4255) | vergil-tooling | Documentation review: batch PRs across the site docs | 2026-11-12 |
| [docs#100](https://github.com/vergil-project/docs/pull/100) | docs | Batch PRs in the task model and the workflow pages | 2026-11-12 |
| [plugin#1031](https://github.com/vergil-project/vergil-claude-plugin/pull/1031) | vergil-claude-plugin | Batch support in the skills index and README | 2026-11-12 |
| [plugin#1032](https://github.com/vergil-project/vergil-claude-plugin/pull/1032) | vergil-claude-plugin | Doctrine §1.7: the batch-stop ruling words | 2026-11-12 |
| [#4258](https://github.com/vergil-project/vergil-tooling/pull/4258) | vergil-tooling | **Batch** (#4253 and #4254): finalize's conflict message no longer calls a submitted PR frozen to agents; the batch worktree form in the CLAUDE.md template | 2026-11-12 |
| [#4259](https://github.com/vergil-project/vergil-tooling/pull/4259) | vergil-tooling | The last frozen-to-agents claim removed; conflict steps documented | 2026-11-12 |

- **Repos touched:** 4 (.github, vergil-tooling, vergil-claude-plugin, docs).
- **Tasks:** 27, plus this retrospective. That is 24 code and doc tasks, 1 deployment and 2 validations, landed in 23 PRs, one of them a batch closing two tasks. Two more PRs were test vehicles for the validations: vergil-tooling#4248 (a batch) and vergil-claude-plugin#1029.
- **Releases cut** (by the human):
  - vergil-tooling v2.1.302 (deployed) and v2.1.303 (the V1 changelog check).
  - vergil-claude-plugin v2.1.64 (deployed) and v2.1.65 (#1022).
  - vergil-tooling v2.1.300 and v2.1.301 were also cut during the epic, but before any epic code merged.
- **Span:** epic opened 2026-11-10T23:21Z; the first PR merged 2026-11-11T19:11Z and the last task closed 2026-11-12T17:32Z.

## 1. How the plan evolved

The plan's fifteen tasks landed in the planned order, with one exception: the human started Task 9 before V1 closed. Every other change was added work. The plan's [Evolution during execution](plan.md#evolution-during-execution) log has the dated entries; the shape of it was:

- **The deploy hit a gate this epic didn't build.** vergil-tooling#4194 had made `vrg-release` refuse any repo without `[publish].release = true`. vergil-claude-plugin had no such key, so the plugin could not be released at all. A one-line task (plugin#1016) unblocked it. The key's name was then questioned, because it confuses releasing with publishing, and that went to intake (.github#1041). A stale dependency-update PR, which had failed on a transient apt mirror error, also blocked the tooling release until it was merged.
- **Live validation found the epic's most important gap.** V1 proved the happy path. Task 9's agent then noticed that a batch extracted down to one issue left that issue's work in a worktree that single-issue resume never looks in. With batches of two being the common case, the human judged this would happen often, and ruled it fixed inside the epic (plugin#1022), with its own live validation (plugin#1024).
- **That validation found a doctrine gap.** The batch-stop choices (extract, continue, hand off, unbatch) had no words in §1.7's closed set of ruling words (plugin#1030).
- **The documentation review found code that was wrong, not just docs.** `vrg-finalize-pr`'s conflict message said a submitted PR is frozen to agents, which contradicts the freeze lifting at submit. The CLAUDE.md consumer template also lacked the batch form. Both were filed in the epic (vergil-tooling#4253, #4254) and run as the first batch epic-implement chose itself. That batch's agent then found the same false claim in one more message, and doc prose that #4253 had made stale (vergil-tooling#4257).

The human's standing instruction for the second half was to close with no known gaps. That is why five follow-up tasks stayed inside the epic instead of going to intake.

## 2. Lessons learned

- **Delegated agents re-add guards the design deleted.** The design review had removed a "tree must match after the fold" check as defence against an impossible failure. Task 8a's agent wrote it back into the skill, and the merge review did not flag it, because nothing in the review's inputs says what the design removed. Task 9's agent also added two refusal conditions beyond the issue, which the human accepted only after checking each. From then on, every brief said "add no checks the issue doesn't ask for, and never reintroduce one the design deleted", and the problem did not recur. The paad feedback note (.github#1028) makes the same point for the pushback method.
- **Tests that read the real repo's HEAD pass locally and fail in CI.** Task 1a's vrg-git change failed `test_checkout_feature_denied_in_main_worktree` only in CI: local validation runs on a branch, CI on a detached merge commit. Every later brief warned about it.
- **Live validation earned its cost.** Review and unit tests passed everything. The two live runs found the collapsed-batch gap and the ruling-word gap. Neither could be seen from the diff.
- **Operational tasks were filed as empty scaffolds.** D1, V1 and #1024 were all created with the template's placeholders. Each needed a scope change, drafted by the agent and posted by the human, before it could run. That cost a human round trip each time. `vrg-issue-create` also refuses a body for `--kind validation`, so the right content cannot be given at filing (.github#1057).
- **Interfaces between parallel tasks need pinning in the plan.** merge-review (8b) was written against `batch-show` (2b) before 2b existed, and the plan did not fix its output format. The two were checked against each other before submit and matched, but only because the gate looked.
- **Same-repo batching ran cleanly once it existed.** The batch with #4253 and #4254 went from agent to merged with no friction, and it is the only time the feature was used for real work rather than a test.

## 3. Compromises & tradeoffs

- **Task 9 started before V1 closed.** The human judged that V1's two pending checks (the changelog entries and a single-issue squash) only confirmed behaviour already shown. Both later passed.
- **The live batch tests used two issues.** The cap of 4, and 8 for tiny items, has not been exercised.
- **A collapsed batch whose remaining issue has no commit yet** still leaves the batch worktree behind. The human judged it not significant enough to hold the epic (intake .github#1046).
- **`vrg-finalize-pr`'s retry when a pinned merge's head moves** keys on GitHub's error text "Head branch was modified". That text was taken from the REST docs and never checked against what `gh pr merge --match-head-commit` prints. If it differs, finalize stops instead of retrying, and nothing unchecked merges. No run so far has hit that path (intake .github#1058).
- **A hand-added `Blocked-by:` link** (issue-implement's extraction path for a colliding issue) is assumed to be read by `vrg-epic-audit` like one written by `vrg-issue-create`. That has not been checked (intake .github#1059).
- **The advisory ID that `vrg-security-record` wrote into a public intake issue** was removed by hand. The conflict between the tool and the doctrine is open (.github#1039).

## 4. New problems & opportunities

The sweep covered 27 tasks, the epic issue and the planning issue #1017. Every "Awaiting a ruling" item has a trusted ruling, with the filed issue existing where the ruling was `file`. The epic issue's security items have a counts line (3 recorded). No PR carried a review or comment. Every finding #1017 did not accept has a reason or an intake reference.

Spun off and tracked:

- .github#1039: `vrg-security-record` writes the advisory ID into the public intake issue, against doctrine §1.7.
- .github#1041: `[publish].release` conflates releasing with publishing; rethink the `[publish]` section.
- .github#1045: `vrg-finalize-pr` waits out its timeout when the semgrep job fails before the required `Semgrep OSS` check is registered.
- .github#1046: a collapsed batch whose remaining issue has no commit leaves the batch worktree stranded.
- One security record, in the intake queue.

Found untracked by the sweep, with the human's ruling on each:

- **U1:** a merge-reviewer reply arrived with HTML-escaped angle brackets (`&lt;…&gt;`), which the run restored by hand (plugin#1024). **Filed:** .github#1054.
- **U2:** `batch-show` prints refs as `#N` and `report-ready` as `N` (vergil-tooling#4228). **Filed:** .github#1055.
- **U3:** D1's acceptance line `vrg-commit --fixup --help` cannot pass as written. **Dropped** by the human: a wording slip in a closed task, with no defect behind it. `vrg-commit --help` shows the flag.
- **U4:** the freeze refusal message names only the first issue of a batch. This was a discarded alignment finding on #1017. **Filed:** .github#1056.
- **U5:** operational tasks are filed as empty scaffolds, and `vrg-issue-create` refuses a body for `--kind validation`, so every deployment or validation needs a scope-change round trip (§2). **Filed:** .github#1057.
- **U6:** the finalize head-moved retry text is unverified against `gh`'s real output (§3). **Filed:** .github#1058.
- **U7:** whether a hand-added `Blocked-by:` link is read like a tool-written one is unverified (§3). **Filed:** .github#1059.

## 5. What's next

- **Release the epic's last merges.** vergil-tooling #4255, #4258 and #4259, and vergil-claude-plugin #1031 and #1032, merged after the last tags, so they are not yet in a release. The consumer CLAUDE.md template change (#4254) reaches other repos only through `vrg-fleet-sync --target claude-md`.
- **The intake items above** go through `intake-review`. The `[publish]` rethink (.github#1041) and the operational-task scaffold problem (.github#1057) are the two worth planning.
- **Batch-size guidelines.** The spec says every batch stop feeds guideline refinement. The V1 batch's #4103 was over the per-issue size guideline (9 files), and that is the first data point.
- **The paad feedback note** ([paad-feedback.md](paad-feedback.md)) is ready for the human to send to the paad author.
