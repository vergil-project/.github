# Upstream-refresh PRs merge themselves — retrospective

- **Epic:** vergil-project/.github#1168
- **Planning issue:** vergil-project/.github#1067
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

The nightly upstream-refresh job in vergil-containers and vergil-python opened
or refreshed one PR on a fixed `upstream-refresh` branch, and a human had to
merge it. We set out to make the job carry its PR through to a merge into
`develop` as the org automation App, and to succeed only when the PR merged or
there was nothing to change. We also set out to keep the App's private key away
from the hook and everything it loads, and to fence the patch so that an
unreviewed merge can only move pins.

That shipped, and it has run live. On 2026-11-17 the first runs on the new
workflow opened and merged their own PRs, opened and merged by
`app/vergil-project-automation`:

- **vergil-python #91:** opened 09:31:44Z, merged 09:34:19Z.
- **vergil-containers #933:** opened 09:43:30Z, merged 10:11:33Z.
- **vergil-containers #938:** a dispatched run, opened 13:00:05Z, merged
  13:28:00Z.

A token renewal has not yet been seen live: both containers waits (1682s and
1675s) ended under the 1800s renewal threshold. See §3.

| PR | Repo | What it did | Merged |
| --- | --- | --- | --- |
| [.github#1203](https://github.com/vergil-project/.github/pull/1203) | .github | Spec and plan | 2026-11-16 |
| [#4520](https://github.com/vergil-project/vergil-tooling/pull/4520) | vergil-tooling | Task 1: the App token source (mint, renew, revoke, in-process; `cryptography`; MIT-0 allowed) | 2026-11-16 |
| [.github#1217](https://github.com/vergil-project/.github/pull/1217) | .github | Spec §6.2 and plan: per-block version comparison with an anchor group, and the removal rule | 2026-11-16 |
| [#4527](https://github.com/vergil-project/vergil-tooling/pull/4527) | vergil-tooling | Task 3a: `prepare`, the `paths` key and the path fence | 2026-11-16 |
| [#4530](https://github.com/vergil-project/vergil-tooling/pull/4530) | vergil-tooling | Task 2: the token source in every `gh` and `git` call | 2026-11-16 |
| [#4531](https://github.com/vergil-project/vergil-tooling/pull/4531) | vergil-tooling | Task 3b: `publish`, the CLI modes and the tooling docs | 2026-11-16 |
| [vergil-actions#1267](https://github.com/vergil-project/vergil-actions/pull/1267) | vergil-actions | Task A1: `ops-upstream-refresh.yml` as `prepare` and `publish` | 2026-11-16 |
| [vergil-python#89](https://github.com/vergil-project/vergil-python/pull/89) | vergil-python | Task P1: the allowlist, with `series` as the anchor | 2026-11-16 |
| [vergil-containers#931](https://github.com/vergil-project/vergil-containers/pull/931) | vergil-containers | Task C1: the allowlist; the hook refuses a digest change with no version change | 2026-11-17 |
| [#4540](https://github.com/vergil-project/vergil-tooling/pull/4540) | vergil-tooling | The fence's hash rule: a hash changes only with its block's version | 2026-11-17 |
| [#4545](https://github.com/vergil-project/vergil-tooling/pull/4545) | vergil-tooling | The non-whole-file refusal names the hash rule | 2026-11-17 |
| [vergil-python#92](https://github.com/vergil-project/vergil-python/pull/92) | vergil-python | `runtime.toml` checksums marked with the hash group | 2026-11-17 |
| [vergil-containers#937](https://github.com/vergil-project/vergil-containers/pull/937) | vergil-containers | Digest and toolchain hash lines marked with the hash group | 2026-11-17 |
| [vergil-containers#941](https://github.com/vergil-project/vergil-containers/pull/941) | vergil-containers | Docs and comments: refresh PRs merge themselves | 2026-11-17 |
| [#4559](https://github.com/vergil-project/vergil-tooling/pull/4559) | vergil-tooling | Documentation review: tooling docs and guides | 2026-11-17 |
| [docs#126](https://github.com/vergil-project/docs/pull/126) | docs | `ci.md` describes the nightly upstream refresh | 2026-11-17 |

- **Repos touched:** 6 (.github, vergil-tooling, vergil-actions,
  vergil-containers, vergil-python, docs).
- **Tasks:** 20, plus this retrospective. The plan had 10 tasks and the 2
  bookends; 8 were added during execution (#1216, vergil-tooling#4539, #4542,
  #4543, vergil-containers#932, vergil-python#90, vergil-containers#939,
  docs#125). They landed in 16 PRs; 4 tasks were operational (3 deployments, 1
  validation).
- **Releases cut** (by the human): vergil-tooling v2.1.319 (2026-11-16, Tasks 1,
  2, 3a, 3b) and v2.1.320 (2026-11-17, the hash rule and its wording);
  vergil-actions v2.1.79 (2026-11-17, the two-job workflow).
- **Span:** epic opened 2026-11-16T12:40Z; the first PR merged 2026-11-16T18:57Z
  and the last 2026-11-17T14:34Z.

## 1. How the plan evolved

The ten planned tasks ran in the planned order, and every planned dependency
held. What changed was the path fence: merge review found it weaker than the
spec's design four times, and each fix grew the design. The plan's [Evolution
during execution](plan.md#evolution-during-execution) log has the dated entries;
the shape of it was:

- **Pairing within a diff hunk was not enough.** Task 3a's first merge review
  found that the spec's no-downgrade rule (pair removed and added versions
  within each hunk) could be bypassed. The first fix, pairing across the whole
  file, was still not enough for files that repeat keys per block, such as
  `runtime.toml`'s `[[series]]`. The second fix compares each file as a whole,
  per block, with an optional anchor group naming the block. The spec and plan
  were updated in place before P1 and C1 ran (.github#1216). That was done
  believing nothing had shipped, but Task 1 had merged 51 minutes earlier, so
  the plan's log records the edit after the fact.
- **The removal rule came from the same thread.** A fresh review then found that
  dropping a series in one run and re-adding it lower in the next passed the
  fence. The human ruled that an automated refresh never removes an existing
  anchored block or `[[package]]` entry, and that dropping a series is a human
  change. That review also found that deleting `upstream-version` from an
  existing entry had the same effect, and it was fixed in the same branch.
- **Hashes were unfenced against their versions.** C1's merge review found that
  vergil-containers' hook wrote new digests for an unchanged version and only
  warned a human to check. The hook now refuses that, and the human ruled the
  matching rule into the fence itself, since the hook is the untrusted half.
  That added four tasks: the hash rule (vergil-tooling#4539), its release
  (#4542), and the two consumer tasks that mark their hash lines
  (vergil-containers#932, vergil-python#90), plus a small wording fix (#4543).
- **Task 2 did not add retries where the plan said.** The plan listed
  `post_json_once` and `post_json_with_status` among callers that retry after a
  401. Both are single-shot by contract, since a retry could duplicate a create
  that already landed. Each takes the token source's current token, and no retry
  was added.
- **D2's cleanup came after the first nightly.** The plan said to close the
  legacy fixed-branch PRs before the next nightly run. The vergil-actions
  release landed at 01:40Z, and the first runs on the new workflow ran at 09:30Z
  and 09:42Z, before the cleanup at 12:21Z. Each repo briefly had two refresh
  PRs open; no harm came of it, since the new PRs merged themselves.
- **V1 accepted the renewal without seeing it live.** See §3.
- **The documentation review spawned two per-repo doc tasks**
  (vergil-containers#939, docs#125), and found stale security-model text older
  than the epic (rulesets were said to require an approving review;
  `required_approving_review_count` is 0), which self-merging PRs made plainly
  false.

## 2. Lessons learned

- **Merge review earned its cost on a security fence.** It found four real ways
  around the fence that the spec's design review and pushback rounds had passed,
  each a medium-severity way to merge a downgrade or a swapped asset unreviewed.
  A design that reasons about diff hunks is easy to get subtly wrong; the
  version that held compares whole before and after files.
- **A fix that takes the weaker of two options needs checking against the
  ruling.** On R1, the human approved the stricter rule, but the fix took the
  whole-file pairing option the finding itself had called insufficient, and the
  check caught it. A security reply carries only its decision word, so the
  direction lives in the private finding file; the run read it but chose
  differently. Checking the fix's approach against the finding's recommendation
  before the check would have saved a round.
- **Threat-model the hook as untrusted, everywhere.** The hook's own digest
  check fixes the honest case (a swapped upstream asset), but under the epic's
  own model the hook and what it loads are the untrusted half. Only a rule in
  `publish` closes the compromised-dependency case. That framing found the hash
  rule.
- **Live runs proved the boundary quickly.** The first nightly after the release
  exercised the whole self-merge path in both repos before V1 started. The key
  boundary (no token in `prepare`, no hook in `publish`) and the `develop`-only
  guard were both confirmed from logs.
- **Agent identities cannot do several operational steps.** Closing PRs,
  deleting branches, dispatching workflows and reading one job's log on a large
  run are all denied to the user identity, so D2 and V1 ran on human-run steps.
  That is by design, but it made validation slower, and the job-log gap left
  part of V1's evidence resting on timestamps (intake .github#1239).

## 3. Compromises & tradeoffs

- **The renewal is accepted as covered by tests, not seen live.**
  vergil-tooling's `test_wait_for_checks_poll_picks_up_renewal` runs the real
  `AppTokenSource` through a poll that crosses the renewal: the renewed token is
  used and the replaced one revoked. vergil-containers' CI now takes about 28
  minutes, under the 1800s threshold, so no live run has renewed. The human
  judged contriving a longer wait not worth the effort: the workflow runs in
  production as is, a long enough wait will come in normal operation (a GitHub
  slowdown is enough), and a renewal failure would show as a red refresh run. No
  tracking item was filed.
- **The renewal's order is judged from token timestamps.** The PR poller's
  output reaches the log only at step end, so poll lines carry the end-of-step
  time (intake .github#1235).
- **The vergil-containers key boundary rests on an earlier run.** Run
  44088128201's logs could not be downloaded by an agent; the boundary evidence
  for containers comes from run 44057969189.
- **A vergil-containers run red only outside the refresh jobs counts.** The
  2026-11-17 nightly failed on a lost hosted runner (`prod-ts-node:24`) and a
  Launchpad lookup in the lifecycle check; the failed-jobs re-run passed. The
  human ruled that V1 judges the refresh jobs.
- **One tool's changed checksums at the same version now blocks the whole
  vergil-containers refresh,** not just that tool. That fails safe, and the
  human accepted it with the hook fix.
- **Nightly runs were red from v2.1.319 until v2.1.79,** as the spec planned
  (§6.4). In practice the window closed before any nightly ran on the new tool.
- **The fence's MIT-0 allowlist entry** is a fleet-wide license-policy change
  made inside Task 1, because `cryptography` needs cffi 2.x on Python 3.14 and
  later. The human kept it.

## 4. New problems & opportunities

Filed or captured during the epic:

- **In the epic:** .github#1216, vergil-tooling#4539, #4542, #4543,
  vergil-containers#932, vergil-python#90, vergil-containers#939, docs#125 (all
  closed).
- **Intake:**
  - .github#1232: the `prod-ts-node:24` amd64 build was lost with its hosted
    runner (closed: the re-run passed).
  - .github#1233: one Launchpad timeout in the lifecycle check fails the whole
    Ops run.
  - .github#1234: the deployment template should say how to check a release tag
    contains a merge.
  - .github#1235: the PR poller's log lines appear only at step end.
  - .github#1239: a read-only `vrg-gh` command to fetch one job's log.
- **Security:** four findings, all fixed in the epic (counts only).

**The unaddressed-findings sweep** read 20 children, the epic issue, 16 merged
PRs and the planning issue's design-review rounds. Every `Awaiting a ruling`
item on the epic has a ruling, and every filed issue exists. No merged PR has a
review or comment. Untracked problems found:

- **D2's ruling list was left open.** The result comment on vergil-actions#1256
  listed the vergil-containers run failures under `Awaiting a ruling` with no
  ruling line. **Filed:** the ruling is now recorded there; the failures are
  .github#1232 and #1233.
- **Two out-of-scope items had no tracking issue.** **Filed:** moving
  `github-config` onto the token source (.github#1247), and moving
  `vrg-update-deps` into automation (.github#1248).
- **Moving the other App-token jobs onto the `automation` mode** (branch-sweep,
  dependabot-audit, epic-sweep, epic-rollup, cd, cd-release) was pointed at
  vergil-actions#1185, which covers only the develop-only credentials
  environment. **Filed:** .github#1249.
- **.github#1145 did not say its merge half was done.** **Found:** a status
  comment now records that the merge half shipped here and the release half
  remains.
- **Alignment round 1 on the planning issue has no public per-finding reasons.**
  **Found:** the round was handled as security material on purpose; the broken
  marker behind it is vergil-project/vergil-claude-plugin#1037.
- **Four spec and plan items from the documentation review** (Task 2's callers,
  D2's ordering, spec §6.1's deadline wording, spec §6.2's hash rule).
  **Found:** fixed in this retrospective's PR.

## 5. What's next

- **The language lifecycle job** (.github#1128), the next epic, reuses the token
  source.
- **Cutting a release automatically once a refresh merges** (the second half of
  .github#1145) depends on release rights for the automation identity and a
  per-repo release lock (.github#1088).
- **The tooling documentation from #4559** is not in a vergil-tooling release
  yet (v2.1.321 was cut before it merged).
