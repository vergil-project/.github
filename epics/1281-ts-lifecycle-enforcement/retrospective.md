# TypeScript lifecycle PRs and toolchain enforcement — retrospective

- **Epic:** vergil-project/.github#1281
- **Planning issue:** vergil-project/.github#1208
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Span:** planning opened 2026-11-16T19:05Z; epic opened
  2026-11-17T21:16Z; the last task before this retrospective closed on
  2026-11-18. The first PR merged at 2026-11-17T23:18Z and the last at
  2026-11-18T17:17Z.

## 0. At a glance

We set out to make Vergil own the Node runtime of a TypeScript repo, as it owns
Python's. New Node majors would arrive through reviewed lifecycle PRs, a repo
could not declare a Node or package-manager version of its own, and every CI
leg would check that it ran the Node major CI chose. All of it shipped:

- `ts-node` is on `pr` lifecycle automation, and the window is 26/24/22.
- `vrg-validate` reports TypeScript toolchain settings as errors and an
  out-of-window `engines.node` range as a warning.
- `vrg-container-run` withholds Node and npm host settings, with a notice.
- `assert-toolchain` checks each TypeScript leg's Node major.

Fixture runs on both TypeScript fixtures validated it end to end against the
released versions. One thing did not happen as planned: Node 26 reached the
window through a hand-made PR the day the plan was approved, not through the
first lifecycle add PR (§1).

**Work delivered** (PR — what it did; all merged):

- **Planned tasks:**
  - Task 1, .github #1303 — the spec and plan.
  - Task 2, vergil-containers #956 — `ts-node` moves to `pr` lifecycle
    automation.
  - Task 4, vergil-containers #957 — the ts-node policy page's window facts
    under `pr` automation.
  - Task 6, vergil-tooling #4674 — TypeScript toolchain-setting errors in
    `vrg-validate`.
  - Task 7, vergil-tooling #4679 — the two-sided `engines.node` warning,
    with a hand-written npm range reader.
  - Task 8, vergil-tooling #4675 — `NODE_OPTIONS` and `npm_config_*`,
    `COREPACK_*` and `VOLTA_*` variables withheld from containers, with a
    notice.
  - Task 9, vergil-actions #1304 — the TypeScript branch of
    `assert-toolchain`.
- **Outside the plan:** vergil-containers #949 — ts-node 26 added by hand,
  before Task 2 (§1).
- **Plan correction:** .github #1308 — the hand-made Node 26 add recorded in
  the plan and spec.
- **Documentation review and its doc tasks:**
  - vergil-tooling #4683 — 26/24/22 wherever the docs list the Node majors.
  - vergil-containers #962 — the `core.node` pin exception for drop PRs.
  - vergil-actions #1306 — the smoke workflow comment names all four
    asserted languages.
  - docs #135 — TypeScript lifecycle, toolchain selection and the window on
    the org docs site.
  - .github #1328 — the spec's wording, and the ruled plan changes in the
    evolution log.
- **Fixture-run changes:** vergil-fixture-ts-lib #28 (`truncateWords`) and
  vergil-fixture-ts-app #29 (the `upper` subcommand).

**Counts:**

- **Repos touched:** 7: .github, vergil-containers, vergil-tooling,
  vergil-actions, docs and the two TypeScript fixtures.
- **Tasks:** 24 children of the epic, of which 23 closed before this
  retrospective. They are 9 planned tasks, 3 bookends, 1 plan-correction task,
  4 doc tasks the documentation review spawned and 6 fixture-run tasks.
- **PRs:** 15 in the epic's tasks, plus vergil-containers#949 outside them.

**Releases cut:**

- vergil-containers v2.1.46
- vergil-tooling v2.1.324
- vergil-actions v2.1.83
- vergil-fixture-ts-lib v0.1.3 and vergil-fixture-ts-app v0.1.3

The plan's vergil-tooling release hold ran from the first Task 6 and Task 8
merges (12:41Z) to v2.1.324 (14:50Z) on 2026-11-18.

## 1. How the plan evolved

The plan held for both tracks' code work. Tasks 2, 6, 7, 8 and 9 shipped as
written, in parallel on the first day, with one plan departure inside a task
(Task 7, below). The evolution was all on Track A's operational side and in
the validation checks, and it came from one event.

**Node 26 arrived by hand.** vergil-containers#949 added ts-node 26 on
2026-11-17 at 23:03Z, fifteen minutes before the spec and plan merged. It used
the lifecycle check's own add path, so it carried what an automated add PR
would have. That removed Track A's centrepiece: Task 3 had planned to review
and merge the first lifecycle add PR, and there would be none. The human ruled
to keep Task 3 for the checks that still applied (the 26 dev images and the
vergil-vm follow-up), to unblock Task 4 on Task 2 instead, and to amend the
spec in place with dated notes (.github#1305).

**Task 5's checks moved with it.** Its release-notes check had to look for
vergil-containers#949, which shipped one release before the one Task 5
attested. The human
ruled that both releases' notes count. Task 3's "no ts-node lifecycle PR"
check had also passed only because no lifecycle run had happened yet under
`pr` automation. So Task 5 gained a check that a run after #956 opened no
ts-node PR. Rather than wait a day for the schedule, the human dispatched the
Ops workflow by hand and ruled that a dispatched run counts. Its log could not
be downloaded, so the human fetched the run's report artifact. It showed
ts-node `in sync`.

**Task 7 read the grammar faithfully over the plan.** The plan listed `">=x"`
as an invalid range. node-semver reads it as `*`, and the spec requires the
grammar be read faithfully, so the code follows node-semver. The plan's
evolution log records it.

**The documentation review spawned four doc tasks** in other repos
(vergil-containers, vergil-actions, docs, .github), which the delivery table
did not list.

**Not in the evolution log:** three rulings made late in execution changed how
tasks closed, not what they did:

- the dispatched-run and artifact rulings on Task 5 (epic items 5 and 6);
- the fixture-run serialization override;
- accepting both fixture releases despite `vrg-release` exiting 1 (item 7).

They are recorded on the epic issue.

## 2. Lessons learned

- **Check the target's state the day the plan is approved, not only the day
  it is written.** The plan was written against a manifest without Node 26,
  and a hand-made add landed between writing and approval. The plan's
  centrepiece task then had nothing to act on. A last-minute probe of the
  facts a plan depends on ("is Node 26 still missing?") would have caught it
  before the task list froze.
- **A "nothing happened" check needs something to have happened.** Task 3
  passed "no ts-node PR" before any lifecycle run under `pr` automation had
  run. A check of absence should name the event that makes it mean something:
  here, a completed run after the switch.
- **Parallel dispatch of an independent frontier worked well.** Four tasks
  across three repos ran at once in the first pass and reached report-ready
  within about ten minutes of each other. Per-repo submission let
  vergil-containers and vergil-actions go ahead of vergil-tooling's two PRs.
- **Getting the release-lock go-ahead up front avoided two stalled runs.**
  Tasks 6 and 8 each had a "stop before report-ready" step. The driver asked
  for the go-ahead once, before dispatch, and passed it to both.
- **The agent did not use a tool that already existed.** Task 5's run retried
  `vrg-gh run view --log` more than a dozen times. A per-job
  `vrg-gh run job-log`, added for exactly this failure (vergil-tooling#4597,
  shipped in v2.1.322), was never tried. Next time, look for the dedicated
  command before escalating a tool failure to the human.
- **Look at the local fleet before asking GitHub.** Task 10 first tried
  `vrg-gh search code` to find the TypeScript repos by their `vergil.toml`,
  and the org code search returned almost nothing. Every fleet repo is
  checked out locally by design, with its default branch, so reading each
  local clone's `vergil.toml` on `develop` is the direct way to answer a
  fleet-wide question about repo contents. The human ruled this a working
  practice, not a tool defect.
- **Write the scope notes for the next epic in plain words.** The first draft
  of the follow-on comments leaned on issue numbers. The human asked for each
  reference to say what it is.

## 3. Compromises and tradeoffs

- **A hand-written npm range reader, not a library.** The only PyPI
  implementation of npm's range grammar failed the plan's criteria: no Python
  3.15 support declared, CI stopping at 3.13, and maintenance mode. The reader
  in `typescript.py` is now Vergil's to maintain. It was cross-checked against
  node-semver 7.7.4 on 16,000 random ranges. It differs on two node-semver
  quirks the spec does not cover: `minVersion` returning null after an empty
  alternative, and a prerelease-only overlap in `intersects`.
- **Task 5 accepted weaker-than-written evidence, by ruling:**
  - a dispatched Ops run in place of a scheduled one;
  - release notes spread over two releases;
  - an artifact the human downloaded, because the agent could not.

  Each ruling is recorded on the epic, and the substance of each check held.
- **The fixture releases count as successful though `vrg-release` exited 1.**
  The post-release `develop` validation (close-finalize) never ran, because
  another epic's vergil-tooling v2.1.325 was released mid-run and the package
  index's CDN cache had not caught up. The human ruled the release successful
  without a re-run. So post-release `develop` validation on both fixtures at
  v0.1.3 was not observed. Each fixture's CI on the release PR and on `main`
  was green.
- **The fixture-run serialization rule was overridden.** The ruling's record
  says epic #1218's validation tasks (vergil-fixture-ts-lib#12,
  vergil-fixture-ts-app#12) were still open. In fact both had closed about a
  minute before the ruling was recorded (17:04:50Z and 17:04:53Z, ruling at
  17:06:09Z). The override was most likely not needed, and the record states
  a fact that was already stale.
- **The dependency-update check is a known gap for TypeScript.**
  `vrg-update-deps` has no npm updater (epic 3, .github#1209), so neither
  fixture run proved the release's dependency update.

## 4. New problems and opportunities

**Unaddressed-findings sweep:**

- **Swept:** 23 tasks, the epic issue's own comments, and the planning
  issue's review rounds.
- **Rulings:** every `Awaiting a ruling` item on the epic (items 1–7) has a
  recorded ruling, and every issue those rulings name exists.
- **PR reviews:** none of the 15 task PRs has a COMMENTED or
  CHANGES_REQUESTED review.
- **Planning rounds:** every rejected design-review finding has a reason or an
  intake ref. The rounds' "discards" lists are topic-only, with no reasons.

What the epic surfaced, and where each went:

- **Already tracked:**
  - Dev-image provisioning failed when a just-released vergil-tooling was
    not yet in the package index (the fixture releases' close-finalize).
    **Found:** vergil-tooling#4706, filed from epic 1218, closed.
  - Stale picomatch audit exceptions in both TypeScript fixtures.
    **Found:** vergil-fixture-ts-lib#20 and vergil-fixture-ts-app#20, open;
    also noted on .github#1209.
- **Handed to the next epics:**
  - The core Node is 22, whose end of life is 2027-04-30, and the ts-node
    drop PR keeps its pin. Noted on epic 4's planning item, .github#959;
    core-Node lifecycle automation is .github#1094.
  - vergil-vm's separate Node 22 install. Noted on .github#959.
  - The npm updater is also the fixture runs' missing dependency-update
    proof. Noted on epic 3's planning item, .github#1209.
- **Ruled by the human in this retrospective:**
  - The fixture-run serialization rule deadlocks when two runs are filed
    before either starts, so the human sequences them by hand (also seen in
    epic 1174). **Filed:** .github#1358.
  - `vrg-gh` does not allow `run download`, so the agent could not fetch a
    run artifact. **Found:** commented on .github#497 (deployment and
    validation steps runnable by the identity that runs them).
  - Python-primary repos that build TypeScript are not checked by the
    TypeScript rules (a spec round 1 discard, with no reason). **Declined:**
    one repo holding both a Python and a TypeScript program is not a
    supported configuration. They belong in separate repos, which is what
    the TypeScript support is for.
  - `vrg-gh search code` over `vergil.toml` is unreliable across the org;
    Task 10 read every repo instead. **Dropped:** the whole fleet is checked
    out locally with its default branches, so a fleet-wide question about
    repo contents is answered from the local clones, not GitHub search (§2).
  - `vrg-container-docs` notices start with `vrg-container-run:`, because it
    shares the passthrough. **Dropped:** cosmetic, and the plan fixed the
    wording.
  - The GitHub release page shows only a docs link; the notes live in the
    tag's `releases/v<version>.md`. **Dropped:** the existing convention.
  - The range reader's two node-semver quirk differences. **Dropped:**
    outside the spec, and recorded in vergil-tooling PR #4679.

## 5. What's next

- **Epic 3, TypeScript supply chain and the npm updater
  (.github#1209):** `epic-create` started on 2026-11-18. The scope notes from
  this epic are on that issue.
- **Epic 4, Node runtime architecture (.github#959):** a planning item. It
  carries this epic's notes on the core Node's 2027-04-30 deadline and
  vergil-vm's Node.
- **The follow-on brainstorm task**, .github#1283, closed with these handoffs.
