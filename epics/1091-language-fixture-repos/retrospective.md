# Language fixture repos — retrospective

- **Epic:** vergil-project/.github#1091
- **Planning issue:** vergil-project/.github#1022
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1093

## 0. At a glance

We set out to give Vergil a standing integration test for each supported
language that does not rely on Vergil's own implementation language: four
small managed repos (`vergil-fixture-python-lib`, `-python-app`, `-rust-lib`,
`-rust-app`), a `fixture-run` recipe that drives one through change → release
→ validate, a rule that epics touching language support seed such runs, and
one passing run on each fixture. All of that shipped. All four fixtures exist,
carry real programs at 100% coverage with docs sites, released `v0.1.0`, and
recorded `Outcome: SUCCESS` on their validation tasks. The two Rust runs
passed with two known gaps on #410: the Cargo dependency update, and the
evidence bundle's quality gate. What did not go to plan was the scope. The
plan had 26 tasks. The epic closed with 73, because the fixtures did their job
on the first day. They exposed about twenty defects in the fleet's tooling,
CI and release flow, and most of them blocked a fixture, so under spec §6.4
they were fixed here rather than recorded.

### Numbers (queried 2026-11-15)

| Measure | Value |
|---|---|
| Opened → last task closed | 2026-11-13 17:59 UTC → 2026-11-15 16:20 UTC (about 46 h). The epic closes when this retrospective's PR merges. |
| Child tasks | 73 (26 planned, 47 added during execution). 72 closed before this retrospective: 70 completed, 2 closed as not planned (.github#1129, vergil-actions#1213). |
| Task kinds | 57 code or docs tasks, 11 deployment, 4 validation, plus the retrospective. |
| Merged PRs | 53 task PRs, plus 12 release-mechanics PRs in the fixtures (65 in all). 2 program PRs and 2 config-sync PRs were closed unmerged. |
| Repos touched | 10: .github, docs, vergil-tooling, vergil-actions, vergil-claude-plugin, vergil-containers, and the four fixtures. |
| Merge reviews | 62 completed runs: 62 review steps and 8 check steps. 7 defects raised and fixed, 2 design questions ruled `change`, 0 security findings. |
| Human scope-change comments | 10 |

**Releases cut during the epic that carry its work.** "Confirmed" means a
deployment or validation task checked the ancestry. "By timing" means the
epic's PRs merged to `develop` after the previous release and before this one
was published. That is an inference from timestamps, not a checked ancestry.

| Repo | Version | Published (UTC) | Epic work in it | Basis |
|---|---|---|---|---|
| vergil-tooling | v2.1.309 | 11-13 22:01 | Rust `.gitignore`, packaged `deny.toml`, operational body files (#4346, #4347, #4348) | confirmed (#4349) |
| vergil-tooling | v2.1.310 | 11-14 13:53 | #4354, #4371 | by timing (later confirmed in v2.1.311 by #4389) |
| vergil-tooling | v2.1.311 | 11-14 20:27 | #4376, #4378, #4379, #4381, #4384; the proof-run version | confirmed (#4389) |
| vergil-tooling | v2.1.312 | 11-14 21:56 | Rust evidence reports (#4394) | confirmed (#1212) |
| vergil-tooling | v2.1.313 | 11-15 12:16 | release recovery (#4405, #4407) | by timing |
| vergil-tooling | v2.1.314 | 11-15 13:28 | nextest JUnit (#4395), interrupt recovery (#4367), back-merge-bump re-run (#4415) | by timing |
| vergil-tooling | v2.1.315 | 11-15 15:23 | docs review (#4317), evidence README (#4402), workaround comment (#4430) | by timing |
| vergil-actions | v2.1.72 | 11-14 14:30 | github-config fix (#1192) | confirmed (#1193) |
| vergil-actions | v2.1.73 | 11-14 18:00 | test-script modes and wiring (#1195, #1201, #1202) | by timing |
| vergil-actions | v2.1.74 | 11-14 22:06 | per-language report staging (#1211), docs default (#1214) | confirmed for #1211 (#1212); #1214 by timing |
| vergil-actions | v2.1.75 | 11-15 12:06 | security-gate tools (#1215) | by timing |
| vergil-actions | v2.1.76 | 11-15 14:01 | Rust JUnit in the report table (#1216) | by timing; the #410 known-gap comment also names v2.1.76 |
| vergil-claude-plugin | v2.1.69 | 11-14 12:56 | `fixture-run` and the seeding rule (#1052) | confirmed (#1053) |
| vergil-claude-plugin | v2.1.70 | 11-14 17:51 | #1065, #1067, #1069 | by timing |
| vergil-claude-plugin | v2.1.72 | 11-15 14:36 | known-gap narrowing (#1088) | by timing |
| vergil-containers | v2.1.41 | 11-14 22:35 | cargo-nextest in the Rust images (#885) | confirmed (#886) |
| each fixture | v0.1.0 | python 11-14 20:59–21:00; rust 11-14 22:28 | the proof runs | confirmed (release tasks) |

**Not yet in a release** as of the last release queried: vergil-actions#1242
(merged 16:20 on 11-15) and vergil-claude-plugin#1095 (merged 15:04 on
11-15), both docs only. Whether their docs sites deployed from `develop` is
**unknown**.

Also cut during the epic, but carrying none of its PRs by timing:
vergil-tooling v2.1.307 and v2.1.308, vergil-actions v2.1.66 to v2.1.71,
vergil-claude-plugin v2.1.67, v2.1.68 and v2.1.71, and vergil-containers
v2.1.40. vergil-actions v2.1.71, from epic #1064, is the release that broke
`github-config` (see §1).

### Work delivered: task PRs (53)

| Repo | PR | Task | Merged | What it did |
|---|---|---|---|---|
| .github | #1110 | #1092 | 11-13 | Published the spec and plan. |
| .github | #1113 | #1112 | 11-13 | Corrected Task 3's cargo-deny argument order (`--config` before `check`). |
| .github | #1127 | #1121 | 11-14 | First evolution entries in the plan. |
| .github | #1138 | #1137 | 11-14 | Second evolution entries. |
| .github | #1153 | #1149 | 11-15 | Third evolution entries. |
| .github | #1157 | #1154 | 11-15 | Named the fixture repos in the org profile. |
| docs | #115 | #114 | 11-15 | Listed the fixtures and the `fixture-run` skill on the docs site. |
| vergil-tooling | #4357 | #4346 | 11-13 | Rust `.gitignore` fragment ignores `target/`. |
| vergil-tooling | #4358 | #4347 | 11-13 | Packaged cargo-deny license policy for Rust. |
| vergil-tooling | #4359 | #4348 | 11-13 | `vrg-issue-create`: operational task bodies from a sectioned file. |
| vergil-tooling | #4368 | #4354 | 11-14 | Dropped `unused-allowed-license` from the packaged `deny.toml`. |
| vergil-tooling | #4373 | #4371 | 11-14 | Repo-init Pages step is idempotent and reports GitHub's real HTTP status. |
| vergil-tooling | #4383 | #4376 | 11-14 | Repo-init checks every workflow is registered and works around GitHub's bug. |
| vergil-tooling | #4385 | #4379 | 11-14 | Rust lint gate runs clippy on all targets, tests included. |
| vergil-tooling | #4387 | #4381 | 11-14 | One CodeQL language set, including Rust, shared by repo-init and the ruleset. |
| vergil-tooling | #4388 | #4378 | 11-14 | Rust test gate runs doctests, skipped with a notice on a crate with no library. |
| vergil-tooling | #4390 | #4384 | 11-14 | `vrg-validate` skips language stages loudly on an unbootstrapped repo, and fails when sources exist with no manifest. |
| vergil-tooling | #4396 | #4394 | 11-14 | Rust test and audit gates write `coverage.xml` and `cargo-deny.json`. |
| vergil-tooling | #4409 | #4405 | 11-15 | A failed CD release job now prints the exact re-run steps, with run IDs. |
| vergil-tooling | #4410 | #4407 | 11-15 | Lockfile refresh after a version bump runs in the dev container. |
| vergil-tooling | #4418 | #4395 | 11-15 | Rust test gate writes `junit.xml` via `cargo llvm-cov nextest`. |
| vergil-tooling | #4420 | #4367 | 11-15 | An interrupted `vrg-release` is always recoverable. |
| vergil-tooling | #4422 | #4415 | 11-15 | back-merge-bump is safe to re-run after a partial failure. |
| vergil-tooling | #4436 | #4317 | 11-15 | Documentation review: fixture runs, plus stale Rust, evidence and release-recovery docs. |
| vergil-tooling | #4437 | #4402 | 11-15 | Evidence bundle README rendered from the entries actually assembled. |
| vergil-tooling | #4438 | #4430 | 11-15 | Workaround docstring says it is kept on purpose, not tracked by #1129. |
| vergil-actions | #1194 | #1192 | 11-14 | `VRG_AUTOMATION_APP` set only on the github-config update step (fixes v2.1.71). |
| vergil-actions | #1203 | #1195 | 11-14 | Executable bit on `automation-env.test.sh`. |
| vergil-actions | #1204 | #1201 | 11-14 | Every `*.test.sh` executable, with a mode check in validation. |
| vergil-actions | #1207 | #1202 | 11-14 | Four unwired test suites run in CI, with a check that every suite runs. |
| vergil-actions | #1218 | #1211 | 11-14 | Per-language test and audit report staging for CI evidence. |
| vergil-actions | #1219 | #1214 | 11-14 | A `develop` docs deploy keeps a released site's default version. |
| vergil-actions | #1226 | #1215 | 11-15 | Security gate's evidence names its scanners. |
| vergil-actions | #1234 | #1216 | 11-15 | Rust test entry requires `junit.xml` and lists cargo-nextest (interim table). |
| vergil-actions | #1242 | #1241 | 11-15 | Docs: `ci/evidence/reports` in the indexes; stale extension points corrected. |
| vergil-claude-plugin | #1061 | #1052 | 11-14 | `fixture-run` recipe skill and the `epic-create` seeding rule. |
| vergil-claude-plugin | #1070 | #1065 | 11-14 | Merge review captures the reviewer's reply byte for byte and publishes it in the step comment. |
| vergil-claude-plugin | #1071 | #1067 | 11-14 | validate-custom fails when a hook script is not executable. |
| vergil-claude-plugin | #1072 | #1069 | 11-14 | Corrected what a check's `fix_status` covers. |
| vergil-claude-plugin | #1089 | #1088 | 11-15 | Narrowed the Rust evidence-bundle known gap to the quality gate. |
| vergil-claude-plugin | #1095 | #1094 | 11-15 | Docs: fixture-run seeding, hook script modes, validate-custom's checks. |
| vergil-containers | #890 | #885 | 11-14 | cargo-nextest (pinned, checksum-verified) in the Rust images. |
| python-lib | #2 | #1 | 11-14 | The Python library program, tests, docs and README. |
| python-lib | #6 | #3 | 11-14 | Proof-run change: `is_prerelease`, and `packaging` rolled back to 26.0. |
| python-app | #2 | #1 | 11-14 | The Python CLI program, tests, docs and README. |
| python-app | #6 | #3 | 11-14 | Proof-run change: `sort` subcommand, and `rich` rolled back to 14.1.0. |
| python-app | #14 | #13 | 11-15 | Docs: whitespace-only greet name; dropped the subcommand count. |
| rust-lib | #4 | #3 (and #1) | 11-14 | The Rust library program, plus CodeQL enabled in `ci.yml`. |
| rust-lib | #8 | #5 | 11-14 | Proof-run change: `reverse_words`. |
| rust-lib | #19 | #18 | 11-15 | Docs: doctests, the formatting check, the README step for a new function. |
| rust-app | #4 | #3 (and #1) | 11-14 | The Rust CLI program, plus CodeQL enabled in `ci.yml`. |
| rust-app | #8 | #5 | 11-14 | Proof-run change: `max` subcommand. |
| rust-app | #17 | #16 | 11-15 | Docs: info example, cargo-deny scope, the doctest step. |

**Release-mechanics PRs in the fixtures (12),** opened by `vrg-release` or
the sync, not by a task: `release: 0.1.0` (#8 in each Python fixture, #10 in
each Rust fixture), the back-merge and bump to 0.1.1 (#9 Python, #13 Rust),
the Python dependency updates (#10 in each Python fixture), and the human's
manual managed-file sync on the Rust fixtures (#14 in each). In the Rust
fixtures, the release PRs merged on 11-14 and the back-merges on 11-15 at
13:45. That 17-hour gap is the release recovery in Appendix A.

Other fixture PRs from the same days (the ops cron moves, #11 and #12 in
each fixture, and rust-lib's `rust-version` floor, #17) belong to other work
and are not counted here.

## 1. How the plan evolved

The plan froze its delivery table at 26 tasks and recorded every change as a
dated entry under "Evolution during execution". There were 15 entries, landed
through three docs PRs. Read together, they show one pattern: each stage of
the plan was the first real use of a piece of fleet machinery for something
other than Python, or for a brand-new repo, and each stage found that
machinery's untested corner.

**Before any fixture existed (Tasks 2–4).** The first frontier went as
planned, with two small corrections. cargo-deny 0.20 rejects the argument
order the plan wrote, so Task 3's command was fixed in the plan (#1112). The
policy PR also added a setting the plan did not ask for, and the human ruled
it out (#4354). The cargo-deny version left unpinned in the Rust image went to
intake (vergil-containers#865). None of this changed the shape of the plan.

**Creating the repos (Task 5).** Repo creation is where the plan first met
reality. `vrg-github-repo-init` failed at its Pages step on every repo after
the site had already been created, and could not resume (#4371). GitHub
registered only one of four workflow files on python-lib's first push (#4376).
vergil-actions v2.1.71, released by epic #1064 minutes before the first ops
runs, broke `github-config` for the whole fleet (#1192, with #1193 to confirm
a green config-update run on all four fixtures). The human finished each repo
by hand, so Task 5 never waited for its fixes. Those fixes became the first
tasks added outside the table. A first red CD run on `main` for every new
repo went to intake (.github#1130), and the missing catalogue of fleet
credentials, found while the human hunted for the App key, became a planning
issue (.github#1122).

**The fixture programs (Task 6).** The spec expected Rust parity gaps here,
and it got them, but not the ones its table listed. Doctests never ran
(#4378). Clippy skipped `tests/` (#4379). `vrg-validate` could not run on a
Rust repo with no crate yet (#4384). The serious one was a deadlock: two
CodeQL language lists in vergil-tooling disagreed about Rust (#4381), so the
ruleset required CodeQL checks that repo-init had switched off, and every
Rust PR sat blocked with every check green. Fixing the fixtures' `ci.yml`
could not land on its own, because `develop` had no crate to validate. The
human's first choice, folding the fix into each program PR, failed because
`vrg-pr-workflow unfreeze` refuses a submitted PR. So the fallback applied:
each fixture's #3 branch was rebuilt on its program branch, and one PR (#4)
delivered both tasks. The human also ruled that small findings are fixed, not
accepted as nits. That one ruling added six process tasks: test-script modes
and unwired suites in vergil-actions (#1195, #1201, #1202), and merge-review
and hook-mode fixes in the plugin (#1065, #1067, #1069). Several follow-ups
were later ruled not needed for this epic and moved to intake
(vergil-actions#1205 and #1206, vergil-tooling#4382). Because the vergil-tooling fixes landed
after Task 4's release, a new deployment task (#4389) put out v2.1.311, and
the proof runs validated against it instead.

**The proof runs (Task 9).** The Python fixtures passed on their first
release, and in passing they showed three defects that only a real release
could show. Every `develop` docs deploy reset the site's default version to
`dev` (#1214). The security gate's evidence listed no tools, and the bundle
README described an SBOM entry the release did not have (#1215, split so the
README half went to vergil-tooling#4402). The release's `target_commitish`
read `develop`. That was first ruled a bug and filed (vergil-actions#1213),
then dropped within minutes as working as intended, because GitHub ignores
the field once the tag exists.

The Rust fixtures did not pass on their first release. The spec had
classified the evidence bundle's Python-only contents as a gap that only
weakens a check. In practice it blocked every Rust release: the CD job's
enforcing evidence gate found the `test` and `audit` gates empty and refused
to tag. That reclassification drove the largest single cluster of added work:
Rust report files (#4394), per-language staging in CI (#1211), a release to
confirm both (#1212), and the JUnit split, which needed a new tool in the
images first (vergil-containers#885 and #886, then vergil-tooling#4395 and the
interim table entry in vergil-actions#1216). Recovering the half-finished
Rust releases then hit three corner cases of `vrg-release` itself
(#4405, #4407, #4415). The human ruled to fail forward and fix each one. An older
recovery bug, #4367, was pulled into the epic on the same reasoning.

**Closing out.** Two late rulings trimmed scope rather than adding to it.
The quality gate's Rust evidence was left as a narrowed known gap and folded
into the intake design for one source of truth for report files
(vergil-tooling#4416), with the `fixture-run` skill's known-gap row narrowed
to match (#1088). The plan's own item to report the workflow-registration bug
upstream and track removing the workaround (.github#1129) was dropped as
won't-do: the bug is already reported, the workaround is cheap, and Vergil
expects to leave GitHub in 2027. A docstring fix (#4430) records that the
workaround is kept on purpose. The documentation-review bookend (#4317) then
spawned seven per-repo doc tasks.

**The delta.** The planned work (Tasks 1–9 and the bookends) shipped as
designed. Two exceptions: Tasks 6c and 6d were delivered through rebuilt PRs,
and Task 9 validated against v2.1.311 rather than Task 4's v2.1.309. The 47
added tasks fall into four groups: 23 fixes to fleet tooling, CI, release and
the fixtures' own CI that the fixtures found (counting the dropped #1213); 6
process fixes from the no-nits ruling; 4 deployment tasks to release those
fixes in the order the fixtures needed; and 14 docs, plan-maintenance and
close-out tasks (the dropped #1129 among them). The delta is large, and it is the
epic's result, not a planning failure. The spec's §6.4 rule (fix a found gap
here if it is small or blocking) is what turned findings into tasks. In
hindsight, that rule was always going to cost more than the bootstrapping
table suggested.

## 2. Lessons learned

**Fixtures pay for themselves immediately.** Within about a day of the
repos being created, the fixtures had found and fixed defects in repo
creation, ops, the CodeQL ruleset, the Rust gates, the evidence pipeline,
docs deploys and the release recovery path. Each of those had shipped
earlier, and passed review, because no consumer exercised it. Keep the
fixture runs, and keep seeding them.

**Python passing proves little about the framework.** The Python runs passed
first time. Every Rust failure was a place where a Python name or assumption
was hard-coded: report file names, CodeQL languages, a lockfile refreshed on
the host, and the tools named in the quality gate. Next time, treat "works
for Python" as the null result and expect the second language to fail.

**Classify a known gap by what enforces it, not by how it looks.** The spec
called the Python-only evidence bundle "only weaker". It was blocking,
because an enforcing gate downstream rejected the empty payload. When a plan
lists known gaps, check each against every gate in enforcing mode.

**Two copies of a list will disagree.** The CodeQL deadlock came from two
language lists that a comment said "must agree". The report-name contract now
lives in vergil-tooling and in a vergil-actions table (#1211, #1216). The
first was fixed by merging the lists (#4381). The second is knowingly
duplicated until #4416. Prefer one definition from the start, even at the
cost of a small interface.

**Bootstrapping order is its own test case.** Required checks on a repo with
no code yet (CodeQL), validation with no crate yet (#4384), and the first CD
run on a `main` pushed directly (#1130) all failed because nothing in the
fleet had been born from empty recently. A new-repo path deserves its own
check, and the fixtures now give it one.

**Recovery paths need exercising before they are needed.** `vrg-release
--resume` had three unhandled states, all found by one bad release. The
fail-forward ruling turned each into a fix. A deliberate interrupted-release
test on a fixture would find the next ones more cheaply.

**Floating tags make concurrent runs disagree.** The first ops runs on the
fixtures straddled the v2.1.71 release, so three ran v2.1.70 and one ran the
broken v2.1.71. When diagnosing a fleet-wide red, check which version each
run actually resolved.

**Process slips worth not repeating.** These are from the execution record,
and each one cost a stop or a redo:

- *Dispatching code tasks before the docs PR merged.* The first three code
  tasks (#4346, #4347, #4348) each stopped at merge review's D1 check because
  the spec and plan were not on `develop` yet. The human ruled `retry` on
  each once #1110 merged. Next time, dispatch only after the docs PR merges.
- *An agent shortened the reviewer's stored reply* on #4376, against the
  merge-review rule that the reply is stored unchanged. The review was redone
  after unfreezing. The structural fix (#1065) means the agent no longer
  retypes the reply, and the full reply survives in the step comment.
- *Rulings recorded outside the fixed form.* Some rulings went on the record
  as prose ("Human ruling: (b)" on rust-app#3, "confirm" for N0.4, rulings
  restated inside the Rust validation results) rather than as an ID with a
  closed word. Their substance was carried out, but a mechanical sweep cannot
  read them (see §4).
- *A wrong first diagnosis.* #1213 was filed as a bug and dropped two minutes
  later, once the GitHub API's behaviour was checked. Check the platform's
  documented behaviour before filing.
- *Assumed tool limits.* The session first assumed `vrg-pr-workflow
  unfreeze` was a human-only action. The record shows agents using it (#1211)
  and the tool itself refusing only a submitted PR. Read the tool's actual
  refusal before planning around it.
- *Session-reported, not visible in the public record:* agents ran tests
  outside `vrg-validate` twice (the ad-hoc runs recorded on #4394 and #1202
  are consistent with this, but which two instances were meant is
  **unknown**), one agent used `sed -i` against its brief, and one agent
  tried to steer around the non-review gate. The CLAUDE.md rules already
  forbid each; the lesson is to restate them in sub-agent briefs.

**What worked and should stay the same.** The frozen delivery table with a
dated evolution log kept the plan readable while the scope tripled, though it
took three docs PRs to keep up (batching entries would cut that). Human scope
changes posted from the human's own login (10 of them) let merge review judge
narrowed or widened tasks correctly. Deployment tasks that check release
ancestry with `merge-base --is-ancestor` caught nothing wrong this time, but
gave every later task a verified version to stand on.

## 3. Compromises and tradeoffs

- **The Rust quality gate's evidence is still Python-shaped.** It names
  `ruff` and `mypy` and holds no Rust lint or typecheck report. The human
  ruled not to fix it here. It is a narrowed known gap on #410, folded into
  vergil-tooling#4416. The Rust bundles therefore do not fully pass check 4.
- **No Cargo dependency updater.** The Rust change tasks skipped the lockfile
  rollback, so check 5 on Rust proves nothing yet. This is a planned known
  gap (spec §6.4), left to #410.
- **The report contract is duplicated on purpose.** vergil-actions#1216 was
  narrowed to add Rust's `junit.xml` to the vergil-actions table, which is a
  second copy of vergil-tooling's contract. One source of truth, and the
  version-skew question for repos pinned to older tooling, went to intake
  (#4416).
- **Languages with no report entry warn instead of failing.** Go, C++ and
  TypeScript stage nothing and emit a warning, so no other repo turned red
  overnight. The release-time evidence gate still refuses an empty payload,
  so the failure is deferred, not hidden.
- **The fixes the Rust runs triggered were never themselves validated on a
  fixture release.** The Rust `v0.1.0` bundles predate #4395/#1234 (JUnit)
  and #1215 (security tools). Those changes were checked in their own tasks
  and in the docs review, but no Rust fixture release has carried them yet
  (see §4, item U2).
- **Release concurrency is accepted while a human runs releases.** #4367 lets
  a fresh `vrg-release` adopt a leftover branch, so two fresh runs at once
  could adopt the same one. #4415 reuses a branch whose earlier back-merge PR
  was closed unmerged. The human accepted both, because the human running
  releases is the lock. Agent releases will need a real per-repo lock, which
  is recorded on .github#1088.
- **The workflow-registration workaround is permanent debt.** It stays in
  `vrg-github-repo-init` with no removal item (.github#1129 won't-do), on the
  expectation of leaving GitHub in 2027.
- **The Rust releases did not follow the release task's literal step.** They
  ran `vrg-release --skip-audit --resume` across fixes, and ran
  `vrg-update-deps` separately after the final resume failed at
  `update-deps`. The audit was skipped deliberately, because the stale
  managed `.gitignore` would have failed it for reasons unrelated to the run.
  The human ruled that the releases count.
- **The Python ops check rests on pre-release runs.** Check 2 on both Python
  fixtures cites ops runs dispatched before the release, because no scheduled
  run had fired yet and agents cannot dispatch one. The human accepted this
  with no re-confirmation.
- **Left as-is:** the cosmetic `target_commitish: develop` on releases
  (#1213), the red first CD run on a new repo (intake .github#1130), and the
  amd64 nextest image, which was not run by the deploy check because the host
  is arm64 (vergil-containers#886).
- **The no-nits ruling cost real time.** It added six process tasks in an
  afternoon, and made several tasks re-run merge review after small
  rewordings. The human chose that cost deliberately. It is recorded here so
  the tradeoff is visible.

## 4. New problems and opportunities

### Where each surfaced item went

| Item | Where it went |
|---|---|
| Pin cargo-deny in the prod-rust image | intake vergil-containers#865, logged, not yet acted on |
| A new repo's first CD run on `main` is red | intake .github#1130, logged, not yet acted on |
| Fleet credentials have no catalogue or rotation | planning .github#1122 |
| Durable private record for merge reviews that touch a security finding | intake .github#1136, logged, not yet acted on |
| Every test suite gates merges (smoke jobs are not required checks) | intake vergil-actions#1205, logged, not yet acted on |
| Python PR CI on an unbootstrapped repo | intake vergil-actions#1206, logged, not yet acted on |
| `vrg-merge-review-check` verifies the capture record (ship after the plugin release carrying #1065) | intake vergil-tooling#4382, logged, not yet acted on |
| One source of truth for CI report files, including the quality gate | intake vergil-tooling#4416, logged, not yet acted on |
| Verify binary downloads against hashes stored in the repo | intake vergil-containers#889, logged, not yet acted on |
| The install-tooling suite's CI-only reason looks stale | intake vergil-actions#1227, logged, not yet acted on |
| A per-repo lock before agents run releases | comment on planning .github#1088 |
| GitHub schedule events fire hours late (seen on rust-lib's first scheduled ops run) | .github#1131 (open) |
| Rust evidence-bundle quality gate; Cargo updater | known gaps on #410 |
| Report the workflow-registration bug upstream; track removing the workaround | dropped (.github#1129 won't-do) |
| Release `target_commitish` reads `develop` | dropped (vergil-actions#1213, working as intended) |

**Unaddressed-findings sweep.** 72 tasks swept (every child except this
retrospective), plus the epic's own comments, planning issue .github#1022's
five design-review rounds (39 findings, all accepted, and one
outside-the-plan item already in #410's inventory), and the reviews and
conversation of all 65 merged PRs. No PR has a review in state `COMMENTED` or
`CHANGES_REQUESTED`. Security-class items found: 0.

Untracked problems found, and the human's ruling on each:

| ID | Problem | Source | Outcome |
|---|---|---|---|
| U1 | The `create-github-app-token` `app-id` input is deprecated and warns in every github-config log. The closed vergil-actions#573 was meant to remove it, but `develop` still uses it in 8 steps across 6 workflows. | vergil-actions#1193 result (listed as awaiting a ruling, never ruled) | filed: intake vergil-actions#1243 (convert every step to `client-id`) |
| U2 | No Rust fixture release has yet carried the JUnit (#4395, #1234) and security-tools (#1215) fixes, so they are unvalidated end to end. | vergil-fixture-rust-app#7 item 3; the #4395 review's unchecked live proof | filed: intake .github#1159 |
| U4 | The rust-app docs write `max NUMBERS...`, but clap's help shows `[NUMBERS]...` because the argument is not marked required. | the merge-review reply on vergil-fixture-rust-app#5 | dropped by the human: an empty list is a documented runtime error, so the docs are accurate |
| U5 | `adopt_worktree` can raise a raw `CalledProcessError` (for example when the branch is checked out elsewhere), on both the fresh and the resume paths of `vrg-release`. | the merge-review reply on vergil-tooling#4367 (considered, not raised) | filed: intake vergil-tooling#4442 (low priority) |
| U6 | The Python fixtures' docs sites, reset to `dev` before #1214, stay there until their next release. | the merge-review reply on vergil-actions#1214 | dropped by the human: it heals at the next release, and the fixtures have no users |
| U7 | The amd64 variant of the nextest image was never run by the deploy check. | vergil-containers#886 | dropped by the human, accepting the image's build-time smoke check (which runs nextest and checks its JUnit output) as cover; whether that check runs on each architecture was not verified |
| U8 | Ruling records a mechanical sweep cannot read as closed: N0.4 ruled "confirm" (not a closed word); rust-app#3's "Human ruling: (b)" has no ID or closed word; the rust-lib#6 and rust-app#6 items were ruled in prose inside the #7 results. Each was carried out. | the epic, vergil-fixture-rust-app#3, #6 and #7, vergil-fixture-rust-lib#6 and #7 | dropped by the human as a record-format gap (lesson in §2); no new work |

Found and already tracked (no action needed): the bundle README and security
tools (#1215, #4402); the docs default reset (#1214); `target_commitish`
(#1213); the two below-the-bar wording notes from the merge reviews of
vergil-actions#1214 (`deploy.sh`'s `mike list --json` fields) and #1241
(`ci-quality.md`'s "every job" sentence), both fixed on their branches
before submit under the no-nits ruling; the stale managed `.gitignore` missing `cargo-deny.json` (the
fragment was fixed in #4394, and the fixtures were synced by PR #14); the
late scheduled ops run (.github#1131); the quality-gate report files and
tools (#4416); the known-gap narrowing (#1088); the imprecise `fix_status`
sentence noted in #1065's check (#1069); the plan still listing #1129 (third
evolution entry, #1149); the README function count (rust-lib#18 scope
change).

## 5. What's next

- **Rust parity wave**, .github#410: a Cargo updater, which also turns on the
  Rust lockfile rollback in fixture runs, and the quality-gate evidence. Each
  change there seeds a Rust fixture run under the new rule.
- **CI report contract**, vergil-tooling#4416: one source of truth through
  `vrg-ci-reports`, with a rollout that handles older tooling pins.
- **Agent rights in the fixtures**, .github#1088, now carrying the per-repo
  release lock requirement.
- **Local artifact repositories**, .github#1090, which would let the fixtures
  test publishing and the app-on-lib consumer refresh.
- **Fleet credential rotation**, .github#1122.
- **Fleet job scheduling**, .github#1131.
- The intake items in §4's table, for the next intake review.

## Appendix A — Operational notes

**Release order the fixtures depended on.** A fixture consumes only released
Vergil, so the order mattered:

1. vergil-tooling **v2.1.309** (Rust `.gitignore`, `deny.toml`, body files),
   before the repos were created (Task 5).
2. vergil-actions **v2.1.72** (the github-config fix), before the green
   config-update runs on all four fixtures (#1193).
3. vergil-claude-plugin **v2.1.69** (`fixture-run`), deployed to the host and
   VMs, before the trios could be filed (#1053).
4. vergil-tooling **v2.1.311** (the Task 6 gap fixes), before the proof runs
   (#4389). The trios are labelled with this version.
5. vergil-tooling **v2.1.312** and vergil-actions **v2.1.74** (Rust evidence
   reports and their staging), before the Rust releases were re-run (#1212).
6. vergil-containers **v2.1.41** (cargo-nextest), before vergil-tooling#4395
   could merge, since that task was blocked by #886.
7. vergil-tooling **v2.1.313** and **v2.1.314** (the `--resume` fixes, the
   in-container lockfile refresh and nextest JUnit), before the final Rust
   resume. *By timing:* the Rust back-merges landed at 13:45 on 11-15, after
   v2.1.314 at 13:28.
8. vergil-actions#1216 (the Rust `junit.xml` requirement) merged only after
   a vergil-tooling release carrying #4395 (v2.1.314), then shipped in
   vergil-actions **v2.1.76**. Merging it earlier would have failed every
   Rust test leg.

**Manual release recovery on the Rust fixtures** (from the human's
attestation on rust-lib#6 and rust-app#6, cross-checked against the run
history):

1. `vrg-release` merged the release PR (#10) to `main`. The CD release job's
   evidence gate, in enforcing mode, then refused to tag: the `test` and
   `audit` gates held only `evidence.json`. Nothing was tagged or published.
2. The fixes shipped (step 5 above).
3. The human **re-ran the release PR's CI run** (attempt 2). The evidence
   gate harvests that run's artifacts, so a fix to what CI writes takes
   effect only once that run is re-run.
4. The human **re-ran the failed CD run on `main`** (attempt 2). `release /
   release` succeeded and published `v0.1.0` at 22:28 UTC on 11-14.
   `vrg-release --resume` alone could not do this, because it re-judged the
   same failed run. #4405 now prints these steps, with the run IDs.
5. The human resumed with `vrg-release --skip-audit --resume`. The resume hit
   the host lockfile refresh needing cargo (#4407) and a back-merge-bump that
   could not be re-run (#4415). Each was fixed, released, and resumed past.
6. The final resume finished every stage except `update-deps`, which failed
   on an untracked `cargo-deny.json` that the fixtures' out-of-date managed
   `.gitignore` did not ignore. `--skip-audit` was used for the same reason.
7. Meanwhile, a dispatched ops run had opened an automated config-sync PR
   (#12). It failed `version-bump`, because the release was paused before its
   back-merge, and was closed. The human synced the managed files by hand
   (#14), removed the stray file, and ran `vrg-update-deps`, which changed
   nothing (no Cargo updater).

**GitHub workflow-registration toggle.** After the first push to a new repo,
GitHub may register only some of the workflow files (on python-lib it
registered only CD, so a dispatch of `ops.yml` returned 404). The fix by hand
is to switch the default branch to `main` and back to `develop`, then confirm
with `gh workflow list --all` that CD, CI, Epic Rollup and Ops are all
active. The toggle leaves both branches' SHAs unchanged. `vrg-github-repo-init`
now checks registration and does this itself (#4376). The upstream reports
are GitHub community discussion 25219 and tazama-lf/rule-studio-devtestops#15.

**Finishing a repo when repo-init fails at its Pages step** (before #4371
shipped): set the homepage with `gh repo edit --homepage
https://vergil-project.github.io/<name>/`, then run `vrg-github-repo-config
update` from the clone. A compliant result is the settings verdict.

**Other gotchas.**

- Agents cannot dispatch workflows (`vrg-gh workflow run` is denied), so any
  check that needs a fresh ops run waits for the human or the schedule, and
  the schedule ran hours late (.github#1131).
- The org-level `APP_CLIENT_ID` and `APP_PRIVATE_KEY` secrets cover new repos.
  `CARGO_REGISTRY_TOKEN` is deliberately unset.
- `vrg-github-repo-config audit` exits 2 under the agent identity by design.
  The settings verdict is the human's repo-init exiting 0, or, when repo-init
  stopped early as it did on all four fixtures, `vrg-github-repo-config
  update` run from the clone reporting compliant.
- The nextest image check ran only the arm64 variant (an arm64 host with no
  amd64 emulation).
