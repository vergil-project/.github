# Multi-series runtimes and a fully automated lifecycle

Epic: `vergil-project/.github#443`. Documentation task:
`vergil-project/.github#444`. Planning issue:
`vergil-project/vergil-python#36` (this epic's origin). Builds on the language
version lifecycle epic, `vergil-project/.github#407` (spec, plan and
retrospective in `epics/407-language-version-lifecycle/`). Related intake:
`vergil-project/vergil-tooling#3522` (pruning policy for EOL-frozen packages)
and `vergil-project/.github#455` (package hosting beyond the GitHub Pages cap).

> **Revised after pushback round 2** (2026-10-24, documentation task
> `vergil-project/.github#456`). A second `paad:pushback` pass on the merged
> spec checked each design area against the code. It changed §2.3, §2.4, §4.2–§4.4,
> §5, §6.1–§6.5, §7.3, §8, §9 and §10. Decisions from that round are marked
> *(decided in pushback round 2)*. The alignment pass that followed is marked
> *(decided in alignment, round 2)* or *(aligned in round 2)*.

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
| `lib/update_deps/updaters/vergil_eco.py` `_declared_runtime` | Reads the raw `vergil.toml` `package` key as a table. With `[[package]]` it is a list, so `.get` raises `AttributeError`, which escapes the orchestrator's error handling (`lib/update_deps/orchestrator.py`), so no fix task is filed |
| `lib/ci_evidence.py`, `lib/github_config.py`, `lib/versions/extensions/python.py`, `lib/update_deps/vergil_index.py` | Read the single `cfg.package` |
| `lib/package/index/collect.py` `collect`, `site.py` | Download and verify every artifact of every stable release of every product **before** retention runs |
| `lib/package/index/site.py` | Site size guard: `SIZE_WARN` 750 MB, `SIZE_FAIL` 900 MB (GitHub Pages caps a site at 1 GB). One `vergil-python<x.y>` version is about 140 MB across its four artifacts (measured, v1.0.3) |

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
- **`cd-docker-publish.yml` uses `cancel-in-progress: true`**, with one group
  per image prefix. The release CD shares the prod group with
  `nightly-rebuild-prod`, and develop CD shares the dev group with
  `nightly-rebuild-dev`. Any of them can cancel another, and a cancelled
  release CD counts as a failed publish.
- **There is no release lock.** The only guards are an existing
  `release/<version>` branch, an open tracking issue (found through search,
  which lags), an existing tag, the pending dependency-update gate and the
  published-package check.
- **One wait is unbounded.** Most waits already have limits: checks
  (1800 s), CD-run discovery (300 s) and the package index (1200 s). But
  `gh run watch` (`lib/release/subprocess.py`) has none.
- **App installation tokens expire after one hour** (GitHub documentation).
  The chain this epic builds runs for hours.
- **`vrg-promote` runs even when a publish job failed.** `confirm-main`
  hard-gates only the `release` job, and a failed `docker-publish` is recorded
  as a deferred publish failure. The `promote` stage then moves `vX.Y`
  without checking that list (`lib/release/orchestrator.py`, `confirm.py`).
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
- **Every reader of the package config moves to the list**, including the
  ones that read raw TOML (§2.3). `vergil_eco.py` goes through the parsed
  config: it collects the runtime pins of the python-builder entries, and an
  ambiguous pin is an `UpdateDepsError` naming the entry. A vergil-tooling test
  runs the `vergil` updater against an array-form `vergil.toml` *(decided in
  pushback round 2)*.
- **An updater crash is never silent.** The `vrg-update-deps` orchestrator
  turns *any* unexpected exception from an updater into a `DepsFailure`, with
  the traceback tail, so it gets its fix task like any other update failure.

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
  index matching uses the **exact filename** derived from that name and
  version. A name-prefix pattern is not enough: packages `foo` and `foo-bar`
  would collide.
  - `collect.py` must still read the **old** manifest shape (no name) for
    releases that are already published.
  - An old-shape manifest with more than one package per `(fmt, arch, suite)`
    is still an error.
- **Install-test runs per package**, with that package's `smoke`.
- **`check-runtime` runs per package.** Each **python-builder** package must
  depend on exactly one `vergil-python<x.y>`, but different packages in one
  release may use different series. Packages from other builders (including
  the runtime packages themselves) record no runtime and are not checked.

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
  - Such a release attaches **no** package manifest. An empty `artifacts`
    list would fail every later index rebuild.
  - The index collect logs it as "all packages skipped", not as a
    pre-packaging release.
- The same rule applies to single-form repos with a static `version`. The
  current "refuse the whole release" behaviour becomes a logged skip *(my
  call: this is a behaviour change, and pushback should confirm it)*.
- **The matrix only builds `build` packages.** The CD matrix is computed after
  the sort, so skipped packages have no cells.
- **A partly published package is an error, with a stated remedy.** If some of
  its identities are published and some aren't (a CD that failed partway),
  classification refuses it loudly. The remedy is to re-run that release's
  CD publish to complete the identities, after which the package classifies as
  `skip`. A runtime version can't be bumped to escape this, because it is
  derived from its inputs.

### 4.4 Retention and the index

- **Retention works per package.** `_by_lines` applies its existing line
  semantics (`keep` releases per line, across `lines` lines) **per
  `(product, name)`**: each name's lines come from the tags that carry that
  name. A single-package repo therefore keeps exactly what it keeps today.
- **The newest published version of every package name is always kept**,
  whether or not later repo releases include it, **unless the name is listed
  in `[retention].retire`**. This is the guard that keeps an EOL-frozen series
  installable (§3, principle 9). `retire` remains the way to drop a name
  deliberately, for example the legacy per-patch `vergil-python3.14.8`.
- **A size budget for runtime packages** *(decided in pushback round 2, as a
  stopgap)*. One `vergil-python<x.y>` version is about 140 MB (§2.3), so line
  retention multiplied across N series would blow through the site guard.
  - Retention gains a **per-name-pattern keep**, configured in the index's
    `[retention]` (the `packages` repo's `packages.toml`). It is separate from
    the repo-line `keep`. Vergil sets `"vergil-python*" = 1`.
    - Generic code can't tell what a "runtime" name is without config
      (principle 8), so there is no built-in default *(aligned in round 2)*.
  - A check projects the site size from the retained set and fails **before**
    a publish can cross `SIZE_WARN`. The threshold is `SIZE_WARN` minus the
    bytes the next release would add, estimated from the newest release's
    assets. So the cliff shows as a red check long before `SIZE_FAIL` breaks
    every repo's index publish.
  - This is a stopgap only. With one version per series, vergil-python alone is
    about 560 MB, and it grows by about 140 MB for each EOL-frozen series
    (around 2028–2029 to `SIZE_FAIL`). GitHub Pages doesn't scale as the
    package host. The replacement is `vergil-project/.github#455` (§10).
- **The collect is lazy** *(decided in pushback round 2)*. Today the index
  rebuild downloads every artifact of every release ever made before retention
  runs (§2.3). With all-series releases that fills a hosted runner's disk
  (about 14 GB) within roughly 25 releases. Instead:
  - artifacts are identified from release-asset filenames (as
    `published.parse_filename` already does);
  - retention runs on those identities;
  - only the kept set is downloaded and verified.
- `_check_conflicts` keeps its cross-release identity check. It compares
  GitHub release-asset digests, and downloads an asset only when no digest is
  available (for example, older assets).
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
    and releases every series **that the tag ships**. More release volume is
    the accepted cost.
  - **Each series resolves on its own, and mixed `pbs_tag`s are allowed**
    *(decided in pushback round 2)*. PBS doesn't ship every series in every
    tag: release `20241008` dropped CPython 3.8 in the same release that
    shipped 3.13.0, and Python EOL and GA both fall in October.
    - Each series resolves against the **newest PBS tag that ships it** (all
      architectures). `runtime.toml` and `build.sh` already carry a tag per
      series.
    - An existing series that the newest tag no longer ships stays at its pin.
      It is reported as `upstream stopped building x.y; frozen at <pin>` and
      doesn't block the other series or a series add.
    - It becomes a tracked issue only if that state outlasts the window's own
      drop of the series by a stated, tested bound.
    - The hook is no longer all-or-nothing across series. It is still
      all-or-nothing **within** a series: both files are rendered before
      either is written.
- **The hook makes the series list match the window** *(decided in pushback:
  one refresh path)*. The rule "a new series is reported, never bumped" is
  replaced by "the series list follows the window".
  - The hook is given the **released** Python window: the versions at
    `[dependencies].vergil`, i.e. `v2.1`, which is what consumers see. It comes
    from the released `window.json`, handed to the hook as a file in the
    workspace (the builder picks the path). `vrg-ci-resolve` can't supply it:
    vergil-python declares no `primary-language`, and an `infrastructure` repo
    resolves as an app, which gets only the newest version. A plain env var
    doesn't reach the hook's container either (vergil-python has no
    `[container].env-prefixes`).
  - **The `VRG_VERSION_WINDOW` override applies only in dry-run** *(decided in
    pushback round 2)*. A run that can merge (`vrg-update-deps`, the
    `vrg-release` stage, the driver in live mode) refuses loudly when it is
    set. So a window exported for local testing, or the §9 fixture window,
    can never auto-merge a series change.
  - Before refreshing, it adds every window version missing from
    `runtime.toml`, and drops every series that is no longer in the window.
  - **Add** resolves the newest available patch and PBS build for the new
    series, then writes both files.
  - **No PBS build yet** for a GA series is a **pending** state. The hook
    reports it in the held-back report, and the driver records it and retries
    on later runs. It is not a failure until a bounded grace period runs out
    (§6.5).
  - **Drop** removes the series from both files and touches nothing in the
    index.
  - **The hook always reconciles, including when run by hand**
    *(decided in alignment)*. `vrg-update-deps` always passes the released
    window, so a hand-run update, or `vrg-release`'s closing `update-deps`
    stage, also keeps the series list in line with the window. There is one
    behaviour, not two. A human release that must *not* change the series
    list passes `vrg-release --without-deps`.
- **So `vrg-update-deps` is the only path that changes the series files.**
  Whether the driver runs it (§6.4 step 5) or `vrg-release` runs it as its
  closing `update-deps` stage, it is the same pipeline: the same
  `deps/update-<v>` branch, label, green-only merge, fix task on failure, and
  held-back report. No second refresh mechanism exists.
- **Backfill.** 3.14 resumes when the series list is first converted: the hook
  is run by hand to fill in its entry, and the release that proves
  multi-package end to end is cut by hand (§9 step 3). Adding 3.12 and 3.13 is
  the driver's first production run, not a hand edit. That proves the add path
  and the patch refresh in one go.
- **Tests** cover:
  - each series refreshes independently, and an unchanged series is left
    alone;
  - the build covers every listed series;
  - a removed series has no build cell;
  - add and drop are idempotent;
  - pending-PBS handling;
  - a series absent from the newest PBS tag is frozen and reported, while the
    other series still refresh;
  - mixed `pbs_tag`s across series.

## 6. Design: the unattended lifecycle

### 6.1 The automation identity (closes a security hole)

`identity_mode` gains an explicit **`automation`** identity. It fixes the
fall-through bug in §2.4 for the whole fleet, not only for this workflow:

- **On a CI runner, resolution never falls through to HUMAN.** A CI runner is
  detected empirically by `GITHUB_ACTIONS=true` (not `CI`, which other
  environments also set). There it resolves to `automation` when the workflow
  declares it, and otherwise to a fail-closed value that every human-only and
  automation-gated command refuses.
- **The same holds inside containers** *(decided in pushback round 2)*.
  `vrg-container-run` always forwards `GITHUB_ACTIONS`, `VRG_IDENTITY_MODE` and
  the authorisation variable below into the container. A nested container (the
  update-deps hook, `vrg-validate`) then resolves exactly as its host does,
  never to HUMAN by default.
- **Human-only commands** (`vrg-submit-pr`, `vrg-finalize-pr`, `vrg-release`
  and `vrg-update-deps` when run by hand) refuse `automation` and agents.
  - `vrg-finalize-pr` has no identity gate today, so this gate is new.
  - It allows `vrg-finalize-pr --cleanup-only`, which `vrg-release`'s
    `close-finalize` stage runs, under the release scope. Otherwise it refuses.
- **Gates allow listed modes; they never deny listed ones.** Every gate is
  written as `is_human()` or "in an explicit scope", never as "not an agent".
  So a new mode (`automation`, the fail-closed value) can never pass a gate as
  if it were human. A test runs every `identity_mode` caller under each new
  mode.
- **Unattended operations must be explicitly authorised.** A command that may
  run unattended (`vrg-release`, the merge engine, `vrg-update-deps`,
  `vrg-epic-audit --close`) accepts `automation` only when **both** are true:
  - the job runs a designated **shared** workflow. That is checked against an
    allowlist of that shared workflow's own `job.workflow_ref` (for example
    `vergil-project/vergil-actions/.github/workflows/ops-epic-sweep.yml@refs/tags/v2.1`),
    which each shared vergil-actions workflow exports into an env var
    *(decided in pushback round 2)*;
  - the operation is in that shared workflow's declared scope.
- **Why the shared workflow, not the caller.** For a reusable workflow,
  `GITHUB_WORKFLOW_REF` names the **calling** top-level file, which differs
  from org to org and file to file. One calling file can also host several
  unrelated jobs (lifecycle, github-config, nightly rebuilds). Keying on the
  shared workflow means:
  - callers' filenames and orgs don't matter, and every org that consumes
    vergil-actions at `v2.1` keeps working through the cutover;
  - every job carries exactly one scope;
  - the check lines up with the OIDC `job_workflow_ref` claim that
    `vergil-project/vergil-tooling#3524` will later verify.
- `VRG_EPIC_SWEEP` moves onto the same mechanism. It must keep working, through
  a compatibility path or a coordinated update of `ops-epic-sweep.yml`.
- **`automation` is never valid on an agent VM or the Mac.** It is refused
  whenever any agent signal is present, and `vrg-whoami --explain` reports the
  disagreement.
  - An **agent signal** is an identity signal: the mode file, `app.pem`,
    `VRG_APP_ID`, or the `/vergil` mount. It is **not** the platform. A hosted
    runner resolves to `cloud-vm`, and that must not count against
    `automation`.
  - `vrg-whoami`'s platform/identity correlation gains a runner ↔
    `automation` case, so a legitimate run doesn't warn.
- The exact env-var names, the allowlist format and where the allowlist lives
  are the builder's call. The requirement: authorisation is checked in code,
  is scoped to named workflows, and fails closed.
- **Threat model: accidents, not deliberate spoofing** *(the human's
  decision)*. Ambient values such as `VRG_JOB_WORKFLOW_REF` can be set by any
  workflow step, so this check stops a runner or workflow from *accidentally*
  acting as a human or as automation. It doesn't stop a deliberate fake.
  Hardening that with a verified GitHub OIDC `job_workflow_ref` claim is
  follow-on work, `vergil-project/vergil-tooling#3524`, and doesn't block this
  epic.
- **What actually bounds `automation`** *(decided in pushback round 2,
  replacing round 1's wrapper add-back list)*. No workflow calls `vrg-gh` or
  `vrg-git` on a runner. The `vrg-*` tools call `gh` and `git` directly, and
  `vrg-git` has no mode-dependent lists at all. So:
  - **The wrapper lists govern only direct wrapper calls.** `automation` and
    the undeclared fail-closed value both get the **agent** lists of `vrg-gh`,
    with nothing added back. This is cheap and consistent, but it is not the
    boundary.
  - **The boundary is the App installation's permissions plus the command
    gates and scopes above.** §8 states the minimum App permission set, and
    the human attests to it per org.
  - **Migration before cutover.** Every shared workflow that runs a gated
    `vrg-*` command on a runner today (at least `ops-epic-sweep.yml` in
    `vergil-actions`) exports its `job.workflow_ref` and declares its scope
    **before** the fail-closed identity change ships. No existing ops workflow
    in any org can break when it lands.

A sweep of every caller of `identity_mode` is part of this work. Any command
whose safety relied on "a runner looks human", or on "anything that isn't an
agent is human", must be found and decided explicitly. Two known cases are
`vrg-pr-fix-body` and `vrg-audit-approve`.

### 6.2 vrg-release runs on a runner

`vrg-release` gains the ability to run unattended under `automation`. All of
its stages, CD confirmation and failure handling are reused unchanged
(principle 3). What has to change is its assumptions about the environment
(§2.4):

- **Checkout:** full history and tags (`fetch-depth: 0`), using the App token so
  that pushes and merges trigger workflows. `GH_TOKEN` is the App token.
- **Tokens are refreshed; none has to outlive an hour** *(decided in pushback
  round 2)*. App installation tokens expire after one hour, and the chain runs
  for hours.
  - The driver holds the App ID and private key. It mints a fresh installation
    token before each step and on a timer inside long waits.
  - Both `gh` and git read the token from a refreshing credential source,
    never a value captured once at job start.
  - **One entry point each for `gh` and git** *(decided in alignment, round
    2)*. Today `gh` is executed from several modules in several ad-hoc ways.
    - All of them consolidate onto one `gh` runner in `lib/github.py`, which
      injects the credential. All git execs go through `lib/git.py`, and git
      gets its credential from a git credential helper.
    - Guard tests forbid executing `gh` or git anywhere else, so a new ad-hoc
      call can't regress silently.
    - A PATH shim that masks the inconsistency is explicitly rejected.
  - Checkout must not persist a token in `.git/config` that outlives the
    refresh.
- **Tools:** git-cliff and the `.worktrees/` layout work on the runner. The
  lifecycle job runs without a container (it needs Docker), so it installs a
  pinned, checksum-verified git-cliff itself. The driver checks every tool
  `vrg-release` needs before it acts, so a missing tool fails in seconds and
  never partway through the chain *(decided in alignment)*.
- **Docker:** the update-deps stage needs a Docker socket.
  - GitHub-hosted Ubuntu runners have one, but the `prod-base` job container
    does not.
  - The builder chooses the runner or container setup, or runs that stage
    separately.
  - **Skipping it silently is not an option.**
- **Waits get bounds.** Every wait (CD appearing, `gh run watch`, the package
  index, checks) has a stated limit, so a hung run fails loudly within the job's
  `timeout-minutes` and never hits GitHub's 6-hour default. Most already have
  one (§2.4). The new work is bounding `gh run watch`. The bounds can't be
  summed safely, so instead the driver runs under a **global deadline**: the
  job timeout minus a margin *(decided in alignment, round 2)*.
  - Every wait is clipped to the time remaining.
  - Hitting the deadline fails the current stage as `failed: deadline`, and
    the tracked issues are still synced.
  - The workflow also files the issue if the job dies before the driver can
    (`failure() || cancelled()`).
  - A timeout kill is never silent.
- **Promote requires every publish job to have succeeded** *(decided in
  pushback round 2; applies to **every** `vrg-release` run, human or
  automation)*.
  - When `deferred_publish_failures` is non-empty, `vrg-release` refuses the
    `promote` stage. The run fails, `vX.Y` doesn't move, and the failure names
    the jobs.
  - **Promote re-verifies from GitHub, never from memory** *(decided in
    alignment, round 2)*. The deferred list lives only in memory, and a
    `--resume` skips the stage that fills it. So `promote` reads the release
    SHA's **main** CD run (or its newest re-run) and refuses unless every
    publish job in it succeeded.
    - Recovery is: re-run the failed jobs, then `vrg-release --resume`. The
      existing "do not resume" texts change to match.
    - A develop-side docs-publish failure is not part of the main CD run, so
      it doesn't block `vX.Y`.
  - Otherwise a vergil-containers release whose prod `docker-publish` failed
    would still move `v2.1`. The window would then list a version whose prod
    image doesn't exist, and window-tested consumers across the fleet would go
    red.
  - This deliberately changes human releases of every repo, following
    principle 3: a release is not done until its CD has succeeded.
- **Prompts:** no prompt runs unattended. An interactive confirmation either
  doesn't apply to `automation` or is a hard refusal, never a hang.
- **Results:** the outcome (release URL, packages built or skipped, CD runs) is
  written to the job summary.

### 6.3 Concurrency and the docker-publish hazard

- **Fix the cancel race, and queue instead of dropping** *(revised in pushback
  round 2)*. This is a correctness fix that stands on its own.
  - Release CD and develop CD can never be cancelled by a nightly rebuild
    (`nightly-rebuild-prod` or `nightly-rebuild-dev`), and the reverse.
  - Turning off `cancel-in-progress` alone isn't enough. GitHub keeps only one
    **pending** run per group by default and cancels the rest.
  - **One group per image prefix; every run queues; nothing is cancelled**
    *(decided in alignment, round 2)*.
    - The group is `docker-publish-<image-prefix>`, with
      `cancel-in-progress: false` and `queue: max` for every trigger, the
      nightlies included.
    - Runs are strictly serialised, so a slower, older build (for example a
      nightly that checked out the previous `main`) can never overwrite
      newer tags.
    - Separate groups for release and nightly runs are rejected because they
      reintroduce that overwrite race.
  - The driver's develop-CD wait (§6.4 step 3) waits on the CD of the
    **newest** merge SHA. A superseded run counts as fine only if a later run
    on a descendant SHA succeeded.
- **A real release lock.**
  - The lifecycle workflow has its own `concurrency:` group, with no
    cancellation.
  - Before acting on a repo, the driver checks for a release in flight: a
    `release/*` branch on origin whose newest PR is open or that has no PR
    (the `vrg-branch-sweep` rules), an open release tracking issue (checked
    directly, not through search), or an unmerged dependency-update PR. A
    finished `release/*` branch that delete-on-merge failed to remove doesn't
    count.
  - **A PR on the automation prefixes that fails the merge rule** *(decided in
    alignment, round 2)*:
    - authored by the App, but failing any other condition (a foreign commit,
      a moved head, cross-repo) → **failed**, with a tracked issue naming the
      condition. Tampering is never a green deferral;
    - authored by a human (for example a hand-run `vrg-update-deps`) →
      deferred, as a human operation in flight.
  - **The driver's own PRs don't block it.** An open `deps/update-*` or
    `lifecycle/*` PR that passes the merge rule in §6.4 step 2, and whose
    checks are still pending, is the driver's own work in progress. The driver
    resumes it (waits for it, then merges it) and doesn't defer. A deps PR
    whose checks **failed** is handled by §6.4 step 5, never resumed.
  - **The driver's own interrupted release doesn't block it either**
    *(decided in pushback round 2)*. A release branch or tracking issue created
    by the App is the driver's own:
    - if `vrg-release` reports it resumable, the driver runs
      `vrg-release --resume`;
    - if it is in the deferred-publish state that `vrg-release` says not to
      resume, the stage stays **failed** (red, with its issue kept open). It is
      never recorded as a deferral.
  - If it finds anything else in flight, the run **backs off** for that repo. It records "deferred:
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
the repos the chain touches, **including `<owner>/.github`**, where fix tasks
are filed (§8). Each run reconciles in this order. It stops at the first stage
that **failed**. A `pending` or `deferred` stage ends only the work that
depends on it *(decided in alignment, round 2)*. For example, a series still
waiting for a PBS build doesn't hold back a vergil-python release of the series
that did refresh. A vergil-containers release deferred by a human release in
flight doesn't stop the vergil-python stages, which need only the **released**
window.

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
     languages only, in its PR text. `publish.py` itself still never merges,
     and its test asserting that no merge or auto-merge call is made stays.
     The driver does the merging *(aligned with the plan, round 2)*.
   - **Every lifecycle branch the automation verified has a recorded SHA**
     *(decided in alignment, round 2)*. `publish.py` pushes a branch only when
     its content changed. So it records the head SHA of every managed branch
     it confirmed equals the commit it built this run, pushed or not. A PR
     with no recorded SHA is a **failed** stage with a named reason. Otherwise
     a PR whose earlier merge failed would be refused on every run, with no
     way to recover.
   - **The unattended merge rule** *(decided in pushback)*. This rule applies to
     lifecycle PRs here, to the `deps/update-*` PRs from §5 and from
     `vrg-release`'s `update-deps` stage, and to the post-release back-merge PR.
     A PR is merged only when **all** of the following hold:
     - its author is the automation App;
     - its head branch is in the same repo (not a fork);
     - every commit on it was authored by the App;
     - the merge is pinned to the head SHA the automation pushed itself
       (`gh pr merge --match-head-commit` or an equivalent), so a later push
       makes the merge fail. When the merge engine updates the branch itself,
       the pin moves to the new head SHA and the rule is checked again.

     A branch name alone never qualifies a PR. Any mismatch is a refusal that
     gets a tracked issue (§6.5).
   - **The release-PR rule** *(decided in pushback round 2)*. A release PR
     (`release/<v>` → `main`) carries every develop commit since the last
     release, including human feature commits (principle 2). So it can never
     satisfy "every commit authored by the App". It merges unattended only when
     **all** of these hold:
     - its author is the App, and its head branch is in the same repo;
     - its head is pinned to the SHA that `prepare` pushed;
     - the release branch's base is exactly the repo's **gated develop SHA**
       *(defined in alignment, round 2)*. Each repo declares its gate kind
       explicitly, as a driver input, never inferred from its workflows:
       - **`cd`** (vergil-containers): the develop SHA whose develop CD
         succeeded (step 3);
       - **`checks`** (vergil-python, whose develop CD builds nothing): develop
         HEAD, the merge commit of a PR that passed the strict required
         checks. Strict protection means its tree is exactly what CI tested.
     - every commit the pipeline added on top of that SHA (changelog, version
       bump) was authored by the App.

     The guarantee is the same in substance: nothing reaches `main` except
     CD-gated `develop` plus the automation's own commits.
   - **What "authored by the App" means** *(decided in pushback round 2)*.
     - The App is per org (`<org>-automation`). Its slug comes from the
       token-minting step, never a hard-coded value.
     - Commits the automation makes, including vergil-containers' lifecycle
       commits (authored as `github-actions[bot]` today), use the App's bot
       identity: `<bot-user-id>+<slug>[bot]@users.noreply.github.com`. The
       bot **user** ID is looked up from `users/<slug>[bot]`; it is not the App
       ID.
     - The check compares logins after normalising the `app/<slug>` (PR
       author) and `<slug>[bot]` (commit author) forms.
     - Before `LIFECYCLE_LIVE` is set, a dry-run assertion checks this against
       a real App-authored PR's `author` and `commits` output.
   - A ruleset that only lets the App push these prefixes is follow-on defence
     in depth (recorded on `vergil-project/.github#445`), not part of this epic.
3. **Wait for the develop CD.** The post-merge develop CD (image build and
   scan) must succeed. This is the gate the human named: green PR CI doesn't
   build images, so nothing is released until develop CD has built them.
4. **Release vergil-containers.**
   - **Trigger** *(decided in pushback)*: runs only when a lifecycle PR (add,
     drop or `runtime`) for an auto-merge language has merged into `develop`
     since the last release. "Since the last release" means the merge commit
     is **not an ancestor of the newest stable `vX.Y.Z` tag on `main`**
     *(corrected in alignment, round 2)*. That tag sits on main's release
     merge, whose develop-side ancestors are exactly the commits cut into the
     release. A lifecycle PR merged after the cut is therefore not an
     ancestor. It is neither a timestamp comparison nor the `develop-v*` tag:
     main CD creates that tag from whatever `develop` is at that moment, after
     the cut, so it would hide such a merge. "Develop is ahead" is never the trigger:
     `develop` is always ahead after a release, because of the post-release
     version bump and the `update-deps` merge.
   - **Content:** whatever is on `develop` ships with it (principle 2).
     Releasing vergil-containers for any reason other than a window change
     stays with its usual owner.
   - It runs `vrg-release` unattended (§6.2), including its CD confirmation
     and the `v2.1` promote, so the window becomes visible to consumers.
5. **Reconcile vergil-python by running `vrg-update-deps` unattended**
   *(decided in pushback)*.
   - The vergil-python hook makes the series list match the **released**
     window, then refreshes every series (§5).
   - `vrg-update-deps` opens its usual `deps/update-<v>` PR and merges it under
     the unattended merge rule (step 2). If validation or the merge fails, it
     files its usual fix task.
   - If the previous step 6 release already left a `deps/update-*` PR open from
     its own `update-deps` stage, and its checks are still pending, this step
     resumes that PR rather than opening a second one (§6.3). This resume is
     new driver code: `vrg-update-deps` itself refuses an open PR on its
     branch.
   - **Clear a landed fix first** *(decided in pushback round 2)*. Before this
     step, the driver runs the same clearing that `vrg-release` preflight
     runs:
     - if the deps PR's fix task has been closed as completed (its fix is in
       `develop`), the driver closes the stale deps PR and deletes its branch,
       and this step opens a fresh one;
     - a deps PR with failed checks and a fix task still open is reported as one
       **failed** stage awaiting a human. It is never retried as a resume.

     So doing what the fix task says is always enough to unblock the chain.
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

**The live switch.** The chain stays in dry-run mode until a human sets the
`vergil-containers` repo variable `LIFECYCLE_LIVE=true`. In dry-run mode every
decision is printed to the job summary and nothing is merged, released or
opened. The human sets the variable once, after reviewing a dry run. That is
the single deliberate moment this ratchet engages. Unsetting it turns the
chain back into a planner.

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
  issue holds the run link and the failing stage.
- **A failure issue closes only on proof** *(decided in pushback round 2)*. It
  closes only when the same (repo, stage) later reports `done` or
  `nothing-to-do`. A run that defers, skips or never reaches that stage leaves
  it open. So a green deferral can never hide a real failure.
- **Pending is not failure, up to a limit.** "No PBS build for a GA series
  yet", "upstream stopped building x.y" (§5) and "deferred: release in
  progress" are recorded in the job summary and stay green. If any of them
  lasts past its bound, it becomes a tracked issue. The defaults are the
  builder's call, and each bound must be stated and tested.
  - **Tracked per item, not per stage** *(decided in alignment, round 2)*.
    - Runs are stateless, so one open **watch** issue per repo holds a
      first-seen marker for each item (`pending:3.16`, `frozen:3.12`,
      `deferred:<reason>`).
    - An item gets its own tracked issue, keyed by item, only once it passes
      its bound.
    - A **frozen** series is timed from its EOL date (from the window, or
      from the manifest if the window lacks it), not from first sight. That
      matches §5's "outlasts the window's own drop".
    - An overdue item turns the job **red** while it lasts. Its issue
      persists, so nothing churns between open and closed.
- **A fallback is a tracked gap.** If any step proves impossible to run
  unattended during implementation, the epic ships that step as "prepared, a
  human acts". The step prints exactly what the human runs, its tracked issue
  records the gap, and a follow-on task or epic is filed (principle 6).

## 7. Scope

### 7.1 In scope

1. Multi-package support in `vergil-tooling`: config and every reader of it,
   build, matrix, manifest, exact-filename index matching, the build and skip
   classification, per-package retention with the runtime size budget, the
   lazy index collect, install-test, `check-runtime`, and the release index
   wait (§4). The `packages` repo sets the runtime retention budget in its
   `packages.toml`.
2. `vergil-actions` package dimension in `ci-package.yml` and `cd-release.yml`
   (§4.5).
3. `vergil-python` series list, per-series build, per-series refresh against
   each series' own newest PBS tag, and the hook reconciling the series list
   against the released window (§5).
4. The `automation` identity and the fail-closed runner fix, including the
   sweep of `identity_mode` callers, the identity forwarding in
   `vrg-container-run`, the shared-workflow authorisation, and migrating
   `VRG_EPIC_SWEEP` (§6.1).
5. `vrg-release` able to run unattended, including token refresh and the
   promote-requires-publish gate for every release (§6.2).
6. The docker-publish cancel and queue fix, and the release lock (§6.3).
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
- **Replacing GitHub Pages as the package host.** This epic ships only the
  stopgap retention budget (§4.4). The replacement is
  `vergil-project/.github#455`.

### 7.3 Anti-goals: the epic fails if any of these happen, even if it technically works

- A failure, skip, deferral or fallback is silent, or can't be told apart from
  success.
- A GitHub runner, or a container started on one, can still resolve to HUMAN.
  Or `automation` can be claimed from an agent VM, from the Mac, or from a
  shared workflow that isn't on the allowlist, without deliberately faking its
  identity.
- A gate lets a new identity mode through because it isn't an agent.
- A lifecycle PR for a language whose auto-merge switch is off is merged
  without a human.
- A PR is merged unattended because of its branch name alone. Or a non-release
  PR is merged unattended when it holds a commit the automation didn't author.
  Or a release PR is merged unattended when its content beyond the CD-gated
  develop SHA holds a commit the automation didn't author.
- An existing ops workflow, in any org, breaks when the runner identity change
  ships.
- A real failure's tracked issue is closed by a run that deferred or skipped
  that stage.
- `vX.Y` is promoted while a publish job of that release failed.
- Two mechanisms rewrite `vergil-python`'s series files.
- vergil-containers is released unattended without a lifecycle change to
  trigger it.
- An unattended release counts as done before its CD has succeeded.
- A package identity is re-published with different bytes, or an unchanged
  package is rebuilt and re-published.
- Retention ages out the newest version of any package not listed in
  `[retention].retire`, including an EOL-frozen series.
- The index site grows past `SIZE_WARN` without a red check first, or the index
  rebuild's disk use grows with the full release history instead of the
  retained set.
- The shared machinery contains Python-specific branches.
- A supported window series goes without patches past one scheduled cycle after
  python-build-standalone publishes a fix, without a tracked issue saying why.
- One series that PBS no longer ships blocks the refresh or add of another
  series.
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
- **The `package-signing` environment check covers every repo the driver
  releases**, vergil-containers included, not only vergil-python.
- **The App's installation and permissions.** This is the real boundary for
  `automation` (§6.1), so it is a **minimum** set, reviewed and attested by the
  human per org.
  - `<owner>/.github` is in the chain too, because `vrg-update-deps` files fix
    tasks under its ad-hoc epic. On it the App needs **issues (write)** only.
  - On every repo the driver releases (vergil-containers, vergil-python):
    - contents (write);
    - pull requests (write);
    - issues (write);
    - actions (read);
    - administration (read), for `vrg-release`'s `audit` stage.

  Anything beyond that needs a stated reason. It is attested by the human in
  the same way.
- **Releases are human-gated today.** Shipping each part of this epic still
  needs human-run releases of `vergil-tooling`, `vergil-actions`,
  `vergil-python` and `vergil-containers`, in the bootstrap order below. The
  plan records these as deployment preconditions.

## 9. Rollout (constraint on the plan's ordering)

1. **The identity fix (§6.1) comes first.** It is a security fix in its own
   right, and everything unattended depends on it. It ships in two steps
   *(decided in alignment)*:
   - the opt-in `automation` identity, released with the first
     multi-package release;
   - the fail-closed cutover, as soon as the ops workflows have migrated to
     declared `automation`, and released together with the driver. That way
     the cutover is live before anything runs unattended.
2. **Multi-package in `vergil-tooling` (§4)**, then **`vergil-actions`
   (§4.5)**, released in the usual order. Single-package repos must behave
   exactly as before at every step.
3. **`vergil-python` series list (§5)**, at first released by hand, which
   proves multi-package end to end. Backfill can happen here or as the driver's
   first run; the plan decides.
4. **The docker-publish fix and release lock (§6.3)**, in parallel with 2–3.
5. **`vrg-release` on a runner (§6.2)**, then **the driver (§6.4–6.5)**.
6. **Live validation.** A full chain run, cold, with the evidence recorded.
   *(Decided in alignment.)* No real Python GA or EOL falls inside the epic:
   3.16 goes GA around October 2027, and 3.12 reaches EOL around October
   2028. So the validation has two parts:
   - **Live run.** The first production run backfills 3.12 and 3.13. Its real
     `runtime_published` lifecycle PRs exercise every stage end to end: merge,
     develop CD, both unattended releases, and the update-runtime path.
   - **Fixture-driven dry run.** A fixture window that adds 3.16 and drops
     3.12 drives the add/drop decisions through the driver and the hook,
     without touching the production window.

   **Known gap** *(recorded in pushback round 2)*: the unattended merge of
   **add** and **drop** lifecycle PRs isn't exercised live, because the
   fixture dry run never merges. The first live GA run (and the first live EOL)
   is that proof, and the retrospective records it as a follow-on observation.

## 10. Closing bookends and follow-ons

- `vergil-project/vergil-tooling#3523`: documentation review, a multi-repo
  sweep.
- `vergil-project/.github#445`: follow-on brainstorm for the next
  unattended-lifecycle ratchet, fed by every gap §6.5 records.
- `vergil-project/.github#446`: retrospective (terminal).
- `vergil-project/vergil-tooling#3522`: pruning policy for EOL-frozen packages
  (intake).
- `vergil-project/.github#455`: brainstorm a package host to replace GitHub
  Pages (intake). §4.4's retention budget is only a stopgap. At about 140 MB
  per series-version, the site crosses `SIZE_FAIL` around 2028–2029, sooner if
  more repos publish packages. **This must land before then.** The §4.4
  size-projection test is its early warning.
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
