# TypeScript through the full cycle — retrospective

- **Epic:** vergil-project/.github#1174
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md) (with its `##
  Evolution during execution` log)
- **Planning issue:** vergil-project/.github#1161
- **Span:** epic opened 2026-11-16T13:54Z; last child closed 2026-11-17T21:00Z
  (about 31 hours). Planning ran from 2026-11-15T19:18Z to 2026-11-16T19:08Z.

## 0. At a glance

**What we set out to do.** Make TypeScript on Node a first-class Vergil
language: a TypeScript library and a TypeScript application each run Vergil's
full cycle (change, PR, merge, release, validate), as the Python and Rust
fixtures do. In scope: the version-window pin rule, a cross-language
tool-ownership guideline, repo-declared TypeScript tools, Vergil's base configs
as managed files, a release path, a language-neutral BUILD check, TypeScript
evidence, and two fixture repos to prove it.

**What shipped.** All of it. Two new fixture repos, `vergil-fixture-ts-lib` and
`vergil-fixture-ts-app`, each went through the full cycle twice. Run 1 released
v0.1.0 on the new tooling. Run 2 released v0.1.1 on `ts-node` images that no
longer carry `tsc`, ESLint or Vitest. Both runs validated SUCCESS, with one
expected known gap (the dependency update: no npm updater yet,
vergil-project/.github#1209). The fixtures also did their job: Run 1 found four
tooling defects no unit test had caught, and all four were fixed and released
inside the epic. The documentation review found a fifth.

**By the numbers** (queried from GitHub):

| | |
| --- | --- |
| Children | 46 closed, plus this retrospective: 31 tasks, 8 deployments, 4 validations, 3 bookends |
| PRs merged | 33, across 7 repos |
| Repos | .github, vergil-tooling, vergil-actions, vergil-claude-plugin, vergil-containers, and the two new fixtures |
| Tasks added during execution | 11, plus the 12 fixture-run tasks the plan seeded |
| Releases | vergil-tooling v2.1.319, v2.1.320, v2.1.321; vergil-actions v2.1.78 (the fixtures consumed v2.1.79); vergil-claude-plugin v2.1.74; vergil-containers v2.1.44; each fixture v0.1.0 and v0.1.1 |

**Work delivered:**

| Task | Repo, PR | What it did |
| --- | --- | --- |
| 1 | .github#1204 | Spec and plan |
| 2 | vergil-tooling#4513 | A `[ci].versions` pin is required only for a multi-family window (today, C++) |
| 3 | vergil-tooling#4514 | A `package.json` that can't be parsed is an error, not "not TypeScript" |
| 4 | vergil-tooling#4515 | The cross-language tool-ownership guideline |
| 5 | vergil-tooling#4516 | A language-neutral BUILD check kind and `build-enabled` in `vrg-ci-resolve` |
| 6 | vergil-actions#1262 | The `quality / build` job, gated on tooling's flag |
| 7 | vergil-tooling#4522 | TypeScript gates run the repo's own tools within accepted majors; `npm ci --ignore-scripts` |
| 8a | vergil-tooling#4517 | Three TypeScript base configs as managed files in the fleet sync and repo-init |
| 8b | vergil-tooling#4526 | Entry-point and single-Prettier checks; LINT reads the managed configs in place |
| 9 | vergil-tooling#4524 | `vrg-version` bumps `package.json` and the lockfile; a version mismatch fails on the PR |
| 10 | vergil-tooling#4528 | TypeScript lint, typecheck, test and audit reports |
| 11 | vergil-tooling#4525 | TypeScript release build, BUILD on for TypeScript, `build` script check |
| 12a | vergil-actions#1261 | `cd-release` releases TypeScript |
| 12b | vergil-actions#1263 | TypeScript rows in the CI report contract |
| 13 | vergil-claude-plugin#1107 | `fixture-run` knows `ts-lib` and `ts-app`; known gaps recorded per gap |
| 14, 15 | (deployments #1205, #1206) | Releases attested and the plugin deployed; the fixture repos created |
| 16a, 16b | vergil-fixture-ts-lib#5, vergil-fixture-ts-app#5 | The library and the command-line app programs |
| 17 | vergil-containers#942 | The `ts-node` image keeps only Prettier and license-checker |
| 18 | (deployment #1207) | The rebuilt prod `ts-node` images confirmed without the tools |
| Runs 1, 2 | ts-lib #6, #16; ts-app #6, #16 | The fixture changes: `longestWord` and `uniqueWords`; `max` and `reverse` |
| Added | vergil-tooling#4544 | Vergil-managed files kept out of LINT's Prettier check (#4541) |
| Added | vergil-tooling#4551 | A container cache rebuilds when the pinned tooling line has a newer release (#4550) |
| Added | vergil-tooling#4554 | repo-init writes a minimal `package.json` for TypeScript (#4552) |
| Added | vergil-tooling#4555 | Prettier-clean release notes; release output kept out of the Prettier check (#4553) |
| Added | vergil-tooling#4632 | repo-init's dependency-free `package.json` counts as not bootstrapped (#4624) |
| Added | .github#1215, #1227, #1278, #1279 | The plan's four evolution-log entries (#1214, #1224, #1275, #1277) |
| Doc review | vergil-tooling#4631, vergil-actions#1292 | The documentation sweep (#4451) and its vergil-actions doc task (#1288) |

## 1. How the plan evolved

The plan held. Every planned task shipped in its planned shape, and the delivery
order needed one change. The four evolution-log entries record the rest, and
nearly all of it came from the fixtures finding things.

- **One dependency the plan missed (2026-11-16).** Tasks 8b and 10 both rewrote
  the TypeScript LINT command, so running them in parallel would have produced a
  conflict a rebase can't resolve. Task 10 was moved to run after 8b. The plan
  had listed Task 10 as blocked only by Task 7.
- **Run 1 found four defects in released tooling (2026-11-17).** The unit tests
  of Tasks 8a, 8b, 9 and 11 each passed, but a real repo that repo-init had just
  created did not. LINT's `prettier --check .` failed on files Vergil itself
  writes: first the managed files (`.claude/settings.json`, `ops.yml`,
  `CLAUDE.md`; #4541), then the release notes `vrg-release` generates (#4553).
  Both fixture programs then failed again on a stale container cache that still
  ran the previous tooling release (#4550). And Task 9's strict version check
  made a new repo's first CD run fail, because repo-init wrote no `package.json`
  (#4552). Each became a task under the epic, with a deployment task (#1223) to
  release and install the first fix, and three tooling releases (v2.1.320,
  v2.1.321) carried them. The fixture programs and the releases waited on them
  and resumed without any change to the fixtures. The recipe's rule that a run
  never weakens the fixture held each time.
- **Run 2 shared its fixtures with another epic.** Epic #1218 filed its own trio
  on both TypeScript fixtures just before Run 2 was filed. The recipe's
  serialization rule would have deadlocked the two runs, so the human sequenced
  them (Run 2 first) and recorded the decision on both runs' change tasks.
- **The documentation review found a regression of the epic's own making.**
  #4552's `package.json` made a newly created TypeScript repo look partly
  bootstrapped, so `vrg-validate` failed at `npm ci` instead of skipping its
  TypeScript stages as #4384 intends. That became #4624, released in
  vergil-tooling v2.1.322 after the epic's runs. The review also corrected a
  plan interface (Task 8a's drift function returns one result, not a list;
  #1277).

What didn't change: no planned task was dropped or split, no gate was lowered,
and no fixture was worked around.

## 2. Lessons learned

- **The fixtures earned their cost.** Four of the five defects found during
  execution exist only when tools meet a real, newly created repo: repo-init's
  output, the generated release notes, a cache built the day before, a first CD
  run. Each task's unit tests were green. Keep fixture runs as the acceptance
  for any language epic, and file them early enough that their findings can
  still be fixed inside the epic.
- **Any file Vergil writes into a repo is a file the repo's gates check.** The
  Prettier failures were one defect in three places: managed files, release
  output, and `package.json`, which repo-init now writes and Prettier checks.
  The fix that lasts is to derive the exclude list from where Vergil defines
  what it writes, with tests that fail when a new written file isn't covered.
  #4544 and #4555 did that. A new language with a whole-repo formatter will meet
  the same thing.
- **A fix can create the next defect.** #4552 fixed the red first CD run and
  caused the bootstrap regression (#4624). The documentation review caught it
  only because it read the code against the docs. A run against a newly created
  repo after each repo-init change would have caught it sooner.
- **Stale caches hide releases.** For a day, the fix the fixtures needed was
  released but not used, because the branch cache said `current`. #4550 makes a
  cache follow its pinned line, at the cost of a small index lookup on every
  run, failing hard offline (the human's ruling).
- **Parallel dispatch works when the plan's independence is real.** Up to seven
  tasks ran at once without trouble. Where tasks shared code, the shared files
  were the plan's own "independent" tasks appending to one tuple
  (`PRE_GATE_CHECKS`) or rewriting one command. Those caused every rebase
  conflict of the epic. Next time, a plan that has several tasks append to one
  registry should say so, or order them.
- **Frozen branches and local rebases don't mix.** Rebasing a reported-ready
  branch moved its head, and `vrg-submit-pr` then refused it until the branch
  was unfrozen and report-ready re-run. The working pattern: let the submit run
  do the rebase. If it stops on a conflict, abort, unfreeze, rebase onto current
  `develop`, validate and re-run report-ready, and submit one conflicting branch
  at a time.
- **Record decisions as they are made.** Rulings N0.1–N0.8, and the scope change
  on #4550, were written on GitHub at the moment the human decided. That made
  every resumed run self-sufficient, and made this retrospective's sweep clean.

## 3. Compromises and tradeoffs

- **The dependency update stays a known gap.** No npm updater exists, so the
  fixtures skip the dependency rollback, and check 5 of every validation is
  recorded on #1209 instead of passing. That was planned (epic 3). It means the
  release's dependency-update stage is unproven for TypeScript.
- **Run 1 didn't consume exactly v2.1.319.** Its CI ran vergil-tooling 2.1.320
  and 2.1.321, and vergil-actions v2.1.79, because fixes landed between the
  release and the runs. The recipe's probe accepts a later release that contains
  the one named, and the validations recorded the actual versions. But the run
  validated v2.1.319 plus its fixes, not v2.1.319 as released.
- **Some evidence rests on the human's word.** #1205's plugin deployment to the
  macOS host and the other VMs, and the release attestations (#1205, #1223,
  #1207), were recorded from the human's statements in the session, with what
  was observed kept apart. The human ruled that sufficient. The records say so.
- **The image check covered arm64 only.** #1207 ran the no-globals checks on the
  arm64 images. amd64 rests on the CD build jobs and Run 2's CI, which ran on
  amd64.
- **The license-checker check was adapted.** `license-checker --version` prints
  its version but exits 1, so the image check and #1207 use `--help`. The
  deviation is recorded on #1207.
- **The whole of `CLAUDE.md` is outside Prettier's check,** including the repo's
  own section, because Prettier can't check part of a file (#4544). Also, an
  editor's Prettier and `prettier --write .` don't know Vergil's excludes. The
  standards page says to format the repo's own paths instead.
- **Offline use got stricter.** A line-pinned repo now needs the package index
  on every `vrg-container-run`, or `VRG_DOCKER_INSTALL_TAG`, after #4550. The
  human accepted this.
- **The fixture-run deadlock was handled by hand.** By the human's decision,
  nothing was filed. The human is the lock for now.

## 4. New problems and opportunities

### Routed during the epic

| Item | Where it went |
| --- | --- |
| The BUILD check leaves no evidence report, for any language (N0.8) | .github#1250 (captured in intake, now in the planning queue) |
| `vrg-validate --check <kind>` passes when no language is detected | captured as .github#1231, now vergil-tooling#4594 |
| Merge-reviewer replies wrapped in a code fence fail the format check | captured as .github#1238, now vergil-claude-plugin#1130 |
| The App secrets are org secrets; the onboarding docs said per-repo (N0.2) | vergil-actions#1268 |
| The next epics' inputs from this one | notes on epic 2's planning item (#1208) and epic 3's (#1209) |

### The unaddressed-findings sweep

Swept: all 46 closed children, the epic issue and the planning issue #1161; 157
comments, all by trusted authors. All 14 `Awaiting a ruling` items have trusted
rulings, and every filed issue exists. No PR in scope has any review, so there
are no inline comments to check.

Already tracked elsewhere: the `consumer-refresh` notice on fixtures
(vergil-tooling#4590), the skill-version mismatch in long sessions
(vergil-claude-plugin#1129), job-log downloads (vergil-tooling#4597, shipped in
v2.1.322), and the delegate's security-marker defect
(vergil-claude-plugin#1037).

Untracked problems found, and how the human ruled on each (2026-11-17):

| # | Problem | Source | Outcome |
| --- | --- | --- | --- |
| U1 | Every fixture GitHub Release has `target` `develop`, while the tag is on `main`'s merge commit. vergil-actions#1213 tracked this, but was closed not-planned on 2026-11-14 with no reason given. | release data for all four fixture releases | **Filed:** vergil-project/.github#1285, linking #1213 |
| U2 | The agent identity can't run `release download` (allowlist) or re-run a workflow (the App token is refused: "Resource not accessible by integration"), so the human re-ran Run 1's release-PR jobs and agents fetched evidence bundles with curl | ts-lib#3, ts-app#3 | **Found:** .github#497, with this evidence added as a comment |
| U3 | After `vrg-release` fails at `merge-release` on a red check, recovery needs a manual re-run of the release PR's CI before `--resume` | ts-lib#3, ts-app#3 | **Found:** .github#451, the `vrg-release` recovery umbrella, with this evidence added as a comment |
| U4 | An editor's Prettier and `prettier --write .` don't know Vergil's excludes | vergil-tooling PR #4544 | **Dropped:** a documented trade-off, with guidance on the standards page |
| U5 | `releasing.md`'s Python `.venv` and host-install rows may be dated; the doc sweep noticed but didn't check | the #4451 run | **Filed:** vergil-project/.github#1286 |
| U6 | The `ts-node` images carry no Vergil release label, so #1207 had to tie digests to v2.1.44 by build timing | #1207 | **Filed:** vergil-project/.github#1287 |
| U7 | `license-checker --version` exits 1 | #1207 | **Found:** recorded on #1209, epic 3's planning item, with the workaround in place |
| U8 | #1161's plan pushback round 2 records "Discards: 10" with no reasons | #1161 comment 6925220464 | **Dropped:** planning is closed, and every other round lists its discards |
| U9 | The fixture-run serialization rule deadlocks two runs that are filed but not started | plan evolution entry 3 | **Dropped** by the human's decision during the epic: the human coordinates for now |
| U10 | `vrg-version show` gives a traceback for a non-UTF-8 `package.json`, and a missing `package.json` is reported twice by the pre-gate | #4507 merge review | **Filed:** vergil-project/.github#1288 (low) |
| U11 | #4553's Prettier and markdownlint tests on generated notes are skipped where those tools are absent | #4553 merge review | **Dropped:** the resumed Run 1 releases passing LINT on v2.1.321, and Run 2's releases, are the live proof |

A security-class item count: 3 design-review security records and 1 follow-up,
each handed to its task or its own issue during planning and execution; none is
untracked.

## 5. What's next

- **Epic 2 of the TypeScript-parity wave** (lifecycle PRs, the catch-up to Node
  26, toolchain enforcement): `epic-create` is under way on its planning item,
  vergil-project/.github#1208, which carries this epic's notes for it. The
  follow-on brainstorm bookend (#1176) closed with that planning started.
- **Epic 3** (supply chain and the npm updater): vergil-project/.github#1209,
  with this epic's notes and the four known-gap records. When the npm updater
  lands, the `fixture-run` TypeScript known-gap row goes and the rollback turns
  on.
- **Epic 4** (the Node runtime architecture): vergil-project/.github#959. **Epic
  5** (the mnemosys CLI from melete's TypeScript tool): its planning item is
  still to come.
- **The remaining planned follow-ups:** TypeScript 7, deferred to #1181;
  Python's BUILD check (#1211); moving Python's standalone tools to the image
  under the new guideline (its planning item); and the BUILD evidence gap
  (#1250).
- **Not yet released:** vergil-actions#1288's doc PR (#1292) is in no
  vergil-actions release yet, as of v2.1.81.

## Appendix A — Operational notes

The order that worked, for the next language:

1. Land the tooling, actions and plugin work, then release **tooling first**.
   The actions `build` job reads tooling's `build-enabled` flag, and the report
   contract needs tooling's reports. Actions-first was harmless here only
   because no TypeScript consumer existed yet.
2. Attest the releases and deploy the plugin (#1205), then create the fixture
   repos with `vrg-github-repo-init` (#1206). The App secrets are org secrets in
   vergil-project, so no per-repo secret step is needed.
3. Write the fixture programs, file Run 1 with `fixture-run`, and expect
   defects. Each fix that the programs or releases need goes in as a task with
   its own release, and the blocked fixture tasks get `Blocked-by:` it.
4. After a tooling release, a branch cache built earlier may still hold the old
   tooling until #4550 ships everywhere. `vrg-container-cache clean`, then
   `build`, refreshes it.
5. If `vrg-release` stops at `merge-release` on a red check: fix the cause,
   re-run the release PR's failed jobs (the human, since the agent's App token
   can't), then run `vrg-release --resume`.
6. Change the image only after Run 1 passes (Task 17), release it, confirm the
   prod images (#1207), then file Run 2. Check for other epics' fixture runs on
   the same fixtures before filing.
