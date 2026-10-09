# Retrospective — binary package publishing

> **Epic:** vergil-project/.github#356 · **Spec:** `epics/356-binary-package-publishing/spec.md` · **Plan:** `epics/356-binary-package-publishing/plan.md`
> Reading order: **spec → plan → retrospective**.

## §0 At a glance

We set out to make Vergil releases build signed `.deb` and `.rpm` packages and
publish them to a per-org package repository on GitHub Pages. The targets were
Ubuntu 24.04/26.04 and RHEL 9/10 on amd64 and arm64. vergil-tooling was to be
the first product, dogfooded by installing it on the agent VMs. That shipped.

- **Repository:** <https://vergil-project.github.io/packages> serves a signed
  apt suite and dnf repositories.
- **Products indexed:** a fingerprint-pinned keyring package, a pinned
  CPython runtime package (`vergil-python3.14.8`), and vergil-tooling
  2.1.231–2.1.233.
- **Agent VMs:** every Lima and cloud VM now installs vergil-tooling from the
  package rather than `uv`.
- **Validation:** a cold rebuild of both VM kinds, clean `dnf` installs on
  UBI 9/10 for both architectures, and a tampered-index rejection test all
  passed.

### Work delivered

Planned tasks (plan IDs in brackets):

| PR | Repo | What it did |
|---|---|---|
| [.github#360](https://github.com/vergil-project/.github/pull/360) | .github | Spec and plan (documentation bookend, #357) |
| [vergil-tooling#3087](https://github.com/vergil-project/vergil-tooling/pull/3087) | vergil-tooling | [T1] Target registry, `[package]` config, `vrg-package matrix` (#3074) |
| [vergil-tooling#3092](https://github.com/vergil-project/vergil-tooling/pull/3092) | vergil-tooling | [T2] Builder interface, staged builder, nFPM, glibc floor guard (#3075) |
| [vergil-tooling#3093](https://github.com/vergil-project/vergil-tooling/pull/3093) | vergil-tooling | [T3] Fingerprint-pinned apt/dnf trust bootstrap (#3076) |
| [vergil-tooling#3095](https://github.com/vergil-project/vergil-tooling/pull/3095) | vergil-tooling | [T4] Python builder: venv from `uv.lock` on the pinned runtime (#3077) |
| [vergil-tooling#3096](https://github.com/vergil-project/vergil-tooling/pull/3096) | vergil-tooling | [T5] `vrg-package install-test` (#3078) |
| [vergil-tooling#3089](https://github.com/vergil-project/vergil-tooling/pull/3089) | vergil-tooling | [T6] Index part 1: collect, verify provenance, retention (#3079) |
| [vergil-tooling#3094](https://github.com/vergil-project/vergil-tooling/pull/3094) | vergil-tooling | [T7] Index part 2: apt/dnf metadata, signing, `vrg-package index` (#3080) |
| [vergil-tooling#3090](https://github.com/vergil-project/vergil-tooling/pull/3090) | vergil-tooling | [T8] `package / evidence` gate (#3081) |
| [vergil-tooling#3097](https://github.com/vergil-project/vergil-tooling/pull/3097) | vergil-tooling | [T9] `vrg-release` deferred package-index stage (#3082) |
| [vergil-tooling#3098](https://github.com/vergil-project/vergil-tooling/pull/3098) | vergil-tooling | [T10] VM provisioning installs from the package repository (#3083) |
| [vergil-tooling#3124](https://github.com/vergil-project/vergil-tooling/pull/3124) | vergil-tooling | [T11] vergil-tooling adopts `[package]` (#3084) |
| [vergil-actions#907](https://github.com/vergil-project/vergil-actions/pull/907) | vergil-actions | [A1] `ci-package.yml` and the package setup action (#904) |
| [vergil-actions#909](https://github.com/vergil-project/vergil-actions/pull/909) | vergil-actions | [A2] `cd-release`: package build, main-only signing, attach, dispatch (#905) |
| [vergil-actions#908](https://github.com/vergil-project/vergil-actions/pull/908) | vergil-actions | [A3] `publish-index.yml` (#906) |
| [packages#3](https://github.com/vergil-project/packages/pull/3) | packages | [P1] Keyring product and index config (#1) |
| [vergil-python#3](https://github.com/vergil-project/vergil-python/pull/3) | vergil-python | [P2] Pinned CPython runtime package (#1) |
| [vergil-tooling#3145](https://github.com/vergil-project/vergil-tooling/pull/3145) | vergil-tooling | Documentation review bookend (#3072) |

Operational tasks:

- **OP1** (vergil-tooling#3073): trust roots, environments and repos;
  human-attested.
- **DEP1** (packages#2): repository live with the keyring.
- **DEP2** (vergil-python#2): runtime indexed.
- **DEP3** (vergil-tooling#3085): packaged tooling live, VMs switched. It
  passed on attempt 2.
- **VAL1** (vergil-tooling#3086): cold-rebuild validation; all 4 checks
  passed.

Added during execution:

| PR | Repo | What it did |
|---|---|---|
| [vergil-tooling#3091](https://github.com/vergil-project/vergil-tooling/pull/3091) | vergil-tooling | Allow the Public Domain license in the dependency audit (#3088) |
| [vergil-actions#914](https://github.com/vergil-project/vergil-actions/pull/914) | vergil-actions | Remove declared-but-unpassed signing secrets (#913; harmless, not the cause, see §1) |
| [packages#7](https://github.com/vergil-project/packages/pull/7) | packages | Restore `secrets: inherit` on the cd and publish-index callers (#6) |
| [vergil-python#5](https://github.com/vergil-project/vergil-python/pull/5) | vergil-python | `secrets: inherit` on the cd-release caller (#4) |
| [vergil-actions#919](https://github.com/vergil-project/vergil-actions/pull/919) | vergil-actions | Document `secrets: inherit` for environment secrets (#918) |
| [vergil-tooling#3108](https://github.com/vergil-project/vergil-tooling/pull/3108) | vergil-tooling | SARIF gate honors accepted in-source and external suppressions (#3107) |
| [vergil-tooling#3115](https://github.com/vergil-project/vergil-tooling/pull/3115) | vergil-tooling | `vrg-sarif-filter`: upload SARIF without accepted-suppressed results (#3114) |
| [vergil-actions#921](https://github.com/vergil-project/vergil-actions/pull/921) | vergil-actions | Filtered SARIF upload, so code scanning agrees with the gate (#920) |
| [vergil-actions#926](https://github.com/vergil-project/vergil-actions/pull/926) | vergil-actions | `cd-release`: fail closed when the CI-evidence gate or attach fails (#925) |
| [vergil-tooling#3120](https://github.com/vergil-project/vergil-tooling/pull/3120) | vergil-tooling | `vrg-ci-evidence assemble`: clear error, not a traceback, on missing harvest state (#3119) |
| [vergil-tooling#3103](https://github.com/vergil-project/vergil-tooling/pull/3103) | vergil-tooling | confirm-main settles on the release job by leaf name (#3102) |
| [vergil-tooling#3126](https://github.com/vergil-project/vergil-tooling/pull/3126) | vergil-tooling | Escape glob metacharacters in nFPM `src` paths (#3125) |
| [vergil-tooling#3128](https://github.com/vergil-project/vergil-tooling/pull/3128) | vergil-tooling | Reduced package matrix: all builds, one install-test per format (#3127) |
| [vergil-actions#932](https://github.com/vergil-project/vergil-actions/pull/932) | vergil-actions | Tiered package CI: reduced on feature PRs, full on release PRs (#930) |
| [vergil-actions#933](https://github.com/vergil-project/vergil-actions/pull/933) | vergil-actions | Package setup: Azure mirror, one apt update, fail-fast apt (#931) |
| [vergil-tooling#3133](https://github.com/vergil-project/vergil-tooling/pull/3133) | vergil-tooling | Scoped apt update and fail-fast apt options in bootstrap and builder (#3131) |
| [vergil-tooling#3140](https://github.com/vergil-project/vergil-tooling/pull/3140) | vergil-tooling | Idempotent trust bootstrap (#3138) |
| [vergil-tooling#3146](https://github.com/vergil-project/vergil-tooling/pull/3146) | vergil-tooling | `repo-init` generates packaging-capable `cd.yml`/`ci.yml`; adopt keeps hand-written `vergil.toml` tables (#3144) |
| [vergil-actions#938](https://github.com/vergil-project/vergil-actions/pull/938) | vergil-actions | Docs: `secrets: inherit` for packaged callers, package workflow pages (#937) |
| [packages#12](https://github.com/vergil-project/packages/pull/12) | packages | Docs: README status, VM bootstrap owner, adding a product (#11) |
| [vergil-python#10](https://github.com/vergil-project/vergil-python/pull/10) | vergil-python | Docs: release status and releasing paragraph (#9) |
| [docs#36](https://github.com/vergil-project/docs/pull/36) | docs | Docs: package repository page, release packaging in CI (#35) |

Two more fixes came out of this work and were filed under vergil-tooling's
ad-hoc epic, not this one:

- [vergil-tooling#3113](https://github.com/vergil-project/vergil-tooling/pull/3113)
  keeps Python bytecode off the bind mount (#3111).
- [vergil-tooling#3139](https://github.com/vergil-project/vergil-tooling/pull/3139)
  retries a 404 on GitHub resources it has only just discovered (#3137).

### By the numbers

- **Repos touched:** 6 — `.github`, `vergil-tooling`, `vergil-actions`,
  `packages`, `vergil-python`, `docs`.
- **Children:** 46. That is 21 planned items (OP1, T1–T11, A1–A3, P1–P2,
  DEP1–DEP3, VAL1), all closed, and 3 bookends (spec, docs review, this
  retrospective). The other **22 were added during execution**. The macOS
  follow-on bookend (#358) was moved to the ad-hoc epic and closed as not
  planned (§5).
- **PRs merged:** 42 feature PRs (40 closing children, 2 ad-hoc fixes) plus
  31 release and back-merge PRs, so 73 in total.
- **Releases:** 16 tags.
  - vergil-tooling v2.1.226–v2.1.233 (8). v2.1.231 is the first packaged
    release.
  - vergil-actions v2.1.37–v2.1.41 (5).
  - packages v1.0.0 and v1.1.1 (2). The origin of the v1.0.0 tag is
    **unknown**: it targets `develop` and matches no release PR.
  - vergil-python v1.0.0 (1).
  - packages 1.1.0 was merged but never tagged (§3).
- **Span:** opened 2026-10-05 17:12Z; the last child before this one closed
  2026-10-09 13:10Z, about 3.8 days. The epic closes when this PR merges.

## §1 How the plan evolved

The spine held: tooling (T1–T11), workflows (A1–A3), the two new repos (P1–P2),
then deploy (DEP1–3) and validate (VAL1). Every planned item shipped. The delta
is the **22 unplanned children**, a little over twice the planned code tasks.
Almost none of them were scope creep. They are what turning the pipeline on
against real infrastructure exposed, and they fall into four clusters.

**1. The `secrets: inherit` reversal.** The plan's callers used
`secrets: inherit`. Semgrep's `secrets-inherit` rule rejected it. The fleet had
also been on explicit least-privilege maps since epics #189 and #197. So the
first correction swapped to an explicit map, and that left `package-sign` with
an empty signing key.

We first blamed declared-but-unpassed secrets shadowing the environment ones
(vergil-actions#913). That was a plausible theory, a real cleanup, and **not
the cause**. A minimal probe settled it (packages#6, run 37507168545):
GitHub delivers *environment* secrets to a job in a cross-repo reusable
workflow **only** under `secrets: inherit`. Both corrections are recorded as
comments on the epic.

The real fix (`inherit` plus a scoped, justified `nosemgrep`) then exposed that
the SARIF gate ignored in-source suppressions (vergil-tooling#3107). Fixing that
exposed that GitHub's code scanning still counted the suppressed results. So the
uploaded SARIF is now filtered (#3114, actions#920) while the evidence bundle
keeps the full file. One wrong premise cost six issues, plus an abandoned
packages 1.1.0.

**2. The release machinery under real load.**

- The CI-evidence gate failed *open*: a failed gate still let the release
  proceed (actions#925). Guard steps now fail closed.
- confirm-main settled on the wrong job (#3102), and 404'd on a run GitHub had
  only just listed (#3137).
- An evidence-assembly traceback became a clear error (#3119).

**3. Packaging on real hosts.**

- nFPM treated `{name}.tmpl` as a glob (#3125).
- amd64 build and sign jobs hung on apt. The cause is a known apt deadlock when
  delayed retries meet the Azure mirror-list failover (Launchpad #2003851). The
  CI setup action got `Retries::Delay "false"`, a 60 s `DPkg::Lock::Timeout`,
  bounded retries and timeouts, and job `timeout-minutes` (actions#931).
  vergil-tooling got `Acquire::Retries=3`, 20 s http/https timeouts and a
  scoped `apt-get update` (#3131); vergil-tooling#3150 adds the other two
  options there.
- The full 8-cell install-test matrix on every PR was too slow and too costly,
  so package CI became tiered (actions#930, #3127). Feature PRs install-test
  the **oldest** release per format; release PRs run the full matrix.

**4. Dogfooding found what tests didn't.**

- DEP3 attempt 1 failed on a box an earlier update had already migrated. The
  trust bootstrap wrote a source that conflicted with the keyring's own
  `Signed-By` (#3138). It passed on attempt 2, twice in a row on every VM.
- The documentation sweep found that `vrg-repo-init` could not generate a
  working packaged repo (#3144). Fixing it surfaced a latent data-loss bug:
  adopt mode rewrote `vergil.toml` and dropped every hand-written table,
  `[package]` included.

**VAL1 adapted to infrastructure reality.** No Vergil VM runs RHEL, so the
"clean `dnf` install" check ran in throwaway UBI 9/10 containers on each
architecture. That was always the check's wording. The single cloud amd64 VM is
shared with the logical-minds-foundry lab. Its rebuild was scheduled around the
lab, and the checks ran read-only plus `--rm` containers, leaving the lab
workspace clean.

**What the plan didn't capture.** `plan.md` has no "Evolution during execution"
log. Deviations were recorded only as two epic comments, plus the per-issue
`Outcome:` records. This narrative was rebuilt from those and from the issue
graph. See §2.

## §2 Lessons learned

- **Probe platform semantics; don't theorize.** The shadowing theory cost a
  release (packages 1.1.0) and a misdirected fix. A minimal probe workflow answered the
  question definitively. For "does GitHub do X across a reusable-workflow
  boundary", write the probe first.
- **A fleet policy can carry a hidden premise.** The explicit-secrets policy
  (#189/#197) was correct for *repository* secrets. It silently assumed no
  caller would need *environment* secrets. Policies like this should state their
  premise, so the first counter-example reads as a scoped exception rather than
  a contradiction.
- **Every gate must agree with every other view of the same data.** The
  internal SARIF gate, GitHub code scanning and the evidence bundle disagreed
  about suppressions. Each disagreement was its own outage. The fix pattern
  (gate on the effective suppression, upload a filtered copy, archive the full
  file) generalizes.
- **Fail closed, and test the failure path.** The evidence gate's fail-open
  existed because nothing exercised a failing gate inside a release. Guard steps
  that assert the success outputs (`passed=true`, `attached=true`) are cheap.
- **Idempotency is a deployment requirement, not a nicety.** Unit tests ran
  the bootstrap once on a clean host. Real VMs run it on every update. Any
  provisioning step should be tested twice in a row on a migrated host.
- **The docs sweep is a real review pass.** It found a code gap (#3144) and a
  latent data-loss bug, not just stale prose. It earns its place before the
  retrospective.
- **CI apt is infrastructure, not a given.** Mirror failover, lock contention
  and full index downloads each turned a routine job into a hang that ran to the job timeout. The
  hardened settings now live in one place, the setup action.
- **Keep the evolution log as you go.** Rebuilding §1 from comments and the
  issue graph worked but was lossy: why each unplanned child was filed had to
  come from memory and issue bodies. Next time, append a dated line to
  `plan.md` (or an epic comment) whenever the plan changes.

## §3 Compromises & tradeoffs

- **`secrets: inherit` on packaged callers.** It hands the first-party callee
  every caller secret, not just the signing pair. It is required by the
  platform, scoped to repos with `[package]`, and suppressed with a justified
  `nosemgrep`. Repos without `[package]` keep explicit maps.
- **Reduced package CI on feature PRs.** A feature PR builds every cell but
  install-tests only the oldest release per format, amd64 first. A defect
  specific to the newest release or to arm64 is caught at the release PR,
  not the feature PR.
- **`.deb` files are covered by the signed index, not signed individually.**
  `.rpm` files are signed individually. Every package carries a build
  attestation that `publish-index` verifies, pinned to `cd-release.yml` on
  `refs/heads/main`.
- **The runtime pin is exact on CPython, floating on the build.** The
  dependency is `vergil-python3.14.8 (>= 3.14.8+20261003)`. The CPython patch
  is in the package name; newer PBS builds of the same patch satisfy it.
- **VAL1's RHEL coverage is userland, not a VM.** UBI containers exercise
  `dnf`, signature checks and installed-file ownership. They do not exercise
  systemd or SELinux enforcing, which the packages don't depend on.
- **packages 1.1.0 was abandoned, not reverted.** It was merged to `main` and
  never tagged after `package-sign` broke (§1). 1.1.1 superseded it, and its
  CHANGELOG and release notes mark 1.1.0 as not published.
- **#925's root cause is unknown.** The fail-open was closed by guard steps
  that assert the gate's outputs. Why the gate step's failure didn't stop the
  job was not established.
- **apt hardening covers bootstrap and CI, not every VM apt call.** The retry
  and timeout options apply to the trust-bootstrap and builder apt calls. The
  plain `apt-get update` and `apt-get install` in `vm_packages.py` run without
  them, and that `apt-get update` is a full, unscoped update (only the
  trust-bootstrap update is scoped to `vergil.sources`), so a slow or dead
  mirror could stall a VM install. Fixed: vergil-tooling#3149 routes both calls
  through the shared `apt_get()` options.

## §4 New problems & opportunities

| Surfaced | Where it went |
|---|---|
| Environment secrets need `secrets: inherit` across reusable workflows | Fixed: packages#6, vergil-python#4, actions#918; documented fleet-wide (#3072, actions#937, docs#35) |
| SARIF gate ignored suppressions; code scanning counted suppressed results | Fixed: vergil-tooling#3107, #3114, actions#920 |
| CI-evidence gate failed open | Fixed with guards: actions#925; root cause logged, not yet acted on |
| Corrupt `.pyc` on the macOS↔container bind mount | Fixed: vergil-tooling#3111 (ad-hoc) |
| GitHub list-before-readable 404 lag | Fixed: vergil-tooling#3137 (ad-hoc) |
| apt mirror deadlock and lock contention in CI | Fixed: actions#931 (all five options, incl. `Retries::Delay`, `DPkg::Lock::Timeout`), vergil-tooling#3131 (retries and timeouts only); vergil-tooling#3150 adds the other two to vergil-tooling |
| Non-idempotent trust bootstrap | Fixed: vergil-tooling#3138 |
| `repo-init` adopt dropped hand-written `vergil.toml` tables | Fixed: vergil-tooling#3144 |
| `vrg-validate` lints shell and Markdown only in fixed directories (`packaging/*.sh`, `docs/*.md` unchecked) | Triage: .github#362, open |
| A retry for GitHub GraphQL "Something went wrong" errors (release recovery needed manual re-runs) | Fixed: vergil-tooling#3148 (was .github#363); vergil-tooling#3151 extends retries to `pr_checks` and `failing_checks` |
| VM-side apt calls lack the retry and timeout options (§3) | Fixed: vergil-tooling#3149 (was .github#364) |
| A vanity domain for the package repository (once the business entity exists) | Idea: .github#365 (needs a keyring release to move the baked-in source URL) |

## §5 What's next

- **macOS as an install target: deferred, not planned** (.github#358, moved to
  the ad-hoc epic). macOS is a development platform only, and is meant to stay
  that way. Deployments target enterprise-scale Linux, which is why agent work
  runs in Linux VMs. The macOS host stays on `uv` (spec D14). A Mac Studio
  joining as an alternate platform is a development and VM host, not a
  deployment target. Reopen through a new intake issue only if a concrete
  macOS deployment need appears.
- **Next consumers.** The framework is product-agnostic: any repo opts in with
  `[package]`, a `package-signing` environment, and an entry in its org's
  `packages.toml`. The next adopters are the logical-minds-foundry products
  this epic was built for (milestone M1 of that architecture), tracked in that
  org.

## Appendix A — Operational notes

- **Trust roots (OP1).**
  - Offline RSA-4096 primary `B3A1D804DC03AB036566A4E367861822166ECABE`, with
    signing subkey `DD67B5BA841442A77A6EC2F704D68CEDB8142851`. The primary is
    backed up offline; only the subkey is in CI.
  - Environments: `package-signing`, main-only, in vergil-tooling,
    vergil-python and packages; `index-signing`, develop-only, in packages.
- **Release order for a new org.**
  1. Keyring product (DEP1).
  2. Runtime (DEP2).
  3. Products: each release dispatches `package-released`, and `publish-index`
     rebuilds the site.
  4. VM switch-over (`vrg-vm update`).
- **VM migration.** `vrg-vm update` removes the uv copy only *after* the apt
  install succeeds. A dev ref (`--tag develop`) installs via uv with a DEV
  banner, and a plain update returns the VM to the package. Re-running is
  idempotent (#3138).
- **Gotchas.**
  - Callers must use `secrets: inherit` (§1).
  - Never re-release an unchanged name and version: the index hard-errors when
    the same name and version arrive with different bytes.
  - Release recovery after GitHub 5xx and GraphQL errors was re-run/resume,
    which worked every time.
- **VAL1 method.**
  - Fresh Lima arm64 and fresh cloud amd64 VMs were checked read-only.
  - `dnf` installs ran in `--rm` UBI 9/10 containers on both architectures.
  - The tamper test copied the live `noble` suite to a local `file://` repo,
    confirmed an untouched copy verifies, then flipped one byte of the signed
    `InRelease`. apt rejected it with `BADSIG` / "is not signed".
