# Retrospective: Ubuntu 24.04 as the shared base platform

- **Epic:** vergil-project/.github#649
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md) · **Spike:** [`spike.md`](spike.md)
- **Planning issue:** vergil-project/.github#605 (design review history in its comments)
- **Retrospective task:** vergil-project/.github#652

## §0 At a glance

We set out to make Ubuntu 24.04 the one operating system Vergil develops and validates on:

- the container image tree, rebuilt on a `pkg-ubuntu:24.04` root instead of Debian and upstream language images;
- the local and cloud agent VMs;
- the GitHub-hosted runners, pinned instead of the floating `ubuntu-latest`;
- an OS-usage table that says where every OS choice is made and why.

All of it shipped. The `prod-` images moved in one cutover release, vergil-containers v2.1.38 (2026-11-11). A no-cache rebuild of the whole tree then passed, and every one of the 18 `prod-` images was confirmed on its expected OS on both architectures. All three agent VMs are rebuilt on 24.04. Every runner label names its release, and `vrg-validate` now refuses `ubuntu-latest`.

| | |
|---|---|
| Span | Epic opened 2026-10-28; spec and plan merged 2026-11-09T21:36Z; last task closed 2026-11-12T18:44Z (execution: about 2 days 21 hours) |
| Tasks | 48 (plus this retrospective): 39 code or docs, 6 validation, 3 deployment. All closed as completed; every operational task recorded `Outcome: SUCCESS` |
| PRs | 38 merged: vergil-containers 16, vergil-tooling 9, vergil-actions 5, .github 4, vergil-vm 3, docs 1 |
| Repos touched | 6: vergil-containers, vergil-tooling, vergil-actions, vergil-vm, vergil-project/.github, vergil-project/docs |
| Key releases | vergil-containers v2.1.38 (the cutover) and v2.1.39 (the apt and OS-upgrade fixes, ahead of the cold rebuild); vergil-vm v2.1.50 (the Lima pin) |
| Rollbacks | None. The cutover's revert path was never used |

The work, in merge order:

| Task | Repo | PR | What it did |
|---|---|---|---|
| #650 | .github | #981 | The spec and plan |
| #394 | vergil-vm | #396 | Pin the Lima agent VMs to Ubuntu 24.04 |
| #777 | vergil-containers | #792 | Declare the scripts' dependencies (PEP 723) and run them with `uv run` |
| #985 | .github | #986 | Decouple the mq-rest-admin repos from the plan |
| #1114 | vergil-actions | #1117 | Install vergil-tooling from its package, and never install uv with pip |
| #4130 | vergil-tooling | #4133 | Install vergil-tooling from its package where the image has the vergil apt repository |
| #988 | .github | (validation) | The amd64 half of the spike |
| #982 | .github | #1001 | The spike report and the plan's first evolution entries |
| #4131 | vergil-tooling | (deployment) | Release the vergil-tooling and vergil-actions install paths |
| #4138 | vergil-tooling | #4158 | Make the packaged cppcheck suppressions comment-free, for cppcheck 2.13 |
| #4139 | vergil-tooling | #4161 | Count only glibc's own libraries in `elf.glibc_requirements` |
| #778 | vergil-containers | #816 | Pin the language toolchains by version and hash, refreshed daily |
| #779 | vergil-containers | #818 | Build `base` on `pkg-ubuntu:24.04`, with vergil-python and a side-by-side Node |
| #780 | vergil-containers | #822 | Build the python images on `base` |
| #781 | vergil-containers | #823 | Build the go images on `base` |
| #825 | vergil-containers | #827 | Wrap semgrep to cap the stack hard limit, and scan at build time |
| #782 | vergil-containers | #830 | Build the rust images on `base` |
| #783 | vergil-containers | #831 | Build the cpp-clang images on `base` |
| #785 | vergil-containers | #832 | Build the cpp-gcc images on `base` |
| #786 | vergil-containers | #833 | Build the ts-node images on `base` |
| #1132 | vergil-actions | #1133 | A `container-prefix` input for `ci-package.yml` |
| #837 | vergil-containers | #838 | Fallback apt mirrors outside Canonical's domain, for both architectures |
| #1134 | vergil-actions | #1139 | Make `container-prefix` a tunable of every reusable workflow, starting with `cd-release.yml` |
| #4202 | vergil-tooling | #4203 | Run package CI on the `dev-` package images until the cutover |
| #4199 | vergil-tooling | #4201 | Add `APT::Update::Error-Mode=any` to the package jobs' apt fail-fast settings |
| #787 | vergil-containers | #834 | Publish the OS-usage table and update the CVE triage page |
| #839 | vergil-containers | (validation) | apt in `dev-base` on amd64, through the fallback mirror list |
| #788 | vergil-containers | (validation) | Consumer CI on the new `dev-` images, before the cutover |
| #789 | vergil-containers | (deployment) | The cutover release, v2.1.38 |
| #1113 | vergil-actions | #1149 | Pin the runners to `ubuntu-24.04` |
| #4237 | vergil-tooling | #4238 | Return package CI to the `prod-` package images after the cutover |
| #4230 | vergil-tooling | #4241 | Publish the apt indexes by hash and keep earlier index generations |
| #776 | vergil-containers | #843 | Pin the runners to `ubuntu-24.04` |
| #407 | vergil-vm | #410 | Ignore provisioner comments in `test_base.sh`'s rc checks |
| #408, #409 | vergil-vm | (validation) | The second and third rebuilt VMs on 24.04 |
| #395 | vergil-vm | (deployment) | Release vergil-vm and rebuild the VMs on 24.04 |
| #844 | vergil-containers | #846 | Choose the apt mirror list from the archive hosts `ubuntu.sources` names |
| #845 | vergil-containers | #847 | Re-run the `pkg-` images' OS upgrade on the first build of each UTC day |
| #4129 | vergil-tooling | #4252 | Refuse `ubuntu-latest` runner labels in `vrg-validate` |
| #790 | vergil-containers | (validation) | The cold rebuild of the whole tree |
| #826 | vergil-containers | (none) | The semgrep upstream report: already filed upstream, so closed by decision |
| #101, #1060, #411, #1150, #739, #4260 | docs, .github, vergil-vm, vergil-actions, vergil-containers, vergil-tooling | #102, #1062, #412, #1151, #852, #4265 | The documentation review and the per-repo doc tasks it spawned |

## §1 How the plan evolved

The plan held up in its shape: one image tree rooted in `pkg-ubuntu:24.04`, families rebuilt on `base` one by one, a single cutover release, then the VMs, the runner pins and the guard. What changed was everything around it: the plan assumed working infrastructure, and for most of execution the infrastructure was the problem.

**The spike paid off.** Running the trial builds first (#982, with the amd64 half split into #988) found two vergil-tooling bugs before any family task hit them: noble's cppcheck rejecting the comments in the packaged suppressions file (#4138), and `elf.glibc_requirements` counting libgcc's version needs as glibc's (#4139). It also settled small details recorded on the task issues, such as the C++ images installing `g++-14` and Rust's linker packages. The plan's log records these.

**The install path changed under the plan.** Tasks 4 and 5 (#4130, #1114) moved containers and CI to installing vergil-tooling from its package. Landing them showed the package path left uv off `PATH`, which broke harness fixtures, so the rule became that every install path leaves uv on `PATH` and never installs it with pip. A deployment task (#4131) released both install paths before the image work depended on them.

**The family tasks were mostly routine; semgrep was not.** The six family rebuilds (#780–#786) landed in an evening. In between, rebuilding `base` on Python 3.15 made every semgrep scan segfault: Python 3.15 reports an unlimited rlimit as `2**64 - 1`, and semgrep derives a ~6 EB stack limit from it. A wrapper in `base` (#825) caps the stack hard limit. Reporting it upstream (#826) turned out unnecessary: semgrep/semgrep#11979 already had the report and #11987 the fix, so the workaround's removal moved to the deferred queue (vergil-containers#851).

**A four-day Canonical DNS outage forced unplanned work.** Canonical's archive hosts became unreachable for days, and we chose not to wait it out. That produced:

- fallback mirrors outside Canonical's domain for both architectures (#837), validated on amd64 as a separate task (#839) because this host is arm64;
- `APT::Update::Error-Mode=any` for package jobs, so a failed index download fails the job (#4199);
- a temporary switch of package CI to the `dev-` package images, and its revert after the cutover (#4202, #4237). That needed a `container-prefix` input on the reusable workflows, which became a tunable of all of them (#1132, #1134).

None of this was in the plan, and the log records none of it.

**Validation was narrowed twice, by decision.** V1 (#788) validated only the python family and `base`, the ones with a real consumer today; the other families' consumer validation was deferred to .github#1022. V2 (#790) counted the VM check from D3's rebuilds, since a rebuild runs the same create path. It also accepted consumer CI failures caused by the consumers' own out-of-date configuration as known issues in those repos (see §3).

**After the cutover, three defects showed up that only a published release could show:**

- **The apt publish race.** GitHub Pages caches each file for ten minutes, separately at each edge, so an apt client could get a new `Release` with old indexes ("Mirror sync in progress?"). The fix publishes the indexes by hash and keeps earlier generations (#4230).
- **The arm64 archive host.** Ubuntu 26.04 serves arm64 from the main archive, not `ports.ubuntu.com`, so the mirror rewrite had to follow the hosts `ubuntu.sources` actually names (#844).
- **A stale upgrade layer.** BuildKit's registry cache reused a `dnf upgrade` layer from before RHEL 9.9, so `dev-pkg-rhel:9` failed the Trivy gate on fixed CVEs. The `pkg-` images now take a daily `UPGRADE_DATE`, so the OS upgrade re-runs on the first build of each day (#845).

v2.1.39 carried the last two onto `main` before the cold rebuild.

**The VM work was mostly mechanical.** The Lima pin (#394), the release, and `vrg-vm rebuild` of all three VMs (#395) went as planned. A bug in `test_base.sh`'s own check (#407) was fixed before the second and third VMs were validated (#408, #409).

**Scope came off at the end, as recorded in the log's last entry.** A private repo's runner-label task was opened in error and dropped. The three follow-on brainstorms (#959, #651, #983) moved to the planning queue instead of holding the epic open.

## §2 Lessons learned

- **Spike first.** The spike cost a day and found real defects in shared tooling before the six family tasks could hit them one by one. Do this again for any change that rebuilds a tree of images.
- **Plan for the infrastructure failing, not only the code.** The largest unplanned cost was external: Canonical's outage, then GitHub Pages' caching. Fallback mirrors and by-hash indexes are now permanent, but the lesson is to ask at planning time what happens when each upstream (package mirrors, registries, the Pages CDN) is down or stale, and to treat that as design, not incident response.
- **Some defects only exist after a release.** The publish race, the arm64 archive host on 26.04 and the stale upgrade layer passed every PR gate and appeared only in published releases and nightly builds. A cold rebuild and a look at develop's CD after each merge belong in the plan as gates, not as afterthoughts.
- **The evolution log fell behind.** It records three entries. Execution changed far more: the outage work, the dev-prefix detour, by-hash, the semgrep wrapper, both validation narrowings, and #844 and #845. The record survived on the issues and in their rulings, so this retrospective could be written, but the plan alone no longer explains the epic. Write a log entry when the plan's assumptions change, not only when its task list does.
- **Rebases of frozen branches were expensive.** Several branches conflicted at submit time with work that landed after their report-ready. Each meant unfreezing, rebasing, revalidating and a fresh merge review. Merging related branches in one batch, or holding a dependent branch's report-ready until its sibling lands, would have saved most of that.
- **Validation tasks need to name their method, and to be checked against what the agent can run.** V2's VM check assumed a free `vrg-vm create`; its by-hash check named an apt debug option that does not log https; its consumer check assumed reruns would be valid. Each was settled by a scope change in minutes, but each was found only at run time.

## §3 Compromises and tradeoffs

- **The semgrep wrapper stays in `base`** until semgrep ships its fix. It is a few lines, documented in place and tracked in the deferred queue (vergil-containers#851), and it could last months. Accepted so the epic could close.
- **Consumer CI failures accepted as known issues.** Three consumers fail on the new images because of their own out-of-date configuration: logical-minds-foundry/mq-resiliency-lab-for-linux#1738 and logical-minds-foundry/mq-resiliency-observability#72 (the docs job's `librt` has no Python 3.15 wheel and `base` has no compiler), and mnemosys-project/melete#265 (its build command needs `npm` on `PATH`, which the new images deliberately do not provide). They are fixed when work on those repos resumes, outside this epic. Both causes are deliberate design choices: the core Python is 3.15 and the core Node stays off `PATH`.
- **Consumer validation covers python and `base` only.** go, rust, cpp-clang, cpp-gcc and ts-node were never exercised by a real consumer before the cutover, because none is active (vergil-project/.github#1022).
- **Daily, not per-run, OS upgrades** for the `pkg-` images: a per-run value would rebuild the whole tree and every consumer's cache after every merge. So a security fix can take up to a day to reach a `dev-` build, and a released `prod-` image picks it up from the nightly cache-disabled rebuild.
- **The mq-rest-admin repos are excluded** throughout (runner labels, the guard's recount, consumer CI): their CI is broken for unrelated reasons and they are not in active development.
- **The plan's Delivery table and dependency graph** still name the dropped private repo's runner-label task. The plan is append-only once execution starts, so the last evolution entry supersedes them instead of rewriting them.

## §4 New problems and opportunities

Surfaced during the epic and already tracked:

| Problem or opportunity | Where it went |
|---|---|
| Review the May decision to run vergil-actions' CI on `dev-` images | intake vergil-project/.github#1016 |
| Consumer validation for the families with no active consumer (an integration-test repo) | vergil-project/.github#1022 |
| Trivy's version pin and its CVE gating | intake vergil-project/.github#1025 |
| The dnf repository's `repomd.xml`/`.asc` publish race | intake vergil-project/.github#1040 |
| `vrg-release` moves `vX.Y` before the package index has the release | intake vergil-project/.github#1044 |
| Agents cannot move a follow-on brainstorm from a development epic to planning | intake vergil-project/.github#1053 |
| Make "record every documentation-sweep deferral on the review issue" a skill rule | vergil-project/vergil-claude-plugin#1008 |
| Remove the semgrep wrapper once semgrep ships its fix | deferred vergil-project/vergil-containers#851 |
| The images page has no sections for cpp-clang, cpp-gcc and ts-node | intake vergil-project/vergil-containers#853 |
| `install.sh`'s header comment still says Debian | vergil-project/vergil-actions#1152 |
| Repo-unscoped cache tags | vergil-project/vergil-tooling#4162 (done) |
| The consumers' out-of-date configuration | logical-minds-foundry/mq-resiliency-lab-for-linux#1738, logical-minds-foundry/mq-resiliency-observability#72, mnemosys-project/melete#265 |

**The unaddressed-findings sweep.** 48 tasks, their 38 PRs, the epic's own comments and the planning issue's 7 design-review rounds were read; every comment was by a trusted author. Every design-review finding not accepted has its reason or intake ref, and no PR has review comments to check. These were found untracked:

| Problem | Outcome |
|---|---|
| vergil-containers#839, note 3: apt failover is slow; each unreachable Canonical host costs the full 20 s timeout on every `apt-get update` and `install` (67 s and 61 s in the test) | Filed: intake vergil-project/vergil-containers#854 |
| vergil-containers#839, note 2: the validation scaffold ran `apt-get update \| tee` without `pipefail` | Dropped: it affected only that one-off checklist, and the run's output showed success |
| vergil-containers#839, note 1: its step-1 criterion read as covering only the Ubuntu archive sources (`vergil.sources` and `github-cli.list` bypass `mirror+file:`) | Dropped, the reading confirmed: those are not Ubuntu archive mirrors |
| vergil-containers#839, note 4: apt's error names one host in the URL and another in the connect error | Dropped: apt's own error text, with a correct exit status |
| vergil-tooling `tests/vergil_tooling/test_retry.py`: `pytest.raises(..., match="")` always passes (seen in #988's amd64 log; predates the epic) | Filed: intake vergil-project/vergil-tooling#4266 |
| vergil-containers#788, ruling A: `vrg-commit` and `vrg-git branch -m` refuse throwaway `validate/*` branches, so a validation run cannot commit its own fixture branches | Filed: intake vergil-project/vergil-tooling#4267 |

The rulings on #839's four notes are recorded on that issue.

## §5 What's next

The follow-on brainstorms moved to the planning queue, vergil-project/.github#439, instead of holding this epic open:

- vergil-project/.github#651: which components can move to Ubuntu 26.04.
- vergil-project/.github#959: a Node runtime architecture modelled on the Python design, and TypeScript app support.
- vergil-project/.github#983: remove the cutover-era vergil-tooling install branch.

## Appendix A: Operational notes

The cutover and its follow-through, in order:

1. Release the package install paths for vergil-tooling and vergil-actions (#4131), so images and CI can install vergil-tooling from its package.
2. Build the new tree on `develop` as `dev-` images; validate consumers on `dev-` (#788) and apt on amd64 (#839).
3. Cut the vergil-containers cutover release (v2.1.38, #789): `prod-` images move to Ubuntu 24.04 in one release. Its revert path (reverting the family PRs) was documented and not needed.
4. Return package CI to `prod-` images (#4237) and pin the runners (#1113, #776).
5. Release vergil-vm with the Lima pin (v2.1.50) and `vrg-vm rebuild` each VM (#395); `vrg-vm update` does not change a VM's OS.
6. Release vergil-containers v2.1.39 with #844 and #845, then dispatch `ops.yml` for a no-cache rebuild of `prod-` and `dev-` (run 43261125313) and validate the result (#790).
7. Merge the `ubuntu-latest` guard (#4129) after a recount of every in-scope repo's default branch showed no remaining uses.

Gotchas:

- A rerun of an old workflow run reuses that run's workflow definition: the reruns in #790 ran on `ubuntu-26.04` runners from before the pins, and a PR CI rerun re-checks the version against today's `develop`.
- `nerdctl` in rootless mode cannot copy files out of a stopped container; checking an image of another architecture without emulation means saving it and replaying its layers.
- apt's `Debug::Acquire::http` does not log https requests; use `Debug::Acquire::https` for the vergil repository.
