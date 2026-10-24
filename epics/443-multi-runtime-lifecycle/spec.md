# Multi-series runtimes and a fully automated lifecycle

Epic: `vergil-project/.github#443`. Documentation task:
`vergil-project/.github#444`. Planning issue:
`vergil-project/vergil-python#36` (this epic's origin). Builds on the language
version lifecycle epic, `vergil-project/.github#407` (spec, plan and
retrospective in `epics/407-language-version-lifecycle/`). Related intake:
`vergil-project/vergil-tooling#3522` (pruning policy for EOL-frozen packages).

## 1. Summary

When a new Python version goes GA, the fleet doesn't move to it in a single
day. Repos that pass their tests on the new version move at once. Repos that
need work stay pinned to an older version until they are fixed. For that
period, **more than one interpreter series is in use, and each one has to keep
receiving patches.**

Today `vergil-python` keeps only one series current. When it moves to a new
series, the previous one is frozen and never patched again. `vergil-python3.14`
is stuck at 3.14.8. The window covers 3.12–3.15, but no runtime is published
for 3.12 or 3.13.

This epic makes two changes:

1. **Multi-series runtimes.** `vergil-python` publishes a co-installable
   `vergil-python<x.y>` package for **every** `x.y` in the supported window
   and keeps each one patched. This needs **generic multi-package support** in
   `vergil-tooling` and `vergil-actions`, because both currently assume one
   package per repo.
2. **A fully automated, end-to-end lifecycle.** A scheduled workflow takes a GA
   or EOL from detection to a published runtime with **no human involved**:
   - it merges the `vergil-containers` lifecycle PR and releases
     `vergil-containers`;
   - it adds or drops the `vergil-python` series and refreshes patches for
     every series;
   - it merges its own PRs and releases `vergil-python`.

   A human hears about it only when a gate fails.

The second change is **deliberately a ratchet**. It is the first Vergil
workflow with no human gate anywhere. It is a step towards the longer-term goal
of planning work, queueing it, and being told when it is done (§3, principle
1).

## 2. The problem in detail

### 2.1 One series at a time (data, recon 2026-10-24)

- `vergil-python/runtime.toml` holds a single `series` (`3.15`), with its
  `cpython`, `pbs_tag` and per-arch `sha256`.
- `vergil.toml` has one `[package]` (`vergil-python3.15`, version
  `3.15.1+20261021`).
- `packaging/build.sh` refuses to build if the two files disagree.
- The update hook (`scripts/dev/update-deps.sh` → `update_deps.py`) refreshes
  the patch and the PBS build **inside** that one series. It only *reports* a
  newer minor version and never changes the series.
- The hook runs only at the end of `vrg-release`. **Nothing runs on a
  schedule**, so even the current series gets patches only when a human cuts a
  release.
- Install layout: `/opt/vergil/python/<x.y>/`. Series are already
  co-installable.

### 2.2 What the lifecycle automation does and doesn't do (data, recon 2026-10-24)

- **Detection is automated.**
  - The `language-lifecycle` job in `vergil-containers/.github/workflows/ops.yml`
    runs daily. It derives the window from endoflife.date and Docker Hub, and
    reads the vergil apt index to set `runtime_published` per version.
  - It opens add, drop and `runtime` PRs against
    `docker/versions/manifest.yml` in `vergil-containers` **only**. It pushes
    with the App token, so CI runs on them.
- **Every lifecycle PR waits for a human merge.** `publish.py` never enables
  auto-merge, citing "a human reviews and merges every PR". The underlying
  reason is that PR CI builds no images: new images are first built and
  scanned by the develop CD *after* merge.
- **A window change reaches consumers only through a `vergil-containers`
  release.** That release moves the rolling `v2.1` tag. Consumers read
  `window.json` at their `[dependencies].vergil` ref.
- **The `vergil-python` half was abandoned.** Spec #407 §7 described the
  lifecycle check adding and dropping `vergil-python` series. As built, a
  series move is a manual PR (`vergil-python#34`), and nothing drops a series
  at EOL.
- **Consumers are already correct.** A packaged library (today only
  `vergil-tooling`) deploys on the newest window version whose
  `runtime_published` is true. Pins are respected. This epic **doesn't change
  consumer resolution or pinning.**

### 2.3 One package per repo, throughout vergil-tooling (data, recon 2026-10-24)

Paths are under `vergil-tooling/src/vergil_tooling/` unless noted.

| Location | What it assumes today |
|---|---|
| `lib/config.py` (`PackageConfig`, `_parse_package_config`) | One `[package]` table; `[[package]]` raises "must be a table"; `version` is one static string; one overlay path, `packaging/nfpm.overlay.yaml` |
| `lib/package/build.py` `run_build`, `naming.package_name` | One name, and version = `pkg.version or repo version` |
| `lib/package/staged.py` | The build command gets only `VRG_STAGING_ROOT` and `VRG_TARGET_ARCH` |
| `lib/package/matrix.py` | Cell ids and `manifest()` entries `(fmt, arch, suite)` have no package name |
| `lib/package/index/collect.py` | `ManifestEntry.pattern()` ignores the name, so two packages per arch fail with "has 2 packages for deb/amd64" |
| `lib/package/index/published.py`, `lib/release/preflight.py`, `bin/vrg_package.py` | check-unpublished covers one name and refuses the whole release if that identity is already published |
| `lib/package/index/retention.py` `_by_lines` | Keeps the newest releases per repo release line. A package missing from later releases ages out |
| `lib/package/install_test.py` | One package, one smoke command |
| `lib/package/runtime.py`, `vrg-package check-runtime` | One runtime for every package in the directory |
| `lib/release/package_index.py` | The release waits on one name and version |
| vergil-actions `cd-release.yml`, `ci-package.yml` | Artifacts are keyed by cell id; `--version "$(vrg-version show)"` |

### 2.4 Obstacles to running unattended (data, recon 2026-10-24)

- **Identity falls through to HUMAN on a GitHub runner.** `identity_mode`
  resolves from the env var, then the mode file, then `app.pem`, then
  `VRG_APP_ID`, and finally **HUMAN**. A runner sets none of the first four.
  So the "human-only" gates in `vrg-release` and `vrg-update-deps` let a runner
  through. The `VRG_EPIC_SWEEP` switch checks only that the value equals `"1"`.
  **This is a security hole:** "human only" isn't enforced off the Mac.
- **`vrg-release` assumes a workstation:**
  - a clean local `develop` equal to `origin/develop`;
  - `.worktrees/` and git-cliff;
  - full history and tags;
  - Docker for the update-deps stage;
  - credentials for pushes, including the force-push of `vX.Y` in `promote`.
- **Pushes made with `GITHUB_TOKEN` don't trigger workflows.** So merges to
  `main` and `develop` must use the App token, or CD never fires.
- **`cd-docker-publish.yml` uses `cancel-in-progress: true`** and shares a
  concurrency group with the nightly `nightly-rebuild-prod`. Either can cancel
  the other, and a cancelled release CD counts as a failed publish.
- **There is no release lock.** The only guards are an existing release
  branch, an open tracking issue (found through search, which lags), and an
  existing tag.
- **Branch protection requires no reviews** (0 approvals, strict required
  checks). The green-only merge engine already merges release PRs without a
  review.
- **Unverified:** whether the `package-signing` deployment environment has
  required reviewers. The human reports reviewers aren't used anywhere. This
  is a precondition to attest (§8).

## 3. Principles

1. **If a lifecycle step can be mechanized, it must be** *(the human's
   decision)*. Automation means a human does nothing and is alerted only when
   something goes wrong. Human gates that have become rubber stamps are
   removed, starting with this workflow.
2. **Trust the CI and CD gates** *(the human's decision)*. Green CI plus green
   post-merge CD is the bar for merging and releasing. An unattended release
   ships whatever is on `develop`.
3. **The unit of work is a `vrg-release`-grade operation** *(the human's
   framing)*. An operation is complete only when its PRs have merged, its CD
   has succeeded, and its post-merge checks pass. That is how `vrg-release` and
   `vrg-finalize-pr` already define success.
4. **Leading edge, not bleeding edge.** GA versions only. Pre-release
   interpreters are out of scope (§7.2).
5. **Consumer behaviour doesn't change.** Unpinned repos try the newest version
   on their next build, and pinned repos stay put. Making every supported
   series available and current is this epic's whole contribution to consumers.
6. **Fail loud, never silent.** Every failed gate stops the chain, turns the job
   red, and leaves a tracked issue. A step that can't yet run unattended falls
   back to "prepared, a human acts" **only** as an explicit, tracked gap filed
   as follow-on work, never as a quiet default.
7. **Converge, don't script.** Each scheduled run compares the desired state
   with the actual state and takes the next safe step. Re-running is always
   safe. A chain that takes several runs (for example, waiting for
   python-build-standalone to publish a new series) is normal and not a
   failure.
8. **Generic machinery.** Multi-package support, the automation identity and
   the driver must not contain Python-specific branches. Python is the only
   runtime that needs them today, because compiled languages ship binaries
   with no runtime package.
9. **EOL stops building and keeps the artifacts** *(the human's decision)*.
   Dropped series stay installable and frozen, because unknown consumers may
   still depend on them. Pruning is deferred
   (`vergil-project/vergil-tooling#3522`).

## 4. Design: multiple packages per repo (vergil-tooling)

### 4.1 Configuration

- `vergil.toml` accepts **either** the existing single `[package]` table
  (unchanged, fully backward compatible) **or** a `[[package]]` array of
  tables. Mixing the two forms is a `ConfigError`.
- Internally both forms become a list of `PackageConfig`.
  `VergilConfig.package` gives way to `VergilConfig.packages`; for single-form
  repos the list has one entry.
- In the array form:
  - every entry must set `name`;
  - names must be unique;
  - every entry sets its own `version`, `summary` and `smoke`;
  - `builder`, `targets`, `native`, `noarch` and the builder sub-tables are
    per entry, with the same defaults as today.
  - *(Builder's call: whether shared defaults can be stated once. Not required.)*
- **The overlay path is per package:**
  - single form: `packaging/nfpm.overlay.yaml`, as today;
  - array form: `packaging/<name>.nfpm.overlay.yaml`;
  - either way the file is optional, with the same shape check as today.
- The `uv.lock` check and the other `read_config` package checks run once per
  entry.

### 4.2 Build, matrix and manifest

- **A package dimension in the matrix.** Each build and test cell carries the
  package name. Cell ids gain a package prefix so artifact names stay unique.
  In single form the prefix may be left out, so existing ids stay stable
  *(builder's call; the requirement is uniqueness)*.
- **Build commands are told what to build.** `staged.py` adds
  `VRG_PACKAGE_NAME` and `VRG_PACKAGE_VERSION` to the environment of the
  build command, so one script can serve N packages.
- **Versions.** Each package's version is its own `version` if set, otherwise
  the repo version, as today.
- **Manifest entries carry identity.** They gain `name` and `version`, and
  index matching uses the name.
  - `collect.py` must still read the **old** manifest shape (no name) for
    releases that are already published.
  - An old-shape manifest with more than one package per `(fmt, arch, suite)`
    is still an error.
- **Install-test runs per package**, with that package's `smoke`.
- **`check-runtime` runs per package.** Each package must depend on exactly
  one `vergil-python<x.y>`, but different packages in one release may use
  different series.

### 4.3 Publishing an unchanged package: skip, and say so

A release of a multi-package repo usually changes only some of its packages.
The rule:

- **Before building**, preflight (`vrg-release`) and `vrg-package
  check-unpublished` (CD) sort every package into one of two cases, keyed on
  `(name, version)` *(my call)*:
  - **`build`:** that identity is not published;
  - **`skip`:** that identity is already published by an earlier stable
    release.
- **Why name and version is the key.** Package builds aren't byte-reproducible
  (`published.py`), so "the bytes would differ" can't be judged before a build.
  A package's version has to change whenever its inputs change.
  - For `vergil-python` that holds by construction: the version is
    `<cpython>+<pbs_tag>`, and the checksums are pinned.
  - The existing index-time check that refuses the same identity with different
    bytes across releases (`retention._check_conflicts`) stays.
  - Known cost: changing a build input without bumping the version gives a
    logged skip, not a refusal. Pushback should confirm this trade-off.
- **Skipped packages are not built or re-published.** Each skip is printed and
  recorded in the release summary or job summary, by name and version. A skip
  is never silent.
- **A release with no packages to build is allowed, and announced
  prominently** *(my call)*. It happens for a docs-only release of a package
  repo. The unattended driver never cuts one: it releases `vergil-python` only
  when at least one package will be built (§6.4).
- The same rule applies to single-form repos with a static `version`. The
  current "refuse the whole release" behaviour becomes a logged skip *(my
  call: this is a behaviour change, and pushback should confirm it)*.
- **The matrix only builds `build` packages.** The CD matrix is computed after
  the sort, so skipped packages have no cells.

### 4.4 Retention and the index

- **Retention works per package.** `_by_lines` keeps the newest releases per
  `(product, name)`, not per repo release line. **The newest published version
  of every package name is always kept**, whether or not later repo releases
  include it. This is the guard that keeps an EOL-frozen series installable
  (§3, principle 9).
- `_check_conflicts` keeps its cross-release identity check, unchanged.
- The release step that waits on the package index waits for **each built
  package**.

### 4.5 vergil-actions

- `ci-package.yml` and `cd-release.yml` carry the package dimension in matrix
  rows and artifact names, and pass `--package` to `vrg-package
  build`/`install-test`.
- The version argument becomes per package. The matrix job emits each row's
  version, so `--version "$(vrg-version show)"` is no longer the only source.
- Callers keep passing only `language:` and `container-suffix:`. **No change is
  needed in consuming repos' workflows.**

## 5. Design: vergil-python publishes every window series

- **`runtime.toml` holds a list of series:**

  ```toml
  [[series]]
  series = "3.15"
  cpython = "3.15.1"
  pbs_tag = "20261021"
  [series.sha256]
  x86_64 = "…"
  aarch64 = "…"
  ```

- **`vergil.toml`** has one `[[package]]` per series: `vergil-python<x.y>`,
  version `<cpython>+<pbs_tag>`, a smoke command that runs
  `/opt/vergil/python/<x.y>/bin/python<x.y>`, the `staged` builder and the
  shared `packaging/build.sh`.
- **`build.sh`** stages the series named by `VRG_PACKAGE_NAME`. It keeps every
  existing guard, now applied per series:
  - the name, version and smoke agree with `runtime.toml`;
  - the checksum is verified;
  - the archive has a single `python/` root;
  - the `EXTERNALLY-MANAGED` marker is written.
- **The update hook refreshes every series independently**, each staying inside
  its own `x.y`.
  - A series whose patch and PBS tag are current is left alone, so its package
    version doesn't change and §4.3 skips it.
  - **Every new stable PBS build counts as a change, even with the same CPython
    patch** *(the human's decision: be aggressive)*. PBS builds regularly
    carry bundled-library security fixes (OpenSSL, SQLite and others), so the
    leading edge is the newest PBS build. Each new PBS tag therefore rebuilds
    and releases every series. More release volume is the accepted cost.
  - A newer minor version is still only *reported* by the hook. Adding series
    is the driver's job.
- **Adding and dropping series are mechanical commands** that the driver calls
  (e.g. `scripts/dev/series.py add 3.16` / `drop 3.12`; the interface is the
  builder's call).
  - **Add** resolves the newest available patch and PBS build for the new
    series, then writes both files.
  - **No PBS build yet** for a GA series is a **pending** state, reported by
    the driver and retried on later runs. It is not a failure, until a bounded
    grace period runs out (§6.5).
  - **Drop** removes the series from both files and touches nothing in the
    index.
- **Backfill is the first real run.** Adding 3.12 and 3.13, and resuming 3.14,
  should be the driver's first production run, not a hand edit. That proves the
  add path and the patch refresh in one go.
- **Tests** cover:
  - each series refreshes independently, and an unchanged series is left
    alone;
  - the build covers every listed series;
  - a removed series has no build cell;
  - add and drop are idempotent;
  - pending-PBS handling.

## 6. Design: the unattended lifecycle

### 6.1 The automation identity (closes a security hole)

`identity_mode` gains an explicit **`automation`** identity. It fixes the
fall-through bug in §2.4 for the whole fleet, not only for this workflow:

- **On a CI runner, resolution never falls through to HUMAN.** A CI runner is
  detected empirically (`GITHUB_ACTIONS=true` / `CI`). There it resolves to
  `automation` when the workflow declares it, and otherwise to a fail-closed
  value that every human-only and automation-gated command refuses.
- **Human-only commands** (`vrg-submit-pr`, `vrg-finalize-pr`, `vrg-release`
  and `vrg-update-deps` when run by hand) refuse `automation`, exactly as they
  refuse agents.
- **Unattended operations must be explicitly authorised.** A command that may
  run unattended (`vrg-release`, the merge engine, `vrg-update-deps`,
  `vrg-epic-audit --close`) accepts `automation` only when **both** are true:
  - the run is a designated workflow, checked against an allowlist of
    `GITHUB_WORKFLOW_REF` values for the org's ops workflows;
  - the operation is in that workflow's declared scope.
- `VRG_EPIC_SWEEP` moves onto the same mechanism. It must keep working, through
  a compatibility path or a coordinated update of `ops-epic-sweep.yml`.
- **`automation` is never valid on an agent VM or the Mac.** It is refused
  whenever any agent signal is present, and `vrg-whoami --explain` reports the
  disagreement.
- The exact env-var names, the allowlist format and where the allowlist lives
  are the builder's call. The requirement: authorisation is checked in code,
  is scoped to named workflows, and fails closed.
- **Threat model: accidents, not deliberate spoofing** *(the human's
  decision)*. Ambient values such as `GITHUB_WORKFLOW_REF` can be set by any
  workflow step, so this check stops a runner or workflow from *accidentally*
  acting as a human or as automation. It doesn't stop a deliberate fake.
  Hardening that with a verified GitHub OIDC `job_workflow_ref` claim is
  follow-on work, `vergil-project/vergil-tooling#3524`, and doesn't block this
  epic.

A sweep of every caller of `identity_mode` is part of this work. Any command
whose safety relied on "a runner looks human" must be found and decided
explicitly.

### 6.2 vrg-release runs on a runner

`vrg-release` gains the ability to run unattended under `automation`. All of
its stages, CD confirmation and failure handling are reused unchanged
(principle 3). What has to change is its assumptions about the environment
(§2.4):

- **Checkout:** full history and tags (`fetch-depth: 0`), using the App token so
  that pushes and merges trigger workflows. `GH_TOKEN` is the App token.
- **Tools:** git-cliff and the `.worktrees/` layout work on the runner.
- **Docker:** the update-deps stage needs a Docker socket.
  - GitHub-hosted Ubuntu runners have one, but the `prod-base` job container
    does not.
  - The builder chooses the runner or container setup, or runs that stage
    separately.
  - **Skipping it silently is not an option.**
- **Waits get bounds.** Every wait (CD appearing, `gh run watch`, the package
  index, checks) has a stated limit, so a hung run fails loudly within the job's
  `timeout-minutes` and never hits GitHub's 6-hour default.
- **Prompts:** no prompt runs unattended. An interactive confirmation either
  doesn't apply to `automation` or is a hard refusal, never a hang.
- **Results:** the outcome (release URL, packages built or skipped, CD runs) is
  written to the job summary.

### 6.3 Concurrency and the docker-publish hazard

- **Fix the cancel race.** Change `cd-docker-publish.yml` so a release CD can
  never be cancelled by the nightly rebuild, and the reverse. Separate
  groups, or `cancel-in-progress: false` for release runs; the builder decides.
  This is a correctness fix that stands on its own.
- **A real release lock.**
  - The lifecycle workflow has its own `concurrency:` group, with no
    cancellation.
  - Before acting on a repo, the driver checks for a release in flight: a
    `release/*` branch on origin, an open release tracking issue (checked
    directly, not through search), or an unmerged dependency-update PR.
  - If it finds one, the run **backs off** for that repo. It records "deferred:
    release in progress" in the job summary and stays green.
  - A deferral that persists past a bound becomes a tracked issue.
- **The reverse direction.** A human starting `vrg-release` while a lifecycle
  release is in flight is refused by the existing preflight guards
  (release branch, tracking issue). This must be checked, not assumed.

### 6.4 The driver

A new `vergil-tooling` command (working name `vrg-lifecycle`; the builder picks
the final name) runs from a scheduled ops workflow. The workflow is a
reusable workflow in `vergil-actions`, called from `vergil-containers` next to
the existing lifecycle job, or folded into it. It uses an App token scoped to
the repos the chain touches. Each run reconciles in this order, stopping at the
first stage that cannot finish yet:

1. **Detect.** The existing lifecycle check runs as it does today and opens or
   refreshes its PRs.
2. **Merge the vergil-containers lifecycle PRs, for Python only** *(the
   human's decision)*.
   - Auto-merge is a **per-language switch** in
     `docker/versions/manifest.yml`, next to the existing `automation: "pr"`
     setting (the field name and shape are the builder's call). This epic turns
     it on for `python` only.
   - Today the check opens add, drop and `runtime` PRs for **both** Python and
     Go, and a human merges them. Go PRs keep working exactly that way until Go
     catches up in a later epic.
   - Each open lifecycle PR for an auto-merge language goes through
     `pr_merge.wait_and_merge`: green only, and never when behind,
     conflicting or draft.
   - `publish.py` drops its "a human merges every PR" stance for those
     languages only.
3. **Wait for the develop CD.** The post-merge develop CD (image build and
   scan) must succeed. This is the gate the human named: green PR CI doesn't
   build images, so nothing is released until develop CD has built them.
4. **Release vergil-containers.**
   - Runs only if `develop` is ahead of the last release. Whatever is on
     `develop` ships (principle 2).
   - It runs `vrg-release` unattended (§6.2), including its CD confirmation
     and the `v2.1` promote, so the window becomes visible to consumers.
5. **Reconcile vergil-python.**
   - It reads the **released** window at `v2.1`, so the runtimes follow
     what consumers actually see.
   - It compares the window with `runtime.toml`. A version missing from
     `runtime.toml` is added; a series no longer in the window is dropped.
   - It runs the per-series patch refresh.
   - Any change becomes one PR on a reserved branch (`lifecycle/<…>`; the exact
     names are the builder's call), which is merged with the same engine.
6. **Release vergil-python**, only if §4.3 classifies at least one package as
   `build`. It runs `vrg-release` unattended, which waits for the package index
   as it does today.
7. **Close the loop.**
   - The next run's step 1 sees the newly published runtime and opens the
     `runtime_published` PR.
   - Steps 2–4 merge it and release `vergil-containers` again.
   - From then on, consumers' existing resolution picks up the runtime.

Each step is idempotent. A run that starts partway through, for example after a
failure has been fixed, picks up from wherever the state is. The patch refresh
in step 5 runs every time, so `vergil-python` series get patches on the
workflow's schedule, not only when a human releases.

**`vergil-tooling` is not in the chain.** Its packaged runtime is derived at its
own release time from the published window. Releasing consumers automatically
is a later ratchet (§7.2).

### 6.5 Failure handling

- **Any failed gate stops the run and turns the job red.** Failed gates include:
  - CI red on a lifecycle PR;
  - develop CD failed;
  - a `vrg-release` stage failed;
  - an index wait timed out;
  - an index-time identity conflict (§4.3);
  - an authorisation refusal.
- **Each failure opens or updates exactly one tracked issue**, keyed by repo and
  failure kind, following the existing `publish.py sync_issues` pattern. The
  issue holds the run link and the failing stage. The next successful run
  closes it.
- **Pending is not failure, up to a limit.** "No PBS build for a GA series
  yet" and "deferred: release in progress" are recorded in the job summary and
  stay green. If either lasts past a bound, it becomes a tracked issue. The
  defaults are the builder's call, and the bound must be stated and tested.
- **A fallback is a tracked gap.** If any step proves impossible to run
  unattended during implementation, the epic ships that step as "prepared, a
  human acts". The step prints exactly what the human runs, its tracked issue
  records the gap, and a follow-on task or epic is filed (principle 6).

## 7. Scope

### 7.1 In scope

1. Multi-package support in `vergil-tooling`: config, build, matrix, manifest,
   index matching, the build and skip classification, per-package
   retention, install-test, `check-runtime`, and the release index wait (§4).
2. `vergil-actions` package dimension in `ci-package.yml` and `cd-release.yml`
   (§4.5).
3. `vergil-python` series list, per-series build, per-series refresh, and the
   add and drop commands (§5).
4. The `automation` identity and the fail-closed runner fix, including the
   sweep of `identity_mode` callers and migrating `VRG_EPIC_SWEEP` (§6.1).
5. `vrg-release` able to run unattended (§6.2).
6. The docker-publish cancel fix and the release lock (§6.3).
7. The driver and its scheduled workflow (§6.4), and the failure handling
   (§6.5).
8. **First real runs:** backfilling 3.12 and 3.13, resuming 3.14 patches, and
   a full GA→runtime chain run, proven by a live validation task (§9).

### 7.2 Non-goals

- **Pre-release (beta or RC) interpreters.** Offering them as an opt-in is a
  separate enhancement.
- **Deleting EOL packages.** Deferred to `vergil-project/vergil-tooling#3522`.
- **Changing consumer resolution or pinning,** or how a repo that fails on a new
  version gets pinned back.
- **Removing human gates from epic feature work** (`vrg-submit-pr`, human
  merge, human-initiated releases of other repos).
- **Automatically releasing consumers** (`vergil-tooling` and others) after a
  new runtime appears.
- **Building images in PR CI** (`vergil-project/.github#213`). Post-merge CD is
  the gate (§6.4 step 3).
- **Runtime packages for other languages.** None is needed today. The machinery
  stays generic.
- **Unattended merge for any language other than Python.** Go and later
  languages keep human-merged lifecycle PRs until they catch up in their own
  epics. They only need the per-language switch turned on once their
  automation is mature.
- **Spoof-resistant automation identity (OIDC).** Follow-on work,
  `vergil-project/vergil-tooling#3524`.
- **The lab's guest runtime pin.**

### 7.3 Anti-goals: the epic fails if any of these happen, even if it technically works

- A failure, skip, deferral or fallback is silent, or can't be told apart from
  success.
- A GitHub runner can still resolve to HUMAN, or `automation` can be claimed
  from an agent VM, from the Mac, or from a workflow that isn't on the allowlist
  without deliberately faking its identity.
- A lifecycle PR for a language whose auto-merge switch is off is merged
  without a human.
- An unattended release counts as done before its CD has succeeded.
- A package identity is re-published with different bytes, or an unchanged
  package is rebuilt and re-published.
- Retention ages out the newest version of any package, including an
  EOL-frozen series.
- The shared machinery contains Python-specific branches.
- A supported window series goes without patches past one scheduled cycle after
  python-build-standalone publishes a fix, without a tracked issue saying why.
- A single-package repo's existing config, cell ids or published index behave
  differently, except for the documented "refuse" → "logged skip" change
  (§4.3).

## 8. Preconditions and things to verify

- **`package-signing` environment protection** in `vergil-python` (and in any
  other repo the driver releases). It must have no required reviewers, or an
  unattended release will block. This can't be checked with the agent's
  credentials. The human attests to it before the first unattended release,
  for example with
  `gh api repos/vergil-project/vergil-python/environments --jq '.environments[] | {name, protection_rules}'`.
- **The App's installation and permissions** cover contents, pull requests,
  issues and actions (read) on every repo in the chain. Attested by the human
  in the same way.
- **Releases are human-gated today.** Shipping each part of this epic still
  needs human-run releases of `vergil-tooling`, `vergil-actions`,
  `vergil-python` and `vergil-containers`, in the bootstrap order below. The
  plan records these as deployment preconditions.

## 9. Rollout (constraint on the plan's ordering)

1. **The identity fix (§6.1) comes first.** It is a security fix in its own
   right, and everything unattended depends on it.
2. **Multi-package in `vergil-tooling` (§4)**, then **`vergil-actions`
   (§4.5)**, released in the usual order. Single-package repos must behave
   exactly as before at every step.
3. **`vergil-python` series list (§5)**, at first released by hand, which
   proves multi-package end to end. Backfill can happen here or as the driver's
   first run; the plan decides.
4. **The docker-publish fix and release lock (§6.3)**, in parallel with 2–3.
5. **`vrg-release` on a runner (§6.2)**, then **the driver (§6.4–6.5)**.
6. **Live validation.** A full chain run, cold, with the evidence recorded. If
   no real GA or EOL falls inside the epic, the run uses a fixture-driven
   window change on a test path; the plan decides how to do that without
   touching the production window. A validation task records it.

## 10. Closing bookends and follow-ons

- `vergil-project/vergil-tooling#3523`: documentation review, a multi-repo
  sweep.
- `vergil-project/.github#445`: follow-on brainstorm for the next
  unattended-lifecycle ratchet, fed by every gap §6.5 records.
- `vergil-project/.github#446`: retrospective (terminal).
- `vergil-project/vergil-tooling#3522`: pruning policy for EOL-frozen packages
  (intake).
- `vergil-project/vergil-tooling#3524`: make the automation identity
  spoof-resistant with OIDC (intake).
- `vergil-project/vergil-tooling#3520`: the planning sentinel doesn't ensure the
  `in-progress` label in the item's own repo (intake; found while setting up
  this epic).

## 11. Sources

- Recon of the local checkouts of `vergil-tooling`, `vergil-actions`,
  `vergil-containers` and `vergil-python` at `develop` on 2026-10-24 (paths
  cited inline).
- `epics/407-language-version-lifecycle/spec.md` §6.2, §7 and its "Amended
  after implementation" notes, and `retrospective.md`.
- `vergil-project/vergil-python#36`, including the feasibility comment of
  2026-10-22.
