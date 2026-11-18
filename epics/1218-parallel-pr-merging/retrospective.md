# Parallel PR merging — retrospective

- **Epic:** vergil-project/.github#1218
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md) (with its `##
  Evolution during execution` log)
- **Planning issue:** vergil-project/.github#1213
- **Span:** epic opened 2026-11-16T21:33Z; last child closed 2026-11-18T18:57Z
  (about 45 hours). Planning ran from 2026-11-16T20:12Z to 2026-11-17T15:42Z.
  The first PR merged at 2026-11-17T15:14Z and the last at 2026-11-18T18:47Z.

## 0. At a glance

**What we set out to do.** Let a batch of ready PRs merge into `develop` in
parallel instead of one after another. Develop drops its up-to-date rule and
`main` keeps it. `vrg-submit-pr` opens every PR with auto-merge on and no
rebase. `vrg-finalize-pr` watches the batch, then syncs, cleans up and validates
`develop` once, under a lock so that concurrent runs cooperate. The release
gets its own develop-CD gate and installs by default. The epic also folded in
vergil-project/.github#1184: finalize by default on submit, install by default
on release.

**What shipped.** All of it, in three releases, and then a round of fixes found
by using it. The live validation (V1) passed on its second pass: five PRs, then
three, merged in parallel with no `BEHIND` update, and two finalize runs settled
side by side, one waiting 19.3s on the other's lock. Its first pass failed on a
real defect: the watch counted a cancelled check from a superseded CI run as a
failure. That was fixed (#4692) and released before the re-run. All six language
fixtures went through the full cycle on the released tooling. The two
TypeScript fixtures did so only after a new time-boxed npm audit exception
mechanism (#4682) let them ship past an advisory with no fix.

**By the numbers** (queried from GitHub):

- **Children:** 74, that is 73 closed plus this retrospective. 47 were
  planned and 27 added during execution: 15 follow-up tasks, 2
  documentation-sweep tasks, and 10 small `.github` doc tasks used as V1's
  test PRs.
- **PRs merged:** 57, across 11 repos, +20,084 / −7,632 lines. vergil-tooling
  took 26 of them and +16,591 / −7,368.
- **Repos:** .github, docs, vergil-tooling, vergil-actions,
  vergil-claude-plugin, and the six language fixtures.
- **Releases:**
  - vergil-tooling v2.1.322, v2.1.323, v2.1.324, v2.1.325;
  - vergil-claude-plugin v2.1.75, v2.1.76, v2.1.77, v2.1.78;
  - vergil-actions v2.1.80, v2.1.81;
  - fixtures: python-app and python-lib v0.1.1; rust-app, rust-lib, ts-app
    and ts-lib v0.1.2.
- **Merge reviews:** 58 runs on 56 tasks. 51 ended `eligible` and 7
  `not-eligible` by a human ruling. They found 7 defects (all fixed), 4
  design questions and 3 out-of-scope items.

**Work delivered** (task: PR, with the issue in brackets where it differs —
what it did):

- **Planned, vergil-tooling:**
  - 2: #4586 — `lib/cd_runs.py`; a release requires a green develop CD run.
  - 3: #4623 — `vrg-release` installs by default; `--no-install` skips it.
  - 4: #4591 — `CI gates` split into develop and main rulesets; develop
    drops up-to-date.
  - 5: #4625 — the config audit clears classic protection's `strict` flag
    on develop.
  - 7: #4626 — `confirm-main` re-dispatches a main CD run superseded before
    it started.
  - 8: #4641 — `lib/checkout_lock.py`; cleanup and update-deps take the lock.
  - 9: #4650 — release preflight fast-forwards develop under the lock
    instead of refusing.
  - 10: #4595 — `lib/auto_merge.py`, shared by submit and mechanized PRs.
  - 11: #4601 — provenance counts an agent's auto-merge events as
    violations.
  - 18: #4642 — `vrg-pr-workflow batch-check` for a submitted batch PR.
  - 12c: #4651 — finalize reports develop CD instead of waiting; `--wait-cd`
    opts in.
  - 12a: #4627 — `lib/finalize_watch.py`, the batch watch phase.
  - 12b: #4660 — finalize watches a batch, then settles develop once under
    the lock.
  - 13a: #4643 — submit opens every PR through one path, without a rebase,
    with auto-merge on.
  - 13b: #4666 — submit finalizes by default; `--no-finalize` opts out.
  - 17: #4667 — the governance and batch-pipeline specs superseded; the
    permission model updated.
- **Planned, vergil-actions:** 6: #1277 — its own CD concurrency group is
  keyed by branch. 20: #1289 — the ruleset docs describe the develop and main
  rulesets.
- **Planned, vergil-claude-plugin:**
  - 14: #1127 — merge-review reviews a fix to an open PR as a whole branch.
  - 15: #1143 — pr-watch watches PRs that auto-merge.
  - 16: #1126 — epic-create seeds a CD-build validation for epics that need
    fresh dev images.
  - 19: #1145 — skills and the agent-merge hook describe submit-time
    auto-merge.
- **Planned, .github:** 1: #1257 — the spec and plan. D1–D3 (deployments
  #1252, #1253, #1254) — Releases 1 and 3 attested and installed, Release 2
  applied to all 15 repos. V1 (#1255) — the live parallel batch, in two
  passes; its test PRs (#1321–#1325, #1327, #1330, #1342–#1344) fixed small
  org-doc gaps.
- **Fixture runs:** each fixture's change PR (python-app#19, python-lib#21,
  rust-app#34, rust-lib#35, ts-app#22, ts-lib#21); the release and
  validation tasks have no PR.
- **Added during the build:**
  - vergil-tooling#4602 (#4584) — the merge-review request carries every
    batch issue's scope.
  - vergil-claude-plugin#1135 (#1131) — merge-review judges each batch
    commit against its own issue's scope.
  - vergil-tooling#4655 (#4585) — the release gate's ancestor-run refusal no
    longer reports a 0s wait.
  - vergil-tooling#4654 (#4648) — `vrg-pr-await` waits for the merge: exit 0
    merged, exit 1 cannot merge.
  - vergil-claude-plugin#1146 (#1142) — pr-watch uses that contract.
  - vergil-claude-plugin#1148 (#1147) — pr-watch waits on `vrg-pr-await` in
    the background.
  - vergil-tooling#4665 (#4662) — `--allow-provenance-violation` never
    releases past a merged violation.
- **Added after Release 3:**
  - vergil-tooling#4680 (#4677) — finalize prints a status table on each
    poll.
  - vergil-tooling#4681 (#4678) — `vrg-repo-status` shows a submitted PR
    that cannot merge, with its reason.
  - vergil-tooling#4685 (#4682) — time-boxed exceptions in the TypeScript
    npm audit.
  - vergil-tooling#4693 (#4692) — a check cancelled by a superseding CI run
    is not a failure.
  - vergil-claude-plugin#1161 (#1159) — pr-watch handles `check_cancelled`
    and `failed: cancelled`.
  - vergil-tooling#4698 (#4694) — the watch is a live progress stage that
    collapses when done.
- **Evolution log:** .github#1265 (#1264) and #1307 (#1304) — the plan's
  three evolution-log entries.
- **Documentation review:** vergil-tooling#4673 (#4529),
  vergil-claude-plugin#1154 (#1152) and docs#134 (#132) — the sweep and its
  plugin and `docs` tasks.

## 1. How the plan evolved

The plan's design held: every one of the 22 planned code tasks shipped in its
planned shape, and the delivery table never changed. What changed was around it:
seven follow-up tasks during the build, a release grouping that did not match
the plan, and six more tasks once the human used the released tooling. The
evolution log has three entries; this section draws them together.

- **The new flow broke assumptions in tools the plan did not touch.** The
  plan rebuilt submit and finalize, but other tools still assumed the old
  model. `vrg-pr-await` treated a PR merging during a watch as an error,
  because under the old flow it meant the audit cycle had been bypassed. Under
  auto-merge it is how every PR lands. pr-watch's first rewrite (Task 15)
  worked around it by re-reading the PR's state on exit 1. The human ruled
  that no tool may lean on an exit code in a roundabout way, and the contract
  was fixed at its source (#4648, then #1142, F5). That fix made the await
  wait up to two hours, which ran into Claude Code's 600s cap on a foreground
  command. So pr-watch moved the wait to the background (#1147, F6).
- **Batch-PR merge review needed per-issue scope.** Task 14's own merge review
  found that a whole-branch review of a fix to a batch PR judged every sibling
  issue's commits as out of scope, so it could never end `eligible`. That took
  two tasks, one in tooling (#4584) and one in the plugin (#1131), and both
  became blockers of Task 15.
- **The spec left one override ambiguous.** Spec §6.3 did not say whether
  `--allow-provenance-violation` covered a PR that had already merged with a
  violation. Task 12b let it cover one, which allowed a chained release past
  that violation. #4662 narrowed it (F7). After the epic's code had shipped,
  the human went further and removed the override and the advisory category
  altogether, in the ad hoc vergil-tooling#4695 (released in v2.1.325). So the
  spec's scope line "`--allow-provenance-violation` stays" no longer describes
  the tooling.
- **The plan's release grouping did not hold, without harm.** Spec §6.12 has
  Release 1 shipping on its own, ahead of everything else, and Release 3
  shipping as one release. In practice Releases 1 and 2 both shipped in
  vergil-tooling v2.1.322. That release also carried four Release 3 pieces
  that had already merged (#4568, #4569, #4572, #4584), and vergil-claude-plugin
  v2.1.75 carried three more (#1120, #1122, #1131). None was in use yet, or
  each changed only mechanized PRs, agent provenance or skill text, so nothing
  regressed. The one grouping that mattered did hold: Tasks 13a and 13b shipped
  together in v2.1.323, by release discipline rather than interim code (the
  human's `keep` ruling on #4574).
- **Release 3 was cut before the documentation sweep** (the human's decision).
  Docs change no code, so the sweep's fixes shipped in v2.1.324 and plugin
  v2.1.77.
- **Real use produced a tail of tasks.** After the first real batch on v2.1.323,
  the human asked for visibility: a status table per poll (#4677), a submitted
  PR that cannot merge shown in `vrg-repo-status` (#4678), and then a live,
  collapsing progress stage (#4694). V1's first pass found the
  cancelled-check defect (#4692), with its pr-watch counterpart (#1159). The
  TypeScript fixtures needed the npm audit exception mechanism (#4682). All six
  are under this epic, and all shipped in v2.1.324, v2.1.325 or plugin v2.1.78.
- **Small changes:** a V1 step was added to confirm, against GitHub, the
  undocumented refusal text that `lib/auto_merge.py` relies on (F1). It
  matched. Task 11's real auto-merge event names were recorded by the human on
  a throwaway PR, because the agent cannot read them. Task 12c's CD report says
  `CD: no CD workflow in this repo` rather than warning `unknown` (R1 on #4571).

## 2. Lessons learned

- **A lifecycle change must trace every tool that encodes the old lifecycle.**
  The plan listed the commands it rebuilt, and the callers that passed removed
  flags. It did not list tools whose *meaning* depended on the old flow:
  `vrg-pr-await`'s "merged means bypassed" and pr-watch's foreground wait.
  Next time, the spec should have a section that asks, for each state the
  change makes normal (here, "a PR merged while we watched"), which tools treat
  that state as an error.
- **Harness limits are design inputs.** The 600s foreground cap is a fact about
  the agent runtime, not the tooling. It surfaced only when a wait became
  long. A spec that changes how long an agent blocks should state the limit it
  must fit under.
- **Using the tool found what tests and validation did not.** The first real
  batch was the first time anyone *watched* a batch merge, and the missing
  feedback was obvious at once (#4677, #4678, #4694). V1's first pass found a
  defect no unit test had (#4692): two CI runs on the same head, one cancelled.
  The validation earned its cost by failing. Keep a live validation that runs
  the real flow with real pushes for any epic that changes the PR flow.
- **Check undocumented external behaviour live, early.** The direct-merge
  path rests on GitHub refusal text that GitHub does not document. Adding a
  V1 step to confirm it cost one PR and turned an assumption into evidence.
  It also showed that `gh pr merge --auto` on a clean PR merges at once.
- **Release discipline can replace interim code, when it is written down.**
  The 13a and 13b gap was closed by a stated rule (no release between them),
  recorded in the plan's evolution log, and it held. The broader "Release 3
  ships as one release" rule was not written as an enforceable rule, and it
  did not hold. A release grouping the plan cares about has to be stated as a
  constraint on cutting releases, not as a description of them.
- **An override is a design question, not a convenience.** The
  `--allow-provenance-violation` override was ambiguous in the spec, narrowed
  in #4662 and then removed in #4695. Next time, the spec should state each
  escape hatch's exact reach, or ask whether it is needed at all.
- **Recording rulings as they happen kept the sweep clean.** Every follow-up
  F1–F7, every merge-review ruling and both V1 passes were recorded on GitHub
  when they happened, with links. This retrospective's sweep found nothing
  left unruled.

## 3. Compromises and tradeoffs

- **The combined-change check moved after the merge.** By design (spec §2), a
  semantic break between two individually green PRs is caught by finalize's
  `vrg-validate` on the synced `develop`, or at the release, rather than
  before the merge. `develop` can be briefly broken. Finalize reports that
  with its own exit status, but does not say which batch broke it (a spec
  non-goal).
- **Release 3 did not ship as one release.** It shipped as v2.1.322's early
  pieces plus v2.1.323 (and plugin v2.1.75 plus v2.1.76). It was harmless, as
  §1 explains, but the plan's guarantee was not kept.
- **The TypeScript fixtures shipped past a known advisory.** A high npm
  advisory with no patched release (in a dev-only transitive dependency) is
  acknowledged with a time-boxed exception, on the human's ruling: the same
  rule as a trivy CVE with no fix. Removing it is tracked in
  vergil-fixture-ts-app#20 and vergil-fixture-ts-lib#20.
- **Some evidence rests on the human's word.** Release 2's fleet apply and the
  ruleset read-back (#1253), and the classic-protection half of the audit, ran
  under the human identity, because the agent cannot read admin-scoped
  settings. The release attestations in D1–D3 were the human's. V1's second
  pass could not read the host's installed tooling version, because no log
  prints it (§4, U2, filed as .github#1346).
- **V1's second pass did not exercise the same-head case live.** Its cancelled
  run was on the previous head, so it may never have shown the cancelled check
  to the watch. The same-head sequence from pass 1 is pinned by a replay test
  in #4693, not by a live run. The lock wait was seen live only in pass 1.
- **The permission-model spec records an accepted gap.** The provenance checks
  cover the windows submit and finalize see, not every moment. Spec §6.9
  accepts that, to be revisited if an agent identity is given server-side
  rights on PRs.
- **Epic #1168's spec was left saying develop requires up-to-date checks.**
  By the spec's own decision (§6.9), it is noted here instead of edited: the
  statement in `epics/1168-upstream-refresh-merge/spec.md` (line 199) is
  superseded by this epic's ruleset split.
- **Seven merge reviews ended `not-eligible` by a human ruling,** each with a
  reasoned `keep`, `file`, `change` or `fix` ruling recorded on its task. That
  is the process working, but it does mean seven PRs carried a ruling rather
  than a clean review.

## 4. New problems and opportunities

### Routed during the epic

- **Pre-existing markdownlint errors in vergil-actions `docs/security/`,**
  outside the lint scope (F4): vergil-actions#1287 (open).
- **Work could start from a stale local develop** now that automated jobs
  advance `origin/develop`: .github#1240 (intake, open), filed during
  planning.
- **The provenance advisory category and `--allow-provenance-violation`** are
  not worth keeping: vergil-tooling#4695, done and released in v2.1.325.
- **The picomatch audit exceptions on the TypeScript fixtures:**
  vergil-fixture-ts-app#20 and vergil-fixture-ts-lib#20 (open).
- **The paad delegate closed its security section with a second start
  marker,** twice during planning: already tracked as
  vergil-claude-plugin#1037 (open).
- **Epic #1168's spec still says develop requires up-to-date checks:**
  recorded in §3, by spec §6.9's decision; not edited.

### The unaddressed-findings sweep

Swept: all 73 closed children, the epic issue (17 comments) and the planning
issue #1213 (22 comments). The children hold 238 comments; every comment in
scope is by a trusted author. All `Awaiting a ruling` items have trusted rulings:
F1–F7 on the epic, N0.1 on #1252, #4584, ts-app#10 and ts-lib#10, and every
merge-review R item. Every issue a ruling filed exists. The two `Filed without a
ruling` lists, in V1's pass 1, name existing issues (#4692, #1159). No PR in
scope has any review or PR comment, so there are no inline comments to check.
The planning issue's five design-review rounds record a reason for every
finding not accepted (one rejected, two dropped as moot, the rest kept as
decided).

Untracked problems found, and how the human ruled on each (2026-11-18):

- **U1** (source: #1253's result comment). Release 2's fleet apply also
  removed extra `astral-sh/*` and `pypa/*` entries from
  `actions_permissions.patterns_allowed` in vergil-fixture-python-app,
  vergil-fixture-python-lib and vergil-tooling. The run noted it as "outside
  this task". **Dropped:** the apply converged the repos on the managed
  config, as it should. No workflow in those repos uses an `astral-sh/` or
  `pypa/` action, and their CI has been green since.
- **U2** (source: #1255, pass 2). Neither `vrg-submit-pr`'s nor
  `vrg-finalize-pr`'s log names the installed tooling version, so V1's second
  pass could not tie the host's run to v2.1.325. **Filed:**
  vergil-project/.github#1346 (intake): print the tooling version in both
  logs, so a validation can cite it.

Security-class items, as counts: 7 design-review security records were created
during planning and retargeted to their tasks when the tasks were filed. One
finding raised during execution was ruled on the epic (0 fix, 0 record, 1
reject). Merge reviews raised none. None is untracked.

## 5. What's next

- **Open follow-ups from this epic:** vergil-actions#1287 (the lint errors and
  scope), .github#1240 (never start from a stale develop), and the two
  exception-removal tasks, vergil-fixture-ts-app#20 and
  vergil-fixture-ts-lib#20. They wait on a patched picomatch release. The
  sweep added .github#1346 (the tooling version in the submit and finalize
  logs).
- **#1184** was folded in and closed. Its two defaults shipped in Release 1
  (install by default) and Release 3 (finalize by default).
- No follow-on epic was planned from this one.

## Appendix A — Operational notes

The rollout order that worked:

1. **Release the standalone release changes first** (the develop-CD gate and
   install by default). In practice they shipped with Release 2 in v2.1.322.
2. **Release the ruleset split and apply it fleet-wide** as the human identity,
   with `vrg-github-repo-config update --repo vergil-project/<r>` for every repo
   (15 here). Before the apply, each repo shows the expected drift (the plain
   `CI gates` ruleset present, `CI gates (develop)` missing). Afterwards the
   agent's `audit` reports no ruleset drift. It still exits 2, because it cannot
   check the 10 admin-scoped settings. The docs-only repos (`.github`, `docs`)
   get `CI gates (develop)` only.
3. **Release the submit and finalize rework as one tooling release,** with the
   plugin release that teaches pr-watch and merge-review the new flow. Never cut
   a tooling release between submit's auto-merge change (13a) and its
   finalize-by-default change (13b). Before Release 2 is live, an auto-merged PR
   waits forever at `BEHIND`.
4. **Run the live batch** in a repo with no CD and fast CI. `.github` worked
   well, with small real doc tasks as the PRs. To push to an open PR during the
   watch, wait for submit to turn auto-merge on first, or the push races it
   (seen in pass 2).
5. **For the direct-merge check,** open a PR without auto-merge and wait until
   it reads `CLEAN` twice, with every required check reported, before calling
   `enablePullRequestAutoMerge`. A script that runs before CI registers its
   checks sees `UNKNOWN`, and auto-merge turns on normally (seen on #1327).
