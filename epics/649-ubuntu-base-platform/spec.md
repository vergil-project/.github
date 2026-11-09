# Ubuntu 24.04 base platform

- **Epic:** vergil-project/.github#649
- **Planning issue:** vergil-project/.github#605 (review history is in its
  comments)
- **Status:** draft, in design review
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
| `dev-java` | Ubuntu 26.04 | Inherited from `eclipse-temurin:<v>-jdk` |
| Package jobs (#607) | Ubuntu 24.04 and 26.04, UBI 9 and 10 | Required: the package targets |
| Cloud agent VMs (Azure, GCP) | Ubuntu 24.04 | Pinned in the OpenTofu modules |
| Local agent VMs (Lima) | Ubuntu 26.04 | `template:_images/ubuntu-lts` floats to the newest LTS, though the template header says 24.04 |

Nobody chose Debian. Every Debian image either inherited it or copied it from a
neighbour, and the one Ubuntu image inherited Ubuntu the same way. We package
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
  with the source of each tool adjusted for Ubuntu.

## 4. Anti-goals (failure even if it works)

- **A child built on a stale parent.** A publish or PR build in which `base` or
  a language image builds on yesterday's `pkg-ubuntu:24.04`, when the same run
  changed or rebuilt that parent.
- **A dev image that passes for a package toolchain image.** #607's setup action
  treats `/etc/vergil/package-toolchain` as proof of a toolchain image. A dev
  image must never carry it.
- **Silent toolchain drift.** The upstream language images used to deliver
  patch releases (CPython 3.14.x, Go 1.27.x, …) for free. Each image now
  installs its own toolchain, so the build must take the newest patch release
  of its series on every build. A pinned patch version that goes stale
  unnoticed is a failure.
- **A switch with no way back.** Consumers have no setting to pin an image, and
  dated prod tags are deleted after 30 days (`package-cleanup.yml`). So the way
  back is a **revert release** of vergil-containers that restores the previous
  templates. The cutover PRs are kept revertible (no unrelated changes mixed
  in), and the release notes name the revert path.
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
         ├─ java:<v>
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
- **Python** comes from §5.3's Python source, installed at
  `/opt/python/<series>`. `base` keeps its current series (3.14) as the
  default: `python3` resolves on `PATH` at `/usr/local/bin/python3`, a link
  into `/opt/python/3.14`, as the `python:*-slim` images placed it.
  vergil-actions calls that path directly (§5.5), and needs only `tomllib`
  (3.11+).
- **`base`'s Python CLI tools run on the core Python runtime** (§5.2.1).
  yamllint, ansible-lint, semgrep and mkdocs-material with mike install as uv
  tools, each in its own environment, created with
  `/opt/python/3.14/bin/python3.14` by its versioned path, never through
  `/usr/local/bin/python3`. A uv tool environment is bound to the interpreter
  path it was created with, so a tool created through the default link would
  break, or silently run on another series, once a python image replaces that
  link.
- **The python family owns `python3`.** In `dev-python:<v>`,
  `/usr/local/bin/python3` (and `python`, `pip`) links to `/opt/python/<v>`, as
  it is series `<v>` today when the image is `FROM python:<v>-slim`. The
  family's own pip-installed tools (pytest-xdist and the rest in today's
  template) install into that interpreter. A build check asserts
  `python3 --version` matches the image's series, so a matrix leg can never
  silently test on `base`'s 3.14. Other language images keep `base`'s 3.14 as
  the default.
- **Node** for `base`'s one Node tool, markdownlint-cli, is the core Node
  runtime at `/opt/node/<core-major>` (today 22), from the nodejs.org prebuilt
  tarball. NodeSource's `.deb` goes, since it allows only one Node on a
  machine. `base` puts no `node`, `npm` or `npx` on `PATH`; only `ts-node`
  images have a default Node (§5.3). `markdownlint` on `PATH` is a wrapper that
  runs the tool with `/opt/node/<core-major>/bin/node` explicitly. npm's
  generated bin links start with `#!/usr/bin/env node`, which would pick up
  whatever `node` is on `PATH`, or fail where there is none.
- **apt-mirror config is inherited on amd64.** #607 bakes the Azure-first mirror
  list and fail-fast timeouts into amd64 `pkg-ubuntu`. Dev images inherit them.
  **My call:** keep them. The mirror list fails over to the default archive, and
  vergil-actions' `system-packages` step, which runs `apt-get install` in dev
  images in CI, gains the same protection against the slow Azure mirror that
  prompted #607. A builder check confirms an `apt-get install` from a non-Azure
  network (a Lima VM) still succeeds.

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
  **core runtime**, installed side by side at a private, versioned path
  (`/opt/python/<series>`, `/opt/node/<major>`), and calls it by that path.
- **Core runtimes have their own pins**, separate from the family windows. A
  core runtime moves when core tooling needs it to, never because a language
  image moved. Each core runtime is an entry in the version manifest, under the
  lifecycle automation like any series.
- **Only a language family's image sets a default command for its runtime.**
  The exception is `python3` in `base`, which consumers already rely on (§5.5).
- **Runtimes are only ever added side by side.** A language image adds its
  series next to the core runtime and does not remove or overwrite it.
- **Build checks assert both halves.** Every core tool runs on its pinned core
  runtime in every image, and the default command matches the family series.

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
channel, never built from source**, and the newest patch release of the
series at build time (§4).

| Image | Source on Ubuntu 24.04 (my call unless noted) | Today |
|---|---|---|
| python | python-build-standalone release for the series, downloaded checksum-verified and unpacked at `/opt/python/<series>` at image build, newest patch of the series (human decision, §5.3.1) | `python:<v>-slim` |
| go | Official `go<v>.linux-<arch>.tar.gz` from go.dev, checksum-verified, newest patch of the series | `golang:<v>` |
| rust | `rustup` with the series' newest stable, minimal profile plus the components the image has today | `rust:<v>-slim` |
| java | Eclipse Temurin JDK from the Adoptium apt repository (`packages.adoptium.net`, which serves noble) | `eclipse-temurin:<v>-jdk` (Ubuntu 26.04) |
| cpp-clang | apt.llvm.org `llvm-toolchain-noble-<v>` | `llvm-toolchain-trixie-<v>` |
| cpp-gcc | **Spike (§8):** `gcc-14` from noble, or the Ubuntu Toolchain PPA, whichever gives a released 14.x | `gcc-14` from trixie main |
| ts-node | nodejs.org prebuilt tarball, checksum-verified, at `/opt/node/<major>`, newest release of the major (human decision, §5.2.1). `node`, `npm` and `npx` link into `/usr/local/bin` | NodeSource `nodistro` channel |

In `ts-node:<major>`, npm's global prefix is set to `/usr` (my call), so
`npm root -g` stays `/usr/lib/node_modules` (§5.5).

#### 5.3.1 Why python-build-standalone baked in, not vergil-python

The two serve different jobs (human decision, 2026-10-28):

- **Development** needs the whole supported window. A library is tested on
  every series from 3.12 to 3.15. python-build-standalone, the CPython builds
  uv itself uses, provides any series with no publishing step.
- **Deployment** needs one runtime per product, chosen by the product. That is
  what the vergil-python runtime packages are for. They cost more to publish and
  roll out, which pays off only where a package is installed on a host. The
  agent VMs use them deliberately, to dogfood the runtime.

So the containers use python-build-standalone for every series, and
vergil-python stays the runtime for installed packages and the VMs. Nothing
here changes vergil-python or the VMs' use of it.

**Installed as a system Python, not through `uv python install`** (human
decision, 2026-10-28). `uv python install` registers the interpreter
as uv-*managed*, and vergil-tooling runs uv with `UV_PYTHON` plus
`UV_PYTHON_PREFERENCE=only-system` (`lib/versions/extensions/python.py`,
`lib/container_cache.py`), which excludes managed interpreters. So the image
build downloads the release tarball itself, verifies its checksum and unpacks
it at `/opt/python/<series>`, where uv sees it as a system interpreter. The
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
| java | Docker Hub `eclipse-temurin:<v>-jdk` | Adoptium apt index for noble has `temurin-<v>-jdk` |
| cpp-clang | apt.llvm.org `trixie` listing | apt.llvm.org `noble` listing |
| cpp-gcc | `debian_source_in_suite("gcc-<n>", "trixie")` | The §8 source has a released `gcc-<n>` for noble |
| ts-node | `deb.nodesource.com/setup_<major>.x` (stale: the image uses `nodistro`) | nodejs.org's release index has a release of the major with `linux-x64` and `linux-arm64` tarballs |

The core runtimes (§5.2.1, core Python and core Node) get manifest entries of
their own and probe the same sources as their family.

The adapters for a family change in the same PR as that family's image. The
dates the adapters record (`ga`, `eol`) and their sources (endoflife.date,
GitHub releases) are unchanged. Their tests and fixtures
(`test_adapters_report_only.py`, `test_sources.py`, the trixie fixtures, and
`test_generate.py`'s base-image doc rows) move with them.

### 5.5 Consumers of image internals

These depend on how the current images are laid out, not just on their names.
Each is fixed in the PR that would otherwise break it, or kept true by the
image:

- **vergil-actions docs deploy** (`actions/cd/docs/deploy/action.yml`,
  `.github/workflows/ci-docs.yml`) calls `/usr/local/bin/python3`. §5.2 keeps
  that path valid in `base` and every language image, and a build check
  asserts it.
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
The scope, counted 2026-10-28: 43 places in vergil-actions'
`.github/workflows`, 4 workflow files in vergil-containers, and none in the
other vergil-project repos, plus doc examples (`getting-started.md`, the action
pages). The arm64 runner is already `ubuntu-24.04-arm`. A check in
vergil-actions' own validation refuses `ubuntu-latest` in a workflow, so the
floating label does not come back. vergil-actions has no `scripts/bin/` today;
where the check lives is the builder's call. Moving runners to 26.04 is part of the
look-ahead in #651.

### 5.8 CVE triage

None of the 37 entries in `.trivyignore` is a Debian package. They are bundled
tool binaries, and they carry over unchanged. Temurin's `pebble` entries
(`.trivyignore:85-88,146-148`) come from the `eclipse-temurin` image. They are
removed if the new `java` image no longer carries pebble, and kept otherwise. The
`linux-libc-dev` rego rule and `ignore-unfixed` stay. The publish gate's Trivy
scan of the new images is the check: any fixable CRITICAL/HIGH finding on the
Ubuntu images is triaged in the PR that introduces it, as today.
`docs/site/docs/operations/trivy-triage.md`'s "Debian fix" wording is updated.

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
appended to that epic's `spec.md` and `plan.md` (in this epic's docs PR),
plus an edit to the affected task's issue body pointing at it. #649's tasks
then use the result. They do not rework it.

### 6.1 #213 — parent-aware build machinery

#213's open tasks assume a flat matrix of independent images. They become
parent-aware:

- **Manifest (`parent:`).** A family, `base` and a toolchain entry may name a
  `parent:`, another manifest entry plus version (for example `base` →
  `pkg-ubuntu` `24.04`). No `parent:` means an upstream image, as today. The
  matrix generator emits each entry's parent and its depth in the tree.
  `generate.py` refuses a cycle or a parent that does not exist. This lands in
  **T7 (#515)**, because #213 §0.3.1 orders T7 first and its tiered publish is
  the first consumer (human decision, 2026-10-28). Today no entry
  declares a parent, so it is a no-op until #649 sets one.
- **T2 (#510) dependency map.** T2 consumes the `parent:` field T7 added.
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
- **T7 (#515) native arm64 and publish.** T7 introduces `parent:` (above).
  Publishing runs in tiers by depth: a
  tier's per-arch builds, stitch and promotion finish before the next tier
  starts, and children `FROM` their parent **by digest** from the same run. This
  holds for the `develop` publish, the release (`prod`) publish, and the nightly
  `no-cache` rebuild. A failed parent stops its descendants in that run. The
  previous images stay published, as #213 §4.5 already requires for any failure.
  T7's "fold `base` into the matrix" makes `base` a depth-1 entry under
  `pkg-ubuntu`.
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
**no vergil-containers release is cut during #649's steps 5–6**. The nightly
rebuild continues; it rebuilds `main`, which the freeze holds on the last
pre-migration release. That makes the single cutover release in step 7
possible.

Each step is safe on its own, and each epic's steps keep their own task
boundaries:

1. **Amend** #213 and #607 (§6): notes merged with this epic's docs PR; issue
   bodies edited.
2. **#607 completes** as amended: the `pkg-*` images are published and public,
   package jobs move onto them, and the release is cut. Its rollout (#607 §5.4)
   is unchanged.
3. **#213 resumes** with the parent-aware T2, T5, T6′ and T7. Until #649 sets
   the first `parent:`, the new support is exercised only by tests with a
   synthetic parent fixture.
4. **#649 spike** (§8) runs **in parallel with steps 2–3**. It needs no
   machinery; it builds trial Dockerfiles locally.
5. **#649 rebases `base`** onto `pkg-ubuntu:24.04` (the first real `parent:`).
   Then one PR per language family rebases that image and its adapter. Each
   publishes `dev-` images on merge. Consumers keep using `prod-`.
6. **#649 consumer fixes** (§5.5) land wherever an image change would break
   them, ahead of the release.
7. **Pre-release consumer check** (§8), on the `dev-` tree.
8. **Release** (deployment task, human-gated): one vergil-containers release
   promotes the whole tree to `prod-` at once and ends the freeze. Every
   consumer moves to Ubuntu on its next image pull. The way back is a revert
   release (§4).
9. **Cold-rebuild validation** (§8).
10. **VM and runner pins** (§5.6, §5.7): a vergil-vm release, then the
    `vrg-vm rebuild` deployment task, and the runner-label PRs. These are
    independent of steps 5–9 and can land any time in the window. The
    runner-label PRs touch `.github/workflows/**`, so GitHub refuses the
    agent's push; the human's `vrg-submit-pr` pushes those branches.

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
  still run on 3.14 in a `dev-python:3.12` trial;
- Node side by side: markdownlint runs on the core Node in a trial with no
  default `node` and in a `ts-node:24` trial, and `npm root -g` is
  `/usr/lib/node_modules` there;
- the Go, Rust, Temurin and LLVM sources and nodejs.org tarballs for every
  series in the manifest;
- noble's cmake and cppcheck versions against what the C++ images ship today;
- one `vrg-validate` run of vergil-tooling itself in a trial `dev-python`.

The spike's report goes into the plan. A failed item becomes a decision for the
human, not a silent substitution.

**Pre-release consumer check (a `validation` task before the release, my call).**
Consumers always pull `prod-`, so the `dev-` images are never exercised by
real consumer CI before a release. Before the cutover release, run `vrg-container-run -- vrg-validate` in representative consuming repos with
`DOCKER_DEV_IMAGE` set to the new `dev-` image. Representative means at least
vergil-tooling for python and `base`, plus each switched family's consumers,
including one repo that declares `[container].system-packages`. This uses the
override that exists today, with no new tooling.

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
- The containers use python-build-standalone CPython for 3.12–3.15.
  vergil-python stays the runtime for deployed packages and the VMs (§5.3.1).
- A release freeze, with one cutover release, in a focused window (§7).
- Python is unpacked from python-build-standalone as a system Python at image
  build, not installed with `uv python install`, so the interpreter is baked
  in and `only-system` finds it (§5.3.1).
- The runtime rule: core tooling runs on its own side-by-side core runtime,
  never on the default runtime, and only a language family sets the default
  command. Node moves to nodejs.org tarballs for this (§5.2.1).
- The `parent:` manifest field lands in #213's T7, which goes first, and T2
  consumes it (§6.1).

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
- The manifest shape of the core runtimes (§5.2.1), and how the markdownlint
  wrapper is generated.
- Where vergil-actions' `ubuntu-latest` check lives (§5.7).
- Whether the C++ and TypeScript READMEs keep their image-structure sections,
  once the "compose fragments, never FROM-chain" convention they record is
  gone.
