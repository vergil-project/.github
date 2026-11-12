# Retrospective: manifest-driven monorepo for scalable container builds

- **Epic:** vergil-project/.github#213
- **Spec:** [spec.md](spec.md)
- **Plan:** [plan.md](plan.md), including its "Evolution during execution" log
- **Span:** opened 2026-08-05, plan rebased 2026-10-26, implemented 2026-11-09 to
  2026-11-10, validated 2026-11-12

## 0. At a glance

**What we set out to do.** vergil-containers built every image in one coupled
pipeline:

- every change rebuilt every image;
- the arm64 half of every image compiled under QEMU;
- one image's failure was noise for the whole run.

The epic aimed for a manifest-driven matrix, native arm64 builds, selective
builds, and failures that stay with the image that failed, all without
changing a single published image name or tag.

**What shipped.**

- Every image now builds natively on its own architecture's runner and is
  pushed by digest. A publish job stitches each two-platform index, then scans,
  attests and promotes it.
- `base` is part of the matrix. Images publish in tiers, ordered by their
  manifest parents.
- PR CI now builds and scans only the images a PR affects, behind one required
  gate, `images / evidence`.
- A failure stays with its own image.

The no-cache nightly went from about 65 minutes to 13–16 minutes per prefix with
one tier, and to about 20 minutes per prefix with the three tiers that epic #649
added later. QEMU is gone from the stack.

| Measure | Before (2026-10-26) | After |
|---|---|---|
| Nightly wall-clock | ~65 min | 13.2–15.6 min per prefix, one tier (2026-11-10); ~19–20 min per prefix, three tiers (2026-11-12) |
| Nightly build and publish runner time | ~566 runner-min | ~108 runner-min per prefix, ~216 in all (2026-11-10) |
| python 3.14 build step | 384s | 71–78s |
| go 1.26 build step | 1792s | 156–203s |
| rust 1.92 build step | 3013s | 212–286s |
| QEMU or binfmt in any workflow | yes (arm64) | none |

**Work delivered.**

| Task | PR | Merged | What it did |
|---|---|---|---|
| Docs .github#214 | .github#219 | 2026-08-05 | Spec and plan |
| Docs .github#528 | .github#529 | 2026-10-26 | Rebased the spec and plan onto the post-#407 tree |
| T7 vergil-containers#515 | vergil-containers#774 | 2026-11-09 | Native per-arch builds, push by digest, a stitched multi-arch manifest, `base` folded into the matrix |
| Docs .github#974 | .github#976 | 2026-11-09 | Amended the spec and plan for epic #649: T7b and parent-aware selection |
| T7b vergil-containers#773 | vergil-containers#775 | 2026-11-09 | Tiered publish from manifest parents, and a per-entry `build-args` map |
| T2 vergil-containers#510 | vergil-containers#784 | 2026-11-09 | The `@include` dependency graph for each image (`includes.py`) |
| T5 vergil-containers#513 | vergil-containers#791 | 2026-11-09 | Selects the images a change affects, from its paths and the matrix diff (`selection.py`) |
| T6′ vergil-containers#514 | vergil-containers#795 | 2026-11-09 | PR CI builds and scans only the selected images, behind the `images / evidence` gate |
| vergil-containers#794 | vergil-containers#796 | 2026-11-09 | A bounded, jittered retry of the per-arch push by digest when GHCR denies it |
| vergil-tooling#4132 | vergil-tooling#4147 | 2026-11-10 | `images / evidence` became a generated required gate behind `[ci].images`, with release evidence harvest |
| vergil-containers#798 | vergil-containers#805 | 2026-11-10 | Uploaded `ci-evidence-images` and set `[ci].images = true` |
| Docs review vergil-containers#497 | vergil-containers#817 | 2026-11-10 | Docs now describe the native per-arch publish and PR image selection |
| .github#1012 | .github#1013 | 2026-11-10 | Recorded the plan's evolution during execution |

**Closed without a PR:**

- Superseded by epic #407 at the rebase: T1 (#509), T3 (#511), T4 (#512).
- Dropped at the rebase: T0 (#508, the nightly already gives the baseline) and
  T8 (#516, won't-do).
- Operational tasks: deployment #799 and validation #517.
- #809, the vehicle for six throwaway PRs (vergil-containers #810 to #815),
  which were closed unmerged.

**Counts:**

- 21 tasks plus this retrospective, across 3 repos: vergil-containers (16),
  .github (5 including this one), vergil-tooling (1).
- 13 merged PRs.
- Releases: vergil-containers v2.1.36 (the code), v2.1.37 (the evidence upload)
  and v2.1.38 (the docs); vergil-tooling v2.1.296 (the gate support).

## 1. How the plan evolved

The plan changed shape twice before any code shipped, then picked up five
smaller changes during execution.

**The rebase (2026-10-26).** The epic sat unstarted for almost three months, and
the repo moved under it. Epic #407 had already delivered most of Stage A:

- a validated manifest (`manifest.yml`);
- the generated `matrix.json` and its drift gate;
- a CD that builds from the matrix.

Measurements also showed that the cost was compiled toolchains under QEMU, not
redundant shared downloads. So the rebase:

- moved native arm64 first, as the biggest win with no dependencies;
- dropped the instrumentation stage and the shared `common-tools` layer;
- kept post-merge CD building everything;
- pointed selectivity at PR CI, where a new requirement from #407 needed it:
  build and scan what a PR changes before it merges.

That turned the old push-triggered "selective publish" into T6′, a real
required check.

**The #649 amendment (2026-11-09).** Epic #649 (Ubuntu 24.04 as the shared base)
needed images to build on parents. Rather than retrofit that later, T7b was added
after T7. It brought `parent` and `depth` in the manifest, tiered publish, and
per-entry `build-args`. T2 and T5 became parent-aware: a parent's inputs flow to
its children, and selecting a parent selects its descendants. Validation
gained a tier-order check.

**During execution** (plan entries 1 to 7):

- **Two problems appeared once the new pipeline ran for real.**
  - GHCR sporadically denied pushes in the bursty push-by-digest pattern, which
    led to a bounded retry (#794).
  - T6′ hit two CI traps. Container jobs default to `sh -e`, which broke a
    bash test. A test that landed in parallel (#777) required `uv run` with
    PEP 723 headers.
- **The plan's one manual step could not work.** T6′ Step 2 said to add
  `images / evidence` to the ruleset by hand. But the fleet's required checks
  are generated by vergil-tooling and converged by a daily audit, so a hand
  edit would be removed as drift. It became three tasks: vergil-tooling#4132,
  vergil-containers#798 and #799.
- **A spec rule was overturned by ruling.** The spec said a `.trivyignore`
  change rebuilds nothing. The plan and code rebuild and scan everything, and
  the human kept that.
- **Validation had to wait for #649.** Tier order could only be shown once #649
  gave the matrix real parents, so #517 was held open for two days.
- **Validation needed a vehicle for its throwaway PRs.** A validation task
  cannot have PRs, so task #809 was added to carry them.

## 2. Lessons learned

- **A plan has a shelf life.** Three months on the shelf made half of it
  redundant. The rebase was the most valuable planning step in the epic. It
  re-measured before re-planning, and the measurement (compiled toolchains under
  QEMU, 55% of runner time in Go and Rust) re-ordered the whole epic. Next time,
  re-validate a plan against the tree and fresh data whenever it has sat for
  more than a few weeks.
- **Measure, then pick the biggest lever first.** Native arm64 alone delivered
  most of the win (build steps 5 to 11 times faster) and needed nothing else.
  The shared-layer optimisation the original spec featured was not worth doing.
- **"Required check" is infrastructure, not a click.** Any plan step that
  changes GitHub settings by hand is suspect in this fleet. Rulesets are
  generated and audited, so a new required gate is a tooling change, an evidence
  change and a deployment. Plans should name the tooling task up front.
- **Concurrent epics that touch the same files need an explicit amendment.**
  Amending #213 for #649 before T7b started (rather than reconciling afterwards)
  kept both epics consistent. The cost was that Validation waited for the other
  epic.
- **Validation needs PR vehicles and human steps planned in.** Throwaway PRs,
  a dispatched nightly and a human-only ruleset apply were all needed. None was
  in the plan, and each cost a round trip.
- **Tests that pin workflow shape pay for themselves.** For example,
  `test_every_run_step_is_bash` and the entry-point tests. The CI traps in T6′
  were caught and locked in rather than rediscovered.

## 3. Compromises and tradeoffs

- **Post-merge CD stays full** (human decision at the rebase). Selectivity lives
  only in PR CI. Every merge still proves every image builds, at the cost of a
  full publish per merge.
- **A `.trivyignore` change rebuilds and scans every image in PR CI** (human
  ruling, 2026-11-10). A suppression change is tested before merge, at the cost
  of a 36-job PR on rare `.trivyignore`-only changes.
- **Tiers trade wall-clock for correctness.** With three sequential tiers, the
  nightly takes about 20 minutes per prefix instead of 13–16. Each child always
  builds on its freshly published parent.
- **A `base` change rebuilds everything.** This is accepted as a property of the
  architecture, not a defect.
- **`selected.json` records the matrix's overall build result per job**, not
  each job's own result (accepted 2026-11-10). That's accurate on a passing run;
  per-job results would need an API call and `actions: read`.
- **T8, the shared `common-tools` layer, is won't-do.** Post-T7 timings did not
  justify reopening it.
- **The release/nightly prod-publish race was found but not fixed here.** They
  share `docker-publish-prod` with `cancel-in-progress: true`. It is tracked
  separately (vergil-containers#820).

## 4. New problems and opportunities

**Unaddressed-findings sweep.**

- Swept: 20 child tasks (all but this retrospective), the epic's comments and
  the 13 merged PRs.
- Every `Awaiting a ruling` item has a trusted ruling, and every issue filed by
  a `file` ruling exists.
- No item was ruled `fix` and left unfixed. No PR has reviews or review
  comments.
- There is no planning issue (the epic predates the planning workflow), so
  spec §9's deferred list was checked instead. Each item there is either
  delivered (T6′, the `build-args` map), taken up by epic #649 (D2) or moot (D1).
- Two candidates were already tracked, so nothing was filed:
  - The release/nightly prod-publish cancellation: found, vergil-containers#820.
    It repeats a CD/nightly clash fixed in vergil-containers#655 (2026-10-24),
    now on the release path.
  - Two depth-0 image failures in the 2026-11-12 nightly
    (`prod-pkg-ubuntu:26.04`, `dev-pkg-rhel:10`): found, vergil-containers#844
    and #845. Both images belong to epics #607 and #649.

**Surfaced while executing, outside this epic's scope.** Release and tooling
friction hit along the way, each routed to its own home:

| Item | Where it went |
|---|---|
| Promote refused vergil-actions' freeze-commit release tags | vergil-tooling#4140, fixed |
| `package-index` failed delivered releases on its 1200s timeout | vergil-tooling#4146 (raised to 1h), fixed; vergil-tooling#4145 (deadline semantics), closed |
| Package publication is too slow | .github#993, closed out of intake |
| Two releases in flight have no supported way out | .github#991, closed out of intake |
| `vrg-release` released a repo with `[publish] release = false` | vergil-tooling#4175, closed |
| The deploy scaffold called a human-only apply "agent-safe" | vergil-tooling#4176, closed |
| `--help` was hidden from agent identities | vergil-tooling#4178, closed |
| The host tools were installed on Python 3.14 | vergil-tooling#4157, fixed |
| Splitting vergil-tooling into independently released parts | .github#994, open (brainstorm) |
| `ci-images.yml` runs an unmanaged `aquasec/trivy:0.72.0` | .github#1025, open |

## 5. What's next

- **The no-emulation policy and guard:** .github#527 (open). This epic removed
  QEMU. #527 keeps it out.
- **Unmanaged Trivy in PR CI:** .github#1025 (open).
- **Prod publish concurrency:** vergil-containers#820 (open).
- **The next language:** .github#410 (open). A new language is now one manifest
  entry plus its templates, which is the "first add-a-language proof" the spec
  deferred.
- **Splitting vergil-tooling:** .github#994 (open brainstorm). This is release
  velocity, not containers, but it surfaced here.

## Appendix A: Operational notes

- **Rollout order:**
  1. T7: native arm64 and push by digest;
  2. T7b: tiers;
  3. T2, then T5: graph, then selector;
  4. T6′: PR CI;
  5. vergil-tooling#4132, released as v2.1.296;
  6. vergil-containers#798;
  7. #799: the human ran `vrg-github-repo-config apply`, because the agent
     identity lacks `administration=read`;
  8. a vergil-containers release (v2.1.37) to show the evidence harvest.
- **The workflows must be pushed by the human.** PRs touching
  `.github/workflows/**` were pushed by `vrg-submit-pr`, because GitHub refuses
  agent pushes of workflow files.
- **Validation evidence:**
  - cold publishes: Ops runs 42853985986 (2026-11-10, one tier) and 43261125313
    (2026-11-12, three tiers, all 118 jobs green);
  - throwaway PRs vergil-containers #810 to #815;
  - the full record is on vergil-containers#517.
- **Gotchas:**
  - GitHub container jobs default to `sh -e`.
  - GHCR denies bursty pushes now and then.
  - A full dev-VM disk blocks `vrg-validate`'s image build (it needs 3 GB
    free).
  - A release CD can cancel a running nightly's prod publish.
