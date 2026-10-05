# Binary package publishing — signed `.deb`/`.rpm` releases into per-org package repositories

Epic: `vergil-project/.github#356`. Seed: `vergil-project/.github#355`.

## 1. Summary

A Vergil-managed repository's release today produces tags, a GitHub Release and a
CI-evidence bundle. This epic adds **signed binary OS packages** to the release,
built by standard tooling and published into an **org-specific package
repository** so that consumers install components with `apt`/`dnf` exactly as an
outside user would:

- `.deb` packages for Ubuntu 24.04 and 26.04, on amd64 and arm64;
- `.rpm` packages for RHEL 9 and 10, on amd64 and arm64.

The goal is to **demonstrate production-grade engineering**. Here that means
owning and automating the whole build → sign → index → publish pipeline, and
producing artifacts a consumer can verify. It does not mean an elaborate hosting
stack, and it is not driven by outside customers; the user base today is one.

The epic delivers the generic, org-agnostic machinery; the vergil-project package
repository; the pinned Python runtime as a package; and vergil-tooling itself
published and installed as a package on the static VMs (the dogfood).

## 2. Context

### 2.1 The three milestones

This is **M2** of an architecture that came out of
`logical-minds-foundry/.github#293`. There, lab components copied as loose source
onto whatever system Python a guest shipped broke every collector-fed dashboard.

| Milestone | Scope | Home |
|---|---|---|
| M1 | Componentize lab components into package-shaped source trees on a pinned runtime | logical-minds-foundry epic (from #293) |
| **M2** | **This epic**: generic binary packaging, publication and per-org package repositories | vergil-project |
| M3 | The lab installs published packages; `mqro` is revived as a real product | logical-minds-foundry follow-on |

M1 and M2 run in parallel; M3 needs both. Cross-org relationships are recorded ad
hoc (there is no epic-to-epic mechanism, and the framework forbids cross-org
linking).

### 2.2 Starting point (verified)

- **Nothing packaging-shaped exists.** No nFPM, fpm, `debian/`, `.spec`, or
  repository signing anywhere in vergil-tooling. `mq-resiliency-observability`'s
  README claims "Packaged as signed .rpm and .deb", but the repo has nothing
  behind that claim.
- **Release today:** `cd.yml` calls `vergil-actions/cd-release.yml@v2.1`, a single
  container job on `ubuntu-latest` (x86_64). It runs the evidence gate, then
  `registry-publish`, then `tag-and-release` (with `release-artifacts`), then the
  evidence attach. The CI-evidence tarball already carries a build-provenance
  attestation (`lib/ci_evidence.py:90`).
- **`vrg-release` confirm:** `confirm.py:189` requires the CD job named
  `release`; any other failed CD job is collected as a deferred publish failure
  (`confirm.py:215`).
- **`vergil.toml`** has no `[package]` section. Unknown keys only warn
  (`config.py:564`).
- **How vergil-tooling is installed today:** `uv tool install` from a git tag in
  three places: VM guests (`lib/vm_guest.py:24`), cached dev-container images
  (`lib/container_cache.py:354`), and the macOS host.

## 3. Decisions

Each decision is labeled **data** (a checked source) or **judgment** (reasoning
agreed in the brainstorm).

| # | Decision | Basis |
|---|---|---|
| D1 | **Hosting v1: GitHub Pages**, one public `<org>/packages` repo per org, at `https://<org>.github.io/packages/`. Self-hosted object storage ("A") comes later, together with a custom domain once the business entity exists. | Judgment: use what we already pay for, at no new cost, and add no lock-in (see the D2 guardrails). |
| D2 | **Guardrails that keep A a reconfiguration:** (a) the package repository is a host-agnostic static tree signed with *our* key; (b) package files are kept as **GitHub Release assets** of each product, and the index is a derived, rebuildable view; (c) consumers reference the repository only through a **repo-config package**. Package bytes never live in git. | Judgment. |
| D3 | GitHub Packages is **not** an option. | Data: it supports only npm, RubyGems, Maven, Gradle, NuGet and Docker ([docs](https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages)). |
| D4 | Pages limits shape retention and the size guard (§7.3). | Data: published site ≤ 1 GB; soft limit of 100 GB/month bandwidth; 10-minute deploy timeout; not for commercial hosting ([limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)). |
| D5 | **Packaging tool: nFPM.** A **builder** produces a staged install tree; a language-neutral step wraps it per format. v1 ships two builders: `python` (venv products) and `staged` (a repo-declared build command and/or overlay files, which covers the runtime package, the keyring package and, later, C++). | Data: nFPM produces deb, rpm and more from one config, as a single binary with no dpkg/rpmbuild dependency, with PGP signing and per-packager `overrides` ([nfpm](https://nfpm.goreleaser.com/), [config](https://nfpm.goreleaser.com/docs/configuration/)). Judgment: this fits the self-contained `/opt` product shape. |
| D6 | **M2 implements the builder interface and the Python builder in vergil-tooling; M1 adopts it.** M1 decides layout and recipe rules for its components, expressed as builder configuration. | Judgment: the builder is generic tooling, and LMF depends on vergil-project, not the other way round. |
| D7 | **The runtime is a side-by-side package per CPython patch** (`vergil-python3.14.N`) from a new `vergil-python` repo. Apps pin the CPython patch exactly and float on python-build-standalone (PBS) rebuilds of that patch. | Judgment: the tested interpreter is the deployed interpreter, and bundled-library security fixes need no product rebuild. Data: PBS Linux builds need glibc ≥ 2.17 ([PBS docs](https://gregoryszorc.com/docs/python-build-standalone/main/running.html)). |
| D8 | **Default target matrix: the full 2×2**: Ubuntu 24.04/26.04 and RHEL 9/10, each on amd64 and arm64. Per-repo `exclude` or `targets` override; a `native` escape hatch provides per-OS builds. | Data: RHEL 9 and 10 support 64-bit ARM ([RHEL 9](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/interactively_installing_rhel_from_installation_media/system-requirements-and-supported-architectures_rhel-installer), [RHEL 10](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/interactively_installing_rhel_from_installation_media/system-requirements-and-supported-architectures)), and aarch64 ISOs are on the no-cost Developer subscription. The x86-only constraint the lab hit is **IBM MQ RDQM**, not RHEL ([MQ 9.4 requirements](https://www.ibm.com/support/pages/system-requirements-ibm-mq-94)). |
| D9 | **Build per architecture, test per OS.** By default one `.deb` serves both Ubuntu releases and one `.rpm` serves both RHEL releases. Changing that is configuration, not re-architecture. | Judgment, with a hard requirement from the human. |
| D10 | **Arm64 builds run on free GitHub-hosted arm64 runners.** | Data: `ubuntu-24.04-arm` and friends are standard runners, "free and unlimited on public repositories" ([runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)). |
| D11 | **Signing: one OpenPGP identity per org.** The offline primary is held by the human; CI holds only a signing subkey, in a **`main`-only protected environment**. It signs every `.rpm`, apt `InRelease`/`Release.gpg`, and dnf `repomd.xml`. Sigstore build-provenance attestations are an additive layer, never the trust root, and the index verifies them pinned to the release workflow on `main`. | Judgment; hardened in pushback (§7.4). |
| D12 | **Pipeline topology:** a product's release attaches its signed packages to its own GitHub Release, then dispatches to `<org>/packages`. That repo rebuilds and signs the **whole** index. | Judgment: no cross-repo writes, no index races, publishing is idempotent. |
| D13 | **Gating:** package build and install test gate PRs; packaging failures in CD are fatal before tagging; index publishing is deferred and retryable. | Judgment. |
| D14 | **Dogfood:** vergil-tooling is published as a package. Lima and cloud VMs install it with `apt`, pinned to the major.minor line. The macOS host and the dev containers stay on `uv` **permanently, by design**. | Judgment, from the human: containers are a dynamic pipeline development tool. |

## 4. Architecture

### 4.1 Components

| Component | Home | Role |
|---|---|---|
| Packaging tooling | vergil-tooling: `vrg-package` CLI + `lib/package/` | Target registry and matrix resolution, the builder interface + the `python` and `staged` builders, nFPM packaging, install tests, index generation. |
| Reusable workflows | vergil-actions | `ci-package.yml` (PR gate), the split `cd-release.yml` (package build → sign/attest/attach → dispatch), `publish-index.yml`. |
| Per-org package repository | new public `<org>/packages` git repo + Pages | A **normal released repo** (`VERSION`, `cd.yml`, `[package]` with `builder = "staged"`) whose one product is the keyring package. It also holds the index config (products to index, retention, public key). Its `publish-index` workflow builds, signs and deploys the index. Package bytes never live in it. |
| Repo-config package | built and released by `<org>/packages` | `<org>-archive-keyring`: public key + `sources.list.d` / `yum.repos.d` entry. The only place a consumer references the repository URL. |
| Pinned Python runtime | new `vergil-project/vergil-python` repo | Repackages a verified PBS build as `vergil-python3.14.N`. Also documents the **runtime-package pattern**, which applies to any language that needs a versioned interpreter or VM (Ruby, Perl, the JVM) and not to languages that compile to native binaries. |
| vergil-tooling package | vergil-tooling itself | `/opt/vergil/vergil-tooling/` venv on the exact runtime, with `vrg-*` shims in `/usr/bin`. |

### 4.2 Data flow for one release

```text
product repo (main)                              <org>/packages
───────────────────                              ──────────────
package-build ×N build cells
  builder → staged tree → nFPM (unsigned)
package-sign + release (fatal before tag):
  rpmsign, attest all, tag,          ──dispatch──►  publish-index (deferred, idempotent):
  attach to GitHub Release                            collect stable releases of configured products
                                                      verify attestations + rpm signatures
                                                      retention + dependency closure
                                                      apt + dnf metadata, sign, size guard
                                                      deploy Pages
```

### 4.3 Cross-org

Each org indexes **only its own products**. An LMF host enables both
`vergil-archive-keyring` and `lmf-archive-keyring`, and LMF Python products depend
on `vergil-python3.14.N` from the vergil repository. Standing up
`logical-minds-foundry/packages` (key ceremony, repo, keyring package) is filed
as a task in **LMF's own epic structure**. This epic supplies org-agnostic
tooling, so that stand-up is configuration plus a key ceremony.

## 5. Configuration and the target matrix

### 5.1 Target registry

Supported targets are a registry in vergil-tooling code, in the style of
`lib/languages.py`. Each target is `<distro>/<version>/<arch>`; the distro implies
the format, and the registry records each target's **glibc version** (used by the
§6.1 guard), its install-test container image, and its apt suite or EL release.

```text
ubuntu/24.04/{amd64,arm64}   ubuntu/26.04/{amd64,arm64}   → .deb
rhel/9/{amd64,arm64}         rhel/10/{amd64,arm64}        → .rpm
```

Adding a target, such as Ubuntu 28.04, is a registry change and a vergil-tooling
release. It needs no workflow change.

### 5.2 `vergil.toml`

```toml
[package]
builder = "python"            # v1: python | staged. Future: cmake
vendor  = "vergil"            # → /opt/<vendor>/<name>/ ; name defaults to the project name
summary = "Shared development tooling for Vergil-managed repositories"
smoke   = "vrg-whoami --mode" # the install test runs this after install

# Matrix: default = every registered target. At most one of:
exclude = ["rhel/*/arm64"]    # drop targets (glob)
# targets = ["ubuntu/*/*"]    # explicit subset replacing the default

native  = []                  # e.g. ["ubuntu/26.04/*"]: own build cell, inside that OS
# name    = "…"               # required for staged; python defaults to pyproject [project].name
# version = "3.14.4+20261001" # explicit package version; default: the repo VERSION
#                             # (needed when the package version isn't the repo's semver, e.g. vergil-python)
# noarch  = true              # architecture-independent (e.g. the keyring): built once, deb `all` / rpm `noarch`

[package.python]
runtime  = "3.14.4"           # exact vergil-python CPython patch to build against and depend on
# commands = ["vrg-git"]      # optional subset of [project.scripts] to shim; default: all

# For builder = "staged" instead of [package.python]:
# [package.staged]
# build-command = "packaging/build.sh"  # optional; populates $VRG_STAGING_ROOT for $VRG_TARGET_ARCH
```

Files beyond the builder's output (systemd units, `/etc` config, maintainer
scripts) go in an optional `packaging/nfpm.overlay.yaml`, merged over the
generated nFPM config. nFPM's schema is not re-encoded in TOML. The overlay
contract for systemd units is in §6.5.

### 5.3 Matrix resolution

`vrg-package matrix` turns the config into explicit JSON lists of **build cells**
and **test cells**; workflows consume only that JSON.

- Every non-`native` target on an architecture shares **one build cell**. That
  cell builds on Ubuntu 24.04 and emits every format its targets need.
- Each `native` target gets its own build cell, inside a container of that OS.
- Every target gets its own **test cell**.

With the defaults this gives 2 build cells, 4 artifacts and 8 test cells.

### 5.4 Validation

These are **hard errors** in `vrg-validate`, never warnings:

- an unknown target or glob that matches nothing;
- `exclude` and `targets` both set;
- a `native` pattern that matches no selected target;
- `builder = "python"` without `[package.python].runtime`;
- a missing `uv.lock` for the Python builder;
- `builder = "staged"` with neither a `build-command` nor an overlay that
  contributes files;
- a `[package.python]` section with `builder = "staged"`, or a
  `[package.staged]` section with `builder = "python"`;
- an overlay file that is not a YAML mapping, or that sets a key outside
  `contents`, `scripts`, `overrides`, `provides`, `conflicts`, `replaces`,
  `recommends`, `suggests` (identity keys such as `name` and `version` belong to
  the tooling).

`vrg-validate` checks the overlay's **shape** only: nFPM is not in the dev
container. nFPM's own config check runs at build time in CI.

At build time, a `staged` build whose staging root ends up empty, or whose
`build-command` exits non-zero, is also a hard error.

## 6. Builders and packages

### 6.1 The Python builder

These steps run as root in a clean container of the build cell's OS:

1. **Install the runtime as consumers do:** enable the vergil package repository
   and install `vergil-python3.14.N` with the cell OS's package manager (`apt` in
   the shared Ubuntu cell, `dnf` in a `native` RHEL cell). The venv is built
   against the deployed bytes.
2. **Build the venv at its final path** (venvs are not relocatable):
   `uv venv --python /opt/vergil/python/3.14.N/bin/python3.14 /opt/<vendor>/<name>/venv`.
3. **Install from the lock only:**
   `uv sync --frozen --no-dev --no-editable --compile-bytecode`.
4. **Apply the glibc floor guard:** scan every ELF object in the venv. If any
   requires a `GLIBC_x.y` symbol newer than the lowest glibc among the cell's
   targets, that is a hard error. This means an Ubuntu-24.04-built tree can never
   silently ship an `.rpm` that breaks on RHEL 9. A dependency that compiles
   native code from an sdist trips this guard in a shared cell; that is the
   signal to use `native` builds (pymqi in M3 is the expected first case).
5. **Add command shims:** `/usr/bin/<cmd>` → `venv/bin/<cmd>` for each command in
   `[project.scripts]`, or the `commands` subset.
6. **Hand off:** the staged tree goes to nFPM with
   `Depends: vergil-python3.14.N (>= <PBS build tested with>)` plus the overlay.

### 6.2 The runtime package (`vergil-python`)

- **Contents:** the PBS `install_only_stripped` build for one CPython patch and
  architecture, at `/opt/vergil/python/3.14.N/`.
- **Name and version:** the package name carries the CPython patch
  (`vergil-python3.14.4`) and the version carries the PBS build
  (`3.14.4+<pbs-tag>`). A PBS rebuild of the same patch is an in-place upgrade;
  a new CPython patch is a new package installed alongside.
- **Never on `PATH`:** nothing goes in `/usr/bin`, so it can never shadow the
  system `python3` that the OS and Ansible rely on.
- **PEP 668 `EXTERNALLY-MANAGED` marker:** nobody can `pip install` into the
  shared interpreter.
- **Provenance:** the repo pins the PBS release tag and a SHA-256 per
  architecture; a mismatch fails the build. Adopting a new patch is a reviewed PR.
- **Built with the `staged` builder (§6.6):** its `build-command` fetches the
  pinned PBS archive for the cell's architecture, verifies the checksum, unpacks
  it into the staging root, and adds the `EXTERNALLY-MANAGED` marker.

### 6.3 Install layout (for the M1 cross-check)

```text
/opt/<vendor>/python/<X.Y.Z>/      runtime (vergil-python)
/opt/<vendor>/<name>/venv/         product venv
/usr/bin/<cmd>  → venv/bin/<cmd>   command shims
/etc/<vendor>/<name>/              config (overlay, noreplace)
/var/lib/<vendor>/<name>/          state
journald                           logs (systemd units via overlay)
```

**Directory ownership:** a package owns every directory **strictly below**
`/opt/<vendor>`, and never `/opt/<vendor>` itself, which is shared. dpkg and rpm
reference-count shared directories such as `/opt/vergil/python`, so removal
leaves nothing behind. That is what makes the install-test residue check
meaningful on rpm.

### 6.4 Versioning

A product's package version is its repo `VERSION` with package revision `1`
(e.g. `2.1.240-1`). Only stable `vX.Y.Z` releases produce published packages;
`develop-*` tags never do.

### 6.5 Systemd units: the overlay contract

Install tests run in plain containers without systemd as PID 1 (§8.1). So:

- Overlay maintainer scripts that enable or start units **must** use the
  standard helpers, which are a no-op when systemd isn't running:
  `deb-systemd-helper`/`deb-systemd-invoke` on Ubuntu, and the
  `%systemd_post`/`%systemd_preun`-equivalent idioms on RHEL. A raw
  `systemctl enable --now` in a maintainer script is a hard error at package
  build time.
- Install-test verifies that each shipped unit file is installed and passes
  `systemd-analyze verify`.
- **"The service actually starts and works"** is not an M2 CI promise. It is
  proven in the lab (M3 validation). See §14.

### 6.6 The `staged` builder

This is the generic builder for anything that isn't a Python venv product. In
the build cell:

1. If `[package.staged].build-command` is set, run it with `VRG_STAGING_ROOT`
   (an empty directory standing in for `/`) and `VRG_TARGET_ARCH` set. It must
   populate the staging root, for example `opt/vergil/python/3.14.N/…`. A
   non-zero exit is a hard error.
2. Merge the overlay's files over the staging root.
3. Fail if the result is empty; otherwise hand off to nFPM.

v1 users: `vergil-python` (fetch, verify and unpack PBS) and
`<org>-archive-keyring` (overlay only: key + source entries). Future users: C++
(`cmake --install` into the staging root), and any pre-built tree.

## 7. Indexing, signing and publishing

### 7.1 The `<org>/packages` repository

```text
VERSION, vergil.toml, .github/workflows/{ci,cd}.yml
                     a normal released repo; [package] builder = "staged"
packages.toml        products to index + retention (default keep = 3, lines = 2)
keys/<org>.asc       public key (primary + current subkey)
packaging/           nFPM overlay for <org>-archive-keyring (this repo's own product)
```

The keyring is `noarch`. It installs the public key at
`/usr/share/keyrings/<org>-archive-keyring.asc` (plus
`/etc/pki/rpm-gpg/RPM-GPG-KEY-<org>` and a `$releasever`-based
`/etc/yum.repos.d/<org>.repo` on RHEL). One `.deb` serves both Ubuntu codenames,
so its **apt source is written by `postinst`** from `/etc/os-release`
(`VERSION_CODENAME`) and removed by `postrm`, not shipped as a static file.

### 7.2 The `publish-index` workflow

`publish-index` is a reusable workflow in vergil-actions; the logic lives in
`vrg-package index`.

- **Triggers:** `repository_dispatch` from a product release; `workflow_dispatch`;
  and a **weekly reconcile**, so a missed dispatch self-heals.
- **Collect:** stable releases of each configured product, plus their assets.
  Every packaged release attaches a `packages-manifest.json` listing its
  expected `(format, arch, suite)` artifacts. A release **without** a manifest
  is pre-packaging history and is ignored, with a printed note. A release whose
  manifest names an artifact that isn't attached is a **hard error**.
- **Verify before indexing:** every asset must pass `gh attestation verify`
  **pinned to the release path**: `--repo <product repo>`,
  `--signer-workflow vergil-project/vergil-actions/.github/workflows/cd-release.yml`,
  and `--source-ref refs/heads/main`. Every `.rpm` must also carry a valid org
  signature. A failure is a **hard error**, never a skip. The index never
  contains a package that our own release pipeline did not build on `main`.
- **Retain per line:** for each product, the newest `lines` major.minor lines
  (default 2), and within each line the latest `keep` releases (default 3),
  **plus the dependency closure** (any runtime a retained product depends on).
  So anything indexed is installable, and a VM pinned to the previous line stays
  installable until two newer lines exist.
- **Index:** apt metadata (`Packages`, `Release`) is generated by vergil-tooling
  in Python, from each `.deb`'s control data and hashes, which gives clean
  per-suite selection for `native` builds. dnf metadata comes from
  `createrepo_c`. Both are stateless, so the index is a pure function of the
  release assets.
- **Duplicates:** two artifacts with the same `(format, name, version-release,
  arch)` but different bytes are a **hard error** naming both releases; an
  identical duplicate is collapsed.
- **Sign:** `InRelease` + `Release.gpg`; `repomd.xml.asc`.
- **Size guard:** a loud warning above 750 MB and a **hard failure above
  900 MB**, before deploying. Crossing it is the explicit trigger for moving to A.
- **Deploy:** Actions-based Pages deploy.

### 7.3 Published layout

Consumers point at their own OS release from day one, so a later `native` split
changes nothing for them:

```text
deb/dists/<ubuntu-codename>/main/binary-{amd64,arm64}/   one suite per Ubuntu release
deb/pool/…                                               shared files
rpm/el{9,10}/{x86_64,aarch64}/repodata/                  per-release metadata
rpm/pool/…                                               shared files
```

**To verify during implementation:** how dnf metadata references a shared
`rpm/pool/` (relative `location href` versus `xml:base`). If that does not work
cleanly, the fallback is one copy of each `.rpm` per EL directory. That costs
bytes but changes no design.

### 7.4 Keys and secrets

- The signing subkey and its passphrase live in a GitHub **environment**,
  `package-signing`, in each participating repo: the `packages` repo (to sign
  metadata) and each product repo (for the `package-sign` job's `rpmsign`). The
  environment's deployment-branch policy allows **only `main`**.
- Only the `package-sign` job of a release (on `main`) declares that
  environment.
- `publish-index` runs on the `packages` repo's **default branch** (`develop`),
  because `repository_dispatch` and `schedule` always do. So the `packages` repo
  also has a second environment, **`index-signing`**, holding the same subkey
  and admitting **only `develop`**. `develop` is protected and changes only
  through human-merged PRs, so the property is the same: no unreviewed branch
  can read the key.
- A workflow on any other branch, including agent feature branches, cannot read
  the key. Org-level secrets are deliberately **not** used, because GitHub does
  not restrict them by branch.
- Build cells and PR CI never see the key.
- Generating the primary key, minting subkeys, creating the environments,
  loading secrets and rotating keys are **human-attested preconditions**; agents
  never handle the primary key.
- **Rotation:** mint a new subkey offline, update `keys/<org>.asc`, and release a
  new `<org>-archive-keyring`. Consumers trust the primary, so rotation flows
  through ordinary upgrades.

## 8. CI/CD integration

### 8.1 PR CI: `ci-package.yml`

| Job | Behavior |
|---|---|
| `matrix` | `vrg-package matrix` → build cells and test cells. |
| `build` | Per build cell, on `ubuntu-24.04` / `ubuntu-24.04-arm`, inside the build OS container: builder → nFPM → **unsigned** artifacts. |
| `install-test` | Per test cell, in a clean `ubuntu:24.04`, `ubuntu:26.04`, UBI 9 or UBI 10 container on the matching arch. For `python`-builder products, first enable the vergil repository as a consumer would (the runtime resolves from the **live** repository); `staged` products install standalone, since the keyring can't depend on its own index. Then: install the artifact; verify any shipped systemd units are installed and pass `systemd-analyze verify` (§6.5); run `smoke` in a **sanitized environment** (`env -i` with a system-only `PATH`), so only the packaged binaries resolve and never the `uv` copy of vergil-tooling that drives the test; check that each shim resolves to `/usr/bin/<cmd>`; uninstall; assert nothing remains under `/opt/<vendor>/<name>` or in the shims. |
| `package / evidence` | One stable, version-agnostic gate (`ci-evidence-package`). Like the other evidence gates, it is **required only where it applies**: the per-repo ruleset computation adds `package / evidence` to repos whose `vergil.toml` has `[package]`, and the release-time harvester learns the `package` gate. A repo without `[package]` does not call `ci-package`. |

Raw `ubuntu`/UBI images ship neither Python nor vergil-tooling, so every build
and test container first runs a shared setup action that installs OS
prerequisites, a pinned `uv`, vergil-tooling (from the checkout in
vergil-tooling itself, otherwise from `[dependencies].vergil`) and, in build
cells, a checksum-pinned nFPM.

Local `vrg-validate` validates the packaging **config** only (§5.4); there are no
container-in-container builds.

### 8.2 CD: the `cd-release.yml` split

1. `package-matrix` resolves the cells and writes `packages-manifest.json`;
   `package-build` runs the same build cells as PR CI.
2. A separate **`package-sign`** job declares the `package-signing` environment
   (§7.4), runs `rpmsign` on every `.rpm`, verifies the signatures, and attests
   every artifact. It is a separate job so that repos without `[package]` never
   reference (and so never auto-create) the environment. The `release` job runs
   only if `package-sign` succeeded (or packaging is disabled), so a failed
   build or signing can never let `release` run. `release` attaches the signed
   packages and the manifest to the release. Install-test evidence joins the
   CI-evidence bundle. Any failure is **fatal before tagging**.
3. After `tag-and-release`, a `repository_dispatch` to `<org>/packages` uses the
   org's GitHub App token (`create-github-app-token`), scoped to the `packages`
   repo. A dispatch failure is deferred.

Repos without `[package]` skip the package jobs entirely and see no behavior
change.

### 8.3 `vrg-release`

- The orchestrator gains a deferred **`package-index`** stage. It polls the
  published index until the new version appears (with a timeout) and reports a
  miss as a deferred publish failure.
- The post-release consumer refresh **waits on that stage**, so
  `vrg-vm update --all` never races the index.

## 9. Consumer side: VM provisioning

`uv tool install` is **replaced**, not kept as a fallback, for Lima and cloud
VMs (`lib/vm_guest.py`).

**Version source.** The version comes from the VM's **resolved identity
version**, `resolve_vergil_version()` (`lib/identity.py:229`). That is the
per-identity `vergil` setting in `identities.toml`, falling back to the
config-level one. It does **not** come from the repo's `vergil.toml`. A
`vrg-vm update --tag` value overrides it, exactly as today.

**Packaged install** (the default, for any release version):

1. **Bootstrap trust:** fetch `keys/<org>.asc` from the Pages site and verify its
   primary-key fingerprint against a value **pinned in vergil-tooling's code**. A
   mismatch fails provisioning loudly. Then write the apt source.
2. `apt install vergil-archive-keyring`. The package then owns the key and the
   source entry.
3. **Pin:** a line version `vX.Y` becomes an `/etc/apt/preferences.d` pin to
   `X.Y.*`, followed by `apt install vergil-tooling`. An exact version `vX.Y.Z`
   becomes `apt install vergil-tooling=X.Y.Z-1`, with a matching exact pin.
4. `vrg-vm update` becomes
   `apt-get update && apt-get install --only-upgrade vergil-tooling`, within the
   pin.

**Explicit dev install** (only for a non-release ref, such as `develop` or a
feature branch, passed via `--tag`):

- This is a deliberate mode chosen by the argument, never a fallback after a
  packaged-install failure. It does `uv tool install vergil-tooling @ git+…@<ref>`
  into a separate, clearly named location that takes precedence on the VM user's
  `PATH`.
- Every `vrg-vm` command that touches the VM reports
  `DEV tooling (ref <ref>) — not the packaged install`.
- Running `vrg-vm update` with no `--tag` removes the dev install and returns
  the VM to the packaged install.

Unchanged: the macOS host (`uv tool install`), the dev-container cache
(`uv tool install` from git), and this repo's dev-tree `.venv` override.

## 10. Error handling

Every failure mode in §5–§9 is a hard, loud error. The single deliberate
exception is **index publishing**: deferred, reported by `vrg-release`, and
self-healing through re-runs and the weekly reconcile, because the index is a
derived view of release assets that already exist. Nothing is silently skipped
or swallowed.

## 11. Testing

- **Unit tests** (vergil-tooling, 100% coverage bar): target registry and matrix
  resolution; every §5.4 config error; the glibc guard against fixture ELF files;
  the `staged` builder (command contract, empty-root failure); the raw-`systemctl`
  maintainer-script check; per-line retention with dependency closure; pinned
  attestation-verification arguments; index generation from fixture packages;
  nFPM config generation and overlay merging; fingerprint-pinned bootstrap logic;
  VM version mapping (`vX.Y` → line pin, `vX.Y.Z` → exact, non-release ref → dev
  install).
- **Integration tests:** the PR install-test matrix itself. Every packaging PR
  proves install, smoke and clean removal on every target cell.
- **Live proof:** the deployment and validation operational tasks (§12).

## 12. Delivery shape

The pipeline is bootstrapped by itself, so the order is:

1. Trust roots (human).
2. The tooling and workflows.
3. `vergil-python`, since every Python product depends on it.
4. The `vergil-project/packages` repo and keyring.
5. vergil-tooling as a package.
6. The VM switch.

The plan sequences the tasks; the operational tasks are:

- **Precondition (human-attested):** the vergil org key ceremony;
  `package-signing` environments (main-only) holding the subkey in each
  participating repo, plus the `index-signing` environment (develop-only) in the
  `packages` repo; `vergil-project/packages` created with Pages enabled; and
  the GitHub App permitted to dispatch to it.
- **Deployment:** the first packaged vergil-tooling release (a human-gated
  release) is indexed, then `vrg-vm update --all` runs across existing VMs.
- **Validation (cold rebuild):** a fresh Lima arm64 VM and a fresh cloud amd64 VM
  provision on packaged tooling; clean `dnf install` and smoke on RHEL 9 and 10,
  amd64 and arm64.

Bookends: documentation (`.github#357`), documentation review
(`vergil-tooling#3072`), the macOS follow-on brainstorm (`.github#358`), and the
retrospective (`.github#359`).

## 13. Out of scope

- **macOS as an install target.** Follow-on brainstorm `.github#358`.
- **Debian proper.** Revisit on a customer case.
- **A custom domain and self-hosted storage (A).** These come with the business
  entity's domain; the §7.2 size guard is the technical trigger.
- **Dev containers and the macOS host on packages.** Permanently out, by design
  (D14).
- **LMF products' packages** (M3), and the LMF `packages` stand-up (an LMF-side
  task, §4.3).
- **Distro-orthodox `debian/` / `.spec` builds.** These can be added per product
  later without disturbing the design.

## 14. M1 cross-check items

M1 (`logical-minds-foundry/.github#293`) is still a seed. These M2 choices are
**proposals** that M1's brainstorm must confirm or correct, through builder
configuration rather than a second implementation:

- the install layout (§6.3), including the `<vendor>` naming for LMF (`lmf`);
- the runtime pin model (D7): the CPython patch is exact, PBS rebuilds float;
- lock discipline: build from `uv.lock` only (`--frozen`);
- the recipe for building the venv from source and lock (§6.1), run in CI;
- where systemd units and config land (overlay), and the smoke-command contract;
- **service-start verification:** M2's CI proves only that units are installed
  and valid (§6.5); proving that services start and work belongs in the lab
  (M3). M1 should confirm that its daemons fit the helper-macro contract;
- native dependencies (pymqi): `native` build cells and their interaction with
  MQ SDK availability per target.
