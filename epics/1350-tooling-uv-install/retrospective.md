# Install vergil-tooling with uv everywhere — retrospective

- **Epic:** vergil-project/.github#1350
- **Planning issue:** vergil-project/.github#466
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1352
- **Span:** epic opened 2026-11-18T19:33Z. Execution began when the spec
  and plan merged, 2026-11-19T12:38Z. The last task before this
  retrospective closed 2026-11-20T00:05Z: about 28.5 hours from opening,
  about 11.5 hours of execution.

## 0. At a glance

**What we set out to do.** Make a vergil-tooling release usable the moment
its tag exists, by installing it everywhere with `uv tool install` from one
resolved release commit, on a uv-managed Python 3.15, constrained by that
commit's `uv.lock`. Every consumer used to install the apt package from
GitHub Pages, whose `max-age=600` held each release back by up to ten
minutes. Along the way:

- move `base`'s core Python from vergil-python to a managed 3.15;
- drop every vergil package and the vergil apt source from the VMs;
- restrict release-tag creation to the Admin role and the automation App,
  since installs now trust the git tag;
- keep building, signing and publishing the packages for outside
  consumers.

**What shipped.** All of it:

- **vergil-tooling** resolves a ref once (`tooling_install`). The cache
  build, `vrg-vm` and the Mac's consumer refresh install that one commit
  with its lock. `vrg-host` reports vergil-tooling and leaves it to the
  release. VMs lose their vergil packages and apt source, and get a
  non-login `PATH` guarantee.
- **vergil-actions:** the setup action and the package setup driver
  install from git. Release tags are pushed with the App's installation
  token.
- **vergil-containers:** `base` carries a managed 3.15 at
  `VERGIL_CORE_PYTHON_DIR=/opt/vergil/uv/python`, with no vergil-python and
  no vergil apt source.
- **The Tag protection ruleset** restricts tag creation in all 15
  vergil-project repos. Six fixture releases logged the App's push as a
  bypass of the creation restriction, and an agent test push was refused.
- **The headline:** v2.1.329 was published at 21:47:46Z. The release's own
  consumer refresh had it on the Lima VM at 21:56:41Z, and a cache build
  resolved `v2.1` to it with no index wait. The package-index stage, no
  longer on any install path, took 3.9s.

**Work delivered** (PR — what it did — merged, UTC):

- **Spec and plan:** .github #1380 — 2026-11-19 12:38.
- **Release tags and the ruleset:**
  - Task 2a, vergil-actions #1322 — release tags pushed with the App's
    installation token — 13:13.
  - Task 2b, nine repos — each `cd.yml` passes the App secrets:
    vergil-claude-plugin #1193 (13:00), vergil-containers #969 (13:37),
    vergil-vm #431 (13:41), fixtures python-app #28, python-lib #30,
    rust-app #43, rust-lib #44, ts-app #45, ts-lib #44 (13:01–13:04).
  - Task 2, vergil-tooling #4754 — the creation rule and the App as a
    bypass actor in the Tag protection ruleset — 14:51.
  - New task, vergil-tooling #4750 — `repo_init`'s generated `cd.yml`
    passes the App secrets — 14:43.
- **vergil-tooling:**
  - Task 6, #4745 — `vrg-host` reports vergil-tooling and skips it — 13:00.
  - Task 3, #4748 — `tooling_install` and the locked Mac refresh — 13:31.
  - Task 3b, #4755 — `consumer-refresh` as the release's last pipeline
    stage, rolled up to one line — 14:51.
  - Task 5a, #4757 — VM install with uv and package cleanup — 14:52.
  - Task 4, #4756 — the cache build installs one release commit — 14:54.
  - Task 5b, #4758 — `~/.local/bin` on `PATH` for non-login commands —
    16:03.
- **vergil-actions:**
  - Task 8, #1320 — the setup action installs from git — 13:06.
  - Task 9, #1321 — the package setup driver installs from git — 13:07.
  - New task, #1333 — uv 0.13.9 and a released-3.15 check — 16:53.
- **vergil-vm:** Task 11, #432 — `vrg-vm-init.sh` drops its own install —
  13:41.
- **vergil-containers:** Task 13, #970 — `base` on a managed 3.15, no
  vergil apt source — 19:30.
- **Fixture-run changes (Task 15):** python-app #33, rust-app #48,
  ts-app #50, rust-lib #49, ts-lib #49, python-lib #40 — 21:25–21:28.
- **Plan evolution log:** .github #1405 — 17:10.
- **Documentation review (R1):** vergil-tooling #4807 — 2026-11-20 00:00;
  per-repo doc tasks vergil-actions #1344 (11-19 23:57), docs #147 (23:58),
  vergil-vm #439 (11-20 00:05).

**Releases** (published, UTC, 2026-11-19):

- vergil-actions v2.1.85 (Tasks 2a, 8, 9) — 14:17.
- vergil-actions v2.1.86 (the ad-hoc fix vergil-actions#1325) — 14:28.
- vergil-actions v2.1.87 (#1333) — 17:01.
- vergil-tooling v2.1.328 (Tasks 2, 3, 3b, 4, 5a, 5b, 6, #4750) — 16:16.
- vergil-tooling v2.1.329 (the Task 16v headline release; also carries the
  ad-hoc vergil-tooling#4791) — 21:47.
- vergil-containers v2.1.47 (Task 13) — 20:25.
- vergil-vm v2.1.54 (Task 11, Task 2b's #431 and the ad-hoc
  vergil-vm#433) — 21:39.
- Fixtures: python-lib v0.1.2 (Task 2ad) — 14:49; python-app v0.1.2,
  python-lib v0.1.3, rust-app v0.1.3, rust-lib v0.1.3, ts-app v0.1.5 and
  ts-lib v0.1.5 (Task 15) — 21:33–21:40.

**Counts:**

- **Repos touched:** 13: .github, docs, vergil-tooling, vergil-actions,
  vergil-containers, vergil-vm, vergil-claude-plugin and the six language
  fixtures.
- **Tasks:** 55 children, all closed once this retrospective merges:
  - 14 code tasks, two of them added during execution;
  - 9 secrets tasks (2b);
  - 5 deployment and 2 validation tasks;
  - 18 fixture-run tasks;
  - the documentation review and its 3 per-repo doc tasks;
  - the spec and plan, the evolution log, and this retrospective.
- **PRs:** 35 before this one. **Releases:** 14.
- **Ad-hoc work filed along the way, outside the epic:** 12 issues
  (§4).

## 1. How the plan evolved

The task list held: every planned task ran and closed. What moved was
order, and the evidence a few tasks rested on. The plan's "Evolution during
execution" log records the first twelve changes; the rest came after it was
written and are summarized here from the issues.

**Order.** The plan assumed one release per step, in dependency order:
tag protection first, then vergil-tooling, then vergil-actions, then
vergil-containers. Tasks 8 and 9 merged into vergil-actions together with
Task 2a, so the release Task 2ad needed (v2.1.85) also shipped the uv
setup action, ahead of Task 7d. Reverting the two to keep the order was
offered and declined. Task 10d ran its checks against that release at once,
with its 7d precondition waived. Then Task 2d, the ruleset, moved after
Task 7d: the human accepted a short window in which installs trusted git
tags before tag creation was restricted, rather than apply the ruleset from
unreleased code. It was applied about half an hour after v2.1.328.

**Scope inside tasks.**

- **Task 3b** was to run `consumer-refresh` right after `confirm-main`.
  Its run found that this would refresh vergil-claude-plugin's consumers to
  the previous release, since their marketplace follows the rolling tag
  that only `promote` moves. The human dropped the move: the gain was
  negligible, because the workflow blocks until the release ends anyway.
  The refresh stays the last stage; the in-pipeline stage and its one-line
  rollup were kept.
- **Task 2a's** smoke-release tag check could not run in PR CI. Two tests
  replaced it, and the live proof moved to Task 2ad, then to Task 2d, then
  to Task 15's six fixture releases, where it landed.
- **Task 2ad** took the fixture's CD log as evidence, because the events
  API gave no creator record for the tag.

**New tasks.** Two were added under the epic: `repo_init`'s generated
`cd.yml` (vergil-tooling#4747), because Task 2a made the App secrets
required, and the setup action's uv pin (vergil-actions#1331), because the
old pin installed a 3.15 release candidate on host runners.

**Validation lightened.** Task 5b's real-VM proof before merge was waived
and folded into Task 7d's per-VM check. Task 7d closed on the Mac and the
one VM that answered. Task 16v's step 1 used the human's rebuilds of the
logical-minds-foundry VMs, one Lima and one cloud, instead of a new Lima
VM, since this session ran inside the shared Lima VM. Its step 2 accepted
indirect evidence that interpreter downloads were off.

**After the log was written** (not yet in `plan.md`):

- A Go standard-library CVE published mid-epic (CVE-2026-97054) failed
  every image build and blocked Task 13. It was triaged ad hoc
  (vergil-containers#971): all six affected tool binaries were suppressed
  with removal conditions.
- The shared Lima VM's disk filled during Task 14d. The human had dead
  images pruned and the default disks doubled, ad hoc (vergil-tooling#4791,
  vergil-vm#433), ahead of Task 16v's rebuilds.
- The fixture runs' "Ready to start" serialization check read literally
  against epic 1384's queued trios. The human ruled that this epic's trios
  go first, since 1384's had not started.
- The rust-lib fixture's stale `rust-version` floor was removed ad hoc
  (vergil-fixture-rust-lib#54), so the fixture tracks the window.

## 2. Lessons learned

- **Setbacks stayed minor, and that is the result to aim for.** Every
  departure from the plan cost a short detour that one ruling resolved;
  none was a showstopper, on a change of this size made to the live system
  while it was in use. The human's read, which the record supports: the
  planning stopped at about the right depth. More pushback rounds could
  always have found another corner case; past some point those cases did
  not matter, and implementation hit no major design problem.
- **A delivery table that orders merges does not order releases.** The
  table allowed Tasks 8 and 9 to merge before Task 2ad's release needed
  them, and nothing said that vergil-actions' next release would carry
  them. The spec's rollout (§5) stated the order, but the table did not
  enforce it. Next time, when one repo's release must carry only some of
  its merged work, the plan should say so at the release task.
- **A security precondition needs to be named at the task that can break
  it.** The tag ruleset was a precondition for installs trusting git tags,
  stated in the spec. It slipped because no task said "do not release the
  setup action before 2d". It was caught late, by the driver, after the
  early release.
- **Latent bugs surface under load, and fixing them was a bonus.** Two CI
  failures were long-standing bugs the epic happened to trigger: an
  early-exit pipeline racing under `pipefail` (vergil-actions#1325) and a
  test that matched a word in branch names (vergil-tooling#4752). Comparing
  passing and failing runs found each in minutes. The human's standing
  rule followed from the first: shell stays minimal, and logic belongs in
  tested Python.
- **Dispatched runs should check what they can before asking.** Two
  fixture runs asked the human to confirm repo-config drift; a third
  showed the agent identity can read the `local:` line itself.
- **The record does the remembering.** Every ruling went onto an issue,
  and the mid-epic evolution log made §1 cheap to write. The changes after
  it was written had to be reassembled from the issues.

## 3. Compromises and tradeoffs

- **A security window, accepted on purpose.** For about two and a half
  hours (v2.1.85 at 14:17 to the ruleset apply at about 16:44), consumer CI
  installed vergil-tooling by resolving git tags while tag
  creation was not yet restricted. The human judged the exposure small and
  chose speed. It closed when Task 2d applied the ruleset.
- **Lighter validation, failing forward.** Task 5b's VM proof was waived;
  Task 7d closed without the cloud VM; Task 16v's disk sizes were not
  confirmed, and its download evidence was indirect. Each was an explicit
  ruling. The cost is that a regression would show up after release, not
  before merge. One did: see #4810 in §4.
- **The refresh still waits for the whole release.** The early refresh was
  dropped (§1), so a new release reaches the Mac and VMs at the end of a
  release of about 20 minutes, not when the tag appears. The install itself
  no longer waits on the package index, and a cache build picks up a new
  release at once.
- **Six suppressed CVE findings.** CVE-2026-97054 is suppressed for six
  third-party binaries until their upstreams ship builds on fixed Go.
- **The disk change shipped incomplete.** vergil-tooling#4791 doubled the
  base Lima disk, but only dedicated VMs get it; the shared base VM still
  takes the template's 50 GiB (vergil-tooling#4810, open).

## 4. New problems and opportunities

**Unaddressed-findings sweep:**

- **Scope:** all 54 children other than this task, the epic issue, their
  35 merged PRs, and the planning issue's six round comments.
- **Rulings:** every `Awaiting a ruling` item on a child has a recorded
  ruling, or is tracked by an existing issue:
  - the `app-id` deprecation warning, raised on six fixture and deploy
    tasks: tracked by vergil-actions#1243 (the driver's duplicate #1332
    was closed in its favour);
  - the rust-lib `rust-version` floor (vergil-fixture-rust-lib#43): filed
    and fixed as vergil-fixture-rust-lib#54, though no ruling line was
    posted on #43 itself;
  - the `develop-v*` tags not covered by the ruleset, and the late `ops.yml`
    schedule: dismissed by ruling.
- **Round findings on #466:** every finding was accepted, or dropped or
  rejected with a reason.
- **PRs:** none of the 35 has a review or conversation comment.
- **Untracked problems found:** five, each noted in a run's report or
  comment and never filed. The human ruled on each:
  - **The refresh's output never reaches the release log.** Since Task
    3b, a successful `consumer-refresh` stage writes only its one-line
    rollup: the full log file holds no install output either, so Task 16v
    could not show the Mac's own install. Filed as a bug, so the stage's
    output is captured like every other command's:
    vergil-project/vergil-tooling#4812.
  - **`vrg-container-cache prune` keeps caches it cannot place.** It kept
    caches named for repos that do not exist (`spike`, `fx`, `ts-app`),
    so dead images had to be removed by hand when the disk filled. Filed:
    vergil-project/.github#1431 (intake).
  - **`vrg-gh release download` is denied to the agent identity.** Every
    fixture validation fetched its public release assets with `curl`
    instead. Filed, to grant the access: vergil-project/.github#1432
    (intake).
  - **`uv tool list` inside a dev container shows no tools,** because
    `UV_TOOL_DIR` is set only while the image builds. Dropped by the
    human: not a problem, since nothing relies on it and the install is
    checked with `vrg-whoami`.
  - **The plan's evolution log stops before the last third of the epic.**
    Dropped by the human: this retrospective's §1 covers every later
    change, and the log is not used consistently enough across epics to
    be worth back-filling.
- **Security findings:** the planning rounds created four advisories
  (GHSA-p7f8-5r4r-v346, GHSA-rpvf-xhfh-pj4r, GHSA-h849-7fff-7mw7,
  GHSA-mjff-mc5q-qrcv), all accepted, one with the follow-on
  vergil-project/.github#1371. Execution raised no new security finding.

**Mistakes made, and what they cost** (agent and human alike, per the
human's request; see vergil-project/.github#1421 for making this a standing
section):

- **Agent: recommended submitting Tasks 8 and 9 with Task 2a.** The table
  allowed it, but it put the uv setup action in the release Task 2ad
  needed. Cost: a release-order deviation, and the security window above.
  Better: check what a repo's next release will carry before advising a
  submit.
- **Agent: missed the security-ordering gap until after the early
  release.** The driver flagged the early setup-action release as an order
  change but not as a security precondition, and caught it only when
  planning the vergil-tooling release. Cost: the window ran longer than it
  needed to; the human then accepted it.
- **Agent: filed vergil-actions#1332 without searching.** #1243 already
  tracked it. Cost: a duplicate issue, closed. Better: search before
  filing, as `epic-implement` already asks.
- **Agent: offered rulings without their questions.** Twice the human had
  to ask for the problem behind a list of options (#4739's two items, then
  the `app-id` item). Cost: a round trip each.
- **Agent: #4791 shipped incomplete.** Neither its run nor its merge review
  noticed that the shared VM path ignores the base disk; a later doc task
  found it (#4810). Cost: the shared VMs did not get the extra space the
  human asked for.
- **Human side, checked against the record.** Every "merged" or "released"
  the human reported in the driver session checked out against git and
  GitHub. The human's risks were explicit rulings, not slips (the security
  window, the waived VM proof, the unchecked cloud VM), and each is in §3.
  The largest human cost was round trips: dozens of human-gated steps
  (submits, releases, reruns, pasted command output). About 30 had been
  counted by mid-epic; a precise total was not kept.

**What the epic surfaced, and where each item went:**

- **Early-exit pipelines racing under `pipefail`:** fixed ad hoc,
  vergil-actions#1325 (closed), with a guard against reintroduction.
- **A test failing on branch names containing "secret":** fixed ad hoc,
  vergil-tooling#4752 (closed).
- **CVE-2026-97054 blocking image builds:** triaged ad hoc,
  vergil-containers#971 (closed).
- **Agent VMs running out of disk:** doubled ad hoc, vergil-tooling#4791
  and vergil-vm#433 (closed). The shared VM still gets 50 GiB:
  vergil-tooling#4810 (open).
- **`NEEDS-REBUILD` right after a dedicated VM's rebuild:**
  vergil-tooling#4806 (open).
- **A stale docstring naming VM provisioning:** vergil-tooling#4811 (open).
- **The `app-id` input deprecation:** vergil-actions#1243 (open).
- **The stale-plugin hook stops agents mid-task:** intake,
  vergil-project/.github#1381.
- **A "Mistakes made" section in every retrospective:** intake,
  vergil-project/.github#1421.

## 5. What's next

- **Package hosting off GitHub Pages:** the deferred brainstorm
  vergil-project/.github#1378, revisited with the home hosting platform.
- **Narrowing which jobs get the App's credentials:**
  vergil-project/.github#1371.
- **Retiring the `develop-vX.Y.Z` tags:** epic vergil-project/.github#1406.
- **The open ad-hoc follow-ups:** vergil-tooling#4810, #4806, #4811 and
  #4812, and vergil-actions#1243, through their repos' ad-hoc queues; and
  the intake items vergil-project/.github#1431 and #1432.
- **Mechanizing the human's round trips:** the retrospective section of
  vergil-project/.github#1421 is the first step; releases that run their
  own downstream checks would remove the most.

## Appendix A — Operational notes

**Release order as it happened (2026-11-19, UTC):**

1. vergil-actions v2.1.85 (14:17), the `merge-release` stage having failed
   once on the pipefail race, then v2.1.86 (14:28) with its fix.
2. The python-lib fixture v0.1.2 (14:49): Task 2ad's App-pushed tag.
3. vergil-tooling v2.1.328 (16:16): its refresh moved the Mac and VMs.
4. The Tag protection ruleset applied to all 15 repos by the human as
   Admin, then audited (Task 2d, about 16:44).
5. vergil-actions v2.1.87 (17:01): the stable-3.15 uv pin.
6. vergil-containers v2.1.47 (20:25): `base` on the managed 3.15, which
   ended the fleet freeze.
7. The six fixture releases (21:33–21:40), then vergil-vm v2.1.54 (21:39)
   and vergil-tooling v2.1.329 (21:47) with the doubled disks.

**Human-gated steps:** every submit, every release, the ruleset apply and
audit, two scope changes, a plugin reload, a CI rerun, the VM rebuilds,
and the Mac and VM checks whose output was pasted back.

**Gotchas:**

- A failed `vrg-release` leaves its release branch, PR and tracking issue
  open; `vrg-release --resume <version>` picks up after the cause is
  fixed.
- `vrg-gh api` and `vrg-gh release download` are denied to the agent
  identity: deploy and validation runs read events and release assets
  another way, or ask the human.
- `!` commands run inside the agent's Lima session, not on the Mac:
  anything that needs the Mac's `vrg-vm` belongs in a Mac terminal.
- Rebuilding the shared Lima VM ends any agent session running inside it.
