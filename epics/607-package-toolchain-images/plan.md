# Package toolchain images — Implementation Plan

> **For epic workers:** These tasks are filed as **GitHub issues under epic
> vergil-project/.github#607** and implemented one PR each via
> `vergil:issue-implement` (which runs its own TDD cycle). Operational tasks
> (`deployment`, `validation`) run via `vergil:issue-deploy` /
> `vergil:issue-validate`. Steps use checkbox (`- [ ]`) syntax.

**Goal:** CI package jobs run in managed toolchain images built by
vergil-containers, so no package job installs its toolchain at run time.

**Architecture:** vergil-containers gains a `toolchains:` manifest section, kept
out of the language model, and two templates, `pkg-ubuntu` and `pkg-rhel`. The
generator emits them into the same `matrix.json` that the publish, the nightly
and epic #213's work consume. The vergil-actions setup action recognises a
toolchain image by a marker file and skips its installs. vergil-tooling's target
registry then names the `prod-pkg-*` images. A deployment step makes the new
packages public before vergil-tooling switches over.

**Tech Stack:** Python 3 (stdlib + `pyyaml`) and Dockerfile templates in
vergil-containers; Bash composite actions in vergil-actions; Python
(vergil-tooling `lib/package/`).

**Spec:** `epics/607-package-toolchain-images/spec.md`

## Global Constraints

- **Image contents:** the upstream image (`ubuntu:<v>` or
  `registry.access.redhat.com/ubi<v>/ubi`), plus the OS upgrade, plus exactly:
  - prerequisites `ca-certificates curl git tar gzip gnupg` (apt) or
    `ca-certificates /usr/bin/curl /usr/bin/gpg git tar gzip` (dnf);
  - uv and nFPM in `/usr/local/bin`;
  - on amd64 `pkg-ubuntu` only, the apt mirror list and
    `/etc/apt/apt.conf.d/80-vergil-fast-fail`;
  - the marker `/etc/vergil/package-toolchain`.

  Nothing else: no interpreter, no vergil-tooling.
- **Package lists are removed** (`rm -rf /var/lib/apt/lists/*`; `dnf clean all`).
- **Image names:** `ghcr.io/vergil-project/{dev,prod}-pkg-ubuntu:{24.04,26.04}`
  and `…-pkg-rhel:{9,10}`. Package jobs use `prod-`.
- **Not languages:** toolchain images never appear in `window.json`, the
  lifecycle check or language resolution (`container.py`).
- **The driver stays dynamic:** vergil-tooling is installed by `uv tool install`
  at run time, from the checkout in vergil-tooling itself, otherwise from the
  pinned release.
- **No silent fallback:** a marker present with a tool missing or at another
  version fails with `::error::`. A marker absent means today's raw-image path.
- **Generated files stay generated:** edit `generate.py` and regenerate; never
  hand-edit `matrix.json` or a `BEGIN generated` block.
- **Workflow pushes are human:** a delta touching `.github/workflows/**` makes
  the agent push fail. `report-ready` records the state anyway.
- **Validation:** `vrg-container-run -- vrg-validate`, from inside the worktree.

## Review Focus

1. **A lifecycle PR rewrites `manifest.yml` through `dump_manifest`.** A
   `toolchains:` section that doesn't round-trip would be silently deleted by
   the next nightly add/drop PR. Task 3 pins the round trip.
2. **The amd64 Azure mirror must not reach arm64.** `ports.ubuntu.com` serves
   arm64, and the Azure mirror does not, so a mirror rewrite on arm64 breaks
   apt. Task 4's build-time check asserts the sources per architecture.
3. **UBI ships `curl-minimal`,** which conflicts with the `curl` package. The
   dnf install must ask for `/usr/bin/curl`, as `os-prereqs.sh` does. Task 4's
   build-time check runs `curl --version` on `pkg-rhel`.
4. **A marker that lies.** An image with a marker but a missing or different
   uv or nFPM must fail the job, never fall through to installing. Task 1's
   harness covers missing, wrong-version and `nfpm: "true"`-without-nFPM.
5. **A new target with no image.** A target added to `targets.py` must not
   keep or invent a raw image silently. Task 5 derives every image from the
   distro and version, and asserts it for every registry entry.

## Task dependency graph

```text
#1087 (ad-hoc, py 3.15) ─→ T2 (driver Python, one input)
T1 (setup marker) ──────────────────────┐
T3 (manifest toolchains) → T4 (images) ─┼→ D1 (deploy: releases + public packages) → T5 (tooling switch) → D2 (tooling release) → T6 (remove raw path)
T2 ─────────────────────────────────────┘                                              T5 → V1 (validation)
Docs review (vergil-tooling#3758) → Retrospective (.github#609)
```

T1, T2 and T3 can start in parallel.

---

### Task 1 (vergil-actions): marker-aware package setup

**Files:**

- Create: `actions/package/setup/check-toolchain.sh`
- Create: `actions/package/setup/tests/check-toolchain.test.sh`
- Modify: `actions/package/setup/action.yml`
- Modify: `.github/workflows/smoke-setup-vergil.yml` (run the new harness
  beside `os-prereqs.test.sh`)

**Interfaces:**

- Produces the marker contract, which Task 4 writes: `/etc/vergil/package-toolchain`
  holds one `<tool> <version>` line per tool, where `<tool>` is a command name
  and `<version>` is the first `[0-9]+(\.[0-9]+)+` token of `<tool> --version`.
  It always lists `uv`, `nfpm`, `curl`, `git`, `gpg`, `tar` and `gzip`.
- `check-toolchain.sh [--nfpm]` exits 0 when every listed tool is on `PATH` at
  its listed version, and when `nfpm` is listed if `--nfpm` is given. Otherwise
  it prints one `::error::` per problem and exits 1. Test seam: `VRG_TOOLCHAIN_MARKER`
  (default `/etc/vergil/package-toolchain`).
- `action.yml` adds a first step with `id: toolchain` that sets
  `managed=true|false` from `[ -f /etc/vergil/package-toolchain ]`. The OS
  prerequisites, uv and nFPM steps gain `if: steps.toolchain.outputs.managed != 'true'`.
  A new step, `if: steps.toolchain.outputs.managed == 'true'`, runs
  `check-toolchain.sh` (passing `--nfpm` when `inputs.nfpm == 'true'`) and
  `git config --global --add safe.directory "$GITHUB_WORKSPACE"` (moved out of
  the skipped prerequisites step).

- [ ] **Step 1: Write the harness** in the framework-free style of
  `tests/os-prereqs.test.sh` (stub tools first on `PATH`, fixture marker via
  `VRG_TOOLCHAIN_MARKER`). Cases:
  - `all_present_matching`: marker `uv 0.12.23` and `nfpm 2.48.0`, stubs print
    those versions → exit 0.
  - `missing_tool`: marker lists `nfpm`, no stub → exit 1, stderr has
    `::error::` and `nfpm`.
  - `version_mismatch`: marker `uv 0.12.23`, stub prints `uv 0.12.24` → exit 1.
  - `nfpm_required_not_listed`: `--nfpm`, marker without `nfpm` → exit 1.
  - `no_nfpm_flag_ignores_absence`: no `--nfpm`, marker without `nfpm` → exit 0.
  - `empty_marker`: zero lines → exit 1 (an empty marker is a broken image).
- [ ] **Step 2: Run** `bash actions/package/setup/tests/check-toolchain.test.sh`
  and expect failures (script absent).
- [ ] **Step 3: Implement `check-toolchain.sh`** (`set -euo pipefail`,
  shellcheck-clean; report every problem before exiting).
- [ ] **Step 4: Run the harness and expect all cases to pass.**
- [ ] **Step 5: Wire `action.yml`** as in Interfaces. Leave the raw-image
  path's step bodies byte-for-byte unchanged.
- [ ] **Step 6: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope package --message "skip the toolchain install in a managed package toolchain image"`.
  The workflow file edit means the human pushes.

---

### Task 2 (vergil-actions): the driver Python comes from one input

Blocked-by vergil-project/vergil-actions#1087 (the 3.15 bump).

**Files:**

- Modify: `actions/package/setup/action.yml`, `actions/package/setup/detect.sh`,
  `actions/package/setup/tests/detect.test.sh`

**Interfaces:**

- `action.yml` input `python` (default `"3.15"`), described as the Python
  version for the driver vergil-tooling. The **only** place the version is
  written.
- The install step uses `uv tool install --python "${{ inputs.python }}"`.
- `detect.sh` reads the version from the environment variable
  `VRG_DRIVER_PYTHON`, set by the step from `inputs.python`. When it is unset or
  empty, it exits 1 with `detect.sh: VRG_DRIVER_PYTHON must be set`; there is no
  default.

Decision recorded here (spec §5.3): the driver follows a **single constant**,
not vergil-tooling's derived package runtime. In a consumer repo, the derived
runtime is the consumer's product runtime, and install-test and staged cells
carry none. The constant lives in one place, and `vrg-update-deps` already
bumps `action.yml` refs.

- [ ] **Step 1: Add `detect.test.sh` cases:** `driver_python_unset_fails`
  (exit 1, message as above), and `driver_python_used`: a `uv` stub records its
  argv, and with `VRG_DRIVER_PYTHON=3.15` the recorded argv contains
  `--python 3.15`.
- [ ] **Step 2: Run and expect them to fail.**
- [ ] **Step 3: Implement** the input, the env pass-through and `detect.sh`'s
  check.
- [ ] **Step 4: Run every setup harness (`detect`, `os-prereqs`,
  `check-toolchain`) and expect a pass.** Then `rg -n -- '--python 3\.' actions/package`
  must print nothing.
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type refactor --scope package --message "take the driver Python from one action input"`.

---

### Task 3 (vergil-containers): a `toolchains:` manifest section

**Files:**

- Modify: `docker/versions/manifest.py`, `docker/versions/generate.py`
- Modify: `docker/versions/test_manifest.py`, `docker/versions/test_generate.py`

**Interfaces:**

- `manifest.py`:
  - `@dataclass(frozen=True) class Toolchain: key: str; build_arg: str; versions: tuple[str, ...]`,
    with versions newest first;
  - `Manifest` gains `toolchains: tuple[Toolchain, ...] = ()`;
  - `_TOP_KEYS` gains `"toolchains"` (optional); `_TOOLCHAIN_KEYS = {"key", "build_arg", "versions"}`.

  Validation reuses `_KEY_RE`, `_BUILD_ARG_RE`, `version_sort_key` and the
  newest-first and duplicate rules. A toolchain key equal to a family key or
  `base` is a `ManifestError`. `dump_manifest` emits a `toolchains:` block
  after `families:` when the section is non-empty.
- `generate.py`:
  - `render_matrix_json` appends `{"language": t.key, "version": v, "build-arg": t.build_arg}`
    for each toolchain version, after the families;
  - `render_build_sh` emits `build <key> <build_arg> <v>` lines for them;
  - `render_package_cleanup` lists their keys;
  - `render_arg_default` applies to their templates;
  - `render_doc_table` adds one row each, with `—` as the language;
  - `render_window_json` is unchanged: families only.

- [ ] **Step 1: Write failing tests** on `tmp_path` manifests (the frozen seed
  stays toolchain-free):
  - `test_parses_toolchains_section`, and `test_toolchains_optional` (a
    manifest without the key loads with `toolchains == ()`);
  - `test_rejects_toolchain_key_colliding_with_family_or_base`;
  - `test_rejects_toolchain_versions_not_newest_first` (`["24.04", "26.04"]`);
  - `test_dump_round_trips_toolchains`: `parse_manifest(yaml.safe_load(dump_manifest(m))) == m`
    for a manifest with toolchains *(Review Focus 1)*;
  - `test_matrix_json_appends_toolchain_entries`: a `pkg-ubuntu` with
    `["26.04", "24.04"]` and build-arg `UBUNTU_VERSION` gives those two
    entries, last, in that order;
  - `test_window_json_excludes_toolchains`: no `pkg-ubuntu` anywhere in the
    rendered window;
  - `test_package_cleanup_includes_toolchains`.
- [ ] **Step 2: Run them and expect failure.**
- [ ] **Step 3: Implement** in the existing module style (no language
  branches; errors name the toolchain and field).
- [ ] **Step 4: Run the tests and expect a pass. Then
  `python3 docker/versions/generate.py --check` passes on the live tree**
  (unchanged, since no toolchains are declared yet).
- [ ] **Step 5: Coordinate with epic #213.** Comment on
  vergil-project/vergil-containers#510 (T2, the `@include` dependency map):
  `dependency_map` should include `manifest.toolchains` keys like families.
  Until it does, T5's fail-closed rule selects the full set for their paths.
- [ ] **Step 6: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope versions --message "add a toolchains section for non-language images"`.

---

### Task 4 (vergil-containers): the `pkg-ubuntu` and `pkg-rhel` images

Blocked-by Task 3.

**Files:**

- Create: `docker/pkg-ubuntu/Dockerfile.template`, `docker/pkg-rhel/Dockerfile.template`
- Create: `docker/common/uv.dockerfile` (curl and `sh` only, so both families
  can include it; installs to `/usr/local/bin` the way
  `common/python-support.dockerfile` does, leading edge)
- Create: `docker/common/apt-mirror.dockerfile` (amd64-only mirror list,
  sources rewrite and fast-fail config, matching
  vergil-actions `actions/package/setup/os-prereqs.sh` exactly: the same three
  hosts, priorities, http scheme, and `Acquire::*` and `DPkg::Lock` values;
  gated on `TARGETARCH`)
- Create: `docker/common/package-toolchain-marker.dockerfile` (writes
  `/etc/vergil/package-toolchain` from `<tool> --version`, per Task 1's
  contract)
- Modify: `docker/versions/manifest.yml` (add `toolchains:`: `pkg-ubuntu`
  `UBUNTU_VERSION` `["26.04", "24.04"]`; `pkg-rhel` `RHEL_VERSION` `["10", "9"]`),
  then regenerate
- Modify: `docker/generate.sh` (add `pkg-ubuntu pkg-rhel` to the default list)

**Interfaces:**

- Consumes: Task 3's `toolchains:` schema; Task 1's marker contract.
- Produces: the images, published by the existing `cd-docker-publish.yml`
  with no workflow change.

`pkg-ubuntu`: `FROM ubuntu:${UBUNTU_VERSION}`. Includes `apt-mirror` first,
then one `apt-get update && apt-get upgrade -y && apt-get install -y --no-install-recommends ca-certificates curl git tar gzip gnupg && rm -rf /var/lib/apt/lists/*`,
then `uv`, `nfpm`, and the marker last. `pkg-rhel`: `FROM registry.access.redhat.com/ubi${RHEL_VERSION}/ubi`.
One `dnf upgrade -y` and `dnf install -y --setopt=install_weak_deps=False ca-certificates /usr/bin/curl /usr/bin/gpg git tar gzip && dnf clean all`,
then `uv`, `nfpm`, and the marker. Each template ends with a build-time check
`RUN` that fails the build:

- on `pkg-ubuntu` amd64, `/etc/apt/sources.list.d/ubuntu.sources` contains
  `mirror+file:/etc/apt/apt-mirrors.txt`; on arm64 it does not, and
  `/etc/apt/apt-mirrors.txt` is absent *(Review Focus 2)*;
- `curl --version`, `git --version`, `gpg --version`, `uv --version` and
  `nfpm --version` all succeed *(Review Focus 3)*;
- the marker has 7 lines, `uv` and `nfpm` among them;
- no `python3` is on `PATH`, and `/var/lib/apt/lists` holds no `*_Packages*`.

- [ ] **Step 1: Write the build-time check `RUN`s first** in each template,
  then the installs.
- [ ] **Step 2: Build locally, both arches if possible:**
  `docker/build.sh pkg-ubuntu` and `docker/build.sh pkg-rhel` (as the
  regenerated `build.sh` block defines them). Every check passes on amd64. The
  arm64 build runs under QEMU until #213 T7 lands; record its time.
- [ ] **Step 3: Regenerate and check:** `python3 docker/versions/generate.py`,
  then `--check` passes. `docker/generate.sh` succeeds and hadolint is clean
  (via validation).
- [ ] **Step 4: Pins:** nFPM rides the existing auto-managed
  `common/nfpm.dockerfile` entry; uv floats like `python-support`, so it needs
  no `pins.yml` entry. `check_pins.py` passes (via validation).
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope images --message "add pkg-ubuntu and pkg-rhel package toolchain images"`.

---

### Task D1 (deployment, vergil-containers): release and publish the toolchain images

`--kind deployment`, Blocked-by Task 1, Task 2 and Task 4.

**Preconditions (human-attested):** vergil-actions released with Task 1 and
Task 2 (the `v2.1` tag moved); vergil-containers released with Task 4
(`prod-` publish ran).

**Procedure:**

1. Confirm `cd-docker-publish` published all eight tags:
   `{dev,prod}-pkg-ubuntu:{24.04,26.04}` and `{dev,prod}-pkg-rhel:{9,10}`.
   Each is a two-platform index with an attestation.
2. **Human:** set `dev-pkg-ubuntu`, `dev-pkg-rhel`, `prod-pkg-ubuntu` and
   `prod-pkg-rhel` to **Public**, and confirm vergil-containers has **Write**
   under Manage Actions access (vergil-containers
   `docs/plans/2026-05-09-release-workflow-and-image-naming.md`, Step 3).
3. For each `prod-` package,
   `curl -s "https://ghcr.io/token?scope=repository:vergil-project/<package>:pull"`
   returns a `token`, not `DENIED`.
4. `docker run --rm ghcr.io/vergil-project/prod-pkg-ubuntu:24.04 cat /etc/vergil/package-toolchain`
   prints the seven tools.

**Acceptance:** every step passes. Record `Outcome: SUCCESS`.

---

### Task 5 (vergil-tooling): package jobs use the toolchain images

Blocked-by D1.

**Files:**

- Modify: `src/vergil_tooling/lib/package/targets.py`,
  `src/vergil_tooling/lib/package/matrix.py`
- Modify: `tests/vergil_tooling/test_package_targets.py`,
  `tests/vergil_tooling/test_package_matrix.py`
- Modify: `docs/site/docs/reference/package-config.md` (the Targets table's
  install-test image column and the "builds in `ubuntu:24.04`" wording)

**Interfaces:**

- `targets.py`: `TOOLCHAIN_REGISTRY = "ghcr.io/vergil-project"`;
  `Target.image` becomes a property, `f"{TOOLCHAIN_REGISTRY}/prod-pkg-{self.distro}:{self.version}"`.
  The `image` column leaves `_UBUNTU` / `_RHEL`, so the rows carry (version,
  suite, glibc).
- `matrix.py`: `SHARED_IMAGE = tg.REGISTRY["ubuntu/24.04/amd64"].image`,
  derived, not a literal.

- [ ] **Step 1: Update and add tests:**
  - `test_format_suite_and_rpm_arch` expects
    `"ghcr.io/vergil-project/prod-pkg-ubuntu:24.04"`;
  - `test_every_target_image_is_its_toolchain_image`: for every
    `REGISTRY` value, `t.image == f"ghcr.io/vergil-project/prod-pkg-{t.distro}:{t.version}"`
    *(Review Focus 5)*;
  - `test_shared_build_image_is_pkg_ubuntu_2404`;
  - the existing matrix tests' expected `image` values change accordingly.
- [ ] **Step 2: Run and expect failure.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run and expect a pass. Update `package-config.md`.**
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope package --message "run package jobs in the managed toolchain images"`.
  This PR's own CI is the end-to-end proof: vergil-tooling installs itself
  from the checkout, so its package gate runs in the new images. The PR body
  records each package job's "Set up package toolchain" time.

---

### Task D2 (deployment, vergil-tooling): release the switch

`--kind deployment`, Blocked-by Task 5.

**Preconditions (human-attested):** a vergil-tooling release containing Task 5
is published.

**Procedure:** check the release's `cd-release` package-build jobs ran in
`prod-pkg-*` images and their setup steps skipped the installs. Check that
each consumer with `[package]` (vergil-python, packages) has a
`dependency-update` PR raising its vergil pin to that release, or has merged
one.

**Acceptance:** release build green on the new images; consumer pins listed
with their state. Record `Outcome: SUCCESS`.

---

### Validation V1 (vergil-tooling): PR elapsed time

`--kind validation`, Blocked-by Task 5.

Over at least 20 consecutive vergil-tooling PR CI runs after Task 5 merged,
measured from each run's per-job step timings (`vrg-gh run view <id> --json jobs`,
the method behind the baseline in vergil-project/.github#606):

1. "Set up package toolchain" takes under 30 s in every package job of every
   run;
2. no package job's log shows `apt-get install` or `dnf install` of the
   prerequisites;
3. no run's package gate finishes more than one minute after its last
   non-package job because of toolchain setup.

Record the numbers and `Outcome: SUCCESS`.

---

### Task 6 (vergil-actions): remove the raw-image path

Blocked-by D2. Preconditions: every repo calling `ci-package.yml` or
`cd-release.yml` with `[package]` pins a vergil-tooling at or past D2's
release (checked by reading each repo's `vergil.toml`).

**Files:**

- Delete: `actions/package/setup/os-prereqs.sh`,
  `actions/package/setup/tests/os-prereqs.test.sh`
- Modify: `actions/package/setup/action.yml` (drop the marker branch, the
  prerequisites, uv and nFPM steps and their pins; keep `check-toolchain.sh`
  unconditional, so a job in a raw image now fails with
  `::error::package jobs require a vergil package toolchain image; /etc/vergil/package-toolchain not found`)
- Modify: `.github/workflows/smoke-setup-vergil.yml` (drop the `os-prereqs`
  harness)
- Modify: `actions/package/setup/tests/check-toolchain.test.sh` (add
  `missing_marker_fails` with that message)

- [ ] **Step 1: Add the `missing_marker_fails` case; run it and expect failure.**
- [ ] **Step 2: Remove the raw-image path; run every setup harness and expect a
  pass.** Then `rg -n 'NFPM_VERSION|UV_VERSION|UV_INSTALLER_SHA256' actions/package`
  must print nothing.
- [ ] **Step 3: Validate, commit, report-ready.**
  `vrg-commit --type refactor --scope package --message "require a package toolchain image"`.

---

### Bookends (already filed)

- Docs: vergil-project/.github#608 (this spec and plan).
- Docs review: vergil-project/vergil-tooling#3758. Add to its sweep: align the
  documented host install's `--python 3.14` (`vergil.toml` comment,
  `CLAUDE.md`, `docs/site/docs/getting-started.md`,
  `docs/site/docs/guides/consuming-repo-setup.md`) with the driver version Task
  2 settles on.
- Retrospective: vergil-project/.github#609.

## Self-Review

**Spec coverage:**

| Spec section | Task(s) |
|---|---|
| §5.1 images and contents | T4, Global Constraints |
| §5.1 manifest placement | T3 |
| §5.1 template reuse | T4 (nfpm and uv fragments) |
| §5.1 apt mirror config | T4 `apt-mirror` |
| §5.1 lists removed | T4 |
| §5.1 OS upgrade | T4 |
| §5.2 target registry | T5 |
| §5.3 marker, skip and check | T1 |
| §5.3 the driver stays dynamic | Global Constraints, T1 |
| §5.3 driver Python from one place | #1087, T2 (decision recorded), docs review for the host docs |
| §5.3 raw-path removal | T6 |
| §5.4 public packages | D1 |
| §5.4 rollout | D1, T5, D2 |
| §5.5 #213 coordination | T3 Step 5 |
| §6 success | V1 |

**Type consistency:** Task 3's `Toolchain` and its `matrix.json` entry shape
(`language`, `version`, `build-arg`) are what Task 4's manifest entries and
`cd-docker-publish.yml` consume. The marker contract (`<tool> <version>`, seven
tools) is written by Task 4 and read by Task 1 and Task 6. The image name
`prod-pkg-<distro>:<version>` is the same in Global Constraints, Task 4, D1 and
Task 5.
