# epic-review and epic-implement adhoc mode — retrospective

- **Epic:** vergil-project/.github#1077
- **Planning issue:** vergil-project/.github#1076
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)

## 0. At a glance

We set out to give queue work and ad-hoc work one pair of `/vergil:epic-*`
commands with verbs that do not overlap. `epic-review` would replace
`intake-review`, `planning-review` and `deferred-review`, with planning review
made prune-only and drop allowed in every queue. `epic-implement` would gain
an `adhoc` mode that works every ad-hoc epic's open tasks, backed by a new
`vrg-queue adhoc` view that can tell which tasks are already in flight. All of
it shipped as planned:

- the two skills and the new view;
- the `one-off` route out of the planning queue;
- queue messages and queue epic bodies that no longer name a retired skill;
- both deployments and both live validations;
- the documentation sweep, which grew by two doc tasks.

| PR | Task | What it did | Merged (UTC) |
|---|---|---|---|
| [.github#1102](https://github.com/vergil-project/.github/pull/1102) | .github#1078 | Spec and plan | 11-13 19:31 |
| [vergil-tooling#4335](https://github.com/vergil-project/vergil-tooling/pull/4335) | vergil-tooling#4330 | `vrg-intake-route` allows `one-off` from planning, drops `in-progress` after the move | 11-13 20:17 |
| [vergil-tooling#4337](https://github.com/vergil-project/vergil-tooling/pull/4337) | vergil-tooling#4331 | Queue refusals, hints and docs name `epic-review`; queue epic bodies name no skill | 11-13 20:21 |
| [vergil-tooling#4338](https://github.com/vergil-project/vergil-tooling/pull/4338) | vergil-tooling#4333 | New `vrg-queue adhoc` view | 11-13 20:25 |
| [vergil-tooling#4350](https://github.com/vergil-project/vergil-tooling/pull/4350) | vergil-tooling#4334 | In-flight detection (open PR, relay ref, branch, worktree, merge-review, batch marks) and `--root` | 11-13 20:58 |
| [vergil-tooling#4369](https://github.com/vergil-project/vergil-tooling/pull/4369) | vergil-tooling#4366 | `github-issues.md` and CLI reference cover `vrg-queue adhoc` and name `epic-review` in the refusal passage | 11-14 12:35 |
| [vergil-claude-plugin#1048](https://github.com/vergil-project/vergil-claude-plugin/pull/1048) | vergil-claude-plugin#1042 | New `epic-review` skill; the three queue review skills retired | 11-13 20:16 |
| [vergil-claude-plugin#1054](https://github.com/vergil-project/vergil-claude-plugin/pull/1054) | vergil-claude-plugin#1043 | `epic-implement adhoc` mode; kind detection and cross-pointing refusals | 11-13 21:09 |
| [vergil-claude-plugin#1059](https://github.com/vergil-project/vergil-claude-plugin/pull/1059) | vergil-claude-plugin#1041 | Documentation review: the lifecycle note counts `epic-review` | 11-14 12:29 |
| [docs#104](https://github.com/vergil-project/docs/pull/104) | docs#103 | Management pages describe `epic-review` and prune-only planning review | 11-13 20:27 |
| [docs#107](https://github.com/vergil-project/docs/pull/107) | docs#105 | Management pages describe `epic-implement adhoc` and `vrg-queue adhoc` | 11-14 12:28 |

| Operational task | Closed (UTC) | Outcome |
|---|---|---|
| vergil-claude-plugin#1044 (D1) | 11-13 20:56 | SUCCESS: group A installed; queue epic bodies .github#438, #439 and #531 rewritten |
| vergil-claude-plugin#1045 (V1) | 11-13 21:01 | SUCCESS: 4 of 4 live checks of `epic-review` passed |
| vergil-claude-plugin#1046 (D2) | 11-13 23:32 | SUCCESS: group B installed (the human had already installed the tooling) |
| vergil-claude-plugin#1047 (V2) | 11-13 23:37 | SUCCESS: 6 of 8 checks passed, 2 not exercised as the plan allows |

- **Counts:** 15 tasks and the retrospective. There are 11 PRs, with +4054/−790 lines changed across 51 files. 9 of the 15 tasks were planned; the sweep added 2 doc tasks, and the 4 operational tasks opened no PR.
- **Repos touched:** `.github`, vergil-tooling, vergil-claude-plugin, docs.
- **Releases:**
  - vergil-tooling v2.1.308 (Tasks 1, 2, 5) and v2.1.309 (Task 6).
  - vergil-claude-plugin v2.1.67 (Task 3) and v2.1.68 (Task 7).
  - Not yet released: vergil-tooling#4369 and vergil-claude-plugin#1059, both docs only. The docs repo has no releases.
- **Span:** the planning issue opened 2026-11-13 15:45Z, and the last task closed 2026-11-14 12:35Z, about 21 hours. Implementation itself took about 4 hours, from 19:31Z to 23:37Z on 11-13. Doc tasks, and review gates waiting on the human, took the rest.

## 1. How the plan evolved

The plan held. Every planned task shipped in the planned shape, and no task
was dropped or re-scoped. Groups A and B ran in parallel as the plan
intended. The one dependency that crossed between them, Task 7 waiting on
Task 3, never cost anything, because Task 3 had merged before Task 6 did.

The only change to the task list came from the documentation review
(vergil-claude-plugin#1041). It found two gaps in other repos and filed one
doc task in each, as the bookend convention says. The docs repo's management
pages had no `epic-implement` entry and nothing on the adhoc mode
(docs#105). vergil-tooling's `github-issues.md` did not list
`vrg-queue adhoc`, and its refusal passage still described queue review in
words that did not match the tool's refusal message (vergil-tooling#4366).
Both gaps had the same root cause. The plan gave the docs repo one task
(Task 4) for `epic-review` only, and gave the adhoc mode's documentation to
the plugin's site docs alone. The plan's evolution log records this.

Smaller points from execution:

- **D2's install step was already done.** The human had installed the
  tooling release everywhere after it was cut, so the run checked the
  install instead of repeating it.
- **One check passed on the first path the plan named.** The Git Data API
  read of a relay ref (Task 6, Step 2) worked, so the fallback the plan
  prepared was never needed.

## 2. Lessons learned

- **The two-group split paid off.** Splitting into two groups with
  disjoint code and only one dependency across them let five tasks start at
  once. All of the code was merged within about 1.5 hours of the spec and
  plan merging.
- **Plan doc tasks by feature as well as by repo.** The plan matched each
  feature to repos, but left the docs repo's coverage of the adhoc mode
  implicit. The sweep caught it, at the cost of two extra PRs and a second
  round of submits. Next time, give each feature an explicit owner on every
  doc surface it touches.
- **Concurrent edits to one docs file were predictable and harmless.** Three
  vergil-tooling branches all edited `cli-tools-overview.md`. They were
  submitted together, and no conflict actually blocked a merge, but the
  agents only noticed the overlap after the fact. A plan can mark a shared
  file ahead of time.
- **The TDD red step was skipped twice.** The runs for #4331 and #4333 did
  not confirm that the new tests failed before they wrote the code. They
  reported it in their final messages, but no public record says so. Their
  final validation passed with 100% coverage. Alignment had already ruled
  against adding ceremony for a separate refactor step (A1.10). The red step
  is cheap, and the plan asks for it, so `issue-implement` should keep
  insisting on it.
- **Run-level ruling items need IDs from the start.** V1 and V2 listed their
  items without the `N0.<i>` ID that §1.7 requires. So V1's ruling line reads
  `Human ruling: 1 file`, and V2's line assigned `N0.1` after the fact. The
  sweep could still match each ruling to its item, but only by reading the
  text.
- **Relaying rulings through a new run works, even on closed tasks.** Both
  validation items were carried out by rulings-only runs on closed tasks
  (vergil-claude-plugin#1045 and #1047), with no worktree and no new
  `Outcome:`. The epic issue's own follow-up (N0.1) was filed directly from
  the main session.

## 3. Compromises and tradeoffs

- **Two validation checks were not exercised.** V2 items 5 (resuming a task
  stopped at merge review) and 8 (`--repo` for a self-homed private repo) had
  no live state to run against. The plan allows "not exercised", and the
  tasks did not manufacture that state on purpose. Both paths are covered by
  the skill text and by the tooling's unit tests, but neither has run live.
- **Some V2 evidence came from applying the skill's rules, not fresh runs.**
  For checks 3, 4 and 7, the run loaded `epic-implement` once and applied its
  rules to each argument. It did not start the skill fresh each time. The
  outputs match the skill text, but the public result says "ran" without
  this qualification.
- **The private-repo enumeration rests on one cross-check.** D1 and V2 both
  found `vrg-gh repo list --visibility private` empty. That could mean the
  org has no private repos, or that the identity cannot see them. The only
  cross-check was the org's public repo count of 9.
- **The `one-off` guard is proven by unit tests only.** As the plan decided,
  a live check would have moved a real planning item if the guard failed.
- **Releases were not ordered.** As ruled in planning (P2.4), the releases
  went out in whatever order was convenient. Nothing broke.

## 4. New problems and opportunities

**Filed during execution (by ruling):**

- vergil-claude-plugin#1055: for a self-homed private repo's queue ref, the
  queue refusal should name `--repo` (epic N0.1, from Task 7's merge
  review).
- vergil-project/.github#1111: reconcile five planning-queue items that
  carry deferral records (V1 item 1).
- vergil-tooling#4209 closed as not planned: a stale failure of an earlier
  dependency update, found by V2. The current failure is
  vergil-tooling#4365, which is open and blocks the next tooling release.

**Already tracked:**

- vergil-claude-plugin#1037: a malformed security marker broke design-review
  round S2 during planning.
- .github#1085: a review delegate ran an out-of-contract `vrg-git fetch` in
  round S3.
- .github#497: `vrg-gh api` is denied for the user identity. This blocked
  sub-issue, milestone and API reads in several runs here; they worked
  around it with `vrg-epic-audit --epic` and `vrg-gh search`.

**Unaddressed-findings sweep:**

The sweep covered 15 tasks, the epic issue, the planning issue's round
comments and all 11 PRs. Every item listed under `Awaiting a ruling` has a
trusted ruling, and every rejected design-review finding has its reason.
None of the 11 PRs has a review. The human ruled on the five untracked
items:

| # | Item | Outcome |
|---|---|---|
| U1 | Private-repo enumeration cannot tell "none" from "cannot see" (D1, V2) | **Filed** as .github#1115 |
| U2 | A validation run applied a loaded skill's rules instead of invoking it fresh per check, and the public result does not say so (V2) | **Filed** as .github#1116 |
| U3 | `vrg-gh search code` returned nothing during V1 | **Dropped**: not reproduced; if it recurs, add it to .github#497 |
| U4 | V1 could not verify the "melete" half of vergil-actions#1019's revival condition | **Dropped**: the outcome (keep) did not depend on it |
| U5 | The TDD red step was skipped in two runs (§2) | **Dropped**, recorded in §2: once is a lesson, not yet a pattern |

## 5. What's next

- **The planning issue:** .github#1076 is still open with `in-progress`, by
  design. `epic-create` closes it on its next touch, once `vrg-queue planning`
  shows it `ready-to-close`.
- **The next tooling release:** it waits on vergil-tooling#4365 (the failed
  2.1.310 dependency update). The documentation PRs #4369 and plugin#1059
  ship in the next tooling and plugin releases.
- **Follow-ups:** vergil-claude-plugin#1055 and .github#1111. U1 and U2 are
  in the intake queue as .github#1115 and #1116.
- No follow-on brainstorm or epic was opened.
