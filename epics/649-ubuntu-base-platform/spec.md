# Ubuntu 24.04 base platform

- **Epic:** vergil-project/.github#649
- **Planning issue:** vergil-project/.github#605 (review history is in its
  comments)
- **Status:** approved by the human (2026-11-09); design review complete
  (2026-11-09)
- **Repos:** vergil-containers (the image tree, manifest, lifecycle adapters),
  vergil-vm (Lima template), vergil-actions (image-internals consumers),
  vergil-tooling (image-internals consumers, site docs)
- **Designed together with:** epic vergil-project/.github#607 (package
  toolchain images) and epic vergil-project/.github#213 (container build
  scalability). §6 amends both. The three epics deploy in the single order
  given in §7.

## 1. Why

The Vergil environment runs on three base operating systems, and only two of
them are needed.

| Where | OS today | How it got there |
|---|---|---|
| `dev-python`, `dev-go`, `dev-rust`, `base` | Debian 13 (trixie) | Inherited from the upstream `python:`, `golang:` and `rust:` images |
| `dev-cpp-clang`, `dev-cpp-gcc`, `dev-ts-node` | Debian 13 (trixie) | `FROM debian:trixie-slim`, chosen to match the images above (`docker/cpp-clang/README.md`) |
| Package jobs (#607) | Ubuntu 24.04 and 26.04, UBI 9 and 10 | Required: the package targets |
| Cloud agent VMs (Azure, GCP) | Ubuntu 24.04 | Pinned in the OpenTofu modules |
| Local agent VMs (Lima) | Ubuntu 26.04 | `template:_images/ubuntu-lts` floats to the newest LTS, though the template header says 24.04 |
| GitHub-hosted runners (`ubuntu-latest`) | Ubuntu 26.04 since early November 2026 | GitHub moved the floating label; our run logs show the `ubuntu-26.04` image from then on |

Nobody chose Debian. Every Debian image either inherited it or copied it from a
neighbour. We package
for Ubuntu and RHEL, never Debian. So we develop on one OS, ship to another,
and track CVEs, image updates and quirks for a third.

Packaging made the cost concrete. #607 adds `pkg-ubuntu:24.04`: `ubuntu:24.04`
plus the package prerequisites, uv and nFPM. It is a second Ubuntu image
lineage, separate from the Debian dev images, that has to be built, scanned and
refreshed. If the dev images were built on top of it, packaging would need no
Ubuntu image outside the dev tree.

**The decision (human, 2026-10-28):** standardize on Ubuntu 24.04 as the single
shared base platform. Debian goes entirely. RHEL/UBI stays only as a package
target. 24.04 rather than 26.04 because we support the current LTS and the one
before it. Developing and validating on the older one gives implicit coverage
of the floor, and the package install-tests cover 26.04.

This is medium priority. It is consistency and simplification, not a fix:
nothing is broken. It belongs to the next wave of container-architecture
changes, together with #607 and #213.

## 2. Goals

1. **One image tree on Ubuntu 24.04.** `base` and every language image, dev and
   prod, build on `pkg-ubuntu:24.04`, which builds on `ubuntu:24.04`. No image
   in vergil-containers builds from a Debian image or an upstream language
   image.
2. **Packaging needs no Ubuntu image outside the tree.** The 24.04 package
   build and install-test image is the tree's root. The images left outside the
   tree are `pkg-ubuntu:26.04`, `pkg-rhel:9` and `pkg-rhel:10`, the 26.04 and
   RHEL test targets.
3. **Every agent VM and GitHub-hosted runner runs Ubuntu 24.04.** Lima is pinned
   like the cloud VMs, and `ubuntu-latest` becomes `ubuntu-24.04`.
4. **The build machinery understands parent images.** A change to a parent
   rebuilds its descendants. A publish builds parents before children. A PR
   that changes a parent builds the children on the changed parent, not on the
   published one.
5. **The lifecycle automation keeps working** with the new toolchain sources.
6. **One published table** records where each OS is used and why.
7. **The three epics fit together.** #213 and #607 are amended so all three
   share one design, with one rollout order (§7).

## 3. Non-goals

- Changing package targets, package-job wiring, or the contents of the `pkg-*`
  images beyond the amendments in §6.2. #607 owns those.
- Moving install-tests onto the dev images. They stay on the minimal `pkg-*`
  images, so they still prove an install on a clean target.
- Moving anything to Ubuntu 26.04. That is the follow-on look-ahead,
  vergil-project/.github#651.
- Changing any image name or tag. #213's §4.6 invariant holds:
  `ghcr.io/vergil-project/{prefix}-{lang}:{version}` is unchanged.
- Trimming the tools an image carries. Each image keeps its current toolset,
  with the source of each tool adjusted for Ubuntu. The one exception is the
  python images' pytest-xdist, which already comes from each repo's own dev
  dependencies (§5.2).
- Narrowing the Python series vergil-tooling itself supports and tests. Once
  this epic runs it on the core Python everywhere in the containers, that is a
  follow-on epic of its own, which also audits every other place a Python
  version is specified for vergil-tooling (hosts, runners, VMs):
  vergil-project/.github#958.

## 4. Anti-goals (failure even if it works)

- **A child built on a stale parent.** A publish or PR build in which `base` or
  a language image builds on yesterday's `pkg-ubuntu:24.04`, when the same run
  changed or rebuilt that parent.
- **A dev image that passes for a package toolchain image.** #607's setup action
  treats `/etc/vergil/package-toolchain` as proof of a toolchain image. A dev
  image must never carry it.
- **Silent toolchain drift.** The upstream language images used to deliver
  patch releases (CPython 3.14.x, Go 1.27.x, …) for free. Each image now
  installs its own toolchain, so its version must stay current without
  changing unreviewed. A toolchain version that changes with no PR, or a
  pinned one that goes stale unnoticed, is a failure. §5.3 meets both with
  pins in source that the daily upstream refresh moves.
- **A switch with no way back.** Consumers have no setting to pin an image, and
  dated prod tags are deleted after 30 days (`package-cleanup.yml`). So the way
  back is a **revert release** of vergil-containers that restores the previous
  templates. The cutover PRs are kept revertible (no unrelated changes mixed
  in), and the release notes name the revert path. A broken cutover may also
  be failed forward (human decision, 2026-11-09): a fix lands on `develop`,
  which republishes the `dev-` images; repos switch to them with
  `container-prefix: dev` until the fix is released; a gate that blocks that
  release is bypassed with the audit-skip option (§7). vergil-containers'
  own CI runs in `prod-base`, so if the breakage is in what its required
  checks run, the fix (or revert) PR is merged by an admin.
- **Install-test fidelity lost.** Nothing in this epic may add to the contents
  of the `pkg-*` images themselves (§6.2 adds only build-time checks and a
  manifest field).

## 5. Design

### 5.1 The image tree

```text
ubuntu:24.04
 └─ pkg-ubuntu:24.04          #607; 24.04 package build and install-test
     └─ base                  shared dev tooling
         ├─ python:<v>        toolchain per §5.3
         ├─ go:<v>
         ├─ rust:<v>
         ├─ cpp-clang:<v>
         ├─ cpp-gcc:<v>
         └─ ts-node:<v>
```

The tree is the same under both prefixes: `dev-*` children build on `dev-*`
parents, and `prod-*` children on `prod-*` parents. A child's `FROM` names its
parent through a build arg, set by the build machinery (§6.1), so the prefix and
digest come from the same run.

**Language images build on `base`, not on `pkg-ubuntu:24.04` directly.** Today
every language image repeats most of `base`'s fragments (`path-defaults`,
`github-cli`, `validation-tools`, `nfpm`, `rpm`, `pandoc`) and some add
`python-support` and `node-markdownlint`. Building on `base` installs them once.
This is the de-duplication #213 deferred as ledger item D2.

- What `base` carries that a language image did not have before (for example
  `opentofu`, `security-tools`, `semgrep`, mkdocs) is accepted as the cost of
  one shared layer.
- **My call:** a language image may not remove anything `base` installs. If a
  tool conflicts with a language, that is a builder question for the plan, not
  a per-image fork of `base`. The runtime rule (§5.2.1) is what makes this hold
  for interpreted tools: a language image adds its own runtime beside `base`'s
  and owns the default command, without removing anything.

### 5.2 `base`

`base` becomes `FROM <parent>` (`pkg-ubuntu:24.04`) and keeps its current
toolset, with these changes:

- **No package toolchain marker.** `base` removes `/etc/vergil/package-toolchain`
  in its first layer, and a build-time check asserts it is absent from `base`
  and from every language image. Package jobs never run in dev images (#607 §5.2
  points them at `prod-pkg-*`), so nothing loses anything.
- **uv and nFPM come from the parent.** `base` stops installing uv (today through
  pip) and drops its `nfpm.dockerfile` include. Each tool then has one pin,
  in `pkg-ubuntu`, as #607 goal 3 intends.
- **The core Python is `vergil-python`** (human decision, 2026-11-09). `base`
  adds the vergil apt repository and installs the `vergil-python<core>`
  package at `/opt/vergil/python/<core>/`, as the agent VMs do. Before it
  writes the apt source, the build checks the downloaded signing key's primary
  fingerprint against a value pinned in the template, the one vergil-tooling
  pins in `lib/package/orgs.py`, and fails on a mismatch; it then installs
  `vergil-archive-keyring`, as `repo_setup.bootstrap` does. A key rotation
  changes both pinned values in reviewed PRs.
- **The core series is pinned** (human decision, 2026-11-09). `<core>` is an
  entry in the version manifest, today 3.15, and moves to a new series only
  through a reviewed PR. In this epic that PR is opened by hand; having the
  lifecycle automation open it is a follow-on (human decision, 2026-11-09;
  vergil-project/.github#978). Within the
  series, the image takes the package vergil-python publishes. vergil-tooling's
  package declares its own `vergil-python<series>` dependency, so when
  vergil-tooling moves series first, the cache build installs that series
  beside the pinned one, which the side-by-side rule allows; the PR that moves
  the pin brings the two back in line. There is no stable link that hides the
  series: callers that need the core Python are created at image build, when
  the series is known (below). No language family uses a path under
  `/opt/vergil/`.
- **`/usr/local/bin/python3` promises the standard library only.** In `base`
  and every non-Python language image it links to the core Python, as the
  `python:*-slim` images placed `python3` there. vergil-actions calls that path
  for `tomllib` (§5.5). No third-party package is installed into any image's
  default interpreter for callers to rely on.
- **`base`'s Python CLI tools run on the core Python** (§5.2.1). yamllint,
  ansible-lint and semgrep install as uv tools, each in its own environment,
  and the docs toolchain as one environment that exposes both `mkdocs` and
  `mike` (`mkdocs` with `--with mkdocs-material --with mike`), since `mike`
  must see the material theme and its extensions. Each is created with
  `/opt/vergil/python/<core>/bin/python<core>` by its versioned path. A build
  check runs `mike --help` and builds a material-themed fixture.
- **No image runs `pip install`** (human decision, 2026-11-09). Today `base`
  pip-installs yamllint, ansible-lint, uv, the docs toolchain and pyyaml, and
  upgrades pip and setuptools; the python family adds pytest-xdist. All of it
  goes: the tools become uv tools (above), uv comes from the parent, pyyaml
  moves into the scripts that use it (§5.5), and pytest-xdist comes from each
  repo's own dev dependencies, which is where `vrg-validate` looks for it
  (the plan confirms no consumer relies on the image's copy). Nothing is
  pip-installed into vergil-python either: its directory belongs to the
  package, and one shared site-packages would let the tools' dependencies
  conflict. The pip that a runtime bundles is the runtime's: it changes only
  with the runtime (§5.3, §5.8).
- **vergil-tooling is installed as its package** (human decision, 2026-11-09).
  When a repo's container cache is built, vergil-tooling installs from the
  vergil apt repository at the repo's pinned version (a line or an exact
  release), the way `vrg-vm update` installs it on the VMs: its venv at
  `/opt/vergil/vergil-tooling/venv` on vergil-python, its `vrg-*` commands
  shimmed into `/usr/bin`. It is installed at cache build, not baked into the
  image, so the image rebuild cadence stays independent of vergil-tooling's
  releases, and each repo keeps its own pin. A dev ref (a branch) is installed
  with `uv tool install` from git, bound to the core Python, as on the VMs. In
  vergil-tooling's own repo it is the project under test, so its checkout is
  used, as today. vergil-tooling is a set of command-line tools, not a library
  any other repo imports, so it runs on vergil-python in every image,
  whatever series the image's family tests. GitHub-hosted host jobs (no
  `container:`) install the same package on the runner, through the same
  fingerprint-checked apt source, so vergil-actions no longer pip-installs uv
  to install vergil-tooling there; the plan settles where a host job's dev-ref
  install gets uv. The one exception is the job that rebuilds and re-signs the
  vergil apt index (vergil-actions' `publish-index.yml`): it always installs
  vergil-tooling from git, never from the index it is about to repair. How the
  install crosses the cutover is §7.
- **Scripts declare their dependencies.** Nothing of ours relies on a package
  being present in an image's interpreter. A script that needs a third-party
  library carries inline script metadata (PEP 723) and runs with plain
  `uv run`, so it names no interpreter path. Anything that needs
  vergil-python declares it as a package dependency. A one-liner that calls
  `python3` gets whatever `python3` is on `PATH` and may use only the
  standard library (human decision, 2026-11-09); one that needs more should
  be a script or a tool, so its breaking is the signal, not a reason to add an
  interpreter path.
- **The python family owns `python3`.** In `dev-python:<v>`,
  `/usr/local/bin/python3` (and `python`, `pip`) links to `/opt/python/<v>`, as
  it is series `<v>` today when the image is `FROM python:<v>-slim`. Nothing
  is pip-installed into it (above). A build check asserts
  `python3 --version` matches the image's series, so a matrix leg can never
  silently test on the core Python. Other language images keep `base`'s link
  to the core Python as the default.
- **Node** for `base`'s one Node tool, markdownlint-cli, is the core Node
  runtime at `/opt/vergil/node/<core-major>` (today 22), from the nodejs.org
  prebuilt tarball. NodeSource's `.deb` goes, since it allows only one Node on a
  machine. `base` puts no `node`, `npm` or `npx` on `PATH`; only `ts-node`
  images have a default Node (§5.3). `markdownlint` on `PATH` is a wrapper that
  runs the tool with `/opt/vergil/node/<core-major>/bin/node` explicitly. npm's
  generated bin links start with `#!/usr/bin/env node`, which would pick up
  whatever `node` is on `PATH`, or fail where there is none.
- **apt-mirror config is inherited on amd64.** #607 bakes the Azure-first mirror
  list and fail-fast timeouts into amd64 `pkg-ubuntu`. Dev images inherit them.
  **My call:** keep them. The mirror list fails over to the default archive, and
  vergil-actions' `system-packages` step, which runs `apt-get install` in dev
  images in CI, gains the same protection against the slow Azure mirror that
  prompted #607. Our amd64 cloud VMs run these images too, outside GitHub's
  runners, so a builder check confirms an `apt-get install` in an amd64 image
  on one of them still succeeds. The check must run on amd64: only amd64
  `pkg-ubuntu` carries the mirror config, and the Lima VMs are arm64.

#### 5.2.1 The runtime rule: core tooling never runs on the default runtime

Human decision (2026-10-28). An interpreted tool is bound to one
installation of its runtime, and a language image's version axis is that same
runtime. Today that applies to Python and Node. It would apply to Ruby or Perl
if they became families. Compiled tools (actionlint, shfmt, trivy, tofu, nfpm,
uv, gh and the rest) and OS packages are bound to nothing but the OS, and are
outside this rule.

A runtime in an image has two kinds of consumer:

1. **The language family's toolchain** owns the **default command** (`python3`,
   `node`): the version the project builds and tests against, which the matrix
   varies.
2. **Core tooling** (linters, doc builders, anything in `base`) needs a runtime
   only to run.

The rule:

- **Core tooling never runs on the default runtime.** It runs on its own
  **core runtime**, installed side by side under `/opt/vergil/`
  (`/opt/vergil/python/<core>/`, `/opt/vergil/node/<core-major>/`), and calls
  it by path. Family runtimes live elsewhere (`/opt/python/<v>`,
  `/opt/node/<major>`), so a core runtime and a family runtime of the same
  series are two installs at two paths, never one install shared by both.
- **Core runtimes have their own pins**, separate from the family windows. A
  core runtime moves when core tooling needs it to, never because a language
  image moved, and only through a reviewed PR. The core Python's series and
  the core Node's major are each an entry in the version manifest, moved by a
  hand-opened reviewed PR in this epic (§5.2). Each is checked against its own
  source, never against a family window.
- **Only a language family's image sets a default command for its runtime.**
  The exception is `python3` in `base`, which consumers already rely on, and
  which promises the standard library only (§5.2, §5.5).
- **Runtimes are only ever added side by side.** A language image adds its
  series next to the core runtime and does not remove or overwrite it.
- **Build checks assert both halves.** At image build, every core tool in
  `base` runs on its pinned core runtime in every image, and the default
  command matches the family series. vergil-tooling is installed later, when
  the container cache is built, so that build checks it: `vrg-*` runs on
  vergil-python, from the package.

Why: tooling choices (how we lint Markdown) must not be forced to move with an
unrelated language toolchain (Node 24 in `ts-node:24`), and a language image
must not inherit a runtime the family never chose. Products that happen to be
Node applications, installed on a real host, are a separate question and are
out of scope: no deployed product needs a non-system Node, so
vergil-project/.github#674 was dropped. A product that needs one starts its
own item.

### 5.3 Language images — toolchain sources

Each language image is `FROM <base>` and installs its own toolchain. The rule
throughout: **prebuilt binaries from upstream or an upstream-maintained apt
channel, never built from source.**

**Downloaded toolchains are pinned in source** (human decision, 2026-11-09).
For python-build-standalone, Go and the nodejs.org tarballs, each series has
an auto-managed pin in vergil-containers holding the release version and the
SHA-256 of each architecture's download, under `check_pins.py` and
`CATALOG.md`. The build verifies the download against that hash and resolves
nothing over the network itself. The daily upstream-refresh hook
(`docker/pins/upstream-refresh.sh`) moves each pin to the newest patch of its
series and writes the new hashes, so every move is one reviewed PR (§4). This
is an upstream refresh, not a dependency update: these toolchains are what
vergil-containers delivers, not something its own code is built on. The
bundled pip of a python-build-standalone series moves with that series' pin.

Rust is pinned the same way (human decision, 2026-11-09). Each series pins
its exact Rust release (`x.y.z`, never a channel such as `stable`), and `rustup-init`
itself is pinned by version and per-architecture SHA-256. The image installs
that exact release; rustup checks each component against the hashes in the
release's channel manifest. The upstream refresh moves the release to the
newest patch of the series.

| Image | Source on Ubuntu 24.04 (my call unless noted) | Today |
|---|---|---|
| python | python-build-standalone release for the series, pinned per series (above) and unpacked at `/opt/python/<series>` at image build (human decision, §5.3.1) | `python:<v>-slim` |
| go | Official `go<v>.linux-<arch>.tar.gz` from go.dev, pinned per series (above) | `golang:<v>` |
| rust | `rustup` (pinned `rustup-init`) installing the series' pinned release (above), minimal profile plus the components the image has today | `rust:<v>-slim` |
| cpp-clang | apt.llvm.org `llvm-toolchain-noble-<v>` | `llvm-toolchain-trixie-<v>` |
| cpp-gcc | **Spike (§8):** `gcc-14` from noble, or the Ubuntu Toolchain PPA, whichever gives a released 14.x | `gcc-14` from trixie main |
| ts-node | nodejs.org prebuilt tarball, pinned per major (above), at `/opt/node/<major>` (human decision, §5.2.1). `node`, `npm` and `npx` link into `/usr/local/bin` | NodeSource `nodistro` channel |

In `ts-node:<major>`, npm's global prefix is set to `/usr` (my call), so
`npm root -g` stays `/usr/lib/node_modules` (§5.5).

#### 5.3.1 Why python-build-standalone for the family, vergil-python for the core

The two serve different jobs (human decisions, 2026-10-28 and 2026-11-09):

- **The family Python** needs the whole supported window. A library is tested
  on every series from 3.12 to 3.15. python-build-standalone, the CPython
  builds uv itself uses, provides any series with no publishing step.
- **The core Python** needs one runtime, chosen by us, for our own tools. That
  is what the vergil-python runtime packages are for: they are what an
  installed product runs on, and what the agent VMs already use, deliberately,
  to dogfood the runtime. With the tree on Ubuntu, the containers can install
  the same packages, and vergil-tooling with them (§5.2).

So the python family's images use python-build-standalone for every series,
`base` uses vergil-python for the core Python, and the two never share a path.
Nothing here changes vergil-python or the VMs' use of it.

**Installed as a system Python, not through `uv python install`** (human
decision, 2026-10-28). `uv python install` registers the interpreter
as uv-*managed*, and vergil-tooling runs uv with `UV_PYTHON` plus
`UV_PYTHON_PREFERENCE=only-system` (`lib/versions/extensions/python.py`,
`lib/container_cache.py`), which excludes managed interpreters. So the image
build downloads the release tarball itself, verifies it against the pinned
hash (§5.3) and unpacks it at `/opt/python/<series>`, where uv sees it as a
system interpreter. The
series is baked in at image build, so installing vergil-tooling or syncing a
project in the container never pays for an interpreter download.
`vrg-validate --all-versions`, which deliberately uses managed interpreters, is
unaffected.

Known version movements the spike must confirm or settle:

- **Unversioned `g++` on noble is 13**, not 14. The C++ images install `g++` for
  libstdc++. The plan picks the libstdc++ the images use and records it.
- **cmake and cppcheck** come "fixed by the Debian release"
  (`common/cpp-analysis.dockerfile`). Noble's are older than trixie's. If a
  consumer needs the newer ones, cmake moves to Kitware's apt channel and
  cppcheck to whatever the spike finds. Otherwise noble's are accepted and
  recorded.
- **`python-support`'s `python3-minimal`** is 3.12 on noble (3.13 on trixie). It
  serves tooling only. Once `base` provides `/usr/local/bin/python3`, the
  fragment is probably redundant in the tree. The plan decides.

### 5.4 Version lifecycle adapters

The adapters in `docker/versions/lifecycle/adapters/` decide when a series is
installable by probing where the image gets it. Each probe moves with its
source:

| Family | Today | After |
|---|---|---|
| python | Docker Hub `python:<v>-slim` + vergil apt index | The newest python-build-standalone release has an install-only build of the series for both architectures; the adapter resolves that release and its checksums itself. The vergil apt index check stays: it records `runtime_published`, which serves packaging, not the image |
| go | Docker Hub `golang:<v>` | go.dev download listing has a `go<v>.x` release |
| rust | Docker Hub `rust:<v>-slim` | The Rust release channel manifest for the series |
| cpp-clang | apt.llvm.org `trixie` listing | apt.llvm.org `noble` listing |
| cpp-gcc | `debian_source_in_suite("gcc-<n>", "trixie")` | The §8 source has a released `gcc-<n>` for noble |
| ts-node | `deb.nodesource.com/setup_<major>.x` (stale: the image uses `nodistro`) | nodejs.org's release index has a release of the major with `linux-x64` and `linux-arm64` tarballs |

The core Node and the core Python (§5.2.1) each get a manifest entry of their
own. In this epic no adapter probes them: a core move is a reviewed PR opened
by hand (§5.2), and the build fails if the chosen core series is not
available. Wiring the core entries into the lifecycle automation is a
follow-on (vergil-project/.github#978).

The adapters say when a series is installable; the patch an image ships is
the series' pin (§5.3), which the upstream refresh moves. The adapters and the
refresh hook share one fetcher per upstream (the plan's Task 6).

The adapters for a family change in the same PR as that family's image. The
dates the adapters record (`ga`, `eol`) and their sources (endoflife.date,
GitHub releases) are unchanged. Their tests and fixtures
(`test_adapters_report_only.py`, `test_adapters_python_go.py`, which asserts
the Docker Hub probes, `test_sources.py`, the trixie fixtures, and
`test_generate.py`'s base-image doc rows) move with them.

### 5.5 Consumers of image internals

These depend on how the current images are laid out, not just on their names.
Each is fixed in the PR that would otherwise break it, or kept true by the
image:

- **vergil-actions docs deploy** (`actions/cd/docs/deploy/action.yml`,
  `.github/workflows/ci-docs.yml`) calls `/usr/local/bin/python3` for
  `tomllib`. §5.2 keeps that path valid in `base` and every language image,
  and a build check asserts it.
- **vergil-containers' own scripts** (`docker/pins/check_pins.py`,
  `generate_catalog.py`, `docker/versions/generate.py`, `lifecycle/check.py`,
  `lifecycle/publish.py`, `report_exposure.py`), run with `python3` from
  `scripts/bin/validate-custom` and `ops.yml` in `prod-base`, import pyyaml.
  They find it only because today's `base` pip-installs it into its one
  interpreter. Before the `base` rebase, each script declares pyyaml in
  inline script metadata (PEP 723) and is run with plain `uv run` (§5.2).
  That names no interpreter path, so it works in the Debian `prod-base` that
  vergil-containers' own CI and `ops.yml` use during the freeze (§7) and in
  the Ubuntu one after it.
- **`[container].build-command` with `npm install -g` outside `ts-node`.**
  `mnemosys-project/melete`, a Python repo, runs one, relying on the `node`
  and `npm` every image carries today. §5.2 leaves no default `node` outside
  `ts-node`, so its container build breaks at the cutover. That is accepted
  (human decision, 2026-11-09): the release notes name the break, and the Node
  side of the runtime design, including TypeScript applications, is the
  follow-on vergil-project/.github#959.
- **vergil-tooling `container_cache.py`** installs vergil-tooling with `uv
  tool install` from git, on the project's resolved Python. It moves to the
  packaged install (§5.2), keeping the dev-ref path, and vergil-actions'
  `shared/setup/vergil` action moves with it, for container jobs and for host
  jobs on the runner. That action's `pip install --break-system-packages uv`
  goes. Across the cutover, both choose the install path by checking the
  environment (§7).
- **vergil-tooling `container.py`** expects global npm modules at
  `/usr/lib/node_modules` (`_NPM_GLOBAL_ROOT`, for a consumer's
  `[container].build-command` global install). §5.3 keeps it true in `ts-node`
  images by setting npm's global prefix to `/usr`, and a build check asserts
  `npm root -g`. No other image has a default `npm`.
- **`[container].system-packages`** (vergil-tooling `container_cache.py`,
  vergil-actions `system-packages`) runs `apt-get install` against the image's
  sources. On Ubuntu some packages live in `universe`, which `ubuntu:24.04`
  enables by default. The cutover validation (§8) runs every consuming repo's
  CI, which exercises each repo's declared packages.

### 5.6 Agent VMs (vergil-vm)

- `templates/agent.yaml` and `templates/agent.yaml.skel` change from
  `template:_images/ubuntu-lts` to `template:_images/ubuntu-24.04`, and the header
  comment is made accurate.
- `tests/test_base.sh` already asserts `VERSION_ID="24.04"`. It now passes on a
  fresh VM.
- The racc seed in `templates/provision/40-profile.sh` exists for 26.04's Ruby
  3.3. Its check passes when the system racc is older than the seed or absent,
  so it is harmless on 24.04. **My call:** keep it, with a comment that it
  matters again when the VMs move to 26.04 (#651).
- Existing 26.04 VMs move with `vrg-vm rebuild` after the vergil-vm release that
  carries the pin. `vrg-vm update` does not change the OS. The release notes
  say so, and the rebuild is a `deployment` task.
- The cloud modules (`opentofu/modules/{azure,gcp}/vm`) are already on 24.04
  and do not change.

### 5.7 GitHub-hosted runners

Every workflow that names `ubuntu-latest` names `ubuntu-24.04` instead (human
decision: the floor is 24.04 everywhere, for the same reason Lima is pinned).
`ubuntu-latest` already runs Ubuntu 26.04 for us: run logs show the
`ubuntu-26.04` image from early November 2026 (§1), although GitHub's
runner-images README still maps the label to 24.04. So the change is not a
no-op: it moves host jobs back to the floor, and the move to 26.04 becomes a
deliberate change under the look-ahead in #651. The arm64 runner is already
`ubuntu-24.04-arm`.

**Scope: every repo that consumes vergil-tooling, in any org**, plus doc
examples (`getting-started.md`, the action pages). Most of vergil-project's
share is in vergil-actions' `.github/workflows`; repos in other orgs that
track vergil-tooling's floating line carry the label too. The plan recounts
it, since new workflows keep adding the label.

**The guard** (human decision, 2026-11-09). A check in the validation every
repo runs (vergil-tooling's common checks) refuses `ubuntu-latest` in a
workflow, so the floating label does not come back. Every consuming repo
picks up a vergil-tooling release on its next CI run, with no change of its
own, so the guard ships only after the label PRs have merged everywhere the
recount found the label, and then as an error directly, with no warning
release (§7).

### 5.8 CVE triage

None of the entries in `.trivyignore` is a Debian package. They are bundled
tool binaries, and they carry over unchanged. The `linux-libc-dev` rego rule and `ignore-unfixed` stay. The publish gate's Trivy
scan of the new images is the check: any fixable CRITICAL/HIGH finding on the
Ubuntu images is triaged in the PR that introduces it, as today.
`docs/site/docs/operations/trivy-triage.md`'s "Debian fix" wording is updated.

**Findings inside a runtime belong to that runtime's owner** (human decision,
2026-11-09). vergil-containers never patches or upgrades anything inside a
runtime, including the `npm` or `pip` it bundles. Each path has one owner:

- `/opt/vergil/python/<core>/` belongs to the vergil-python package: fixed by
  a vergil-python release, triaged in vergil-python. How vergil-python keeps
  itself current is vergil-project/.github#965.
- `/opt/vergil/node/<core-major>/` is a pinned nodejs.org tarball: fixed by
  moving the core Node pin (§5.2.1).
- vergil-tooling's package venv belongs to vergil-tooling: fixed by a
  vergil-tooling release.
- A family runtime from python-build-standalone or nodejs.org is fixed by
  moving that series' pin (§5.3).

Where no fixed runtime exists yet, the finding gets a `.trivyignore` entry
under that file's rules until one does.

### 5.9 The OS-usage table

One page in the vergil-containers site docs lists every place an OS is chosen,
the OS, and the reason:

- the image tree;
- the `pkg-*` images outside it;
- the agent VMs (Lima, Azure, GCP);
- the GitHub-hosted runners (`ubuntu-24.04`, `ubuntu-24.04-arm`; §5.7).

Each row says whether the OS is **required** (a package target, the glibc floor)
or a **choice** (the shared platform). The site docs of the other repos link to
it, not copy it. The documentation-review bookend
(vergil-project/vergil-containers#739) checks that it is current.

## 6. Amendments to #213 and #607

These are applied before either epic resumes. Each is a correction note
appended to that epic's `spec.md` and `plan.md`, in a docs PR of that epic's
own (vergil-project/.github#974 for #213, vergil-project/.github#975 for #607;
human decision, 2026-11-09, so both epics can resume before this epic's
planning ends), plus an edit to the affected task's issue body pointing at it.
This epic's tasks then use the result. They do not rework it.

### 6.1 #213 — parent-aware build machinery

Epic #213's open tasks assume a flat matrix of independent images. They become
parent-aware:

- **Manifest (`parent:`).** A family, `base` and a toolchain entry may name a
  `parent:`, another manifest entry plus version (for example `base` →
  `pkg-ubuntu` `24.04`). No `parent:` means an upstream image, as today. The
  matrix generator emits each entry's parent and its depth in the tree.
  `generate.py` refuses a cycle or a parent that does not exist. This lands in
  a new #213 task, **T7b**, together with the tiered publish that is its first
  consumer. T7 (#515) stays as #213's plan defines it (native per-arch builds,
  the stitch, `base` folded into the matrix) and goes first, as #213 §0.3.1
  orders; T7b follows it and comes before T2 (human decision, 2026-11-09:
  T7 does not wait for this epic's planning). Today no entry declares a
  parent, so it is a no-op until #649 sets one.
- **T2 (#510) dependency map.** T2 consumes the `parent:` field T7b added.
  `dependency_map()` adds parent edges: an entry's
  inputs include its parent's inputs, transitively. A change under
  `docker/pkg-ubuntu/**` reaches `base` and every language image.
- **T5 (#513) selector.** Selecting an entry selects all its descendants. The
  fail-closed rule for unclassified paths is unchanged.
- **T6′ (#514) PR build.** When a PR selects a parent, it builds the parent and
  then its selected descendants **in the same job, on the locally built
  parent** (buildx `--build-context` or a local load, the builder's choice),
  because T6′ never pushes. When a PR selects only children, they build on the
  published `dev-` parent.
- **T7b tiered publish.** T7b introduces `parent:` (above) and reworks
  T7's native per-arch publish into tiers. Publishing runs in tiers by depth: a
  tier's per-arch builds, stitch and promotion finish before the next tier
  starts, and children `FROM` their parent **by digest** from the same run. This
  holds for the `develop` publish, the release (`prod`) publish, and the nightly
  `no-cache` rebuild. A failed parent stops its descendants in that run. The
  previous images stay published, as #213 §4.5 already requires for any failure.
  T7 folds `base` into the matrix; once #649 gives it a parent, `base` is a
  depth-1 entry under `pkg-ubuntu`.
- **T7b build arguments.** Today each matrix entry carries one build argument
  (`build-arg` = version), and the publish, the PR build and `build.sh` pass
  only that. T7b gives each entry a `build-args` map (`{NAME: value}`), `base`
  included, and all three pass every entry in it, so the toolchain pins and
  core series (§5.3) reach the build.
- **Ledger.** #213 §9's item D2 is marked as taken up by #649. The §4.3 note
  ("inverting today's official-upstream-base model") is answered by this spec.
- **Local parity.** `build.sh` builds in the same tier order, so
  `build.sh python` builds or pulls its parents first.

### 6.2 #607 — the root image

- **T3 (#727) manifest.** The `toolchains:` section accepts the `parent:` field's
  counterpart: a `toolchains:` entry may be named as a parent. Nothing in the
  section forbids it. `pkg-ubuntu` itself has no parent.
- **T4 (#728) `pkg-ubuntu` template.** Adds a build-time check that the image
  sets no `ENTRYPOINT`, `CMD`, `USER` or `WORKDIR` that a child would inherit
  unexpectedly. The image's contents are unchanged.
- **The marker stays as #607 designed it.** §5.2 removes it in `base`, so
  #607's setup action and marker format need no change.

## 7. One rollout order for #607, #213 and #649

**A focused window under a release freeze (human decision, 2026-10-28).** The
container images are the foundation for everything else, so the three epics
are worked start to finish, in one focused window of a day or two, with no
other epic in flight. vergil-containers releases are always human-gated, so
**no vergil-containers release is cut during #649's steps 6–9**, until the
cutover release itself. The nightly
rebuild continues; it rebuilds `main`, which the freeze holds on the last
pre-migration release. That makes the single cutover release in step 9
possible.

**During the freeze** (human decision, 2026-11-09), `prod-` images change
only through that nightly rebuild, which still takes upstream OS patches. A
nightly that fails its gates leaves the previous images published (#213
§4.5), so consumers keep working. A fix that needs a vergil-containers change
waits for the cutover release; there is no hotfix path. If a gate blocks the
cutover release itself, it is bypassed with the release tooling's existing
option to skip the audit.

Each step is safe on its own, and each epic's steps keep their own task
boundaries. #213's T7 (#515) has no dependency on this epic: §6 does not
amend it, so it starts at once. It is still on the critical path, because
T7b reworks T7's publish and T2, T5 and T6′ follow T7b. So T7 must merge
before step 3 starts. (T7 merged on 2026-11-09.)

Step 6 needs only T7b of step 3, not T2, T5 and T6′ (human decision,
2026-11-09). A child can be built on a stale parent only where something
builds images, and until T6′ nothing in PR CI does: T6′ adds PR image builds,
and it follows T2 and T5, so it arrives with parent-aware selection. Every
publish, meanwhile, builds children on the parent built in the same run
(T7b's tiers). The anti-goal of §4 cannot occur in that window.

**Runner labels first, outside the numbered order** (human decision,
2026-11-09). `ubuntu-latest` already runs 26.04 (§5.7), so the label PRs
start now, independent of every step below. They touch
`.github/workflows/**`, so GitHub refuses the agent's push; the human's
`vrg-submit-pr` pushes those branches. Once they have merged everywhere the
recount found the label, a vergil-tooling release ships the guard as an
error.

1. **Amend** #213 and #607 (§6): notes merged in each epic's own docs PR
   (#974, #975); issue bodies edited.
2. **#607 completes** as amended: the `pkg-*` images are published and public,
   package jobs move onto them, and the release is cut. Its rollout (#607 §5.4)
   is unchanged.
3. **#213 resumes** with T7b and the parent-aware T2, T5 and T6′. Until #649 sets
   the first `parent:`, the new support is exercised only by tests with a
   synthetic parent fixture.
4. **#649 spike** (§8) runs **in parallel with steps 2–3**. It needs no
   machinery; it builds trial Dockerfiles locally.
5. **Install paths that work on both image generations** (human decision,
   2026-11-09), before any image changes:
   - vergil-containers' scripts move to PEP 723 and plain `uv run` (§5.5);
   - vergil-tooling (`container_cache.py`) and vergil-actions
     (`shared/setup/vergil`) choose the vergil-tooling install by checking the
     environment: where the vergil apt source and the `vergil-python` package
     are present, the packaged install; otherwise today's `uv tool install`
     from git. Each logs which path it took, and a failure on the chosen path
     fails the job; nothing falls back after a failure. Host jobs take the
     packaged path on the runner (§5.2). Consumers track vergil-tooling and
     vergil-actions on a floating line, so this ships in one release of each
     before step 6, and it keeps working on the Debian images for a revert
     (§4).
6. **#649 rebases `base`** onto `pkg-ubuntu:24.04` (the first real `parent:`).
   Then one PR per language family rebases that image and its adapter. Each
   publishes `dev-` images on merge. Consumers keep using `prod-`.
7. **#649 consumer fixes** (§5.5) land wherever an image change would break
   them, ahead of the release.
8. **Pre-release consumer check** (§8), on the `dev-` tree.
9. **Release** (deployment task, human-gated): one vergil-containers release
   promotes the whole tree to `prod-` at once and ends the freeze. Every
   consumer moves to Ubuntu on its next image pull. The way back is a revert
   release (§4).
10. **Cold-rebuild validation** (§8).
11. **VM pin** (§5.6): a vergil-vm release, then the `vrg-vm rebuild`
    deployment task. It is independent of steps 6–10 and can land any time in
    the window.
12. **Remove the cutover scaffolding, in a follow-on epic** (human decisions,
    2026-11-09). Once the human judges that the cutover has held, so a revert
    to the Debian images is no longer an option, vergil-tooling and
    vergil-actions drop step 5's `uv tool install` branch for released
    versions, which nothing can reach any more. The dev-ref path stays. This
    epic ends at step 11 and seeds that follow-on epic as a brainstorm
    bookend, so the branch and the revert path stay until the human's call.

## 8. Risks, spike and validation

**Spike (first implementation task, throwaway Dockerfiles, a recorded
report).** Build on `ubuntu:24.04` plus the planned `pkg-ubuntu` contents, on
amd64 and arm64:

- the GCC 14 source: a released 14.x on noble, which channel, and the libstdc++
  question in §5.3;
- python-build-standalone: every window series unpacks at
  `/opt/python/<series>` on both architectures; uv, with
  `UV_PYTHON_PREFERENCE=only-system`, finds it; `/usr/local/bin/python3`
  resolves; the python family's default (§5.2) works; and `base`'s uv tools
  and vergil-tooling still run on the core Python in a `dev-python:3.12`
  trial;
- vergil-python and vergil-tooling: the signing key's fingerprint check
  passes, `vergil-python<core>` and a pinned vergil-tooling install from the
  vergil apt repository in a noble container on both architectures, a
  dev-ref install uses `uv tool install` on the core Python, and step 5's
  environment check (§7) picks the right path in a Debian and an Ubuntu
  image;
- Node side by side: markdownlint runs on the core Node in a trial with no
  default `node` and in a `ts-node:24` trial, and `npm root -g` is
  `/usr/lib/node_modules` there;
- the Go, Rust and LLVM sources and nodejs.org tarballs for every
  series in the manifest;
- noble's cmake and cppcheck versions against what the C++ images ship today;
- one `vrg-validate` run of vergil-tooling itself in a trial `dev-python`;
- whether vergil-tooling's package venv needs pip inside vergil-python
  (§5.8, vergil-project/.github#965);
- a Trivy scan of the trial core Node, `ts-node` and python-build-standalone
  layers, recording each fixable CRITICAL/HIGH finding in a bundled `npm` or
  `pip`, so each affected task states its remedy in advance: a newer runtime
  pin or a `.trivyignore` entry (§5.8), never an in-place upgrade.

The spike's report goes into the plan. A failed item becomes a decision for the
human, not a silent substitution.

**Pre-release consumer check (a `validation` task before the release, my call).**
Consumers pull `prod-` by default, but vergil-actions' reusable workflows take
`container-prefix: dev`, which exists for exactly this. Before the cutover
release, run representative consumers' real CI on the new `dev-` images by
setting `container-prefix: dev` on a throwaway branch, including one docs
deploy dry run, so the CI-only paths (§5.5: docs deploy, `system-packages`,
`build-command`) are exercised. Set it back to `prod` once the release ships.
Where a path does not pass the prefix through, check it by hand in a
`dev-base` container. Also run `vrg-container-run -- vrg-validate` in
those repos with `DOCKER_DEV_IMAGE` set to the new `dev-` image. Representative means at least
vergil-tooling for python and `base`, plus each switched family's consumers.
Where no repo covers a case (today no repo consumes ts-node or cpp-gcc, and
none declares a non-empty `[container].system-packages`), a throwaway fixture
repo or branch stands in, so every family and the `system-packages` path run
through real CI once. In the new
`dev-base`, also run vergil-containers' `scripts/bin/validate-custom` and
`lifecycle/check.py`, which run only in its own CI. This uses the override
that exists today, with no new tooling.

**Validation (cold-rebuild, a `validation` task).** After the release:

- a no-cache rebuild of the full tree succeeds in tier order on both
  architectures;
- every image's `/etc/os-release` reports Ubuntu 24.04, and none carries the
  package toolchain marker;
- every consuming repo's PR CI is green on the new `prod-` images;
- a fresh `vrg-vm create` gives a 24.04 VM that passes `tests/test_base.sh`;
- the OS-usage table matches what the validation observed.

## 9. Decisions record

Human decisions (2026-10-28):

- Ubuntu 24.04 is the shared base; Debian goes; RHEL/UBI stays a package
  target only (§1).
- `pkg-ubuntu:24.04` is the root of the tree. #649 builds on #607 and does not
  duplicate it (§5.1).
- Parent-image support is folded into #213's and #607's open tasks now, rather
  than reworked later (§6).
- Install-tests stay on the minimal `pkg-*` images (§3).
- Lima is pinned to 24.04 (§5.6), and `ubuntu-latest` runners are pinned to
  `ubuntu-24.04` (§5.7).
- The python family's images use python-build-standalone CPython for
  3.12–3.15 (§5.3.1).
- A release freeze, with one cutover release, in a focused window (§7).
- Python is unpacked from python-build-standalone as a system Python at image
  build, not installed with `uv python install`, so the interpreter is baked
  in and `only-system` finds it (§5.3.1).
- The runtime rule: core tooling runs on its own side-by-side core runtime,
  never on the default runtime, and only a language family sets the default
  command. Node moves to nodejs.org tarballs for this (§5.2.1).
- The `parent:` manifest field and the tiered publish land in a new #213
  task, T7b, after T7 and before T2, which consumes it. T7 proceeds as #213
  planned it, without waiting for this epic (§6.1).

Human decisions (2026-11-09):

- The core Python is vergil-python, and core runtimes live under
  `/opt/vergil/`, apart from every family runtime (§5.2, §5.2.1). Its series
  is pinned in the version manifest (today 3.15) and moves only through a
  reviewed PR, not automatically (§5.2).
- vergil-tooling is installed in the containers as its package, at the repo's
  pinned version, when the container cache is built (§5.2).
- Nothing of ours relies on a package in an image's interpreter; scripts
  declare their dependencies (§5.2, §5.5).
- Making vergil-tooling itself support only the newest series is a follow-on
  epic, outside this one (§3).
- No default `node` outside `ts-node`; the resulting break of melete's
  `npm install -g` build-command is accepted, and Node's runtime design follows
  in vergil-project/.github#959 (§5.5).
- No stable link hides the core series. Scripts declare their dependencies
  (PEP 723, plain `uv run`), and a one-liner gets the `python3` on `PATH`,
  standard library only (§5.2).
- The vergil-tooling install chooses its path by checking the environment
  across the cutover, and a follow-on epic removes that branch once the human
  judges the cutover has held (§7, steps 5 and 12).
- `ubuntu-latest` already runs 26.04, so the runner-label PRs go first and
  cover every repo that consumes vergil-tooling; the guard follows them as an
  error, with no warning release (§5.7, §7).
- Downloaded toolchains, and Rust's exact release with a pinned
  `rustup-init`, are pinned in source with their hashes and moved by the
  upstream refresh, as upstream inputs rather than dependencies (§5.3).
- No image runs `pip install` (§5.2).
- Findings inside a runtime are fixed by its owner; vergil-python's own
  rebuild model is vergil-project/.github#965 (§5.8).

My calls, open to correction: §5.1 (no removals from `base`), §5.2 (marker
removed in `base`; amd64 apt-mirror config inherited; Python CLI tools as uv
tools; the python family owns `python3`; no default `node` outside `ts-node`),
§5.3 (npm's global prefix `/usr` in `ts-node`), §5.6 (keep the racc seed), §6.2
(the `pkg-ubuntu` FROM-ability check), §8 (pre-release consumer check), and the
toolchain sources in §5.3 other than Python and Node.

## 10. Left to the builder

- The `parent:` field's exact schema, and how the matrix carries depth.
- How T6′ hands a locally built parent to its children (`--build-context` or a
  local load).
- Which `base` fragments language images keep including after the rebase,
  provided the result has no duplicate installs.
- The exact build checks (marker absent, `/usr/local/bin/python3`, npm root,
  core tools on their core runtimes, OS release), and where they live.
- The manifest shape of the core Node (§5.2.1), and how the markdownlint
  wrapper is generated.
- Where in vergil-tooling's common checks the `ubuntu-latest` check lives
  (§5.7).
- Where a host job's dev-ref vergil-tooling install gets uv, other than pip
  (§5.2).
- The pin format for the downloaded toolchains and Rust (§5.3).
- Whether the C++ and TypeScript READMEs keep their image-structure sections,
  once the "compose fragments, never FROM-chain" convention they record is
  gone.
