# Retrospective: remove Java support

**Epic:** [vergil-project/.github#952](https://github.com/vergil-project/.github/issues/952)
**Planning issue:** [vergil-project/.github#951](https://github.com/vergil-project/.github/issues/951)
**Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to remove Java as a supported language from the vergil fleet, the way Ruby was removed under #407. Java had zero users. Its only consumer, `mq-rest-admin-java`, was idle, and its support was dead weight in three repos.

All of it shipped, and the fleet now supports five languages: Python, Go, Rust, C++ and TypeScript.

- **vergil-containers:** the Java image, its lifecycle adapter, `osv-scanner` (only the Java image used it), and Java's entries in the version matrix, pins, package workflows and `.trivyignore`.
- **vergil-tooling:** `java` is a retired primary language, so a `vergil.toml` that names it fails loudly. All Java code, tests and site docs are gone.
- **vergil-actions:** the Maven Central publish path and the Java branches of the release and security actions. Package signing is unchanged.
- **Operational:** the `dev-java` and `prod-java` GHCR packages are deleted, the Maven Central secrets are deleted, and `mq-rest-admin-java` is archived.
- **Cleanup beyond the plan:** the org profile's language and repository counts, #384's title and body, and the retired Ruby commands in the deprecation-warning hook.

| PR | Repo | Task | What it did | Merged | Release |
|---|---|---|---|---|---|
| [#956](https://github.com/vergil-project/.github/pull/956) | .github | #953 | Spec and plan | 2026-11-09 | n/a |
| [#769](https://github.com/vergil-project/vergil-containers/pull/769) | vergil-containers | #767 | Removed the Java image, its lifecycle adapter and osv-scanner, and regenerated the matrix, window and package workflows | 2026-11-09 | not yet released |
| [#1107](https://github.com/vergil-project/vergil-actions/pull/1107) | vergil-actions | #1106 | Removed the Maven Central publish path and the Java branches | 2026-11-09 | not yet released |
| [#4113](https://github.com/vergil-project/vergil-tooling/pull/4113) | vergil-tooling | #4111 | Retired `java` and removed its code and tests | 2026-11-09 | not yet released |
| [#4114](https://github.com/vergil-project/vergil-tooling/pull/4114) | vergil-tooling | #4112 | Removed Java from the site docs | 2026-11-09 | not yet released |
| [#957](https://github.com/vergil-project/.github/pull/957) | .github | #955 | Listed the five languages in the org profile and dropped Java from #384 and #410 | 2026-11-09 | n/a |
| [#92](https://github.com/vergil-project/docs/pull/92) | docs | #91 | Removed Java from the org docs site | 2026-11-09 | n/a |
| [#996](https://github.com/vergil-project/vergil-claude-plugin/pull/996) | vergil-claude-plugin | #995 | Dropped the Java and Ruby test commands from the deprecation-warning hook | 2026-11-09 | not yet released |
| [#961](https://github.com/vergil-project/.github/pull/961) | .github | #960 | Corrected the org profile's repository count and containers link, and logged the plan's evolution | 2026-11-09 | n/a |

**Counts:** 12 tasks under the epic.

- 9 closed by a merged PR, as listed above.
- 1 deployment task, #768, closed on `Outcome: SUCCESS`.
- 1 documentation review, #4107, closed without a PR: vergil-tooling needed no change.
- 1 retrospective, #954, closed by this PR.

**Size:** the eight implementation PRs add 183 lines and delete 1,294. That excludes the spec and plan PR.

**Repos touched:** .github, vergil-containers, vergil-tooling, vergil-actions, docs and vergil-claude-plugin, plus operational changes in mq-rest-admin-project: the archive, the secret deletions and #18's rescope.

**Releases:** none was cut during the epic. The latest releases of vergil-tooling, vergil-containers, vergil-actions and vergil-claude-plugin predate every merge. Until the next release of each, the released tools still carry Java support.

**Span:** from the planning issue opening on 2026-11-08 to this PR, about one day.

## 1. How the plan evolved

The plan held. All six planned tasks shipped as written, and the changes were to precision, not shape.

- **The planned edits missed spots.** Task 5's merge review found a second stale language count in the org profile (line 93), which the Files list missed. Checking the profile then surfaced two older stale statements, a "five repositories" count and the `vergil-docker` link, so #960 was added for them.
- **Validation covers less than the plan assumed.** `vrg-validate` runs no strict mkdocs build in vergil-tooling, so Task 3 checked links by search. It also doesn't run vergil-actions' shell tests, which only CI runs.
- **The operational facts were simpler than expected.** The Maven Central token had already expired, and the two secrets existed only on `mq-rest-admin-java` itself.
- **The documentation review needed no vergil-tooling change.** It closed without a PR and spawned two sibling tasks. A human scope change widened one of them, #995, to also remove the retired Ruby commands from the same hook.
- **Retired Ruby was cleaned up too.** The human ruled to drop it from #384's title and body and from the hook, since it sat in the same lines as Java.

## 2. Lessons learned

- **Design review earned its keep, even on a removal.** The first instinct was that a removal needs little review. The single rounds still caught real breakage before any code ran:
  - the substring grep would have forced stripping TypeScript's CodeQL mapping;
  - the deployment check named a command the agent identity can't run;
  - no task owned the MQ-side filing or the #639 closure;
  - the word-bounded search couldn't see snake_case leftovers.

  One round per stage was the right amount.
- **"Delete everything named Java" needs two searches.** The word-bounded gate keeps `javascript` safe but hides `_JAVA_VERSION_RE` and `temurin`. An unbounded review search, read hit by hit, catches what the gate can't. Use the pair for the next language removal.
- **Line numbers in a plan go stale, and can point at the wrong thing.** A `repo_init.py:562` reference pointed at `javascript`, the one entry that had to stay. Plans should name the entry and treat line numbers as hints.
- **A count in prose drifts with the thing it counts.** The profile's language count was wrong twice (seven while there were six, then a second copy missed), and its repository count was wrong as well. Removal tasks should search for the count as a number, not just for the language name.
- **Parallel agents need their own scratch space.** Two agents shared one scratchpad and one overwrote the other's helper script, harmlessly this time. Later dispatches gave each agent its own subdirectory.

## 3. Compromises and tradeoffs

- **The idle MQ repo was allowed to break.** The vergil removal did not wait for `mq-rest-admin-java` to be archived, so its CI broke for the few hours in between. With zero users, that was accepted.
- **GPG was left entirely alone.** The Maven-only forwarding of the GPG secrets was removed from the code. The GPG key and its secrets were not touched, including those on the archived repo, because the key signs more than Maven artifacts.
- **Shell tests ran outside `vrg-validate`.** Because `vrg-validate` doesn't cover vergil-actions' shell tests, the Task 4 agent ran two of them directly. That breaks the rule that `vrg-validate` is the only validation command. CI ran them on the PR as well.
- **One red/green step was skipped.** Task 2 wrote the rejection test and the fix together, without the failing-first run. The test pins the full message and passes.
- **The attestation was posted by the agent.** The human authorised the agent to post the deployment evidence, since attestations are not authenticated.
- **Not yet released.** The removal reaches users only with the next release of each repo.
- **Three MQ-side doc issues are still open.** They are mq-rest-admin-common#234, mq-rest-admin-dev-environment#177 and mq-rest-admin-project/.github#49. They are tracked in that org, on its own schedule, as the spec intended.

## 4. New problems and opportunities

**Unaddressed-findings sweep:** 11 tasks swept, plus the epic issue and the planning issue.

- **Rulings:** every `Awaiting a ruling` item has a ruling. #955's N0.1 was ruled `fix` and carried out. #4107's two items were ruled (`dismiss`, and `file` into #995), and their ruling lines were recorded at the sweep.
- **Design review:** every finding in the planning issue's three rounds was accepted.
- **PRs:** none of the nine merged PRs had a review comment.

Problems the epic surfaced:

1. **`vrg-validate` does not run vergil-actions' shell tests.** CI runs them, but local validation gives a false sense of coverage, and agents are tempted to run tests outside it. Filed: vergil-project/vergil-actions#1108.
2. **`vrg-validate` runs no strict mkdocs build.** The plan assumed one, so broken links to deleted pages are caught only by search. Filed: vergil-project/vergil-tooling#4115, to decide whether `vrg-validate` should build the site strictly.
3. **The retrospective bookend was created without the `retrospective` label.** `epic-create` seeds it by title, and this run used plain `vrg-issue-create` instead of `--kind retrospective`, so the gate fell back to the title. Filed: vergil-project/vergil-claude-plugin#997.
4. **No clean close for a documentation review that changes nothing.** #4107 had no PR to close it, and closing it needed a ruling. Filed: vergil-project/vergil-claude-plugin#998, to document closing as completed with the search evidence as the standard path.
5. **Parallel sub-agents share one scratchpad.** Filed: vergil-project/vergil-claude-plugin#999.

## 5. What's next

No follow-on epic. The forward work is the three open MQ-side doc issues listed in §3, plus the five intake items in §4 (vergil-actions#1108, vergil-tooling#4115, vergil-claude-plugin#997, #998 and #999).
