# Language version lifecycle — Retrospective

## §0 At a glance

We set out to replace the hand-maintained language version lists scattered through the fleet with **one rolling support window, derived from upstream**. The plan:

- one manifest in vergil-containers that every version list is generated from;
- a nightly lifecycle job that notices upstream releases and end-of-life dates and opens the add and drop PRs;
- one resolver, `vrg-ci-resolve`, that every CI run and local validation uses;
- deployed Python runtimes that follow the same window.

**What shipped:** all of it, plus a long tail the epic surfaced on the way:

- **The manifest and window:** `docker/versions/manifest.yml` generates `window.json`, published at `v2.1`, with upstream GA and EOL dates.
- **The lifecycle job:** it opened the first real adds, Python 3.15 and Go 1.27, and the 3.15 runtime flip, and those were validated end to end. It reports gaps for Rust, Java, Clang and GCC.
- **The resolver:** `vrg-ci-resolve` drives every CI leg, `cd-release`'s container tag and image suffix, and local `vrg-container-run`. `[ci].versions` is an optional pin; an app builds one version, and a multi-pin is fatal.
- **The runtime:** vergil-tooling deploys on `vergil-python3.15`, which it derived from the window rather than a pin.

### Work delivered

- **Span:** epic opened 2026-10-21 (spec and plan merged that day, .github#419). Implementation ran 2026-10-21 → 2026-10-23.
- **Repos touched (9):** vergil-containers, vergil-tooling, vergil-actions, vergil-python, packages, docs, .github, logical-minds-foundry/mq-resiliency-lab-for-linux, mq-rest-admin-project/.github.
- **Tasks:**
  - 59 sub-issues: 58 closed, plus this retrospective. 6 were operational (deploy or validate), with no PR.
  - 2 cross-org children closed: lab#1681 (PR lab#1686) and mq-rest-admin .github#43 (archive `mq-rest-admin-ruby`).
  - 36 tasks were added mid-flight (§1), counted by issue number above each repo's bootstrap batch.
- **Releases cut in the window:**
  - vergil-tooling **12** (v2.1.251 → v2.1.262);
  - vergil-actions **11** (v2.1.45 → v2.1.55);
  - vergil-containers **3** (v2.1.29 → v2.1.31);
  - vergil-python **2** (v1.0.2, v1.0.3).

  Some tooling and actions releases also carried other epics' work, such as the Dependabot epic, so these are window counts, not attribution.
- **PRs (53 merged), grouped by repo.** 52 closed epic sub-issues; one, lab#1686, closed the cross-org lab child.

| Repo | PR | Task | What it did |
| --- | --- | --- | --- |
| .github | #419 | #408 | Spec and plan for the epic |
| .github | #434 | #433 | Spec and plan corrected to the final behaviour |
| vergil-containers | #623 | #613 | Every version list generated from one manifest |
| vergil-containers | #627 | #614 | Per-language support policies |
| vergil-containers | #628 | #615 | Lifecycle check core with the Python and Go adapters |
| vergil-containers | #633 | #616 | Report-only adapters: Rust, Java, Node, Clang, GCC |
| vergil-containers | #635 | #617 | Nightly `language-lifecycle` job, with issues for gaps and failures |
| vergil-containers | #636 | #618 | Add and drop PRs for Python and Go |
| vergil-containers | #625 | #619 | Ruby and GCC 13 dropped from the image matrix |
| vergil-containers | #638 | #621 | `runtime_published` recorded from the apt index |
| vergil-containers | #658 | #654 | Lifecycle tests on a frozen manifest, not the live one |
| vergil-containers | #663 | #656 | Pins report tests on the frozen manifest |
| vergil-containers | #670 | #637 | #622 merge hold removed |
| vergil-containers | #672 | #610 | Documentation review |
| vergil-containers | #674 | #673 | Upstream GA and EOL dates for the seeded versions |
| vergil-tooling | #3393 | #3367 | One resolver for CI versions, from the published window |
| vergil-tooling | #3401 | #3368 | Python extension: toolchain-selection and floor checks |
| vergil-tooling | #3382 | #3369 | Go 1.25 retirement and Ruby removed from docs |
| vergil-tooling | #3388 | #3377 | Ruby support removed from the code (breaking) |
| vergil-tooling | #3402 | #3392 | `VRG_VERSION_WINDOW` honoured in `vrg-container-run` |
| vergil-tooling | #3404 | #3400 | vergil-tooling made a `library` |
| vergil-tooling | #3408 | #3371 | vergil-tooling's CI pin removed; follows the window |
| vergil-tooling | #3445 | #3372 | An app multi-pin is fatal (breaking) |
| vergil-tooling | #3413 | #3410 | uv `python-preference` allowed; `UV_PYTHON_PREFERENCE=only-system` in containers |
| vergil-tooling | #3422 | #3421 | Cache-build warmup gets `UV_PYTHON` / `UV_PYTHON_PREFERENCE` |
| vergil-tooling | #3448 | #3414 | `vrg-container-docs` pins the interpreter source |
| vergil-tooling | #3452 | #3442 | Strict-pins transition switch and resolution warnings removed |
| vergil-tooling | #3458 | #3455 | Runtime pin installs the per-series package; vergil-tooling's runtime derived |
| vergil-tooling | #3468 | #3456 | `vrg-update-deps` bumps pins against the per-series packages |
| vergil-tooling | #3483 | #3479 | Index `[retention].retire` |
| vergil-tooling | #3485 | #3480 | `vrg-vm update` purges obsolete runtimes |
| vergil-tooling | #3489 | #3482 | Docs and the rewritten Python standards pages |
| vergil-tooling | #3495 | #3484 | `repo_init` and vergil-tooling stop passing `container-tag` |
| vergil-tooling | #3496 | #3478 | Runtime resolved once per release; `check-runtime` |
| vergil-tooling | #3497 | #3494 | `vrg-release-validate-inputs` accepts the resolver's tag for every language |
| vergil-actions | #990 | #962 | `derive.sh` via `vrg-ci-resolve`; `assert-toolchain` on every leg |
| vergil-actions | #995 | #994 | `pin-python`: `UV_PYTHON_PREFERENCE` on Python legs |
| vergil-actions | #1005 | #1002 | Interpreter source pinned in the docs workflows |
| vergil-actions | #1010 | #1003 | `detect.sh` ignores repo uv settings (`--no-config`) |
| vergil-actions | #1021 | #1016 | Docs: resolver-driven CI, removed inputs |
| vergil-actions | #1023 | #1022 | `derive.sh` harness: stub resolver; app multi-pin is fatal |
| vergil-actions | #1024 | #1017 | `cd-release` container tag from the resolver |
| vergil-actions | #1030 | #1020 | `cd-release` image suffix from the resolver |
| vergil-actions | #1034 | #1028 | CI `common` and `version-bump` images from the resolver |
| vergil-actions | #1038 | #1029 | Recorded runtime passed to package builds; `check-runtime` before signing |
| vergil-python | #27 | #25 | One runtime package per Python series |
| vergil-python | #35 | #34 | Series moved to 3.15 (CPython 3.15.1) |
| vergil-python | #42 | #41 | Runtime package pattern docs |
| vergil-python | #44 | #43 | `cd.yml` stops passing `container-tag` |
| packages | #21 | #17 | `vergil-python3.14.8` retired from the index |
| packages | #20 | #18 | `cd.yml` stops passing `container-tag` |
| docs | #45 | #43 | Version window and per-series runtime docs |
| lab | #1686 | #1681 | Lab follows the window's toolchain rules (keeps a 3.14 pin) |

## §1 How the plan evolved

The plan had no separate "Evolution during execution" log. The dated "As built" notes added to `plan.md` (#433) and the decision comments on #407 are the record. This narrative is drawn from them.

**The core sequence held.** It ran T1 → T7 → D1 (containers), then T8 → T10 → D2 (tooling), T11 → D3 (actions), T15 (sweep), T14 (fatal multi-pin) and T12 (first-adds validation), in that order. What moved was scope at the edges, nearly all of it from **contact with real repos and real releases**:

- **vergil-tooling became a `library` early (#3400).** Under the app rule, its CI would have dropped to one version when D3 shipped. It moved forward, kept its pin until D3, and then the pin was removed (T15a, which doubled as D3's probe PR because no other vergil-tooling PR existed yet).
- **Packaged libraries keep the whole matrix.** The human decided that the runtime rule picks an app's CI version, but only a *library's deployed runtime*.
- **The fleet sweep shrank.** Four of six rows moved to their repos' ad-hoc epics, because those repos are on hold and can't take PRs. The lab, the one active consumer, kept a CI pin (lab#1685) for a reason the spec hadn't anticipated (see §2).
- **Spec §6.5 was amended.** uv `python-preference` (and later `python-downloads`) choose the interpreter *source*, not the version, so repos may keep them. Vergil now sets `UV_PYTHON_PREFERENCE=only-system` everywhere it runs Python. Finding "everywhere" took five tasks: the container run (#3410), the cache build (#3421), the CI legs (#994), the docs jobs (#1002) and `vrg-container-docs` (#3414). Each showed up only when the previous fix met a repo that used the setting.
- **T12 needed blockers the plan didn't have.** The lifecycle tests were built on the live manifest, so the very first add PR failed its own CI (#654). vergil-containers' PR CI never builds images, so the check moved to CD on develop, and PR-CI image builds were deferred to #213. And no task existed for "the vergil-python 3.15 runtime is released" (#34).
- **The runtime model was only half built.** Moving vergil-python to 3.15 exposed four problems:
  - an explicit pin had been installing the **pre-T13 per-patch package**, so vergil-tooling never used the T13 runtime (#3455);
  - the old series freezes (multi-series, #36, deferred: it needs multi-package support in vergil-tooling);
  - one release shipped different runtimes per architecture (#3478, plus actions #1029);
  - the legacy package needed retiring from the index and the VMs (#3479, #3480, packages#17).
- **`cd-release` hadn't followed CI onto the resolver.** It still took a caller tag that `repo_init` had frozen at init time (#1017, #3484), still built its image suffix from caller inputs (#1020, #1028), and rejected the resolver's tag for C++ and TypeScript (#3494).
- **The #622 hold** was added (T6), proved itself on the real adds, and was removed (#637), as planned.

## §2 Lessons learned

- **The first real run is the real test.** The design was sound, but each "first" exposed an assumption: the first real add PR, the first runtime move, the first consumer with `only-managed`, the first release after a window flip. Next time, plan a probe for each first-of-a-kind event, and keep validation tasks (T12) blocking until those probes have passed, rather than until the code merges.
- **Tests must not read live data the thing under test is about to change.** #654 and #656 were the same bug twice. A test that asserts today's manifest breaks the moment automation changes the manifest. Frozen fixtures should be the default for any data-driven automation.
- **"Everywhere" needs an inventory.** The `UV_PYTHON_PREFERENCE` rollout took five tasks because no one listed every place Vergil runs Python up front. When a rule has to hold "wherever X happens", list the places first; #1002's repo-wide guard test is the model.
- **Cross-repo releases break each other silently.** T14's release broke vergil-actions' `derive.sh` harness. Nothing caught it until an unrelated PR ran (#1022). vergil-actions#1042 adds a nightly harness run so the next one shows up the same day.
- **GitHub's eventual consistency is a real input.** On 2026-10-22, delayed Actions results produced "orphaned" check-runs (#3431). Commit-indexed run lookups lagged behind runs that had finished (#3434). A cron fired 5–7 hours late, and concurrency groups cancelled each other's runs (#655). Gates that read GitHub state need bounded grace periods, not first-answer failure.
- **Fail-loud paid off repeatedly.** Silent fallbacks would have hidden:
  - the wrong-Node fallback (#657);
  - the unpatched legacy runtime (#3455);
  - the mixed per-arch release (#3478);
  - an unmatched `retire` entry (#3479's added check).

  The places that weren't loud (`only-managed` silently downloading in docs and cache builds) caused most of the rework.
- **Same next time:**
  - the human-gated release sequence (D1–D4);
  - the per-repo batching;
  - recording decisions as issue comments when they're made. The §1 narrative could be reconstructed only because of them.

## §3 Compromises & tradeoffs

- **The lab keeps a pin (lab#1685).** It's an unpackaged app whose real runtime is the VM's uv venv at the floor. Unpinned, CI would test 3.15 while the lab runs 3.14. It's a pin with a linked issue, waiting on the host-runtime design (.github#425).
- **vergil-python maintains one series.** 3.14 is frozen at 3.14.8+20261013. Accepted, because nothing deploys on 3.14 any more. Multi-series (#36) needs vergil-tooling to support several packages per repo, which is real work and deferred.
- **No image builds in PR CI.** A version add that fails to build or scan is caught in CD after merge (#213).
- **`cd-release` keeps a deprecated `container-tag` input.** GitHub rejects undeclared inputs, and the remaining callers are on-hold repos. The input is ignored with a warning, and removal (#1019) waits for their modernization.
- **The sweep left the on-hold repos unswept.** Each has a record issue, and the dates, pins and `container-tag` items were added to them.
- **Docs builds pin only the source, not the version.** They run on `prod-base`'s Python. A repo whose floor is newer than the image's Python fails loudly; there's no fallback.
- **`check-runtime` is tested offline for .deb only.** The rpm path is first exercised by the next real vergil-tooling release.

## §4 New problems & opportunities

| Surfaced | Where it went |
| --- | --- |
| Multi-series runtime packages | vergil-python#36 (needs vergil-tooling multi-package support) |
| PR-CI image builds for version adds | epic .github#213 |
| Advisory pre-release toolchain legs (human's idea) | .github#424 (intake) |
| Host-run app runtimes | .github#425 (intake) |
| `rust-toolchain.toml` as a toolchain setting | note on .github#410 |
| Orphaned check-runs from late results | vergil-tooling#3431 (open) |
| Evidence gate vs. lagging run index | vergil-tooling#3434 (open) |
| CD and nightly cancel each other | vergil-containers#655 (open) |
| NodeSource silently falls back to distro Node | vergil-containers#657 (open) |
| Cache images fill agent-VM disks | vergil-tooling#3481 (open) |
| `vrg-container-docs` skips CI's docs staging | vergil-tooling#3439 (open) |
| Local validate misses `docker/pins` tests | vergil-containers#662 (open) |
| `packages.toml` not validated before publish | packages#19 (open) |
| uv guard misses `uv --quiet run` | vergil-actions#1009 (open) |
| Bump-tools doesn't retire tool CVEs | vergil-containers#653 (open) |
| Package cleanup 429s | vergil-containers#652 (open) |
| Conflicts on an already-submitted PR | .github#435 (triage) |
| `vrg-gh` allowlist review (`pr update-branch` and more) | vergil-tooling#3420 (open) |
| Bare `#N` epic refs in `vrg-epic-audit` | vergil-tooling#3415 (open); `epic-move` and `adhoc` fixed (#3409, #3411) |
| Stale README, CLAUDE.md and links | vergil-containers#671, vergil-actions#1018, docs#44 (open) |
| Lab guest-runtime pin | lab#1689 (on hold) |
| A tooling release breaking an actions harness | vergil-actions#1042: nightly harness run against the latest `v2.1` (open) |

## §5 What's next

These follow-on brainstorms were moved to the intake queue rather than run inside the epic:

- **.github#424:** advisory pre-release toolchain support, promoted to required at GA;
- **.github#425:** version resolution for apps whose runtime lives on a host;
- **.github#410:** which language gets full lifecycle support next (Rust first);
- **.github#409:** per-language sample repos as an integration testbed.

Related: epic .github#213 (manifest-driven container builds, including PR-CI image builds) and vergil-python#36 (multi-series runtimes). The on-hold repos' modernization records (observability#49, replay-lab#314, melete#246, diogenes#248, mq-protocol-gateway#25 and the mq-rest-admin record issues) hold the sweep's remaining work.

## Appendix A — Operational notes

- **Release order that worked:**
  1. vergil-containers (window);
  2. vergil-python (runtime);
  3. vergil-tooling (resolver);
  4. vergil-actions (CI);
  5. consumers.

  Each D-task verified the previous release from the next layer down.
- **The packaged runtime moves only when the *window* says the runtime is published.** That's the lifecycle job's `runtime` PR (vergil-containers#666) plus a vergil-containers release. Then the next vergil-tooling release builds on it. A vergil-tooling release cut minutes after the window changed (2.1.258) split by architecture; #3478 and #1029 now prevent that.
- **Retiring a package:** add it to `[retention].retire` in `packages.toml`, then dispatch `publish-index.yml`. The rebuild is from scratch, so the package disappears from every format and arch (verified on 2026-10-23 for deb resolute amd64 and arm64, and rpm el9 and el10).
- **VM cleanup:** `vrg-vm update` (2.1.261 or later) purges obsolete `vergil-python*` runtimes. Cache images still need manual pruning until #3481 lands; 106 finished-branch caches were removed on 2026-10-23.
