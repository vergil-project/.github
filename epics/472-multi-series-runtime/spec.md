# Multi-series Python runtime

- **Epic:** vergil-project/.github#472
- **Planning issue:** vergil-project/vergil-python#50
- **Status:** Draft design (2026-10-25). Brainstorming, the builder check and
  one round of paad pushback are done. The builder check replaced carry forward
  with "every release publishes every series" (§5.5). Pushback replaced the
  dependency-update hook path with a new generic `vrg-upstream-refresh` command
  (§5.6), which vergil-containers' `bump-tools` job also adopts. Approved by the
  human. Alignment against the plan is done, and its decisions are folded in
  (§9, decisions 8–10).
- **Repos:** vergil-project/vergil-tooling, vergil-project/vergil-actions,
  vergil-project/vergil-python, vergil-project/vergil-containers,
  vergil-project/packages. The epic is homed in `.github`.
- **Related work:**
  - vergil-project/.github#407 (language version lifecycle): its spec §7 wants
    one co-installable runtime per supported `x.y` with the patch floating
    inside it. What shipped builds one series at a time.
  - vergil-project/.github#470 (intake): unattended merge of automation-authored
    PRs. Deliberately out of scope here.
  - vergil-project/.github#471 (intake): making sure automation PRs and daily
    ops results get noticed.
  - vergil-project/.github#466 (intake): a package-hosting replacement for
    GitHub Pages, and the place to revisit retention.
  - vergil-project/vergil-containers#692 (ad-hoc): adds `rpm` to every dev
    image, which the rpm version-order tests (§5.5) need.
- **Seeded tasks:** .github#473 (documentation), vergil-tooling#3623
  (documentation review), .github#474 (retrospective).

## 1. Summary

vergil-python builds **one `vergil-python<x.y>` package per series in an
explicit list**, 3.14 and 3.15 to start, and keeps every listed series patched.

A new, language-neutral vergil-tooling command, **`vrg-upstream-refresh`**,
runs a repo's own refresh hook on a schedule. If the hook changed nothing, the
command exits quietly. Otherwise it opens or refreshes **one PR** and stops. A
human merges the PR and cuts the release. Two repos use it:

- **vergil-python:** the hook refreshes each series within itself (new CPython
  patch, new python-build-standalone build) and adds a new **leading-edge**
  series once the vergil-containers window has it.
- **vergil-containers:** its weekly `bump-tools` job moves onto the command,
  replacing the same logic written as inline bash.

Every vergil-python release publishes every listed series. Each package version
carries both the upstream inputs and vergil-python's own release version.

All of this needs **multi-package-per-repo support in vergil-tooling** first.
Today `[package]` describes exactly one package, and the release and index
paths assume it.

## 2. Why

A language runtime has no flag day. When a new CPython series goes GA, products
move to it one at a time. For a while more than one series is in use, and every
one in use must keep receiving CPython and OpenSSL security fixes.

Today vergil-python maintains one series at a time: `runtime.toml` has a single
`series`, and the update hook refreshes only that series. Moving the series to
3.15 (vergil-python#34) left `vergil-python3.14` in the index **frozen** at
`3.14.8+20261013`, although CPython 3.14.9 is out. Every future series move does
the same to the series it leaves behind.

The new series is not picked up automatically either. The hook already notices
a newer CPython minor, but only reports it as held back. Moving the series is a
hand-written PR. vergil-containers detects a GA and opens its own PR daily.
vergil-python has no equivalent, so it waits until someone remembers.

There is also a category error. vergil-python's refresh rides the
`[dependency-update]` hook, which `vrg-update-deps` runs right **after** a
release. That mechanism exists to roll a repo's **development** dependencies
forward at the start of a development cycle. vergil-python has no such cycle:
refreshing its upstream inputs and then publishing them **is** its work, so the
order is inverted (refresh, then release). A separate mechanism fits it better.

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
5. A generic **`vrg-upstream-refresh`** command, run on a schedule, opens one
   PR for a human to merge whenever a repo's upstream inputs moved. vergil-python
   and vergil-containers' `bump-tools` both use it.
6. **Every release publishes every listed series**, under package versions that
   include vergil-python's own release version (§5.5).
7. Adding or removing a series is **one reviewed change**.

### Non-goals

- **Unattended merge or release.** Every PR is merged by a human, and every
  release is cut by a human (.github#470).
- **Trailing-edge automation.** No automatic drop, no EOL detection or
  reporting. Dropping a series is a human decision, made through its own
  human-driven epic when the time comes.
- **Retention redesign.** The retention algorithm is unchanged. The only change
  is the `keep` value in `packages.toml` (§6.3). Retention as a whole is
  revisited with .github#466.
- **Index hosting.** The design gets the package namespace right for the long
  term and does not bend to the GitHub Pages size limit beyond §6.3's stopgap.
  Replacing the hosting is .github#466.
- **Back-fill.** Window versions older than the newest listed series (3.12 and
  3.13 today) never get a runtime added.
- **The vergil-containers language lifecycle job.** It opens one PR per change
  and manages issues, which is a different shape. It stays as it is.
- **Other languages.** Python is the only language with a runtime package. The
  multi-package support and `vrg-upstream-refresh` are generic, but no other
  runtime is built here.
- **A tracking issue per refresh.** The refresh PR is the record, as for the
  vergil-containers jobs.
- **The notification gap** for automation PRs (.github#471).

### Anti-goals

- **Overbuilt automation.** This supports a small window of runtimes for
  binaries the project ships to itself. Prefer the existing vergil-containers
  pattern (scheduled job, App-authored PR, human merge) over new machinery. If a
  design step needs new identity, authorization or merge logic, it belongs to
  #470, not here.
- **Breaking single-package repos.** Every existing repo with a single
  `[package]` table must keep working with no edit.
- **Coupling upstream refresh to development dependencies.** `vrg-update-deps`
  and the `update-deps` stage of `vrg-release` never run an upstream-refresh
  hook, and `vrg-upstream-refresh` never runs dependency updaters.

## 4. Concepts

- **Series:** a CPython `x.y`, for example `3.14`. It names the package
  (`vergil-python3.14`) and its install root (`/opt/vergil/python/3.14/`).
- **Listed series:** a series in vergil-python's `runtime.toml`. Only listed
  series are built.
- **The window:** the vergil-containers `window.json`, read at the repo's
  `[dependencies].vergil` ref. That ref (`v2.1`) is a rolling tag that moves
  only when vergil-containers releases, so a new version reaches the window only
  after a vergil-containers release. It lists 3.12–3.15 today, and only 3.14 and
  3.15 have `runtime_published`.
- **Leading edge:** the newest listed series. A window version is a
  **leading-edge candidate** when it is newer than the leading edge.
- **Package identity:** `(name, version, format, arch)`, as the index already
  defines it. Different bytes at the same identity are a hard index error.
- **Inputs and engine.** A runtime package is built from **inputs** (the
  upstream CPython patch and the PBS build) by an **engine** (vergil-python's
  code: `build.sh`, its config, and the vergil-tooling packaging it runs on).
  A change to either produces a different package, so either one leads to a
  release that republishes every series. Upstream refresh watches inputs only.
  Engine changes arrive through normal development PRs.
- **Upstream refresh:** pulling a repo's upstream inputs forward so that the
  repo can republish them. It is distinct from a **dependency update**, which
  rolls development dependencies forward after a release.

## 5. Design

### 5.1 Multi-package config (vergil-tooling)

`vergil.toml` accepts either today's single `[package]` table or an array of
tables, `[[package]]`. Each entry is a full `PackageConfig`. The parser yields a
list, so a single table is a one-element list, and every consumer iterates it.

Rules:

- With more than one entry, every entry must set an explicit, unique `name`.
  The `pyproject.toml` fallback name applies only to a single entry.
- Every entry in a repo has the **same `vendor`**. The release wait and the
  tracking issue resolve one `<org>/packages` repo from it.
- Repo-level checks that today assume one package apply per entry: the root
  `uv.lock` check for the `python` builder, and the fixed overlay path
  `packaging/nfpm.overlay.yaml`. A repo with more than one entry **refuses**
  the shared overlay file, because it would otherwise be applied to every
  package. A per-entry overlay is future work. vergil-python has no overlay.
- A single `[package]` table parses and behaves exactly as today.

### 5.2 Package dimension in build, test and publish (vergil-tooling, vergil-actions)

Thread the package name through every stage:

- **Matrix:** build and test cells carry the package name and that package's
  own runtime. Cell ids are package-qualified, so artifact names
  (`package-build-<id>`, `package-unsigned-<id>`) don't collide. A
  single-package repo may keep today's ids.
- **Manifest:** every artifact entry in `packages-manifest.json` names its
  package. The runtime moves from one top-level value to a per-package value
  (null for a `staged` package). The index reads both the old and new manifest
  shapes, because already-published releases keep the old one.
- **Index matching:** `ManifestEntry.pattern()` and `_match()` in
  `lib/package/index/collect.py` match on the **exact** name, so two packages
  for one `(fmt, arch, suite)` no longer fail with "has 2 packages". A name
  that is a dash-prefix of another (`foo`, `foo-bar`) must not match the
  other's files. The same holds for install-test's artifact selection.
- **Build:** `vrg-package build --cell` selects the entry from the cell. Each
  entry gets its own staging directory. The `staged` builder passes
  `VRG_PACKAGE_NAME` and `VRG_PACKAGE_VERSION` (the **full** version, §5.5) to
  the build command, alongside `VRG_STAGING_ROOT` and `VRG_TARGET_ARCH`.
- **Install-test:** runs each entry's own `smoke` and residue check for that
  entry's artifact.
- **check-runtime:** checks each `python`-builder package against its own
  runtime. `staged` packages are not checked.
- **Release wait** (`lib/release/package_index.py`): waits until every
  package's identity is visible in the index, and records each one.
- **update-deps:** `served_runtime_for`, `_declared_runtime` and `bump_runtime`
  read the packages list. **At most one** `python`-builder entry per repo may
  pin a runtime (`[package.python].runtime`, an exact `x.y.z`). More than one
  pin is refused loudly, never guessed. No repo pins a runtime today, and
  libraries never have a `[package]` at all.
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

`vergil.toml` has one `[[package]]` entry per listed series. Each entry gives
the `name` `vergil-python<x.y>`, the series' inputs (`<cpython>+<pbs_tag>`),
its `summary`, and a `smoke` running `/opt/vergil/python/<x.y>/bin/python<x.y>`.
The full package version adds the release version (§5.5). The two files must
agree. `packaging/build.sh` keeps its consistency guard per entry, and fails the
build on any mismatch, including a series present in one file but not the
other.

`build.sh` selects its series from `VRG_PACKAGE_NAME` and stages it exactly as
today.

Starting list: 3.14 and 3.15. The 3.14 entry starts from the current pin, and
the first upstream refresh brings it current.

vergil-python's `[dependency-update].hook` entry is removed. Its refresh logic
moves to the upstream-refresh hook (§5.4), so a normal `vrg-release`, including
its `update-deps` stage, never touches the series.

### 5.4 The vergil-python refresh hook

vergil-python's `[upstream-refresh].hook` (for example
`scripts/dev/upstream-refresh.sh`) carries today's `update_deps.py` logic, moved
and extended. It edits files only, and writes notes for the PR body. Per run it:

1. **Refreshes each listed series within itself.** The newest PBS release (the
   unmetered `/releases/latest` redirect, as today) is checked for the newest
   CPython patch of each series, shipped as `install_only_stripped` for both
   architectures. Where present, the hook rewrites that series' `cpython`,
   `pbs_tag` and checksums, plus its `[[package]]` inputs and `summary`.
2. **Leaves a series PBS no longer ships at its current pin.** The hook doesn't
   search older PBS releases, because that needs the rate-limited GitHub API.
   The series keeps its pin and gets a distinct **"no longer built upstream"**
   note, not a held-back note. Typically this means the series is past EOL. It
   is not a failure and doesn't block the other series.
3. **Adds a leading-edge series** when the window contains a version newer than
   the leading edge **and** the PBS release ships it for both architectures.
   The new entry is added to both files in full. A window version that PBS does
   not ship yet gets a held-back note, and is never added half-built. Back-fill
   never happens.
4. **Never removes a series.**

The hook reads the window itself, fetching `window.json` at the
`[dependencies].vergil` ref, because it runs without vergil-tooling importable.
It honours `VRG_VERSION_WINDOW` as a local override, so tests can inject a
fixture window.

All existing failure rules carry over: any failure (network, parse, a missing
architecture or checksum, a series mismatch) exits non-zero before either file
is written.

### 5.5 Every release publishes every series (vergil-python, vergil-tooling)

**Every vergil-python release builds and publishes every listed series.** There
is no "unchanged series" case and no skipping.

To make that possible, a runtime package's version carries **both** its inputs
and the vergil-python release that built it: `<cpython>+<pbs_tag>` plus the
repo's semver `X.Y.Z` from `VERSION`. So a release that changed only the engine
(for example a `build.sh` fix) still publishes new identities, and the index's
same-identity-different-bytes check never fires.

**One join, in vergil-tooling.** `vergil.toml` holds only the inputs, in a new
per-entry key, `upstream-version` (for example `"3.15.1+20261021"`), which is
all the refresh hook ever writes. `upstream-version` is mutually exclusive with
`version`. `version` keeps its meaning as a full-version override, so no other
repo's packaging changes. A single vergil-tooling helper computes the
full version from an entry's inputs and the repo version. The build, the
`check-unpublished` identities and the release's index wait all use that helper
(today they each read `pkg.version or version` separately: `build.py`,
`index/published.py`, `release/package_index.py`). The repo version is fixed
for the whole release, because the release packages the tagged commit and bumps
`VERSION` only afterwards on the post-release branch. The advice in
`published.py` that suggests a `.1` rebuild counter is updated to match.

Constraints on the version format (the exact spelling is the builder's call):

- It sorts correctly in both dpkg and rpm, **verified by tests that run both**.
  A missing tool fails the test, never skips it. `rpm` reaches the dev images
  through vergil-containers#692.
  A newer CPython patch sorts higher. At the same inputs, a newer vergil-python
  release sorts higher. A new-format version also sorts above every old-format
  `<cpython>+<pbs_tag>` version already in the index, for example
  `3.15.1+20261021`, so hosts upgrade.
- The inputs come **first**, as `<cpython>+<pbs_tag>` followed by the release
  suffix, so `apt policy` shows which CPython and PBS build is installed.
- It passes the existing `[package].version` pattern, or that pattern is
  widened deliberately.
- Every reader of a runtime version keeps working:
  - the `python` builder's runtime dependency (today
    `vergil-python<x.y> (>= <version>)`) and its pinned-patch check, which
    splits on `+` (`lib/package/python_builder.py`);
  - the served-runtime parser in `lib/update_deps/vergil_index.py`, whose
    `<cpython>+<pbs>` regex today raises on anything else. It must accept the
    release suffix, or any product with a `[package.python].runtime` pin fails
    `vrg-update-deps` once a new-format version is indexed;
  - vergil-containers' `runtime_published` check, which reads only the package
    name.

Because every release contains every series, retention keeps every listed
series. A **removed** series is no longer built. Its published packages stay
until retention ages them out, or for as long as a retained consumer depends on
them (the existing dependency closure).

### 5.6 `vrg-upstream-refresh` (vergil-tooling, vergil-actions)

A new, language-neutral command. The repo supplies the logic in its hook, and
the command supplies the mechanics.

**Config.** `vergil.toml` gains `[upstream-refresh]` with `hook = "<command>"`.
It is separate from `[dependency-update]`, and neither command runs the other's
hook.

**Hook contract.** The hook runs from the repo root and only edits tracked
files. It may write notes to a report file whose path the command passes in an
env var (the format is the builder's call). Non-zero exit is a failure.

**What the command does:**

1. Runs the hook on a clean checkout of the default branch.
2. **No change** (per git): exits quietly, and closes any open refresh PR with a
   comment and deletes its branch, because develop already has it or it no
   longer applies.
3. **A change:** commits it to **one** fixed refresh branch (not `deps/update-*`,
   which `vrg-update-deps` owns), force-pushes only when the content differs, and
   opens or refreshes **one** PR with the hook's notes in the body. A rerun
   refreshes the same PR, and superseded refresh PRs are closed, as `bump-tools`
   does today.
4. **Never merges**, and never runs dependency updaters, validation gates or a
   release. The PR's own CI does the checking.

**Identity.** The command runs from a scheduled ops workflow under the per-org
automation App token. That follows the precedent of the scheduled epic sweep,
which runs `vrg-epic-audit --close` from `ops-epic-sweep.yml`. A human may also
run it by hand. Agent sessions (`user`/`audit`) are refused, because the
command opens PRs. It needs nothing beyond what the vergil-containers jobs use
today: contents and pull-requests write on its own repo.

**Workflow.** A reusable vergil-actions workflow (for example
`ops-upstream-refresh.yml`) mints the App token, checks out with it so pushes
are App-authored and trigger CI, and runs the command in the `prod-base` image,
which has `uv`, as the vergil-containers jobs do. Each consumer calls it from
its own `ops.yml` on its own schedule. vergil-python's schedule is daily.

A failed run is visible as a red scheduled run. Making sure someone notices is
.github#471.

### 5.7 vergil-containers `bump-tools` on upstream refresh

vergil-containers declares `[upstream-refresh].hook` as a script that runs
today's `docker/pins/resolve_latest.py --write` and
`docker/pins/generate_catalog.py`. `bump-tools.yml` is replaced by a call to the
reusable workflow, on its current weekly schedule. The PR lifecycle is the same:
one PR or nothing, refreshed in place, human-merged. Three details change:

- the branch becomes `upstream-refresh` (was `bot/bump-tools-<run id>`);
- the title becomes the refresh commit subject;
- a no-change run now closes a stale bump PR.

The inline bash for branch, commit, PR and supersede handling is deleted.

### 5.8 Adding and removing a series

- **Add (automatic):** the vergil-python refresh PR adds the series once the
  window has it (§5.4 step 3). What puts the version in the window is merging
  the vergil-containers `add python 3.x` PR and releasing vergil-containers.
  After vergil-python publishes the new package, the vergil-containers
  lifecycle job's existing `runtime` PR sets `runtime_published`, and unpinned
  products move to the new series at their next build.
- **Add (by hand):** still possible. The hand edit must add the entry to both
  files, and `build.sh` refuses any mismatch.
- **Remove (by hand only):** delete the series' entry from `runtime.toml` and
  its `[[package]]` from `vergil.toml` in one reviewed PR.

A release is still always a human action. A merged refresh PR is published by
the next normal `vrg-release`.

## 6. Constraints and interactions

1. **Release ordering.** vergil-python and vergil-containers consume
   vergil-tooling and vergil-actions at `v2.1`. The multi-package support and
   `vrg-upstream-refresh` must be **released** before they can be used. These
   are human-gated releases, modelled as deployment tasks at plan time.
2. **Back-compatibility.** Every other packaging repo (vergil-tooling itself,
   and the `packages` repo) keeps a single `[package]` table and must build,
   release and index exactly as today.
3. **Index size: stopgap.** `vrg-package index` refuses to publish above
   `SIZE_FAIL` (900 MB, `lib/package/index/site.py`), and GitHub Pages caps a
   site at 1 GB. One vergil-python series is about 140 MB per release (the four
   v1.0.3 artifacts). With two series and today's `keep = 3`, the shared index
   would cross 900 MB by about the third two-series release. Every repo's
   publish would then fail. So `packages.toml` `[retention]` moves to
   **`keep = 1`** (lines stay at 2) **before** the first two-series release. That
   projects to about 620 MB worst case. It's acceptable at a scale of one user
   who controls the whole install base. The plan adds a size projection, so the
   headroom is measured, not assumed. Retention is revisited with .github#466.
4. **App permissions.** The App installation must grant pull-requests write on
   vergil-python, not only on vergil-containers. It is checked as a precondition
   at plan time. Today's token mint in vergil-python's `ops.yml` already works.
5. **Public relay.** Nothing in this epic handles secrets beyond the App token
   that vergil-containers already uses the same way. The refresh PR body carries
   only the hook's notes.

## 7. Acceptance

- vergil-tooling builds, tests and publishes a repo with two `[[package]]`
  entries, and an unchanged `[package]` repo passes as a regression check.
- vergil-python's release publishes `vergil-python3.14` at the newest 3.14.x
  that PBS ships (3.14.9 or later) and `vergil-python3.15` at its newest, and
  both install side by side.
- A release with unchanged inputs (engine change only) publishes every series
  under new versions, and the index accepts it. The first two-series release
  proves this for any series whose inputs haven't moved, which is likely 3.15.
  If every series moved first, the check carries to the next release.
- `vrg-upstream-refresh`:
  - opens one PR for a pending refresh;
  - refreshes it on rerun;
  - closes a stale one on a no-change run;
  - exits quietly when there's nothing to do.
- vergil-containers' tool bumps run through `vrg-upstream-refresh`, with the same
  PR lifecycle (§5.7).
- A normal `vrg-release` of vergil-python no longer touches the series.
- Tests show:
  - the hook refreshes each series independently;
  - the hook adds a leading-edge series from a fixture window that has one,
    and never back-fills;
  - a series PBS no longer ships keeps its pin and gets a note;
  - the build covers every listed series;
  - removing a series drops its package from the build;
  - version ordering holds under both dpkg and rpm.
- `vrg-container-run -- vrg-validate` passes in every touched repo.

## 8. Left to the builder

- The exact `[[package]]` schema details (per-entry overlay path, and whether a
  shared table supplies defaults).
- The cell-id format, and whether vergil-actions' packaging workflows change at
  all.
- The exact package version spelling, within §5.5's constraints.
- The upstream-refresh report format and env var, the refresh branch name, and
  the PR title.
- The names of the hook scripts in vergil-python and vergil-containers.

## 9. Decisions and calls (for human review)

**Human decisions (2026-10-25):**

1. Runtimes are packaged only for installed, Vergil-built apps: 3.14 and 3.15.
2. Leading edge only, no back-fill. Removing a series is a manual, reviewed
   decision.
3. Every release publishes every series, with the release semver in the
   package version (replacing the brainstorm's carry-forward idea).
4. Index size is not a design constraint. Stopgap: `keep = 1`, revisited with
   .github#466.
5. Upstream refresh is its own generic command and hook, not the
   dependency-update mechanism. vergil-containers' `bump-tools` adopts it. The
   language lifecycle job is left alone.
6. The version join happens once, in vergil-tooling.
7. The served-runtime parser is fixed now, although no repo pins a runtime
   today.
8. At most one pinned `python`-builder package per repo, with more refused
   loudly (alignment).
9. The first two-series release (D3) is the real-index proof of an engine-only
   release (alignment).
10. `rpm` goes into every dev image as dpkg's peer
    (vergil-project/vergil-containers#692), so the rpm ordering tests fail
    rather than skip (alignment).

**Agent's calls (correct any that are wrong):**

1. **Why (§2):** #50's "vergil-tooling is pinned to 3.14.8" is out of date.
   vergil-tooling already ships on 3.15, so the gap is framed as one that
   reopens at every GA, not as an active exposure.
2. **Co-install check:** the side-by-side install of both series (§7) is a
   one-off check in the closing validation task, not a new CI install-test mode.
3. **No tracking issue** per refresh. The PR is the record.
4. **A reusable vergil-actions workflow** for upstream refresh, since it has two
   consumers.
5. **The command refuses agent sessions**, and allows a human or the ops
   workflow.
