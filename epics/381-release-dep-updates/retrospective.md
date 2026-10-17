# Retrospective — release-integrated dependency updates

> **Epic:** vergil-project/.github#381 · **Spec:** `epics/381-release-dep-updates/spec.md` · **Plan:** `epics/381-release-dep-updates/plan.md`
> Reading order: **spec → plan → retrospective**.

## §0 At a glance

We set out to make within-bounds dependency updating part of every `vrg-release`, not a separate step someone has to remember. The release itself must never fail because of the update, but a failed or pending update must never be silently ignored. That shipped:

- **The release stage.** Every `vrg-release` now ends with a fail-deferred `update-deps` stage. When dependencies change, it opens a `deps/update-<next>` PR labelled `dependency-update` and merges it when green.
- **Failure handling.** A failed validation or merge leaves the PR open and files a fix task. The PR body carries the failing `vrg-validate` stage's output.
- **The gate.** A fresh `vrg-release` refuses to start while a `dependency-update` PR is open. `--allow-pending-deps` bypasses it, and the bypass is logged on the release tracking issue.
- **Updaters and hooks.** There are built-in `python`, `vergil` and `images` updaters and a held-back report. vergil-vm and vergil-python each gained a per-repo hook.
- **Validation.** It was validated on real releases of vergil-tooling, vergil-vm and vergil-python, including two real update failures handled end to end and a deliberate gate trip.

The epic also took on the long-stalled leading-edge tool bump in vergil-containers (PR #445). Merging it exposed six weeks of silently failing image publishes, which were fixed before close.

- **Span:** epic opened 2026-10-15 21:41 UTC; last task closed 2026-10-17 12:21 UTC (about 39 hours).
- **Children:** 32. 28 were completed, 3 brainstorms were closed as superseded (§5), and this retrospective is the last. One brainstorm (#385) was moved to the `.github` ad-hoc epic (#101) so it can run on its own.
- **PRs:** 26 merged under the epic's tasks, across 6 repos (.github, docs, vergil-tooling, vergil-vm, vergil-python, vergil-containers). Two more came from fix tasks that the new stage filed itself.
- **Releases cut:**
  - vergil-tooling v2.1.242–v2.1.247 (6);
  - vergil-vm v2.1.43–v2.1.46 (4);
  - vergil-python v1.0.1 (1).

### Work delivered

Planned tasks (plan IDs in brackets):

| PR | Repo | What it did |
|---|---|---|
| [.github#387](https://github.com/vergil-project/.github/pull/387) | .github | Spec and plan (documentation bookend, #382) |
| [vergil-tooling#3263](https://github.com/vergil-project/vergil-tooling/pull/3263) | vergil-tooling | [T1] Held-back report, `[dependency-update]` config, repo-hook updater (#3251) |
| [vergil-tooling#3265](https://github.com/vergil-project/vergil-tooling/pull/3265) | vergil-tooling | [T2] Failed updates stay open as labelled PRs with a fix task (#3253) |
| [vergil-tooling#3270](https://github.com/vergil-project/vergil-tooling/pull/3270) | vergil-tooling | [T3] `update-deps` as the last `vrg-release` stage, `--without-deps` (#3257) |
| [vergil-tooling#3272](https://github.com/vergil-project/vergil-tooling/pull/3272) | vergil-tooling | [T4] Release gate on open `dependency-update` PRs, `--allow-pending-deps` (#3258) |
| [vergil-tooling#3266](https://github.com/vergil-project/vergil-tooling/pull/3266) | vergil-tooling | [T5] Python held-back report (#3254) |
| [vergil-tooling#3267](https://github.com/vergil-project/vergil-tooling/pull/3267) | vergil-tooling | [T6] `action.yml` ref normalization; vergil-python runtime bump from every serving index (#3255) |
| [vergil-tooling#3264](https://github.com/vergil-project/vergil-tooling/pull/3264) | vergil-tooling | [T7] `repo_init` templates derive the vergil series (#3252) |
| [vergil-tooling#3268](https://github.com/vergil-project/vergil-tooling/pull/3268) | vergil-tooling | [T8] `images` updater (#3256) |
| [vergil-vm#320](https://github.com/vergil-project/vergil-vm/pull/320) | vergil-vm | [T10] Dependency-update hook; `vrg-vm-init` reads the vergil series (#317) |
| [vergil-python#12](https://github.com/vergil-project/vergil-python/pull/12) | vergil-python | [T11] Dependency-update hook for the PBS runtime pin (#11) |
| [vergil-containers#445](https://github.com/vergil-project/vergil-containers/pull/445) | vergil-containers | Leading-edge tool bump, reviewed and merged by the human (#599) |
| [vergil-tooling#3298](https://github.com/vergil-project/vergil-tooling/pull/3298) | vergil-tooling | Documentation-review bookend (#3245) |

Operational tasks:

- **[T9] DEP** (vergil-tooling#3259): tooling deployed via v2.1.242.
- **[T12] VAL** (vergil-tooling#3260): live release-cycle validation. SUCCESS on all 4 checks, plus the real failure cycles in §1.

Added during execution:

| PR | Repo | What it did |
|---|---|---|
| [vergil-tooling#3285](https://github.com/vergil-project/vergil-tooling/pull/3285) | vergil-tooling | Failure section carries the failing validate stage's output; a deps-only failure counts once (#3283) |
| [vergil-tooling#3287](https://github.com/vergil-project/vergil-tooling/pull/3287) | vergil-tooling | Preflight closes a deps PR once its fix task is completed (#3286) |
| [vergil-tooling#3299](https://github.com/vergil-project/vergil-tooling/pull/3299) | vergil-tooling | Per-PR refusal advice in the gate (#3297) |
| [vergil-vm#336](https://github.com/vergil-project/vergil-vm/pull/336) | vergil-vm | Hook test hermetic to the live Lima floor and NodeSource major (#334) |
| [vergil-python#19](https://github.com/vergil-project/vergil-python/pull/19) | vergil-python | Hook tests run in `vrg-validate` via `validate-custom` (#17) |
| [vergil-vm#337](https://github.com/vergil-project/vergil-vm/pull/337) | vergil-vm | Docs: the hook and the `vrg-vm-init` series source (#335, from the sweep) |
| [vergil-python#20](https://github.com/vergil-project/vergil-python/pull/20) | vergil-python | Docs: the hook; stop duplicating the runtime pin (#18, from the sweep) |
| [docs#38](https://github.com/vergil-project/docs/pull/38) | docs | `vrg-release` and `vrg-update-deps` in tools-and-skills (#37, from the sweep) |
| [.github#398](https://github.com/vergil-project/.github/pull/398) | .github | Spec amended to match the shipped gate, failure output and hook (#395, from the sweep) |
| [vergil-containers#603](https://github.com/vergil-project/vergil-containers/pull/603) | vergil-containers | Verify hadolint against `checksums.sha256` (#602) |
| [vergil-containers#607](https://github.com/vergil-project/vergil-containers/pull/607) | vergil-containers | Install vergil-tooling before the image Trivy scans (#604) |
| [vergil-containers#608](https://github.com/vergil-project/vergil-containers/pull/608) | vergil-containers | Clear the image publish Trivy gate (#605) |
| [vergil-containers#609](https://github.com/vergil-project/vergil-containers/pull/609) | vergil-containers | Fix the dev-go 1.26 build; retire the Go 1.25 image (#606) |

These PRs were produced by the new stage itself, through the fix-task flow it files. They live in the ad-hoc epics:

| PR | Repo | What it did |
|---|---|---|
| [vergil-tooling#3284](https://github.com/vergil-project/vergil-tooling/pull/3284) | vergil-tooling | Landed the failed 2.1.244 update (ty 0.0.89 typing fixes; fix task #3282, deps PR #3281) |
| [vergil-vm#330](https://github.com/vergil-project/vergil-vm/pull/330) | vergil-vm | Landed the failed 2.1.44 update with a hermetic hook test (fix task #329) |

Related ad-hoc work: [vergil-tooling#3317](https://github.com/vergil-project/vergil-tooling/pull/3317) (#3316) raised `vrg-finalize-pr`'s CD wait and made it report a timeout as result-unknown. vergil-python#21 listed `validate-custom` in the README.

## §1 How the plan evolved

The plan's twelve tasks landed largely as written, in the planned order, and in under a day. The core of the design held: one fail-deferred stage, a labelled PR, a fix task, and a preflight gate. What changed came almost entirely from running it for real. The live validation task (T12) found three design defects in a few hours, each caught by an actual failure rather than a test.

1. **A failure section without the failure.** The first real failure was vergil-tooling 2.1.244, where a new ty release raised type errors. Its PR body carried the last ~200 lines of output, which didn't reliably include the failing stage's output. #3283 changed it to carry the failing stage's output plus the summary, falling back to a labelled raw tail. The same run showed a deps-only failure being counted twice in the release summary, and that was fixed too.
2. **Squash merges never mark the deps PR merged.** Spec §5.2 assumed that merging the fix PR would mark the deps PR merged and clear the gate. Under squash merges it doesn't. The fix landed (#3284), but deps PR #3281 stayed open and the next release refused. #3286 made preflight close a deps PR whose linked fix task is closed as COMPLETED, and fail closed on any lookup error. This was the most important correction in the epic: the gate would otherwise have needed a manual step after every fix, which is exactly the "remember to do it" problem the epic exists to remove.
3. **Generic refusal advice.** The refusal ended with one shared hint ("land the fix task, merge the PR, or close it"). For squash merges that hint was wrong, and it said nothing when a PR had no fix task. #3297 made the advice specific to each PR's case.

The vergil-vm releases exposed a fourth issue, in a test rather than the design. The vm hook's test asserted against the live `NERDCTL_VERSION`. The updater's own success therefore broke its own test, so the 2.1.44 update failed validation. That was fixed hermetically in #330, then extended to the Lima and NodeSource baselines in #334. vergil-python had the opposite gap: its hook tests weren't run by `vrg-validate` at all (#17).

The documentation-review sweep (#3245) found that the tooling docs were stale beyond this epic. The release guide still described the long-removed `vrg-prepare-release` and `vrg-merge-when-green`. The sweep spawned per-repo doc tasks in vergil-vm, vergil-python, docs and .github, and amended the spec to record what shipped (#395), marking each change as post-implementation.

The largest unplanned stretch was in vergil-containers. Task #599 (review and merge the stalled bump PR #445) was planned to follow brainstorm #385. The human reviewed and merged #445 directly. Its CD run then failed in every image, and fixing that uncovered three stacked problems:

- **hadolint's checksums.** hadolint 2.15.1 dropped its per-binary `.sha256` files (#602).
- **The Trivy gate.** It had blocked every image publish since 2026-09-02, unnoticed. Alongside it, a 2026-10-07 vergil-actions change made the shared Trivy action need `vrg-sarif-filter` on PATH (#604, #605).
- **The dev-go builds.** The newest Go linters raised their Go floor past what the dev-go images ship (#606).

The Go 1.25 image would have needed permanent CVE suppressions because Go 1.25 is end of life. The human chose to retire it rather than accept the risk (#606, minimum change). No image was published from 2026-08-29 until CD run 39028546675 on 2026-10-17.

## §2 Lessons learned

- **Live validation earns its cost.** Three of the four design corrections came from the first real failures, within hours. Unit tests passed throughout, and every defect lived in the interaction with GitHub (merge mode, issue state) or with real tool output. Keep a live-cycle validation task in any epic that changes the release path, and budget for the fix tasks it will produce.
- **Check the repo's merge mode before designing around "merged".** The squash-merge assumption in spec §5.2 was plausible and wrong. Any design that relies on GitHub's merge state should name the merge mode it assumes.
- **An updater's tests must not read live upstream state.** A hook test that compares against the current pin breaks the moment the hook succeeds. Make it hermetic from the start, as #330 and #334 did.
- **A hook isn't covered until `vrg-validate` runs its tests.** vergil-python's tests existed but didn't gate anything until `validate-custom` called them.
- **A green PR doesn't mean the artifact builds.** vergil-containers PR CI doesn't build images, so a bump that breaks every image merges green. Six weeks of red CD went unnoticed because nothing pulled it to a human's attention. A gate that can sit red for weeks unnoticed isn't protecting anything.
- **Same next time:** small, sequenced tasks with an explicit filing map. Parallel sub-agents per task with the prompt contract. Per-repo batched submission. Real releases as the test bed.

## §3 Compromises and tradeoffs

- **An `update` failure leaves nothing open.** When an updater or hook fails before validation, nothing is pushed. No PR or fix task exists, the failure appears only on the tracking issue and in publish-status, and it doesn't gate the next release. We accepted this to keep the stage simple, and recorded it on #385.
- **Accepted Trivy suppressions.** #605 cleared the gate by bumping what it could: nfpm (which also fixed the SSH authentication bypass, CVE-2026-56854), trivy and npm. Fixed CVEs in tools with no newer release were suppressed under the existing `.trivyignore` policy (gh, actionlint, shfmt, tofu, pebble, two Ruby default gems, x/image, x/net and x/text in go-licenses and go-test-coverage). The weakest argument is CVE-2026-78669 (HTTP/2 client DoS in gh and tofu), and the human accepted it by merging #608. Each entry carries a removal condition.
- **A temporary npm pin.** It is `^12.2.1` (ahead of npm's `latest` tag). The pin checker doesn't track it, because it only recognises exact versions. A Dockerfile comment carries the condition for removing it.
- **amd64 verified only by CD.** The agent VM is arm64 with no emulation, so local image builds and scans were arm64 only. The green CD run proved amd64.
- **The minimum Go 1.25 retirement.** The image is no longer built, but published `dev-go:1.25` and `prod-go:1.25` tags remain in GHCR, and consumers weren't migrated (§5).
- **Gate bypass used once in validation.** `--allow-pending-deps` was used deliberately during the vergil-vm gate-trip exercise, and the bypass was logged as designed.
- **Out of scope, by design:** language updaters beyond Python, pinning Actions to SHAs, and reconciling `bump-tools` with the release cadence (spec §11).

## §4 New problems and opportunities

| What surfaced | Where it went |
|---|---|
| An `update`-kind failure leaves no PR, no fix task and no gate | Recorded on brainstorm #385 |
| vergil-containers PR CI doesn't build images; red CD can go unnoticed for weeks | Recorded on brainstorm #385 |
| `.trivyignore` works by CVE ID across the whole image, so an entry justified for one tool hides the same ID in another (how the Go 1.25 toolchain findings passed unexamined) | Recorded on brainstorm #385 |
| Several languages' release cadence is faster than assumed; EOL versions remain in the matrix (Go 1.25 was one) | The human's language-support umbrella epic (folds in #383, #384, #403) |
| `vrg-finalize-pr`'s 30-minute CD wait was shorter than vergil-containers' ~43-minute CD, and its batch summary blamed validation for a CD failure | Fixed: vergil-tooling#3316 / #3317 |
| The vergil-actions SARIF filter (#921) silently became a requirement for every image-scan caller | Fixed for vergil-containers (#604). vergil-actions' own callers are unaffected; other repos calling the action directly weren't audited |
| The release guide documented tools removed long ago | Fixed in the sweep (#3298) |
| Epic #52, superseded by this epic, is still open | Logged, not yet acted on |

## §5 What's next

- **Brainstorm #385** (now in the `.github` ad-hoc epic #101): reconcile scheduled and release-driven updates, `bump-tools` ownership, and how update reports and red pipelines reach the human. It also carries the three gaps above. It is the next piece of work.
- **The language-support umbrella epic** (being created at close-out time) takes over and supersedes brainstorms #383 (lifecycle monitoring), #384 (restarting the language updaters) and #403 (EOL retirement, including finishing the Go 1.25 retirement).
