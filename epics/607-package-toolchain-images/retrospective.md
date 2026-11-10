# Package toolchain images — Retrospective

## §0 At a glance

We set out to stop CI package jobs from installing their toolchain on every run. Each job started from a raw distro image and ran `apt-get install` or `dnf install` of the prerequisites, then installed uv and nFPM. On amd64 Ubuntu, apt from the Azure mirror intermittently trickled at about 44 kB/s. In the baseline (planning issue .github#606), 22 of the last 40 vergil-tooling PR runs had a slow package job and took 8–17 minutes, against 2.2–4.2 minutes for everything else.

**What shipped:** the toolchain is now built once, into managed images, and package jobs run in them.

- **Images:** vergil-containers publishes `{dev,prod}-pkg-ubuntu:{24.04,26.04}` and `{dev,prod}-pkg-rhel:{9,10}`. They are multi-arch and public, from a new `toolchains:` manifest section that sits outside the language model. Each image carries a marker, `/etc/vergil/package-toolchain`, listing the seven tools it guarantees.
- **The setup action:** vergil-actions' `actions/package/setup` checks the image against its marker and installs only the driver vergil-tooling, at a Python version set by one action input. It started with a skip path for toolchain images and now requires one; the raw-image path is gone.
- **Package jobs:** vergil-tooling's target registry derives every target's image from its distro and version: `ghcr.io/vergil-project/prod-pkg-<distro>:<version>`.
- **Result (V1, 19 consecutive PR runs):**
  - "Set up package toolchain" takes **3–8 s** in all 94 package jobs;
  - no job runs `apt-get install` or `dnf install`;
  - runs take **2m48s–5m09s**.

### Work delivered

- **Span:** epic opened 2026-10-27, with its spec and plan merged the same day (.github#612). Nothing happened until 2026-11-09. Implementation then ran from 2026-11-09 22:01Z (vergil-actions#1115) to 2026-11-10 21:51Z (vergil-containers#829), about 24 hours.
- **Repos touched (4):** .github, vergil-actions, vergil-containers, vergil-tooling.
- **Tasks:**
  - 19 sub-issues: 18 closed, plus this retrospective.
  - 3 were operational, with no PR: D1 vergil-containers#729, D2 vergil-tooling#3766 and V1 vergil-tooling#3767, all `Outcome: SUCCESS`.
  - 6 were added during execution: .github#987, #990 and #1007 (the evolution log), vergil-actions#1130 and vergil-containers#821 (both spawned by the docs sweep), and vergil-containers#824 (from #821's merge review).
  - A seventh, .github#975, amended the plan for epic #649 before execution started.
- **Releases carrying epic work:**
  - vergil-actions **v2.1.61** (2026-11-09): T1, T2;
  - vergil-containers **v2.1.36** (2026-11-10): T3, T4, which first published the `pkg-*` images;
  - vergil-tooling **v2.1.297** (2026-11-10): T5;
  - vergil-actions **v2.1.62** (2026-11-10): T6, plus the `publish-index` live check (vergil-actions#1125).
  - **Not yet released:** vergil-actions#1131, vergil-containers#828 and #829, vergil-tooling#4183.
- **PRs (16 merged):**

| Repo | PR | Task | What it did |
| --- | --- | --- | --- |
| .github | #612 | #608 | Spec and plan |
| .github | #977 | #975 | Amended the spec and plan for epic #649: toolchain keys can be parents; no inherited image settings |
| .github | #989 | #987 | Evolution log, entries 1–8 |
| .github | #992 | #990 | Evolution entry 9: Task 4's PR image pipeline fix |
| .github | #1008 | #1007 | Evolution entry 10: pin checks for rolling minor tags |
| vergil-actions | #1115 | #1088 | T1: setup checks a toolchain image against its marker and skips the installs |
| vergil-actions | #1116 | #1089 | T2: the driver Python comes from one `python` input (default 3.15) |
| vergil-actions | #1124 | #1090 | T6: setup requires a toolchain image; raw-image path and `os-prereqs.sh` removed; full tool-set check |
| vergil-actions | #1131 | #1130 | Docs: `package/setup` described as an image check, not an installer |
| vergil-containers | #793 | #727 | T3: `toolchains:` manifest section, generator support, `dependency_map`, lifecycle round-trip fix |
| vergil-containers | #797 | #728 | T4: the `pkg-ubuntu` and `pkg-rhel` images, pins exposure report, PR image pipeline support |
| vergil-containers | #828 | #821 | Docs: what the `pkg-*` images contain and who uses them |
| vergil-containers | #829 | #824 | Docs: the marker fragment's header states the current setup contract |
| vergil-tooling | #4152 | #3765 | T5: package jobs run in `prod-pkg-*`; every target image derived and asserted |
| vergil-tooling | #4183 | #3758 | Docs review: the nFPM hint and the fast-fail docstring point at the toolchain image |

The table has 15 rows. The sixteenth PR is vergil-actions#1125, the `publish-index` live check, which merged for vergil-actions#1123. That issue was filed during this epic but isn't one of its children (§4).

## §1 How the plan evolved

The plan's "Evolution during execution" log has eleven entries. They fall into four kinds.

**Develop moved under the plan.** Epic #213 (the manifest-driven image build) was landing work in vergil-containers at the same time, and four of its changes reached develop before or during our tasks:

- **T2 (#510)** had already merged, so Task 3 added the toolchain keys to `dependency_map` itself instead of asking #213 to (entry 1).
- **T7b (#775)** had changed `matrix.json` entries to a `build-args` map with `parent` and `depth`, and Task 3 followed it (entry 8).
- **T7 (#774)** made the arm64 build native, so Task 4's QEMU note no longer applied (entry 7).
- **T6′ (#795), PR-time selective build,** merged while Task 4 was in progress. Its job planner knew only families, so Task 4's PR failed `images / select` until `pr_jobs.smoke_dirs` covered toolchain entries (entry 9).

"Develop moved" notes on the task issues carried most of this to the runs. Only #795, which landed mid-task, surfaced first as a CI failure.

**The plan's wording was wrong, and rulings corrected it.**

- **The fail-fast apt config (entry 5):** the spec and plan said it was "amd64 only". `os-prereqs.sh` actually writes it on every architecture, so the toolchain image would have silently dropped arm64's apt retries and timeouts. Task 4's merge review caught it, and the ruling was **change**.
- **The pin checks (entry 10):** D2's consumer check asked for "a `dependency-update` PR raising its vergil pin", and Task 6's precondition said to read `vergil.toml` for a pin "at or past D2's release". The fleet only ever pins the rolling minor tag (`v2.1`), so neither check can be met or means anything. The ruling replaced both with "the rolling tag resolves to the release or later".
- **V1's sample size:** the plan asked for "at least 20 consecutive" PR runs, and V1 ran on 19 by ruling (N0.1 on #3767, **dismiss**). At the baseline's ~55% slow rate, 10 consecutive fast runs would already have only about a 0.03% chance if nothing had been fixed. And the images no longer run apt at all. It is recorded on #3767 and as the log's entry 11, which this retrospective's PR adds.

**Scope added by ruling (entries 3 and 4).** Both came from follow-ups that execution raised, and both were folded into existing tasks as scope changes rather than filed as new ones:

- Task 4 gained the pins exposure report, so `pkg-*` would not be left out without warning.
- Task 6 gained a required-tool check, so a marker that leaves out `uv` fails at once instead of at the install step.

**Found and fixed in passing.**

- **Lifecycle bug (entry 2):** Task 3 found that `lifecycle/apply.py` rebuilt the manifest from the families alone, so the next nightly lifecycle PR would have deleted `toolchains:` and `base`'s parent. This was the risk Review Focus 1 had named.
- **`pkg-rhel` and Python (entry 6):** Task 4 kept UBI's own `python3`, because dnf depends on it. `pkg-rhel` checks that it is the only Python and that it is distro-owned.

**What held:** the task graph and its order. T1, T2 and T3 ran in parallel, and the rest followed T3 → T4 → D1 → T5 → D2 → T6, with V1 after T5. The marker contract written in Task 1 was read unchanged by Tasks 4 and 6. No task was dropped.

## §2 Lessons learned

- **A plan that leans on another epic's in-flight work goes stale within days.** Four of the ten evolution entries came from epic #213 moving develop. Recording "develop moved" on the issue before dispatch worked well. A merge that lands mid-task can't be caught that way, and showed up only in PR CI. Next time: before dispatching a task in a repo with another active epic, re-read that epic's merges since the plan was written.
- **Local validation is not CI, and handoffs found out the hard way.** `vrg-validate` doesn't lint the generated `docker/*/Dockerfile` files (vergil-containers#761) and doesn't run the `*.test.sh` harnesses, which only run in smoke workflows. Task 4 reported ready with green validation and then failed hadolint in CI, and #1125's new tests first ran in its PR's CI. The closer local validation gets to CI, the fewer red PRs after a handoff.
- **Check that a verification step fits how the fleet actually works.** D2's pin-PR check would have looked plausible in any fleet with exact pins. It was wrong here, and it got through the spec, plan, pushback, alignment and the dispatch brief. The D2 run noticed that no PR existed but called it a wording issue, and the driver leaned towards dismissing it. The human caught it. Next time: a deployment check that names a mechanism (a PR, a file, a field) should be confirmed against one real consumer before the plan is approved.
- **"Success" from a platform isn't the same as delivery.** The first release after the switch, vergil-tooling v2.1.297, failed at `package-index`. A GitHub Pages deployment reported `success`, and GitHub marked it active, but the Pages origin kept serving the previous deployment (packages#46). The only end-to-end check was `vrg-release`'s 60-minute poll. Once the Pages API output made the hidden step visible, the next run went from guessing to a real test, and a re-run went live in 24 s. The lasting fix is to check what's actually served, right after the deploy (vergil-actions#1125).
- **Errors that get swallowed cost the most time.** `vrg-semgrep-scan` dropped semgrep's stderr, so a `semgrep-core` segfault in the rebuilt `base` (#649's work) showed up only as "exit code 2". Finding it took a manual reproduction in the image. The same no-silent-failures principle applies to tools as to product code (vergil-tooling#4188).
- **The driver should finish the CI loop itself.** Twice, a red PR was handed back to the human with a `/vergil:pr-watch` command to run. The human's view: the driver should just run it. Once it did, a CI failure was diagnosed, fixed and green within one exchange.
- **Parallel task runs and per-repo batching worked.** The 24-hour execution ran up to three tasks at once in separate worktrees, and every handoff went through `report-ready` and one `vrg-submit-pr` per repo. T1 and T2 both edited `action.yml` and merged two minutes apart without a reported conflict.

## §3 Compromises & tradeoffs

- **Inline hadolint ignores:** SC3054 on the marker step, which runs under bash and must match `check-toolchain.sh`'s bash regex. DL3006 on the UBI `FROM`, where the RHEL version is in the repository name and the image floats within it by the pinning doctrine. Each has its reason in the template.
- **`pkg-rhel` keeps a Python:** the strict "no python3" constraint was relaxed because UBI's dnf needs `/usr/bin/python3`. The check now proves it is the distro's one copy, and that nothing else is in `/usr/local/bin`.
- **uv pinned at 0.13.7,** the latest release, not the plan's 0.12.23. The plan allowed either.
- **V1's sample was 19 runs, not the plan's 20**, by ruling (§1).
- **D1 didn't check "Write under Manage Actions access" directly,** because the package settings need a login. It inferred write access from the `prod-` publish having pushed all four packages; T5, D2 and V1 then pulled them in CI.
- **T5's end-to-end timings went on the issue, not the PR body.** vergil-tooling's CI runs only on `pull_request`, so the plan's step (record the timings before handoff) couldn't run before a PR existed. They are on vergil-tooling#3765.
- **The `publish-index` live check names the run, not the Pages deployment ID.** Getting the ID needs an API lookup and more permissions, which were left out on purpose.
- **#1131 waited on an unrelated fix rather than being overridden.** It changed only `.md` files but was red on the semgrep segfault. It merged after vergil-containers#827 fixed the base image.

## §4 New problems & opportunities

| Problem / opportunity | Where it went |
| --- | --- |
| A GitHub Pages deployment reported `success` but never went live, blocking v2.1.297's index | packages#46, planning queue, **open**. Debugging is in a separate session; a re-run went live, so it looks like a one-off GitHub fault so far |
| `publish-index`'s index build takes 20–26 minutes | Being reworked as part of the packages#46 work (the human found the index algorithm at fault) |
| No check that the deployed index is the one just built | vergil-actions#1123 → #1125, **merged**, in v2.1.62 |
| `semgrep-core` segfaults in the rebuilt `base` (epic #649), breaking semgrep in `dev-` images | vergil-containers#825 → #827, **merged**; not yet released as `prod-base` |
| `vrg-semgrep-scan` drops semgrep's stderr | vergil-tooling#4188, **open**; its branch is reported ready |
| hadolint doesn't lint generated Dockerfiles locally | vergil-containers#761, **open**, filed before this epic. It describes local validation as not running hadolint at all; plan entry 9 says it runs only on root `Dockerfile*`. The difference isn't resolved |
| `vrg-validate` doesn't run the `*.test.sh` harnesses, which only run in smoke workflows | Raised by the #1123 run; filed at this retrospective as vergil-tooling#4197, in intake, **not yet acted on** |
| Plans may assume exact patch pins elsewhere | Raised in session; the human declined an intake item |

**Unaddressed-findings sweep.** All 18 children other than this one were swept, along with the epic issue and planning issue #606. All comments are by trusted authors. Every `Awaiting a ruling` item has a closing ruling, and each `fix` or `file` was confirmed landed or filed:

- epic N0.1–N0.6;
- #987 N0.1;
- D2 N0.1;
- V1 N0.1;
- merge-review R1 on #728;
- R1 on #821;
- R1 and R2 on #824.

All 16 PRs have no reviews or PR comments, and no item was security-class. #606's pushback round dropped five candidates with one shared reason. Two of them, pins and hadolint coverage, partly came back during execution as N0.2 and entry 9, so the drop was reasoned but optimistic. Four candidates were never tracked. The human ruled on each on 2026-11-10:

1. **D1 didn't check Actions access directly.** **Dropped:** the `prod-` publish wrote all four packages, and T5, D2 and V1 then pulled them.
2. **T5's timings couldn't go in the PR body.** **Dropped:** a plan-versus-workflow mismatch, recorded in §3.
3. **The marker fragment's header leaves out that `check-toolchain.sh` also requires the six contract tools.** **Dropped:** the header is incomplete, not wrong, and it still lists the contract's tools.
4. **`vrg-validate` doesn't run the `*.test.sh` harnesses.** **Filed** as vergil-tooling#4197, in intake.

## §5 What's next

- **Epic .github#649 (standardize on Ubuntu 24.04)** builds `base` and every language image on `pkg-ubuntu:24.04`. That makes this epic's image the parent of the whole fleet. Its amendment is in .github#975 and the plan's "Amendment from epic #649".
- **packages#46** is the open follow-on for the package index (deploy reliability and build time).
- **Releases still to carry this epic's tail:** vergil-containers (#827's semgrep fix plus the #828 and #829 docs), vergil-actions (#1131) and vergil-tooling (#4183, and #4188 once merged).
- **Standardizing OS choice across containers, VMs and runners** (.github#605) stays out of scope, as the spec's non-goals said.

## Appendix A — Operational notes

The rollout sequence, as executed:

1. vergil-actions v2.1.61: T1, the marker-aware skip, and T2, the driver Python input. The raw-image path stayed in place.
2. vergil-containers v2.1.36: T3 and T4. The `prod-` publish created the four `pkg-*` packages. **They came out public, with the repo's write access, without any manual GHCR step**, as the `dev-` ones had. The spec's §5.4 human step to make the packages public turned out to be unnecessary.
3. D1: all eight tags are two-platform indexes with SLSA provenance. Anonymous pulls work, and the marker lists all seven tools.
4. vergil-tooling v2.1.297: T5. Its `package-index` stage failed on the Pages fault (packages#46). A manual `publish-index` re-run (packages run 42846562592) went live, and then the held consumer refresh was completed by hand.
5. D2: the release's package builds ran in `prod-pkg-ubuntu:24.04` and skipped the installs. Consumers pin `v2.1`, which resolves to v2.1.297.
6. vergil-actions v2.1.62: T6, which now requires the image, and the `publish-index` live check.
7. V1: 19 PR runs.

**Gotchas:**

- **Workflow-file pushes:** a task delta touching `.github/workflows/**` is refused on the agent push (T1, T4, T6, #1123). `vrg-submit-pr` pushes those branches.
- **Rolling tags:** consumers follow `vX.Y` tags, so "has the consumer picked up the release?" means "does the tag resolve to it?" (entry 10).
- **Pages deploys:** a deploy can report success and not be live. Check what's actually served (#1125), and read `deployments/<id>/statuses` before re-running.
