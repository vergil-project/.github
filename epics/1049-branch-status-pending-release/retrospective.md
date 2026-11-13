# vrg-repo-status pending release — retrospective

- **Epic:** vergil-project/.github#1049
- **Planning issue:** vergil-project/.github#1047
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

The status command showed `develop` and its worktrees but not what `main` was missing. We set out to rename `vrg-worktree-status` to `vrg-repo-status` and give it a **Pending release** section: every change in `develop` but not `main`, mapped to its PR by commit SHA, with the latest `vX.Y.Z` release and where the rolling `vX.Y` tag points. The section shows only where `[publish].release` is true.

That shipped in vergil-tooling v2.1.305 and passed its live validation (vergil-tooling#4272). Two batch PRs, each rebase-merged as two commits, showed as one row each under their own PR number. The back-merge showed as `release`, and every first-parent commit on `develop` was accounted for. vergil-actions showed its latest tag, which sits on a freeze commit off `main`, and `docs` showed no section.

| PR | Repo | What it did | Merged |
|---|---|---|---|
| [.github#1068](https://github.com/vergil-project/.github/pull/1068) | .github | Spec and plan | 2026-11-12 |
| [#4273](https://github.com/vergil-project/vergil-tooling/pull/4273) | vergil-tooling | Rename `vrg-worktree-status` to `vrg-repo-status`, with no alias (#4269) | 2026-11-12 |
| [#4274](https://github.com/vergil-project/vergil-tooling/pull/4274) | vergil-tooling | `lib/pending_release.py`: one `ls-remote`, a paginated compare and batched GraphQL, with any failed or incomplete lookup an error (#4270) | 2026-11-12 |
| [#4275](https://github.com/vergil-project/vergil-tooling/pull/4275) | vergil-tooling | The Pending release section, gated on `[publish].release`, with the CLI reference (#4271) | 2026-11-12 |
| [#4281](https://github.com/vergil-project/vergil-tooling/pull/4281) | vergil-tooling | Documentation review: the releasing guide points at the new section (#4256) | 2026-11-12 |

Three more vergil-tooling PRs were ad-hoc follow-ups, run before the validation so that it had real batch PRs to check (§1). They are not epic tasks:

| PR | What it did | Merged |
|---|---|---|
| [#4284](https://github.com/vergil-project/vergil-tooling/pull/4284) | The freeze refusal names every issue of a batch (#4280) | 2026-11-12 |
| [#4285](https://github.com/vergil-project/vergil-tooling/pull/4285) | **Batch** (#4279 and #4266): pluralise the `vrg-repo-status` summary counts, and fix a test whose `match=""` always passed | 2026-11-12 |
| [#4286](https://github.com/vergil-project/vergil-tooling/pull/4286) | **Batch** (#4282 and #4283): put the Pending release heading and release line on one line, and say "every change" in the releasing guide | 2026-11-12 |

- **Repos touched:** 2 (.github, vergil-tooling). The documentation review found the old name in other repos only in historical records.
- **Tasks:** 6, plus this retrospective: the spec and plan, 3 code tasks, the documentation review and 1 validation. They landed in 5 PRs; the validation needed none.
- **Releases cut** (by the human): vergil-tooling v2.1.305, which carries #4273, #4274 and #4275, and v2.1.306 (2026-11-12T23:03Z), which carries #4281 and the three follow-up PRs.
- **Span:** the planning issue opened 2026-11-12T16:59Z and the epic at 17:11Z. The spec and plan merged at 20:52Z, the first code PR at 21:18Z, and the last task closed at 22:48Z, all on the same day.

## 1. How the plan evolved

The plan held. Its four tasks ran in the planned order: Tasks 1 and 2 in parallel, then Task 3, then the validation. No task was added, dropped or split, so the plan has no Evolution during execution log.

The departures were inside tasks, and each is stated in its PR's notes:

- **Task 1 left the docstring alone.** The plan's new docstring named the Pending release section, which the command didn't have until Task 3. Task 3 then added it.
- **Task 3's refactor step assumed code that didn't exist.** Step 6 had the new summary share a pluralisation helper with `_summary` and `_development_summary`, but neither pluralised. The helper served only the new summary, and the old "1 worktrees" wording was filed and fixed as a follow-up (#4279).
- **Task 3 settled two points the plan left to the builder.** When nothing is pending, the section prints no summary line. The release-tag regex is defined again in the command rather than imported from the library's private one. Task 3 also updated the reference index's one-line description, which the plan had left to the documentation review.
- **The documentation review found almost nothing left to do.** Each task had carried its own docs step, so the review added only one pointer, in the releasing guide.

The larger change came at the validation. Its precondition needed a batch PR pending release, and after v2.1.305 the only pending change was the release back-merge. The human chose to make the case with real work instead of waiting for it to happen. We picked small vergil-tooling items from the intake queue and ran them as two batches and one single PR (§0). Two of the batch items came from the human's rulings on follow-ups this epic raised, one of them a display change the human asked for after using the released command. The validation then ran against five pending PRs, two of them batches.

## 2. Lessons learned

- **The design review's most important finding was the right one.** Spec round 1's S1.1 replaced reading PR numbers from commit subjects with mapping each commit to its PR by SHA. Batch PRs are rebase-merged, and their commit subjects carry issue numbers, so the subject approach would have mislabelled every batch. A read-only probe during plan review (P2.1) confirmed the mapping on one real batch, and the validation confirmed it on two more.
- **A validation precondition that depends on repo state can be manufactured with real work.** Waiting for a batch PR to happen would have held the epic open. Next time, the plan should say how such a precondition will be produced, not only that it must hold.
- **Using the released command found what review did not.** The one-line heading (#4282) came from the human running v2.1.305 against live data, after every review had passed.
- **The plan's reality check reads code shape, and can still miss it.** The plan's assumption about `_summary` was wrong. The cost was small, because the builder noticed it and said so, but plan steps that rely on how existing code looks should be checked against it.
- **Every merge review passed with no findings.** All nine merge reviews (the four epic tasks with a PR, and the five follow-up issues) raised nothing. This may mean five design-review rounds and 25 accepted findings left little to find, or that the reviewer is lenient on small, well-specified changes. This epic can't tell which.

## 3. Compromises & tradeoffs

- **The validation ran the v2.1.305 binary.** The one-line heading (#4282) and the pluralised summaries (#4279) shipped later, in v2.1.306, so the validation never ran with them. When this retrospective was drafted, the installed v2.1.306 showed the one-line heading against live data. Unit tests cover the rest.
- **Some rows were not seen during the validation.** No dependency update was pending then, so the `deps` row was checked only in tests. One appeared afterwards: the installed v2.1.306 typed #4290 `deps` and counted "1 dependency update". This was an informal check, not a validation run. A `PROBLEM` row can come only from bypassing branch protection, so it has not been seen live, by design.
- **The validation checked GitHub's commit-to-PR associations indirectly.** `gh api` is denied to the user agent, so it compared the rows with `vrg-gh pr view --json commits` and the first-parent log, matching rebased commits by headline. That was enough for the check, but it is not the same query the command makes.
- **The release-tag regex is in two places**, the library and the command. It is small, but the two could drift.
- **Internal names keep "worktree"**, such as `gather_worktree_status`, as the plan intended. Only the command, module and tests were renamed.
- **Task 3 skipped the plan's failing-tests-first run.** The new tests could only have failed on import.

## 4. New problems & opportunities

The sweep covered the 6 tasks, the epic issue and the planning issue #1047.

- **Epic issue:** three items were listed under Awaiting a ruling (N0.1–N0.3). All three were ruled `file`, filed as vergil-tooling#4279, #4282 and #4283, and have since merged.
- **Task issues:** every merge review raised nothing, and the validation's result comment proposed no fix tasks. None of the epic's PRs carried a review or a comment.
- **Planning issue:** all 25 design-review findings across five rounds were accepted, with none rejected or dropped. The alignment round's security count is 1 finding, 1 accepted, recorded and fixed in its task.

Already tracked: what a dependency update changed was out of scope from the start (.github#1048).

Found untracked by the sweep:

- **U1:** spec pushback round 1 was broken by the security cut, not by any security content. An ordinary discard line began with the `Security:` label outside the security markers, so the cut refused the result, and the human had to rule the kept copy ordinary before the round could be re-run. The related issues, .github#1031 and the closed #939, are about the section's markers, not this. **Filed:** .github#1073.

## 5. What's next

- **Close the planning issue.** .github#1047 is `ready-to-close` in the planning queue, since its docs PR (#1068) has merged.
- **What a dependency update changed** (.github#1048) is the natural follow-on, since dependency updates are the one TYPE whose rows this command can't explain yet.
- **U1** (.github#1073) goes through `intake-review`.

## Appendix A — Operational notes

How the validation's precondition was staged:

1. The human cut and installed v2.1.305, which carries Tasks 1–3. After the release, `develop` held only the back-merge (#4278).
2. Small items were taken from the intake queue and routed as vergil-tooling one-offs with `vrg-intake-route … one-off`: #4279 (filed from N0.1), #4266 and #4280. Later, #4282 and #4283 were filed from N0.2 and N0.3.
3. They ran as a batch (#4279, #4266 → PR #4285), a single PR (#4280 → #4284) and a second batch (#4282, #4283 → #4286). #4283 had to wait for #4281 to merge, because it rewords the line #4281 added.
4. `vrg-finalize-pr` rebase-merged each batch as its two commits and squash-merged the rest.
5. The validation then ran the installed v2.1.305 against the five pending PRs.
