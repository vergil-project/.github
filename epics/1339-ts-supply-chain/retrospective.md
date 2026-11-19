# TypeScript supply chain and the npm updater — retrospective

- **Epic:** vergil-project/.github#1339
- **Planning issue:** vergil-project/.github#1209
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1341
- **Span:** epic opened 2026-11-18T18:38Z; the last task before this
  retrospective closed 2026-11-19T12:17Z, about 17.6 hours later.

## 0. At a glance

**What we set out to do.** Give TypeScript the supply-chain checks and
dependency updates that Python and Rust already had:

- a license gate that fails the build and leaves evidence;
- a `vrg-update-deps` updater for `package-lock.json` that reports what its
  ranges hold back;
- fixture runs that prove both and close the TypeScript dependency-update
  known gap, open since epic vergil-project/.github#1174.

**What shipped.** All of it:

- `vrg-npm-license-gate` replaces the non-gating `license-checker --summary`
  in the TypeScript AUDIT check and writes `licenses.json`.
- `NpmUpdater` moves the lockfile within the `package.json` ranges and
  reports held-back packages with their reasons.
- vergil-actions requires `licenses.json` in the `typescript/audit` evidence.
- `fixture-run` has no known gaps left.
- Both TypeScript fixture runs passed every check with no known gap.
- The removal of license-checker from the images is filed as ad-hoc work, as
  scoped.

**Work delivered** (PR — what it did — merged, UTC):

- **Spec and plan:** .github #1368 — the spec and plan — 2026-11-18 22:09.
- **Planned tasks:**
  - Task 2, vergil-tooling #4727 — the time-boxed exception rules shared out
    of the npm audit gate, with no behaviour change — 2026-11-18 22:38.
  - Task 4, vergil-tooling #4728 — `NpmUpdater` and its held-back report in
    `vrg-update-deps` — 2026-11-18 22:39.
  - Task 3, vergil-tooling #4729 — `vrg-npm-license-gate` in the TypeScript
    AUDIT check, writing `licenses.json` — 2026-11-18 23:04.
  - Task 5, vergil-actions #1311 — `licenses.json` and
    `vrg-npm-license-gate` in the `typescript/audit` report contract —
    2026-11-19 00:55.
  - Task 7, vergil-claude-plugin #1182 — `fixture-run` turns on the
    TypeScript dependency rollback and drops its known-gap row —
    2026-11-19 00:55.
- **Fixture-run changes:**
  - vergil-fixture-ts-lib #35 — `countWord`, with an acorn rollback —
    2026-11-19 01:29.
  - vergil-fixture-ts-app #36 — the `lower` subcommand, with a semver
    rollback — 2026-11-19 01:29.
- **Documentation review:** vergil-tooling #4735 — the TypeScript license
  report and the npm updater in the guides — 2026-11-19 12:17.

**Releases** (published, UTC):

- vergil-tooling v2.1.327 (#4727, #4728, #4729) — 2026-11-18 23:17.
- vergil-actions v2.1.84 (#1311) — 2026-11-19 01:02.
- vergil-claude-plugin v2.1.80 (#1182) — 2026-11-19 01:03.
- vergil-fixture-ts-lib v0.1.4 (#35) — 2026-11-19 01:35.
- vergil-fixture-ts-app v0.1.4 (#36) — 2026-11-19 01:34.

Each fixture release's dependency update reversed its rollback:

- ts-lib #39 moved acorn from 8.18.0 back to 8.19.0.
- ts-app #40 moved semver from 7.8.5 back to 7.8.6.

**Counts:**

- **Repos touched:** 6: .github, vergil-tooling, vergil-actions,
  vergil-claude-plugin and the two TypeScript fixtures.
- **Tasks:** 18 children, all closed once this retrospective merges:
  - 5 code tasks;
  - 1 documentation review;
  - 4 deployment and validation tasks in .github;
  - 6 fixture-run tasks;
  - the spec and plan task;
  - this retrospective.
- **PRs:** 9. **Releases:** 5.

## 1. How the plan evolved

The plan held. Its task list never changed, so its "Evolution during
execution" log stayed empty. Every task ran in the order and grouping the
Delivery table set:

- Tasks 2 and 4 ran in parallel.
- Task 3 waited for Task 2.
- Tasks 5 and 7 waited for the vergil-tooling release (6a), not its merge.
- The fixture runs waited for 6b and 8.

The deviations were inside tasks, each recorded in its PR notes:

- **Task 3: brackets in canonical expressions.** The plan said to keep
  brackets "only where AND sits inside OR". Applied literally, that would
  print `(A OR B) AND C` as `A OR B AND C`, which means something else. The
  gate brackets any compound operand of the other operator instead. The merge
  reviewer agreed it was necessary.
- **Task 3: where the exceptions file is read.** `.npm-license-exceptions` is
  read from the lockfile's directory; the command takes only `--lockfile` and
  `--output`, as specified.
- **Task 4: how a change is detected.** `NpmUpdater` compares the lockfile's
  bytes before and after, where the Cargo updater uses `git status`.
- **Task 4: two more docs.** It updated `dependency-update-config.md` and
  `cli-tools-overview.md`, which also list the updaters and would otherwise
  have been wrong.
- **The fixture runs: concrete versions in the validation bodies.** The
  bodies named vergil-tooling v2.1.327 and vergil-actions v2.1.84 for the
  deployed-version probe, rather than only the plan's "the releases of Tasks
  6a and 6b".
- **The follow-on brainstorm (#959) moved out of the epic.** It went to the
  planning queue before the retrospective, as the plan intended ("worked
  through planning once this epic is nearly done"), so it no longer blocked
  the definition-of-done gate.

## 2. Lessons learned

- **The release-ordering constraint worked.** Making Tasks 5 and 7 wait for
  the vergil-tooling *release* (6a), not its merge, meant no TypeScript repo
  saw CI require a `licenses.json` that no released tooling wrote yet. Keep
  this pattern whenever a contract in one repo depends on a producer in
  another.
- **Live checks earned their place.** vergil-tooling's own validation runs no
  npm. The plan's in-image check for Task 3 and host-driver check for Task 4
  ran the real thing before handoff, and the fixture runs then confirmed it
  in CI. Task 4's check also showed what `npm update` really does: seven
  other packages moved with the rolled-back one, all patch releases
  published after the spec's spike. The `fixture-run` validation check was
  worded to expect this from the start.
- **The human sequences releases; trust it.** The deploy tasks for 6a, 6b
  and 8 asked for human attestation and then re-verified it: that releases
  contained their PRs, and that installs reached the host and VMs. A
  vergil-tooling release updates the host and every VM itself, and the human
  releases only after the merges it carries. Next time, write deploy tasks
  that take the human's "merged and released" as given, and check only what
  the release is meant to change.
- **The host's tools are not the image's tools.** The ts-app rollback found
  that the host's npm 10.9.9 drops the `libc` lockfile fields the image's
  npm 11.20.0 writes. Any command that writes a file the image will later
  read belongs in `vrg-container-run`.

## 3. Compromises and tradeoffs

- **license-checker still ships in the ts-node images.** Nothing runs it now.
  Removing it was scoped out of this epic and filed as ad-hoc work (§4).
- **`NpmUpdater.applies` does not check the primary language.** It applies
  wherever a root `package-lock.json` exists. This was discarded with a
  reason in four review rounds on #1209: no fleet repo besides the two
  fixtures has a root lockfile. A non-TypeScript repo that gains one would
  get npm updates. The Cargo and uv updaters key off file presence the same
  way. The sweep turned it into a broader question, filed as
  vergil-project/.github#1377 (§4).
- **The proof is two fixtures.** No production TypeScript repo has run the
  gate or the updater yet. The fixtures are real, managed repos, but they are
  small.
- **Change detection differs from Cargo.** `NpmUpdater` compares the
  lockfile's bytes where Cargo uses `git status`. This is simpler and correct
  for one file, but the two updaters now detect change in two different
  ways.

## 4. New problems and opportunities

**Unaddressed-findings sweep:**

- **Scope:** all 17 children other than this task, the epic issue, their 9
  merged PRs and the planning issue's round comments.
- **Untracked problems found:** one.
  - Every `Awaiting a ruling` item on the epic has a recorded ruling.
  - Every round finding on #1209 was accepted, or discarded with a reason.
  - The PRs carry no review comments.
  - One discard came back in four review rounds on #1209 and had no tracking
    issue: `NpmUpdater.applies` has no language check. The human ruled to
    file it, more broadly, as vergil-project/.github#1377.
- **Security findings:** the epic had 1 security finding, from alignment
  round 1 on #1209. It was accepted and is recorded on vergil-tooling#4722.

**What the epic surfaced, and where each item went:**

- **license-checker left in the ts-node images.** Filed by Task 9 as
  vergil-project/vergil-containers#964 (open).
- **license-checker in the vergil-tooling docs.** Filed by Task 9 as
  vergil-project/vergil-tooling#4734 (open), blocked by #964.
- **The `fixture-run` TypeScript rollback runs npm on the host**, whose older
  npm strips lockfile fields. Ruled "file": vergil-project/vergil-claude-plugin#1186
  (open).
- **The picomatch `.npm-audit-exceptions` entry (GHSA-fc5q-39xc-vjpf) is
  stale in both fixtures.** Ruled "no new issue": tracked by
  vergil-project/vergil-fixture-ts-app#20 and
  vergil-project/vergil-fixture-ts-lib#20 (both open), each with a comment
  that its removal condition looks met.
- **Deploy steps assume release notes are in the GitHub release body.** For
  vergil-tooling they are in `releases/vX.Y.Z.md` and `CHANGELOG.md`. Ruled
  "dropped": the human sequences each release after the merges it carries.
- **Updaters infer the language from files.** `NpmUpdater.applies` has no
  language check, and the Cargo and uv updaters also key off the files
  present. Ruled "file", broadened: vergil-project/.github#1377 (intake). It
  asks to key language-dependent behaviour off what `vergil.toml` declares
  instead of inferring it from the files present.

## 5. What's next

- **Epic 4: Node runtime architecture and TypeScript app support.** The
  follow-on brainstorm is vergil-project/.github#959, now in the planning
  queue. It is the next epic of the TypeScript-parity wave.
- **The license-checker removal:** vergil-containers#964, then
  vergil-tooling#4734.
- **Declared over inferred configuration:** vergil-project/.github#1377, in
  intake. It covers every place vergil-tooling infers a language or shape
  from files.
