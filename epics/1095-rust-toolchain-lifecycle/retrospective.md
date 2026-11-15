# Rust toolchain on the leading edge — retrospective

- **Epic:** vergil-project/.github#1095
- **Spec:** [spec.md](spec.md) · **Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to give Rust the same lifecycle automation and toolchain
enforcement as Python. New Rust series should arrive, and old ones leave,
through reviewed lifecycle PRs that carry their toolchain pins. And a Rust repo
should be unable to pick its own toolchain behind Vergil's back. All of it
shipped:

- **Lifecycle PRs carry their pins.** Add and drop PRs now write and remove
  their toolchain pins.
- **The Rust image pins rustup** to its own release, with auto-install off.
- **`vrg-validate` covers Rust.** It warns on a stale `rust-version` floor and
  rejects `rust-toolchain.toml` and the legacy `rust-toolchain` file.
- **`vrg-container-run` never forwards rustup's selection variables.**
- **Rust allowed actions** drop `actions-rust-lang/*`.
- **CI asserts each Rust leg's `x.y`**, including three-digit minors.

Rust then moved to `automation: "pr"` and caught up from 1.93/1.92 to 1.99/1.98
through four of its own lifecycle PRs. The end-to-end check on the two Rust
fixtures passed.

| PR | Repo | What it did | Merged (UTC) |
|---|---|---|---|
| [#1108](https://github.com/vergil-project/.github/pull/1108) | .github | Spec and plan | 2026-11-13 |
| [#1180](https://github.com/vergil-project/vergil-actions/pull/1180) | vergil-actions | Task 7: `assert-toolchain` checks a Rust leg's `x.y` | 2026-11-13 |
| [#863](https://github.com/vergil-project/vergil-containers/pull/863) | vergil-containers | Task 2a: the matrix and pin catalog render from pins passed in | 2026-11-13 |
| [#4352](https://github.com/vergil-project/vergil-tooling/pull/4352) | vergil-tooling | Task 4: Rust floor and toolchain-file checks; the Rust standards page | 2026-11-13 |
| [#864](https://github.com/vergil-project/vergil-containers/pull/864) | vergil-containers | Task 3: `RUSTUP_TOOLCHAIN` set to the image's release, auto-install off | 2026-11-13 |
| [#4353](https://github.com/vergil-project/vergil-tooling/pull/4353) | vergil-tooling | Task 5: rustup's selection variables are never forwarded into a container | 2026-11-13 |
| [#4355](https://github.com/vergil-project/vergil-tooling/pull/4355) | vergil-tooling | Task 6: Rust allowed actions drop `actions-rust-lang/*` | 2026-11-13 |
| [#868](https://github.com/vergil-project/vergil-containers/pull/868) | vergil-containers | Task 2b: lifecycle add and drop PRs write and remove their pins | 2026-11-13 |
| [#874](https://github.com/vergil-project/vergil-containers/pull/874) | vergil-containers | Task 9: Rust moves to `automation: "pr"` | 2026-11-14 |
| [#887](https://github.com/vergil-project/vergil-containers/pull/887) | vergil-containers | Task 11: the Rust policy page describes versions under `pr` automation | 2026-11-14 |
| [#4417](https://github.com/vergil-project/vergil-tooling/pull/4417) | vergil-tooling | Documentation review: Rust enforcement in the validation, container and image docs | 2026-11-15 |
| [#113](https://github.com/vergil-project/docs/pull/113) | docs | Doc task: `ci.md` names Rust as a `pr` family and adds Rust toolchain selection | 2026-11-15 |
| [#895](https://github.com/vergil-project/vergil-containers/pull/895) | vergil-containers | Doc task: the Rust policy page's dated window facts | 2026-11-15 |
| [#1232](https://github.com/vergil-project/vergil-actions/pull/1232) | vergil-actions | Doc task: the actions index and the assert page's intro cover Rust | 2026-11-15 |
| [#1150](https://github.com/vergil-project/.github/pull/1150) | .github | The plan's evolution log (Task 12 acceptance wording) | 2026-11-15 |

The Rust catch-up was done by the lifecycle job's own PRs, which are not epic
tasks: vergil-containers
[#875](https://github.com/vergil-project/vergil-containers/pull/875) (add 1.99),
[#876](https://github.com/vergil-project/vergil-containers/pull/876) (add 1.98),
[#877](https://github.com/vergil-project/vergil-containers/pull/877) (drop 1.93)
and [#878](https://github.com/vergil-project/vergil-containers/pull/878)
(drop 1.92). All four merged on 2026-11-14.

- **Repos touched:** 5 (.github, vergil-containers, vergil-tooling,
  vergil-actions, docs).
- **Tasks:** 18, plus this retrospective:
  - 9 code tasks;
  - 2 deployments (Tasks 8 and 10) and 1 validation (Task 12);
  - the spec-and-plan task;
  - the documentation review and the 3 per-repo doc tasks it spawned;
  - 1 plan-evolution task.

  That is 15 PRs, or 16 counting this one.
- **Releases cut**, all by the human:
  - vergil-tooling v2.1.309 (2026-11-13), carrying Tasks 4, 5 and 6;
  - vergil-actions v2.1.70 (2026-11-13), carrying Task 7;
  - vergil-containers v2.1.40 (2026-11-14), carrying Tasks 2a, 2b and 3;
  - vergil-containers v2.1.41 (2026-11-14), carrying the caught-up window and
    the first `prod-rust:1.99`/`1.98` images.
- **Span:** the epic opened 2026-11-13T18:21Z, and the last task closed
  2026-11-15T14:32Z, about 44 hours later.

## 1. How the plan evolved

The task list held. Every planned task shipped in the planned order, and no
task was added, dropped or re-sequenced. What moved was timing and acceptance
wording.

- **The fixture dependency only gated the end.** When the epic started, the
  fixture repos (epic vergil-project/.github#1091) did not exist yet. The
  plan's choice to make only Task 12 depend on them paid off: Tasks 2a to 11
  and the documentation review all landed while 1091 was still in flight.
  Task 12 waited for the fixtures' CI and CD to go green. One run was stopped
  part-way when the human reported a Cargo packaging fix still landing in
  1091. It had posted nothing, and a fresh run redid every step later.
- **Task 12's acceptance wording didn't match how the system works**, in two
  places. Both were settled by human rulings and logged in the plan's
  evolution section (#1147):
  - **Rust app CI legs:** an application's CI runs only the newest series, by
    design, so "CI on both window series" applies to libraries only.
  - **Prod image digests:** prod tags are rebuilt nightly from the release
    commit, so a current digest rebuilt from that commit counts as a match.
- **The catch-up took four rounds, not one.** The plan expected the remaining
  PRs to rebase after each merge. In practice, each merge needed its `develop`
  CD run to finish, then a full Ops run to regenerate the remaining branches,
  then green CI. After #877 merged, #878 conflicted with `develop`, because two
  changes to one family edit adjacent lines in the same generated files.
  GitHub's "Update branch" can't resolve that, but the lifecycle job's
  regeneration did.
- **The documentation review spawned three per-repo doc tasks**
  (vergil-containers#894, vergil-actions#1231, docs#112), as the bookend
  convention expects. It also confirmed two reviewer notes from earlier tasks
  and fixed them: the `floors.py` docstring, and the stale dates on `rust.md`.

## 2. Lessons learned

- **"One PR per change" is not independence.** Lifecycle add and drop PRs for
  one family touch the same generated lines, so they conflict once merged in
  sequence, and each merge forces a CD run plus a full regenerate. Next time:
  one PR per family per run, merged in order by the job itself (#1128).
- **Write acceptance against what the code resolves.** Both Task 12 rulings
  came from wording that assumed behaviour the tooling doesn't have:
  per-repository-type series resolution, and movable prod tags. Reading
  `resolve.py` and the nightly rebuild while writing the plan would have caught
  both.
- **A scheduled job is not a scheduled time on GitHub.** Every scheduled
  workflow in the fleet fired 7 to 9 hours late. That moved the "overnight"
  jobs into the working day. There they collided with merges through a shared
  `cancel-in-progress` concurrency group and cancelled publishes without
  failing (vergil-containers#879). Unattended work needs a schedule we control
  (#1131) and a check that catches a late or missed start.
- **Retention of one release is a race.** With `keep = 1`, every vergil-tooling
  release deleted the previous `.deb` while CDN-cached indexes still named it,
  and CI across the fleet 404'd until the cache expired. Keeping two
  (packages#54) closed it, but the size cap behind it is now a constraint
  (packages#55, .github#466).
- **Parallel dispatch works when the plan's dependencies are honest.** Six
  tasks across three repos ran at once and landed in one hand-off per repo.
  The one miss was a batching error by the orchestrator: it batched a
  security-labelled task with another, which the batching rules forbid. The
  run caught it before doing any work, and the two were sent separately.
- **The leading edge is only as fresh as its source.** Rust 1.100 was on the
  stable channel on 2026-11-12, but endoflife.date still didn't list it on
  2026-11-15, so the window couldn't move.

## 3. Compromises and tradeoffs

- **Fourteen krb5 CVEs suppressed in `dev-pkg-rhel:9`** (vergil-containers#866).
  The fixed `krb5-libs` wasn't in the UBI 9 repos, and the Trivy gate was
  blocking CD. Twelve are in server code the image never runs. Two
  (CVE-2026-119079 and CVE-2026-71199) are in libkrb5 itself, the library curl
  and git load, and the human ruled to suppress them too rather than wait on
  Red Hat. The whole group comes out when UBI 9 ships `1.21.1-10.el9_9.2`. That
  removal is a manual step nobody is reminded of.
- **`keep = 2` trades space for safety** (packages#54). It fits today (about
  482 MB of the 900 MB limit), but a second vergil-python minor line would push
  the site past the limit on that line's second release. It is tracked in
  packages#55, and the human raised the priority of .github#466 (new package
  hosting).
- **Review Focus items 3 and 5 were not exercised live.** Item 3 is a
  workspace root that inherits its `rust-version` floor. Item 5 is a
  `rustc 1.100.0` leg checked against `1.100`. Neither fixture is a workspace,
  and the window had no 1.100 series yet. Both are covered by unit tests
  (Tasks 4 and 7), and the first 1.100 lifecycle PRs will exercise item 5 for
  real.
- **amd64 image checks were read from CI, not run.** The orchestrating host is
  arm64-only. Tasks 8 and 10 ran the rustup pin checks locally on arm64 and
  took amd64 from each published digest's CI `checks.sh` output.
- **Catch-up was manual.** The human merged four PRs one at a time, with a
  workflow dispatch between each, over about seven hours. The automation that
  would remove this (#1128) is deliberately out of this epic's scope.
- **Two merge reviews ended `not-eligible (human-ruling)`.** vergil-containers#866
  and packages#54 both shipped on human rulings that overrode the reviewer,
  and both rulings are recorded.

## 4. New problems and opportunities

| What surfaced | Where it went |
|---|---|
| `dev-pkg-rhel:9` Trivy gate blocked on krb5 CVEs not yet fixable in UBI 9 | vergil-containers#866 (fixed) |
| `vrg-release` interrupted before its progress block can't be resumed or restarted through the wrappers | vergil-tooling#4367 (intake) |
| Publish runs cancel each other through the `docker-publish-<prefix>` concurrency group | vergil-containers#879 (ad hoc) |
| GitHub schedule events fire 7 to 9 hours late | .github#1131 (planning); crons moved to 01:xx UTC fleet-wide (vergil-tooling#4380 and siblings); docs example .github#1134 (intake) |
| Lifecycle PRs: merge automatically, in order, one PR per family | .github#1128 (intake; input to #1097) |
| `vrg-repo-status` can't see PRs on `origin` with no local worktree | .github#1132 (intake) |
| Lifecycle and pin fetchers didn't decode gzip (cpp-clang check failed on apt.llvm.org) | vergil-containers#883 (fixed) |
| Package pool deleted the previous release while cached indexes still named it | packages#54 (fixed); vergil-python retention packages#55 (open); package hosting .github#466 (raised priority) |
| Upstream-refresh PRs are never merged or released (vergil-python ran six days behind on CPython) | .github#1145 (intake); vergil-python v1.0.5 released |
| The Rust window lags upstream through endoflife.date | comment on .github#1097 |
| rust-lib fixture floor 1.92 below the window | vergil-fixture-rust-lib#15 (fixed) |
| The disk on the agent host filled (95%) during the doc review | old Rust images removed by hand; logged, not acted on |

**Unaddressed-findings sweep.** All 18 tasks, the epic issue and the planning
issue (#410) were swept against the trusted-author records. No merged PR had
reviews or conversation comments. The results:

- **Found** (already tracked): GHSA-xwj6-fh27-fmjh, the spec round's security
  follow-up, is .github#1103. The fixture-run gaps posted to #410 (the ruff and
  mypy names in the evidence bundle, and the missing Cargo updater) are in Epic
  B's scope, .github#1087. The review notes on vergil-containers#862 were
  fixed by #894.
- **#1105's `cpp-clang` gzip item** has no `Human ruling:` line on that issue.
  It was filed as vergil-containers#883 before the comment was posted, and is
  fixed.
- **F4 on the epic** (scheduled Ops runs failing with no alert) was ruled
  "drop: already tracked elsewhere" with no ref. **Dropped by the human:**
  they track it themselves, outside the issue tracker, and chose not to file
  it. It remains an open problem.
- **Dropped:** four planning review rounds give discard counts
  without per-item reasons. Two minor wording points reviewers noted without
  raising them, on docs#113 and vergil-actions#1232, are below the bar and
  still accurate.

## 5. What's next

- **Follow-on brainstorm, .github#1097 (now in planning):** pick up Epic B
  (.github#1087, Rust validation, supply-chain and dependency-update parity).
  Its inputs from this epic are #1128 and taking the Rust window from Rust's
  own release channel.
- **.github#1131 (planning):** own the fleet's job scheduling.
- **.github#466:** package hosting off GitHub Pages, now a higher priority.
- **The 1.100 move:** once endoflife.date lists 1.100, the lifecycle job opens
  "add rust 1.100" and "drop rust 1.98". That is the first live test of the
  three-digit-minor handling.

## Appendix A — Operational notes

- **Release order for Task 8:** vergil-tooling and vergil-actions first, then
  vergil-containers once Tasks 2b and 3 had merged. Prod images build only from
  vergil-containers `main`, so the containers release is what puts the pin in
  front of consumers.
- **Catch-up loop:** merge one lifecycle PR, then wait for its `develop` CD run
  to finish. Then run `gh workflow run ops.yml` (raw `gh`, because `vrg-gh`
  refuses `workflow run`). Wait for the lifecycle job and the PRs' CI, then
  merge the next PR. Never merge while an Ops run's nightly rebuild is
  publishing: a CD run would cancel it (#879).
- **Recovering an interrupted `vrg-release`:**
  1. Close the stale tracking issue.
  2. Remove the release worktree.
  3. Delete the local `release/X.Y.Z` branch with raw `git`, since `vrg-git`
     refuses to.
  4. Run the release again (#4367).
- **Package releases and retention:** with `keep = 2` live, release
  vergil-python and packages one at a time. Let each `publish-index` run finish
  before the next.
