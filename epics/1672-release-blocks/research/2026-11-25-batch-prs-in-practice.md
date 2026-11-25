# Batch PRs in practice — research, 2026-11-25

- **Epic:** vergil-project/.github#1672
- **Planning issue:** vergil-project/.github#1663
- **Status:** research input to the brainstorm, not a design

This note records an analysis of how the batch PR mechanism of epic
vergil-project/.github#1019 (closed 2026-11-12) has worked in real use, made
to decide whether batching can become the unit of release for tasks that
must ship together. Each section separates **data** (what GitHub, git and
the documents show) from **judgment** (marked **[J]**).

Method: three read-only analyses on 2026-11-25. One covered the
vergil-tooling batch PRs, one covered the batch PRs in vergil-claude-plugin,
vergil-actions, vergil-containers and docs, and one traced the mechanism's
rules and its fixes since epic 1019. Every PR whose head branch contains
`batch` was listed. The analyses could not call `vrg-gh api`, so rewrites
after a PR opened were inferred from commit dates and the head SHAs of CI
runs, not from GitHub's force-push timeline events.

## 1. Summary

- 44 real batches merged or open: 25 in vergil-tooling, 19 elsewhere. Of
  the PRs with `batch` in the branch name, the rest were single issues.
- Every merged batch landed on its base as one linear commit per issue, each
  subject ending in its issue's `(#N)`. No filed issue reports a batch that
  merged wrongly.
- A fix to a non-last commit of the chain, made after the PR opened, has
  happened four times, and each time the later commits came through
  correctly.
- Scale exercised: at most 4 issues per batch (median 2). The largest was
  vergil-tooling#4972: 4 issues, +2057/−313 lines, 28 files.
- **[J]** The routine path is reliable. The hard paths (fixes to a middle
  commit, rebases, fixes after submit) work but rest on a small sample,
  and nothing beyond 4 issues has been tried.

## 2. The rules and where they live

Data:

- **Tooling hard rules** (`lib/pr_workflow/batch_rules.py`,
  `check_hard_rules`): 2 to 8 issues (`MAX_BATCH = 8`), no duplicates, all
  in the PR's repo, tasks only, a buildable parent, trusted authors, not
  operational, not blocked by an open deployment task, and one shared
  parent epic. There is no size or security check.
- **Guidelines, in the skills only** (spec of epic 1019, lines 187–204):
  each issue about 150 lines and 6 files or fewer, with no new command,
  flag or schema; never batch a security-labelled issue, design work or a
  cross-repo contract; a cap of 4 (8 when every item is tiny);
  `epic-implement` batches only independent ready tasks.
- **"Small"** is stated in `skills/issue-implement/SKILL.md` (the batch
  criterion, the cap and the "proves bigger" batch stop), in
  `skills/epic-implement/SKILL.md`, in the plugin's site docs and in the
  tooling's `git-workflow.md` and `standards/github-issues.md`.
- **"No security issues"** is stated in one line,
  `skills/issue-implement/SKILL.md:651`. The spec, plan and retrospective
  of epic 1019 give no reason for it.
- **Precedent:** in epic vergil-project/.github#1467 the human set the
  security guideline aside for three batches, vergil-tooling#4961, #4972
  and #4973, which merged and shipped in v2.1.336 without a recorded
  failure.
- **Related security mechanics:** `batch-extract` refuses while a
  `security-*.md` details file exists; a security finding is a batch stop;
  security items appear publicly only as counts.

**[J]** Relaxing "small" and "no security issues" is a change to skill text
only. The rules that matter more for batching as the unit of release are the
same-repo rule, the cap of 8 and "independent tasks only".

## 3. Fixes to the mechanism since epic 1019

| Ref | Trigger in real use | Status |
|---|---|---|
| vergil-claude-plugin#1022 | A batch collapsed to one issue left its work where resume does not look | Fixed (#1023) |
| vergil-claude-plugin#1038 | A collapsed batch whose last issue had no commit stranded the batch worktree | Fixed |
| vergil-claude-plugin#1030 | Batch-stop choices had no ruling words | Fixed (#1032) |
| vergil-tooling#4253, #4257 | Finalize called a submitted PR frozen to agents | Fixed |
| vergil-tooling#4280 | The freeze message named only the first batch issue | Fixed |
| vergil-tooling#4297 | `batch-show` and `report-ready` wrote refs differently | Fixed |
| vergil-tooling#4584, vergil-claude-plugin#1131 | A fix to an open batch PR could never end `eligible`: the review saw one issue's scope | Fixed |
| vergil-tooling#4570 | Pushes to a submitted batch PR were unchecked | Fixed (`batch-check`) |
| vergil-claude-plugin#1236 | pr-watch held a fix on a ruling that changed nothing | Fixed |
| vergil-tooling#4988 | `batch-check` read PR bodies outside the trust layer | Fixed (#5015) |
| vergil-tooling#5021 | A misleading rule name | Fixed (#5049) |
| vergil-tooling#5110 | Native blocked-by links to a deployment task were ignored | Fixed (#5121) |
| vergil-claude-plugin#1262 | The never-batch list may be too broad | Open |
| vergil-project/.github#1023 | The `rebase -i` exception exists only for git 2.43 | Open |

**[J]** Every fix came from edge cases, wording or trust reads; none traces
to a batch's size.

## 4. Per-PR data

### 4.1 vergil-tooling

| PR | Issues | +/− (files) | Rewrites after open | CI fails | Conflict | Hours to merge |
|---|---|---|---|---|---|---|
| 5197 | 3 | 217/16 (9) | 0 | 0 | no | 0.06 |
| 5121 | 2 | 344/46 (6) | 0 | 0 | no | 0.07 |
| 5120 | 2 | 402/69 (7) | 1 | 1 | no | 0.61 |
| 5106 | 2 | 697/32 (8) | 0 | 0 | no | 0.08 |
| 5098 | 2 | 137/31 (7) | 0 | 0 | no | 0.07 |
| 5077 | 2 | 13/12 (6) | 0 | 0 | no | 0.07 |
| 5073 | 3 | 332/36 (9) | 0 | 0 | no | 0.09 |
| 5072 | 2 | 309/5 (6) | 0 | 0 | no | 0.09 |
| 5069 | 2 | 112/7 (3) | 0 | 0 | no | 0.47 |
| 5056 | 4 | 622/63 (22) | 0 | 0 | no | 0.07 |
| 5054 | 2 | 376/52 (8) | 0 | 0 | no | 0.08 |
| 5050 | 2 | 325/76 (12) | 0 | 0 | no | 0.13 |
| 5049 | 2 | 110/16 (8) | 0 | 0 | no | 0.13 |
| 5047 | 3 | 395/31 (13) | 0 | 0 | no | 0.12 |
| 5040 | 3 | 534/35 (12) | 0 | 0 | no | 0.06 |
| 5028 | 2 | 19/11 (2) | 0 | 0 | no | 0.07 |
| 5026 | 2 | 481/44 (6) | 0 | 0 | no | 0.10 |
| 5023 | 2 | 267/15 (9) | 0 | 0 | no | 0.09 |
| 4973 | 2 | 920/72 (15) | ≥1 | 1 | likely | 0.41 |
| 4972 | 4 | 2057/313 (28) | ≥1 | 1 | likely | 0.93 |
| 4961 | 3 | 1059/129 (19) | ≥1 | 0 | likely | 0.22 |
| 4817 | 2 | 125/6 (2) | 0 | 0 | no | 0.06 |
| 4815 | 3 | 1530/43 (9) | 1 | 0 | yes | 0.48 |
| 4715 | 2 | 40/4 (5) | 0 | 0 | no | 0.06 |
| 4606 | 3 | 233/4 (7) | 0 | 0 | no | 0.08 |

Not batches (single-issue, squash-merged): 5071, 4642, 4602, 5015, 4406.
"Likely" conflicts are inferred from a rebase onto a newer `develop` with no
CI run on the head the PR opened with; 4815's conflict is confirmed by its
merge-review record.

### 4.2 Other repos

| PR | Issues | +/− | Rewrites after open | CI fails | Conflict | Hours to merge |
|---|---|---|---|---|---|---|
| vergil-claude-plugin#1387 | 4 | 135/64 | 0 | 1 | no | open |
| vergil-claude-plugin#1347 | 4 | 93/22 | 0 | 0 | no | 0.03 |
| vergil-claude-plugin#1318 | 3 | 97/17 | 0 | 0 | no | 0.03 |
| vergil-claude-plugin#1305 | 2 | 59/47 | 0 | 0 | no | 0.05 |
| vergil-claude-plugin#1286 | 2 | 91/12 | 1 | 0 | probable | 0.11 |
| vergil-claude-plugin#1282 | 2 | 219/57 | 1 | 0 | unknown | 0.14 |
| vergil-claude-plugin#1274 | 3 | 53/10 | 0 | 0 | no | 0.04 |
| vergil-claude-plugin#1273 | 3 | 40/16 | 0 | 0 | no | 0.04 |
| vergil-claude-plugin#1132 | 4 | 58/7 | 0 | 0 | no | 0.04 |
| vergil-claude-plugin#1117 | 2 | 17/4 | 0 | 0 | no | 0.04 |
| vergil-claude-plugin#1085 | 2 | 90/4 | 0 | 0 | no | 0.03 |
| vergil-claude-plugin#1079 | 2 | 75/9 | 0 | 0 | no | 0.03 |
| vergil-actions#1368 | 2 | 42/42 | 0 | 0 | no | 0.07 |
| vergil-actions#1280 | 2 | 8/6 | 0 | 0 | no | 0.05 |
| vergil-actions#1223 | 2 | 26/8 | 0 | 0 | no | 0.03 |
| vergil-containers#1052 | 2 | 729/100 | 0 | 0 | no | 0.39 |
| vergil-containers#1020 | 2 | 80/39 | 0 | 0 | no | 0.03 |
| vergil-containers#913 | 2 | 129/74 | 0 | 0 | no | 0.04 |
| docs#167 | 2 | 15/5 | 0 | 0 | no | 0.05 |

vergil-claude-plugin#1387 is open because `vrg-submit-pr` could not turn
auto-merge on during a GitHub outage (vergil-tooling#5198), not for a batch
reason; a single-issue PR shows the same state.

## 5. Fixes to a middle commit after the PR opened

1. **vergil-tooling#4972, commits 1 and 2 of 4.** Merge reviews found
   defects, and CI failed on tests that fail only under GitHub Actions. The
   fixes were folded into each owning commit, the branch was rebased onto
   a newer `develop`, the later members were reviewed again with no items,
   and the PR went green.
2. **vergil-tooling#4973, commit 1 of 2.** A CI failure was fixed in
   commit 1; commit 2's patch came through byte-identical. This is the
   cleanest evidence that later commits survive a middle fix.
3. **vergil-tooling#5120, commit 1 of 2.** An Actions-only test failure
   was fixed in commit 1; commit 2 was changed separately for its own
   review finding.
4. **vergil-claude-plugin#1282, commit 1 of 2.** The first issue was
   reviewed again after the PR opened, one finding was fixed, and both
   commits were rebuilt on the latest `develop`; the second commit kept its
   subject and content.

Before submit, every observed fixup went to the newest commit: issues were
built one at a time, each reviewed and folded before the next began.

## 6. Gaps and rough edges

Data:

- **CI tests only the branch tip.** No intermediate commit of a batch has
  been validated on its own.
- **Actions-only test failures** caused red CI on 3 of the 25 tooling
  batches, each costing a rewrite and a fresh review.
- **A fresh review after a rewrite covers only the changed commit's
  issue.** In vergil-claude-plugin#1282 only the first issue was reviewed
  again. After-open close records carry "Branch as reviewed: no".
- **No path for rebasing after merge review:** vergil-claude-plugin#1140
  (open).
- **Merge-review records name pre-autosquash fixup commits** that no longer
  exist; submit accepted them.
- **No fix pushed by pr-watch after submit** appears on any batch PR, so
  that path is untested in real use.
- **Tests:** a fixup folded into the first of two commits is tested
  (`test_vrg_git.py`), as are tip fixups and whole-change records. Nothing
  tests a fixup to a middle commit of three or more end to end, a fold that
  conflicts, or a whole-change record after a later clean rebase.
- **A security batch in flight**, vergil-containers #974, #994 and #995,
  had no PR after its handoff (see vergil-tooling#5199).
- **Report-ready printed `ok: true` on a failed relay push**
  (vergil-tooling#5035), since fixed.

**[J]** Before batches grow, the edge cases need listing and testing: fixes
to a middle commit across longer chains, folds that conflict, long-lived
batches drifting from `develop`, a fresh review of every later commit after
a rewrite, rebasing after review, and fixes pushed after submit. Tip-only CI
matters less once a batch is the unit of release, since its intermediate
states are never released; it still matters for bisecting.

## 7. Limits on scaling up

Data:

- The cap of 8 is hard-coded; nothing above 4 has run.
- Same repo is a hard rule and a stated non-goal of epic 1019, because a PR
  closes only issues in its own repo and CI savings are per repo.
- `epic-implement` batches only the runnable frontier, tasks independent by
  construction; a task that depends on another being merged or deployed is
  a batch stop.
- One merge review runs per issue before handoff; after submit, every fix
  triggers a whole-branch review that grows with the batch.
- Release tooling groups a batch's commits into one row of the pending
  release, and takes contents from the PRs' closing references.

**[J]** "Independent tasks only" is backwards for this purpose: a chain
where A builds on B and B builds on C is the best case for a batch, built
C, B, A on one branch and merged together, instead of three merge-and-wait
round trips. Work that must ship together across repos cannot be one batch,
which is the case the follow-on ownership design covers.
