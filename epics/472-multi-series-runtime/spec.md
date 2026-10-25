# Multi-series Python runtime

- **Epic:** vergil-project/.github#472
- **Planning issue:** vergil-project/vergil-python#50
- **Status:** Draft design (2026-10-25): superpowers brainstorming and builder
  check done. The builder-check round replaced carry forward with "every release
  publishes every series" (§5.5). Pending pushback and human review.
- **Repos:** vergil-project/vergil-tooling, vergil-project/vergil-actions,
  vergil-project/vergil-python. The epic is homed in `.github`.
- **Related work:**
  - vergil-project/.github#407 (language version lifecycle): its spec §7 wants
    one co-installable runtime per supported `x.y` with the patch floating
    inside it. What shipped builds one series at a time.
  - vergil-project/.github#470 (intake): unattended merge of automation-authored
    PRs. Deliberately out of scope here.
  - vergil-project/.github#471 (intake): making sure automation PRs and daily
    ops results get noticed.
  - vergil-project/.github#466 (intake): a package-hosting replacement for
    GitHub Pages.
  - vergil-project/.github#385 (planning): reconcile scheduled and
    release-driven updates. This epic adds a scheduled update path, so the two
    meet (§6.4).
- **Seeded tasks:** .github#473 (documentation), vergil-tooling#3623
  (documentation review), .github#474 (retrospective).

## 1. Summary

vergil-python builds **one `vergil-python<x.y>` package per series in an
explicit list**, 3.14 and 3.15 to start, and keeps every listed series patched.
A daily job in vergil-python refreshes each series within itself (new CPython
patch, new python-build-standalone build) and adds a new **leading-edge**
series once the vergil-containers window has it. It opens one PR, and a human
merges it and cuts the release, the same way the vergil-containers lifecycle PRs
are handled.

This needs **multi-package-per-repo support in vergil-tooling** first, because
today `[package]` describes exactly one package and the release and index paths
assume it.

## 2. Why

A language runtime has no flag day. When a new CPython series goes GA, products
move to it one at a time, so for a while more than one series is in use, and
every one in use must keep receiving CPython and OpenSSL security fixes.

Today vergil-python maintains one series at a time. `runtime.toml` has a single
`series`; the update hook refreshes only that series. Moving the series to 3.15
(vergil-python#34) left `vergil-python3.14` in the index **frozen** at
`3.14.8+20261013`, although CPython 3.14.9 is out. Every future series move does
the same to the series it leaves behind.

The new series is not picked up automatically either. The hook already notices
a newer CPython minor, but only reports it as held back; moving the series is a
hand-written PR. vergil-containers detects a GA and opens its own PR daily;
vergil-python has no equivalent, so it waits until someone remembers.

**Current impact (correction to the planning issue).** #50 says vergil-tooling
is packaged on a pinned 3.14.8. That is no longer true: vergil-tooling#3455
removed the pin and derives the runtime, and the released vergil-tooling
v2.1.265 manifest records runtime `3.15`. So no vergil-project package deploys
on 3.14 today. The gap is still real, and it reopens at every GA (3.16 will
leave 3.15 frozen the same way), but it is not an active exposure on the agent
VMs.

## 3. Goals, non-goals, anti-goals

### Goals

1. One repo can build, test and publish **several packages** (vergil-tooling).
2. vergil-python's `runtime.toml` holds a **list of series**, and each builds
   its own `vergil-python<x.y>` package.
3. Every listed series is **refreshed within itself**: CPython patch, PBS build
   and checksums.
4. A **new leading-edge series** is added automatically, once the window has it.
5. A **daily job** in vergil-python runs the refresh and opens one PR for a
   human to merge.
6. **Every release publishes every listed series**, under package versions that
   include vergil-python's own release version (§5.5).
7. Adding or removing a series is **one reviewed change**.

### Non-goals

- **Unattended merge or release.** Every PR is merged by a human, and every
  release is cut by a human (.github#470).
- **Trailing-edge automation.** No automatic drop, no EOL detection or
  reporting. Dropping a series is a human decision, made through its own
  human-driven epic when the time comes.
- **Index retention changes.** `packages.toml` `[retention]` and the retention
  algorithm are unchanged.
- **Index size and hosting.** The design gets the package namespace right for
  the long term and does not bend to the current GitHub Pages size limit.
  Replacing the hosting is .github#466.
- **Back-fill.** Window versions older than the newest listed series (3.12 and
  3.13 today) never get a runtime added.
- **Other languages.** Python is the only language with a runtime package. The
  vergil-tooling multi-package support is generic, but no other runtime is
  built here.
- **The notification gap** for automation PRs (.github#471).

### Anti-goals

- **Overbuilt automation.** This supports a small window of runtimes for
  binaries the project ships to itself. Prefer the existing vergil-containers
  pattern (scheduled job, App-authored PR, human merge) over new machinery. If a
  design step needs new identity, authorization or merge logic, it belongs to
  #470, not here.
- **Breaking single-package repos.** Every existing repo with a single
  `[package]` table must keep working with no edit.

## 4. Concepts

- **Series:** a CPython `x.y`, for example `3.14`. It names the package
  (`vergil-python3.14`) and its install root (`/opt/vergil/python/3.14/`).
- **Listed series:** a series in vergil-python's `runtime.toml`. Only listed
  series are built.
- **The window:** the vergil-containers `window.json`, read at the repo's
  `[dependencies].vergil` ref (the existing loader,
  `vergil_tooling/lib/versions/window.py`). It lists 3.12–3.15 today; only 3.14
  and 3.15 have `runtime_published`.
- **Leading edge:** the newest listed series. A window version is a
  **leading-edge candidate** when it is newer than the leading edge.
- **Package identity:** `(name, version, format, arch)`, as the index already
  defines it. Different bytes at the same identity are a hard index error.
- **Inputs and engine.** A runtime package is built from **inputs** (the
  upstream CPython patch and the PBS build) by an **engine** (vergil-python's
  code: `build.sh`, its config, and the vergil-tooling packaging it runs on).
  A change to either produces a different package, so either one leads to a
  release that republishes every series.

## 5. Design

### 5.1 Multi-package config (vergil-tooling)

`vergil.toml` accepts either today's single `[package]` table or an array of
tables, `[[package]]`. Each entry is a full `PackageConfig`. The parser yields a
list, so a single table is a one-element list, and every consumer iterates it.

Rules:

- With more than one entry, every entry must set an explicit, unique `name`.
  The `pyproject.toml` fallback name applies only to a single entry.
- Repo-level checks that today assume one package (the fixed overlay path
  `packaging/nfpm.overlay.yaml`, the root `uv.lock` check for the `python`
  builder) apply per entry. Making the overlay path per entry is the builder's
  call. vergil-python has no overlay, so it is not needed for this epic.
- A single `[package]` table parses and behaves exactly as today.

### 5.2 Package dimension in build, test and publish (vergil-tooling, vergil-actions)

Thread the package name through every stage:

- **Matrix:** build and test cells carry the package name. Cell ids are
  package-qualified, so artifact names (`package-build-<id>`,
  `package-unsigned-<id>`) don't collide. A single-package repo may keep
  today's ids.
- **Manifest:** every artifact entry in `packages-manifest.json` names its
  package. The runtime moves from one top-level value to a per-package value
  (null for a `staged` package). The index reads both the old and new manifest
  shapes, because already-published releases keep the old one.
- **Index matching:** `ManifestEntry.pattern()` and `_match()` in
  `lib/package/index/collect.py` match on the name, so two packages for one
  `(fmt, arch, suite)` no longer fail with "has 2 packages".
- **Build:** `vrg-package build --cell` selects the entry from the cell. Each
  entry gets its own staging directory. The `staged` builder passes
  `VRG_PACKAGE_NAME` and `VRG_PACKAGE_VERSION` to the build command, alongside
  `VRG_STAGING_ROOT` and `VRG_TARGET_ARCH`.
- **Install-test:** runs each entry's own `smoke` and residue check for that
  entry's artifact.
- **check-runtime:** checks each `python`-builder package against its own
  runtime. `staged` packages are not checked.
- **Release wait** (`lib/release/package_index.py`): waits until every
  package's identity is visible in the index, and records each one.
- **update-deps:** `served_runtime_for`, `_declared_runtime` and `bump_runtime`
  handle a list of entries, each with its own optional runtime pin.
- **Version resolution:** the Python version extension
  (`lib/versions/extensions/python.py`) reads the packages list instead of the
  single `cfg.package`.

- **check-unpublished:** unchanged in behaviour. It checks every package's
  identities, and refuses a release if any is already published. Under §5.5
  every release's identities are new, so the guard keeps catching a genuine
  double publish.

vergil-actions changes only as needed for the package-qualified cells. If
tooling emits the new cells and manifest itself, the reusable workflows may need
little or no edit.

### 5.3 Series list (vergil-python)

`runtime.toml` becomes a list of series, each with its own `series`, `cpython`,
`pbs_tag` and per-arch `sha256`. Different series may sit on different
`pbs_tag`s.

`vergil.toml` has one `[[package]]` entry per listed series: `name`
`vergil-python<x.y>`, the series' inputs (`<cpython>+<pbs_tag>`), its `summary`,
and a `smoke` running `/opt/vergil/python/<x.y>/bin/python<x.y>`. The full
package version adds the release version (§5.5). The two files must agree. `packaging/build.sh` keeps its consistency guard per entry, and fails the
build on any mismatch, including a series present in one file but not the
other.

`build.sh` selects its series from `VRG_PACKAGE_NAME` and stages it exactly as
today.

Starting list: 3.14 and 3.15. The 3.14 entry starts from the current pin and is
refreshed by the first run of the hook.

### 5.4 The refresh (vergil-python hook)

`scripts/dev/update_deps.py` remains the single refresh logic. It is run both
by `vrg-update-deps` (the human path, including the `update-deps` stage of
`vrg-release`) and by the daily job (§5.6). Per run it:

1. **Refreshes each listed series within itself.** For each series, it takes
   the newest PBS release that ships the newest CPython patch of that series as
   `install_only_stripped` for both architectures. It rewrites that series'
   `cpython`, `pbs_tag` and checksums, plus its `[[package]]` `version` and
   `summary`.
2. **Adds a leading-edge series** when the window contains a version newer than
   the leading edge **and** PBS ships it for both architectures. The new entry
   is added to both files in full. A window version that PBS does not ship yet
   is reported as held back, never added half-built. Back-fill never happens.
3. **Never removes a series.**

All existing failure rules carry over: any failure exits non-zero before either
file is written. A series that PBS no longer builds (typically past EOL) is
**not** a hook failure. It keeps its current pin and is reported as held back,
so it doesn't block the other series.

### 5.5 Every release publishes every series (vergil-python, vergil-tooling)

**Every vergil-python release builds and publishes every listed series.** There
is no "unchanged series" case and no skipping.

To make that possible, a runtime package's version carries **both** its inputs
and the vergil-python release that built it: `<cpython>+<pbs_tag>` plus the
repo's semver `X.Y.Z`. So a release that changed only the engine (for example a
`build.sh` fix) still publishes new identities, and the index's
same-identity-different-bytes check never fires.

Constraints on the version format (the exact spelling is the builder's call):

- It sorts correctly in both dpkg and rpm. A newer CPython patch sorts higher.
  At the same inputs, a newer vergil-python release sorts higher.
- It passes the existing `[package].version` pattern, or that pattern is
  widened deliberately.
- The inputs stay readable from the version, so `apt policy` shows which
  CPython and PBS build is installed.
- The vergil-tooling `python` builder's runtime dependency (today
  `vergil-python<x.y> (>= <pbs>)`) and the version-window `runtime_published`
  check still work with the new format.

Because every release contains every series, retention keeps every listed
series without any change to it. A **removed** series is no longer built. Its
published packages stay until retention ages them out, or for as long as a
retained consumer depends on them (the existing dependency closure).

The index cost (a full package set per release, even with unchanged inputs) is
accepted (§3, non-goals).

### 5.6 Daily refresh job (vergil-python)

The daily job looks only at **inputs**: a new leading-edge series, a new
CPython patch for a listed series, or a new PBS build for one. It never looks at
vergil-python's own version or code. Engine changes reach a release through the
normal development PR path. If no input moved, the job does nothing.

vergil-python's `ops.yml` gains a scheduled job modelled on the vergil-containers
`language-lifecycle` and `bump-tools` jobs:

1. Mint the App token and check out with it, so pushes are App-authored and
   trigger CI (as vergil-containers does).
2. Run the hook (§5.4).
3. If it changed nothing, stop. Otherwise commit to one fixed branch, force-push
   only when the content changed, and open or refresh **one** PR. A rerun
   refreshes the same PR; it never opens a second one.
4. Never merge. A human reviews and merges the PR, then cuts the vergil-python
   release.

The PR's CI builds and install-tests every listed series, so a merged refresh
is known to build.

The job's failure is visible as a red scheduled run, as for the
vergil-containers jobs. Making sure someone notices is .github#471.

### 5.7 Adding and removing a series

- **Add (automatic):** the daily PR adds the series once the window has it
  (§5.4 step 2). Merging the vergil-containers `add python 3.x` PR and releasing
  vergil-containers is what puts the version in the window. After vergil-python
  publishes the new package, the vergil-containers lifecycle job's existing
  `runtime` PR sets `runtime_published`, and unpinned products move to it at
  their next build.
- **Add (by hand):** still possible. The hand edit must add the entry to both
  files, and `build.sh` refuses any mismatch.
- **Remove (by hand only):** delete the series' entry from `runtime.toml` and
  its `[[package]]` from `vergil.toml` in one reviewed PR.

## 6. Constraints and interactions

1. **Release ordering.** vergil-python consumes vergil-tooling and
   vergil-actions at `v2.1`. Their multi-package support must be **released**
   before vergil-python can list two series. These are human-gated releases,
   modelled as deployment tasks at plan time.
2. **Back-compatibility.** Every other packaging repo (vergil-tooling itself,
   and the `packages` repo) keeps a single `[package]` table and must build,
   release and index exactly as today.
3. **Index size.** Every release adds a full package set per listed series.
   The builder measures it and reports how much headroom the current Pages
   site has left, but it is not a gate (§3; hosting is .github#466).
4. **Scheduled and release-driven updates (.github#385).** The daily PR and
   `vrg-release`'s `update-deps` stage run the same hook. The builder must
   decide how the two meet. For example: label the daily PR
   `dependency-update`, so that `vrg-release`'s existing pending-deps gate
   refuses to release while a refresh is unmerged, and the release's own deps
   stage then finds nothing to do. This is a design point for pushback, not a
   reason to widen scope into #385.
5. **Public relay.** Nothing in this epic handles secrets beyond the App token
   that vergil-containers already uses the same way.

## 7. Acceptance

- vergil-tooling builds, tests and publishes a repo with two `[[package]]`
  entries, with an unchanged `[package]` repo as a regression check.
- vergil-python's release publishes `vergil-python3.14` at the newest 3.14.x
  that PBS ships (3.14.9 or later) and `vergil-python3.15` at its newest, and
  both install side by side.
- A release with unchanged inputs (engine change only) publishes every series
  under new versions, and the index accepts it.
- The daily job opens one PR for a pending refresh and refreshes it on rerun.
- Tests show:
  - the hook refreshes each series independently;
  - the hook adds a leading-edge series from a fixture window that has one,
    and never back-fills;
  - the build covers every listed series;
  - removing a series drops its package from the build.
- `vrg-container-run -- vrg-validate` passes in every touched repo.

## 8. Left to the builder

- The exact `[[package]]` schema details (per-entry overlay path, and whether a
  shared table supplies defaults).
- The cell-id format, and whether vergil-actions changes at all.
- The exact package version spelling, within §5.5's constraints, and where the
  release version is joined to the inputs (config or tooling).
- How the hook reads the window (the loader's URL, or a path passed in).
- The daily job's branch name, label and schedule.
- Whether the daily job lives entirely in vergil-python or as a reusable
  vergil-actions workflow. Default: vergil-python only, matching
  vergil-containers.

## 9. Calls made by the agent (for human review)

These were settled during the builder check without a human decision. Correct
any that are wrong.

1. **Why (§2):** #50's "vergil-tooling is pinned to 3.14.8" is out of date.
   vergil-tooling already ships on 3.15, so the gap is framed as one that
   reopens at every GA, not as an active exposure.
2. **Co-install check:** the side-by-side install of both series (§7) is a
   one-off check in the closing validation task, not a new CI install-test mode.
3. **A series PBS no longer ships:** the hook searches back for the newest PBS
   release that ships that series' newest patch, and reports the series as held
   back when nothing newer exists (§5.4).
4. **Daily job report:** the daily job sets up `VRG_DEPS_REPORT` for the hook
   itself, as `vrg-update-deps` does.
5. **Pending-deps gate:** how the daily PR meets `vrg-release`'s gate stays a
   pushback topic (§6.4).
