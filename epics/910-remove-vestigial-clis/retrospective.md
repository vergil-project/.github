# Retrospective: remove vestigial CLIs and skills

**Epic:** [vergil-project/.github#910](https://github.com/vergil-project/.github/issues/910)
**Planning issue:** [vergil-project/.github#831](https://github.com/vergil-project/.github/issues/831)
**Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to clean up vergil-tooling's command-line surface. Commands and skills that nothing in the fleet still runs would go, together with their code, tests and docs. The goal was hygiene and a slightly faster test suite.

All of it shipped:

- **Seven `vrg-*` commands removed:** `vrg-adhoc-migrate`, `vrg-wait-until-green`, `vrg-changelog`, `vrg-container-test`, `vrg-resolve-tooling-version`, `vrg-pr-fix-body` and `vrg-epic-unlink`.
- **One standalone command dropped:** `vrg-repo-profile` is no longer a separate command. `vrg-validate` still runs the check.
- **One skill removed:** `migrate-repo`, from vergil-claude-plugin.
- **One code path removed:** after `vrg-wait-until-green` went, only tests reached `wait_for_checks`' self-lookup path, so a follow-up task removed it.

| PR | Repo | Task | What it did | Merged | Release |
|---|---|---|---|---|---|
| [#913](https://github.com/vergil-project/.github/pull/913) | .github | #911 | Spec and plan | 2026-11-06 | n/a |
| [#85](https://github.com/vergil-project/docs/pull/85) | docs | #84 | Dropped the removed tools and the `migrate-repo` skill from the tools-and-skills page | 2026-11-06 | n/a |
| [#932](https://github.com/vergil-project/vergil-claude-plugin/pull/932) | vergil-claude-plugin | #931 | Removed the `migrate-repo` skill and the stale `vrg-wait-until-green` host-tool entry | 2026-11-06 | not yet released |
| [#3964](https://github.com/vergil-project/vergil-tooling/pull/3964) | vergil-tooling | #3952 | Removed the seven commands, with their modules, tests, test-only library code and docs | 2026-11-06 | not yet released |
| [#3970](https://github.com/vergil-project/vergil-tooling/pull/3970) | vergil-tooling | #3965 | Made `wait_for_checks`' `required` argument mandatory and removed its self-lookup path | 2026-11-07 | not yet released |
| [#3971](https://github.com/vergil-project/vergil-tooling/pull/3971) | vergil-tooling | #3953 | Dropped the `vrg-repo-profile` command and renamed its module to `bin/repo_profile.py`. Its reference page now documents a `vrg-validate` check | 2026-11-07 | not yet released |
| [#3973](https://github.com/vergil-project/vergil-tooling/pull/3973) | vergil-tooling | #3947 | Documentation review: found no stale mentions of removed tools in the fleet. Corrected one doc that named a nonexistent `vrg-repo-init` | 2026-11-07 | not yet released |

**Counts:** 8 tasks under the epic.

- 7 closed by a merged PR: the docs task, four implementation tasks (one of them the follow-up #3965) and the documentation review.
- This retrospective.

**Repos touched:** vergil-project/.github, vergil-tooling, vergil-claude-plugin and docs.

**Releases cut:** none. The last vergil-tooling release, 2.1.286, came out at 16:33 UTC on 2026-11-06, before any of this epic's code merged. The changes ship in the next vergil-tooling and vergil-claude-plugin releases.

**Span:** the epic was created on 2026-11-06 at 15:41 UTC, and the last task closed on 2026-11-07 at 12:36, about 21 hours later.

### What we removed

This quantification was a spec requirement (§6). The line counts are taken from each PR's merge-commit diff, split by top-level directory.

| Area | Added | Deleted | Net |
|---|---:|---:|---:|
| vergil-tooling `src/` | 22 | 786 | −764 |
| vergil-tooling `tests/` | 36 | 1,302 | −1,266 |
| vergil-tooling `docs/` | 47 | 268 | −221 |
| vergil-tooling other (`pyproject.toml`) | 0 | 8 | −8 |
| vergil-claude-plugin | 2 | 169 | −167 |
| docs | 1 | 6 | −5 |
| **Total (excluding the spec and plan)** | **108** | **2,539** | **−2,431** |

**As a share of vergil-tooling:** the baseline is develop just before #3964, and the share is the epic's net change against it.

| Area | Baseline lines | Net change | Share |
|---|---:|---:|---:|
| `src/` | 67,092 | −764 | 1.1% |
| `tests/` | 122,885 | −1,266 | 1.0% |
| `src/` and `tests/` | 189,977 | −2,030 | 1.1% |

**Test overhead:**

- **Test count (measured):** `vrg-validate`'s pytest stage went from 10,432 passed to 10,316. That is 116 fewer tests, or 1.1%. #3964 removed 113 tests and #3970 removed 3, and the rename in #3971 changed no counts.
- **Test-stage time (measured, inconclusive):** the median of five runs on the perf harness went from 36.4 s to 42.6 s. The per-run spread was 35.0–45.4 s before and 33.9–48.8 s after, and a second harness setting moved the other way (40.5 s to 38.9 s). The harness cannot resolve a change of about 1%.
- **Time saved (estimate, not measured):** if test time scales with test count, removing 1.1% of the tests saves about 0.4–0.5 s of a test stage that takes 36–42 s. Checking 2,030 fewer lines of source and tests with lint and type checks saves a similarly small amount. Put plainly, the payoff is hygiene, not speed: about 1% of vergil-tooling no longer needs maintaining, reading or migrating.

## 1. How the plan evolved

The plan has no evolution log. This account comes from the execution record: the task issues and their comments, the PRs, and the epic-implement session.

**Scope changed during design, not during execution.**

- **`vrg-audit-approve` kept:** the brainstorm had put it on the removal list. Spec pushback round 1 raised one security finding, which showed that removing the command would break a required check the plugin docs cite. The human then kept the command as a placeholder for the AUDIT identity, which is on hold until the agentic APIs exist.
- **Module rename:** plan pushback round 1 found that keeping the repo-profile module under its old `vrg_` name after dropping its console script contradicts vergil-tooling's CLI entry-point standard. The module was renamed instead.
- **Retrospective quantification:** the human added the requirement at the end of alignment.

**One task was added during execution.** #3952 pointed out that once `vrg-wait-until-green` was gone, `wait_for_checks`' self-lookup path was reached only by tests. Under the spec's "tests are not callers" rule that path is dead code. The plan had kept `wait_for_checks` whole and applied the rule one function at a time, so the task left the path alone and asked for a ruling. The human ruled to file a follow-up task (#3965), which removed it.

**The order held.** Three tasks ran in parallel first: #3952 (tooling), #931 (plugin) and #84 (docs). #3953 waited for #3952 because they edit the same files. #3953 and #3965 then ran in parallel in separate worktrees, since they touch no common files. The documentation review ran last. Every task reached `report-ready` with `vrg-validate` passing on its first implementation, and none needed rework after submission.

## 2. Lessons learned

- **"Tests are not callers" needs to be applied to code paths, not just to functions.** The plan applied the rule to whole functions. That missed a parameter default inside a kept function that had become dead. Next time, the dead-code step should also ask: which branches of the kept functions did only the removed commands reach?
- **Check what the validation pipeline actually runs.** The plan assumed `vrg-validate` builds the mkdocs site. In vergil-tooling it runs only markdownlint and yamllint on docs, so broken links and nav entries had to be checked by search. Before a plan leans on a gate, a quick read of the repo's validate configuration would confirm the gate exists.
- **Running the perf harness through `uv run` can rewrite `uv.lock`.** In #3964 it quietly downgraded two dependencies, and the change was swept into a commit before it was caught and reverted. A task that runs tools with `uv run` should check `uv.lock` before committing.
- **The timing measurement was the wrong instrument for a 1% change.** Five-run medians varied by more than 10 s between runs. The count and line metrics carried the result. Future cleanup epics should quantify with counts and treat timing as context only.
- **Batch the obvious calls.** During brainstorming, the "are tests callers?" question was asked on its own when it was a no-brainer. It belonged in a batch of obvious decisions for the human to confirm or correct.

## 3. Compromises and tradeoffs

- **History keeps the old names:** CHANGELOG entries, release notes, dated specs and plans, and two recorded test fixtures still name the removed commands. They were left alone on purpose: they record what was true at the time.
- **The docs repo lost one historical note.** Removing the `vrg-adhoc-migrate` entry also removed the page's only mention that the old `standing` epic alias is retired. The human declined to restore it, since the current docs describe only `ad-hoc` (docs#84, ruling N0.1).
- **No speed claim:** we report a measured reduction in tests and lines, but no measured speed-up, because the harness could not resolve one.
- **Some tools kept that may be removed later:** `vrg-reword`, `vrg-github-repo-init`, `vrg-adhoc-epic`, `vrg-ensure-label` and `vrg-audit-approve` stay. `vrg-audit-approve` is a placeholder whose future depends on the AUDIT identity.

## 4. New problems and opportunities

**Spun off during planning:**

- **Automatic fleet sync** ([.github#909](https://github.com/vergil-project/.github/issues/909), planning). The five ad-hoc sync tools stay until this replaces them with an `ops.yml` job that propagates master-file changes as auto-merged PRs.
- **Moving `vrg-generate-commands` into the mq-rest-admin project** ([.github#908](https://github.com/vergil-project/.github/issues/908), deferred).

**Surfaced during execution:**

- **`wait_for_checks`' self-lookup path:** became dead with `vrg-wait-until-green`, filed as #3965 and done in this epic.
- **A doc naming a nonexistent `vrg-repo-init`:** found by the documentation review and fixed in #3973.
- **Transient GitHub App 401s:** `vrg-gh`/`vrg-git` occasionally failed GitHub App authentication with HTTP 401 and succeeded on an immediate retry. This happened several times during the epic. The open issue [vergil-tooling#3825](https://github.com/vergil-project/vergil-tooling/issues/3825) already tracks it, so this epic's occurrences were added there as a comment.

**Unaddressed-findings sweep:** 7 tasks swept (every child except this retrospective), plus the epic issue and the planning issue.

- **Rulings:** every `Awaiting a ruling` item has a trusted ruling.
  - docs#84 N0.1: declined.
  - vergil-tooling#3952 N0.1: filed as #3965.
  - vergil-tooling#3952 N0.2 and N0.3: dismissed. Both are recorded as lessons in §2.
- **Design review:** every finding from the planning issue's three design-review rounds was accepted. The one security finding was resolved by a scope change, and the human ruled it not be recorded.
- **PR reviews:** none of the seven PRs has review comments.
- **Untracked problems found:** one, the transient 401s. An existing issue was found for it, vergil-tooling#3825.

## 5. What's next

- **Automatic fleet sync** (.github#909) is the natural follow-on: it retires the five remaining one-off sync tools.
- A similar sweep of the vergil-claude-plugin skills and hooks could follow once the plugin's surface grows again. Nothing is planned yet.
