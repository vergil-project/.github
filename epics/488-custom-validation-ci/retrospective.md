# Custom Validation in PR CI — Retrospective

Epic: [vergil-project/.github#488](https://github.com/vergil-project/.github/issues/488) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Close the parity gap between local `vrg-validate` and PR CI. Local
validation ran each repo's custom validator (the `[validation]` custom stage), but PR CI did not,
so a repo could merge a change its own validator rejected. The epic set out to:

- add `vrg-validate --check custom`;
- run it on every PR from the vergil-actions quality workflow;
- fold its result into `quality / evidence`;
- get every active-fleet repo that has a custom validator green under it;
- correct the docs that claimed a parity that didn't exist.

**What shipped.** All of it, plus one change the spec had ruled out:

- vergil-tooling v2.1.268 added `--check custom`.
- vergil-actions v2.1.58 runs it as `quality / custom` on every PR and writes `quality-custom.log`
  into the evidence artifact. `quality / evidence` now needs common, custom, lint and typecheck.
- All five validator repos passed on a real PR.
- vergil-tooling v2.1.269 changed the rulesets. `quality / evidence` is now required in every
  repo's CI-gates ruleset, replacing `quality / common`. Without that, a failing custom stage would
  not have blocked a merge in the repos the epic was for (see §1).
- The ruleset change was applied to 15 of the 22 active-fleet repos. The other 7 were deferred
  (see §3).

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#492](https://github.com/vergil-project/.github/pull/492) | #489 | Spec and plan |
| vergil-tooling | [#3653](https://github.com/vergil-project/vergil-tooling/pull/3653) | #3650 (T1) | `vrg-validate --check custom` |
| vergil-vm | [#366](https://github.com/vergil-project/vergil-vm/pull/366) | #364 (T5) | Missing `tofu` fails the custom validator instead of skipping |
| vergil-actions | [#1073](https://github.com/vergil-project/vergil-actions/pull/1073) | #1072 *(added)* | Fix package-runtime fixtures for the reworded check-runtime message |
| vergil-actions | [#1071](https://github.com/vergil-project/vergil-actions/pull/1071) | #1067 (T3) | `custom` job in `ci-quality.yml`, gated through `quality / evidence` |
| vergil-containers | [#719](https://github.com/vergil-project/vergil-containers/pull/719) | #715 (T11) | Drop the duplicate `quality / pins` job (also the T8 probe PR) |
| vergil-claude-plugin | [#726](https://github.com/vergil-project/vergil-claude-plugin/pull/726) | #725 *(added)* | Doc note on `quality / custom` (also the T7 probe PR) |
| vergil-vm | [#368](https://github.com/vergil-project/vergil-vm/pull/368) | #367 *(added)* | Doc note on `quality / custom` (also the T10 probe PR) |
| packages | [#34](https://github.com/vergil-project/packages/pull/34) | #33 *(added)* | Doc note on `quality / custom` (also the T6 probe PR) |
| vergil-python | [#64](https://github.com/vergil-project/vergil-python/pull/64) | #63 *(added)* | Doc note on `quality / custom` (also the T9 probe PR) |
| vergil-tooling | [#3662](https://github.com/vergil-project/vergil-tooling/pull/3662) | #3660 *(added)* | Require `quality / evidence` in every CI-gates ruleset, replacing `quality / common` |
| vergil-tooling | [#3672](https://github.com/vergil-project/vergil-tooling/pull/3672) | #3649 | Documentation review: site docs and `CLAUDE.md` |
| vergil-actions | [#1078](https://github.com/vergil-project/vergil-actions/pull/1078) | #1077 *(spawned)* | CI-gates required-check docs and ruleset examples |
| docs | [#58](https://github.com/vergil-project/docs/pull/58) | #57 *(spawned)* | `ci.md`: the quality gate covers common and custom |

- **Repos touched by PRs (9):** .github, vergil-tooling, vergil-actions, vergil-vm,
  vergil-containers, packages, vergil-claude-plugin, vergil-python, docs. The ruleset change was
  also applied to 15 repos in vergil-project and logical-minds-foundry.
- **Children (25):**
  - 14 planned (T1–T11 and three bookends). 11 were added during execution.
  - 4 deployments: vergil-tooling#3651, vergil-actions#1068, vergil-tooling#3661, and
    vergil-containers#720 (closed not planned).
  - 5 validations: packages#30, vergil-claude-plugin#724, vergil-containers#716, vergil-python#62,
    vergil-vm#365.
  - 2 closed not planned as superseded: vergil-containers#720 and vergil-tooling#3659.
- **PRs merged:** 14, including the spec/plan PR and excluding this one.
- **Releases cut (by the human):**
  - vergil-tooling **v2.1.268** (`--check custom`) and **v2.1.269** (the ruleset change).
  - vergil-actions **v2.1.58** (the `custom` job).
- **Span:** epic opened 2026-10-26 11:44Z; last task before this one closed 2026-10-26 19:12Z,
  about 7.5 hours.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is reconstructed from the issue
record and the session. The planned tasks all shipped in the planned order. The deviations were
two unplanned fixes, a way to get a real PR in quiet repos, and one premise in the spec that
turned out to be false.

- **A fix task blocked T3.** The vergil-actions PR for the `custom` job (#1071) failed
  `package-runtime fixtures`. The test fixtures expected the old check-runtime message, which
  vergil-tooling#3640 (epic #472) had reworded. vergil-actions#1072/#1073 updated the fixtures, and
  #1071 merged twenty minutes later. The failure was unrelated to this epic's code; it was the
  first vergil-actions PR to run against the reworded message.
- **Validation needed real PRs, so doc tasks were added to supply them.** T6–T10 each required a
  real PR opened after v2.1.58, but four of the five validator repos had no PR in flight. The
  human chose to file one small doc task per repo (#33, #725, #63, #367) saying the custom
  validator now runs in PR CI as `quality / custom`. Each PR doubled as the validation probe.
  vergil-containers used T11's PR (#719) instead. All five validations recorded SUCCESS.
- **The spec's ruleset premise was wrong.** The spec's first non-goal was "no change to branch
  protection or rulesets", on the assumption that `quality / evidence` was already required
  everywhere. It was not. `desired_ci_gates_ruleset` required it only when the primary language
  had a lint or typecheck command. All five validator repos set no primary language, so none of
  them required it, and a red `quality / custom` would not have blocked a merge in exactly the
  repos the epic was for. This surfaced in two places at once. The human pasted vergil-containers'
  live required checks while confirming T11's precondition. The documentation review found the
  same condition in the code (#3659).
- **First wrong turn: a manual ruleset edit.** The agent's first response was
  vergil-containers#720, a task to edit the develop ruleset by hand. The human corrected this:
  rulesets are generated from code and asserted by `vrg-github-repo-config`, so the change belongs
  in code. #720 was closed in favour of the code fix.
- **The fix was made fleet-wide, not limited to language-less repos.** #3659 had proposed
  requiring `quality / evidence` in language-less repos too, alongside `quality / common`. The
  human chose to replace `quality / common` outright, so no gate is kept only for historical
  reasons. #3660 makes `quality / evidence` required in every repo and drops `quality / common`
  from the required set. The `quality / common` job still runs and still gates through evidence.
  #3659 was closed as superseded.
- **A proactive sweep instead of waiting for the audit.** A ruleset change leaves every repo out
  of step with the code until it is applied, and the scheduled audit would then fail repo by
  repo. The human asked for a sweep instead: deployment task #3661, a diff → apply → audit loop
  over the 22-repo inventory, run by the human. Diff found drift beyond the expected swap in all
  seven mq-rest-admin-project repos (see §3), so those were held back. The other 15 converged and
  audit compliant.
- **The documentation review was paused for the ruleset work and resumed after it.** It then
  absorbed the gate change: it removed its own note about the language-less gap and spawned two
  cross-repo doc tasks (vergil-actions#1077, docs#57) for docs that still described the old
  required set.

## 2. Lessons learned

- **Check a spec's premises against what is live.** The non-goal rested on "`quality / evidence`
  is required everywhere". Running `vrg-github-repo-config diff`, or reading
  `desired_ci_gates_ruleset`, during planning would have disproved it in a minute. When a spec
  says some existing mechanism already does X, the plan should include a step that checks it.
- **Rulesets are code. Never plan a manual ruleset edit.** Any change to the required set is a
  `github_config.py` change plus a fleet sweep. Agents should go to `desired_ci_gates_ruleset`
  first, not to the ruleset UI.
- **Plan a probe PR for validation tasks in quiet repos.** "Validate on a real PR" assumes a PR is
  coming. In repos with little traffic it isn't, and the plan should name the probe PR up front.
  Small, useful doc PRs worked well here.
- **Fleet commands must work in the human's shell.** The first version of the sweep used
  `for r in $REPOS` over a newline-separated string. zsh doesn't word-split that, so the loop ran
  once with all 22 names as one `--repo` value. Only `diff` ran, so nothing changed. Handed-off
  loops now use arrays (`REPOS=(…)` and `"${REPOS[@]}"`), which behave the same in zsh and bash.
- **A tooling message change can break fixtures in a downstream repo.** #3640 reworded a message
  in vergil-tooling, and vergil-actions' fixtures pinned the old text. It was caught only by the
  next vergil-actions PR.
- **Workflow-touching branches are still submitted from inside the worktree.** vergil-actions#1077
  touched `.github/workflows/README.md`. GitHub refused the agent push, so the worktree-free
  `vrg-submit-pr <branch>` form couldn't be used. This is the same friction epic #472 recorded.
- **Same next time:** sub-agents per task with the worktree prompt contract, per-repo batches,
  and spawning the doc-review tasks in parallel. The two spawned doc PRs merged in the same batch
  as the review's own PR.

## 3. Compromises & tradeoffs

- **Seven mq-rest-admin-project repos are not converged.** The sweep's diff found drift that
  predates this epic:
  - vulnerability alerts are off;
  - `require_extra_approval_for_unattributed_changes` is `true` where the code expects `false`;
  - in mq-rest-admin-java, -python and -rust, the old per-version required checks from epic #338
    are still required;
  - mq-rest-admin-go's `vergil.toml` pins Go 1.25, outside the window, so the tool can't run
    against it.

  These repos have had no active work for months. The human decided not to block the epic on them
  and filed one ad-hoc issue per repo instead, to be handled when each repo is modernized. Until
  then, those repos still require `quality / common`, and the scheduled audit keeps reporting
  them.
- **The spec's ruleset non-goal was knowingly broken.** The epic's goal (a failing custom stage
  blocks the merge) could not be met without it. The human approved the change, and it is
  recorded here rather than in an amended spec.
- **`quality / common` stays in the producible-gate catalog.** It is no longer required, but it
  still runs on every PR. Keeping it in the catalog means a ruleset that still requires it, during
  rollout or in the deferred mq-rest-admin repos, is reported as ordinary drift. Without it, the
  same ruleset would be reported as requiring a check that can never be produced.
- **Out of scope by design:** diogenes-project/diogenes, which has a custom validator but is pinned
  to vergil-actions v2.0, has no `quality / custom`. With current tooling, `vrg-github-repo-config
  apply` would refuse that repo until its `ci.yml` moves to v2.1.

## 4. New problems & opportunities

| Item | Where it went |
|---|---|
| mq-rest-admin-project ruleset/settings drift and the stale Go pin | Ad-hoc issues: [.github#46](https://github.com/mq-rest-admin-project/.github/issues/46), [common#233](https://github.com/mq-rest-admin-project/mq-rest-admin-common/issues/233), [dev-environment#176](https://github.com/mq-rest-admin-project/mq-rest-admin-dev-environment/issues/176), [go#329](https://github.com/mq-rest-admin-project/mq-rest-admin-go/issues/329), [java#330](https://github.com/mq-rest-admin-project/mq-rest-admin-java/issues/330), [python#532](https://github.com/mq-rest-admin-project/mq-rest-admin-python/issues/532), [rust#128](https://github.com/mq-rest-admin-project/mq-rest-admin-rust/issues/128) |
| `vrg-github-repo-config` prints a raw `ResolutionError` traceback, not a clean error | [vergil-tooling#3670](https://github.com/vergil-project/vergil-tooling/issues/3670) (intake) |
| Worktree-free `vrg-submit-pr <branch>` can't submit a branch whose agent push was refused | Logged, not yet acted on (also recorded in epic #472) |
| vergil-containers CD failed on Trivy CRITICAL/HIGH findings in the dev-rust 1.92 image | Logged, not yet acted on; unrelated to this epic |
| diogenes-project/diogenes is pinned to vergil-actions v2.0 | Out of scope (on-hold org); needs its `ci.yml` moved to v2.1 when the org is revived |

## 5. What's next

- **Watch:** the first real PR in each validator repo where `quality / custom` fails, to confirm
  that the failing evidence gate actually blocks the merge.
- The mq-rest-admin-project issues above are picked up when that org is modernized.
- No follow-on brainstorm was spun out of this epic.

## Appendix A — Operational notes

The deploy order that worked:

1. **T1:** release vergil-tooling (v2.1.268). Confirm that `vrg-validate --check custom` runs
   cleanly and exits 0 in a repo with no custom validator.
2. **T3 (after the #1072 fixture fix):** release vergil-actions (v2.1.58) under `v2.1`.
3. **T5, T11:** merge only.
4. **T6–T10:** validate on a real PR opened after v2.1.58, using the docs PRs as probes. Each
   PR's evidence job listed `quality-custom.log`.
5. **#3660:** release vergil-tooling (v2.1.269) and upgrade the host install.
6. **#3661:** from any directory, using an array of repos:

   ```text
   for r in "${REPOS[@]}"; do vrg-github-repo-config diff  --repo "$r"; done
   for r in "${REPOS[@]}"; do vrg-github-repo-config apply --repo "$r"; done
   for r in "${REPOS[@]}"; do vrg-github-repo-config audit --repo "$r"; done
   ```

   Stop and record anything in the diff beyond the expected swap. The "Local checks skipped"
   warning outside a repo's own checkout is expected.

**Gotchas:**

- `vrg-gh run download` is denied for the user identity. Validation read the evidence job's
  staging listing instead of downloading the artifact.
- `vrg-issue-create --kind deployment` doesn't accept `--body-file`, because the kind stamps its
  own scaffold. Create the issue first, then replace the body with `vrg-gh issue edit --body-file`.
