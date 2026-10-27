# Package toolchain images

- **Epic:** vergil-project/.github#607
- **Planning issue:** vergil-project/.github#606 (review history is in its
  comments)
- **Status:** design approved 2026-10-27; pushback round 1 done (2 findings,
  both accepted); alignment done (3 findings, all accepted)
- **Repos:** vergil-containers (the images), vergil-actions (the setup action),
  vergil-tooling (the package target registry)
- **Builds on:** epic vergil-project/.github#213 (manifest-driven container builds)
- **Related, out of scope:** vergil-project/.github#605 (OS consistency review)

## 1. Why

Every CI package job starts from a raw distro image and installs its toolchain
on every run. The images are `ubuntu:24.04`, `ubuntu:26.04`, `ubi9/ubi` and
`ubi10/ubi`. The jobs are vergil-actions `ci-package.yml` (build and
install-test) and `cd-release.yml` (release build). The image names come from
vergil-tooling `lib/package/matrix.py` (`SHARED_IMAGE`) and
`lib/package/targets.py`. The toolchain is installed by vergil-actions
`actions/package/setup`:

1. OS prerequisites through `os-prereqs.sh`: `ca-certificates curl git tar gzip
   gnupg`, from apt (with an Azure-first mirror list and fail-fast apt config)
   or dnf;
2. uv 0.12.23, through its checksum-pinned installer;
3. vergil-tooling via `uv tool install --python 3.14 <spec>`. This downloads a
   uv-managed CPython 3.14, because none of these images has one;
4. nFPM 2.47.0, checksum-pinned (build jobs only).

On amd64 Ubuntu, step 1 intermittently trickles from `azure.archive.ubuntu.com`
at about 44 kB/s. One build log shows the index at 27 MB/s, then
`Fetched 20.0 MB in 7min 32s (44.2 kB/s)` for the packages. The fail-fast
config from vergil-actions#931 only trips on stalled connections, not slow
ones. Over the last 40 vergil-tooling PR CI runs:

- 22 runs had at least one slow amd64 Ubuntu package job and took 8–17 minutes.
- Excluding the package jobs, every run finished in 2.2–4.2 minutes.
- arm64 (ports.ubuntu.com) and RHEL (dnf) jobs were never slow.
- On the reduced PR tier, the two slow jobs (build `shared-amd64` and
  install-test `test-ubuntu-24.04-amd64`) run in sequence, so one run can pay
  twice.

The human decision: stop installing the toolchain at run time. Build it once
into managed images in vergil-containers, and maintain them. Run-time
workarounds were rejected: an apt cache would be empty for most PRs, because
PRs only read caches saved on `develop` and nothing runs package jobs there.
Forcing a mirror switch on slow downloads was rejected as fragile.

## 2. Goals

1. **No run-time toolchain install in package jobs.** Every build,
   install-test and release-build job runs in an image that already has
   everything the setup action installs today except vergil-tooling itself: the
   OS prerequisites, the apt mirror and fail-fast config (amd64 Ubuntu), uv and
   nFPM. The driver vergil-tooling and its Python stay dynamic (§5.3).
2. **PR elapsed time follows the other gates.** On a normal PR, the package
   gate stops being the long pole because of toolchain setup.
3. **One place for each toolchain pin.** uv and nFPM versions live in
   vergil-containers, under its existing pins machinery (`docker/pins/`), not
   in the setup action as well. Today nFPM is pinned twice and the pins
   disagree (2.47.0 in the action, 2.48.0 in `common/nfpm.dockerfile`).
4. **Ride epic #213's machinery, don't duplicate it.** The images are built,
   scanned, published and refreshed nightly by the same manifest-driven matrix
   as the dev images. They get native arm64 builds (#213 T7) and PR-time
   selective build and scan (#213 T6′) when those land.

## 3. Non-goals

- Changing which OS targets packages support, or their versions.
- Standardizing OS choice across containers, VMs and runners. That is #605,
  and this design must work whatever #605 decides. Package images exist
  because packages target specific distros, so they stay per-target.
- Trimming the prerequisite set (for example, dropping `git`). The images carry
  exactly what the setup action installs today.
- Changing the dev images (`dev-*` / `prod-*` language images and `base`).
- Run-time caching or mirror tuning.

## 4. Anti-goals (failure even if it works)

- **Install-test drifting further from a consumer machine.** Today an
  install-test container is the upstream distro image plus the six
  prerequisites, uv and (on amd64 Ubuntu) the apt mirror and fail-fast config,
  with a uv-managed Python and the driver under uv's directories. The image must contain that, plus the OS upgrade every
  vergil image applies (§5.1), and nothing more.
- **Polluting the language model.** Package images are not languages. They
  must not appear in `window.json`, in `vrg-ci-resolve` matrices, in
  language-to-image resolution (`vergil_tooling/lib/container.py`), or in the
  version-lifecycle automation that opens add/drop PRs for language versions.
- **A silent fallback.** If a job runs in an image that claims to be a
  toolchain image but lacks a tool, the job must fail loudly, never quietly
  install it.

## 5. Design

### 5.1 The images (vergil-containers)

Two image families, one per distro family, each built per target version and
architecture:

| Image | Versions | `FROM` |
|---|---|---|
| `pkg-ubuntu` | 24.04, 26.04 | `ubuntu:<v>` |
| `pkg-rhel` | 9, 10 | `registry.access.redhat.com/ubi<v>/ubi` |

They are published as `ghcr.io/vergil-project/{dev,prod}-pkg-ubuntu:<v>` and
`…-pkg-rhel:<v>`, multi-arch (amd64 and arm64), with the same datestamp alias
and provenance attestation the dev images get. *(Names are my call; the builder
may adjust them before the first publish. After that they are an interface.)*

Each image contains, on top of the upstream image:

- the prerequisites `os-prereqs.sh` installs today (`ca-certificates curl git
  tar gzip gnupg` on apt; `ca-certificates /usr/bin/curl /usr/bin/gpg git tar
  gzip` on dnf, keeping UBI's `curl-minimal` handling);
- uv at the pinned version, in `/usr/local/bin`;
- nFPM at the pinned version, in `/usr/local/bin`. It is in both the build and
  install-test variants; one image per target is simpler than two, and nFPM is
  one static binary;
- a marker file, `/etc/vergil/package-toolchain`, listing each provided tool and
  its version (§5.3 uses it).

- on amd64 `pkg-ubuntu` only: the apt mirror list (`/etc/apt/apt-mirrors.txt`
  plus the rewritten sources) and `/etc/apt/apt.conf.d/80-vergil-fast-fail`,
  exactly as `os-prereqs.sh` writes them today. Install-test containers already
  have these today, because setup writes them before install-test runs, so
  baking them in leaves install-test as it is now. Every later apt call keeps
  the Azure-first mirror with failover and fail-fast timeouts. *(My call.)*

**Package lists are not kept.** As every template does today, the image
removes `/var/lib/apt/lists` after installing. Kept lists would go stale
between nightly rebuilds and can 404. So `vrg-package install-test`
(`install_test.py`) and `repo_setup.bootstrap` still run their `apt-get update`
at run time. That is accepted. In every slow run the index fetch was fast
(27 MB/s); the trickle was in package downloads, and with the prerequisites
built in the only remaining downloads in a vergil-tooling install-test come
from the vergil package repository. The exception is a package that ships
systemd units, which still installs `systemd` from the archive. *(My call.)*

**The image runs the OS upgrade** (`apt-get upgrade -y` / `dnf upgrade -y`),
as every other vergil-containers template does (human decision). The publish
gate fails on any fixable CRITICAL/HIGH finding, so an image without the
upgrade would fail whenever a distro fix ships before the upstream tag is
rebuilt. The setup step already pulls updated libraries today (for example
`openssl` and `libcurl` from `noble-updates`), and real consumer machines are
usually patched, so install-test fidelity is not reduced. The nightly
`no-cache` rebuild keeps the image current.

**Template reuse.** `pkg-ubuntu` reuses `common/nfpm.dockerfile` (its apt
`curl` and `TARGETARCH` assumptions hold on Ubuntu). `pkg-rhel` can't use the
apt-only fragments, so its template carries the dnf equivalents. uv gets one
new fragment, usable by both families provided it uses only `curl` and
`sha256sum`. No changes to fragments the dev images use.

**Manifest placement.** The images enter the #213 build matrix through the
generated `matrix.json`, so `cd-docker-publish.yml`, the nightly, native arm64
(T7) and PR selection (T2/T5/T6′) handle them like any other entry. The source
of truth is a new, separate section of `docker/versions/manifest.yml` (for
example `toolchains:`), **not** `families:`, which is the language model
(families carry `language`, a lifecycle adapter and `runtime_published`).
`generate.py` emits matrix entries for both sections, but `window.json`, the
lifecycle check and the docs' language tables read only `families:`. The exact
schema is left to the builder, within the anti-goal in §4.

**Build cost under QEMU.** Until #213 T7 lands, the arm64 half builds under
emulation. These images only install packages and download binaries, with no
compilation, so the emulation cost is small. This epic does not wait for T7.

### 5.2 Pointing package jobs at the images (vergil-tooling)

`lib/package/targets.py` and `lib/package/matrix.py` name the toolchain images
instead of the raw ones:

- every target's test image becomes `ghcr.io/vergil-project/prod-pkg-<distro>:<version>`,
  derived from the target's distro and version rather than hand-listed per
  row, so a new target (for example Ubuntu 28.04) can't silently keep a raw
  image;
- `SHARED_IMAGE` becomes `prod-pkg-ubuntu:24.04`;
- native build cells use their target's toolchain image.

Package jobs use the `prod-` prefix, the released images, as `container.py`
already does for the dev images. A target with no published image fails at
container pull, loudly, in the PR that adds the target.

The registry's glibc values stay correct: the toolchain images use the same
upstream base, so `ldd --version` is unchanged.

### 5.3 The setup action (vergil-actions)

`actions/package/setup` keeps its interface. Its behaviour depends on the
marker file:

- **Marker present (toolchain image):** skip the OS prerequisites, uv and nFPM
  steps. Check that every tool the marker lists is on `PATH` at the listed
  version, and that nFPM is present when `nfpm: "true"`; fail with `::error::`
  otherwise. Then run the vergil-tooling install (below).
- **Marker absent (raw image):** today's behaviour, unchanged.

**The driver install stays dynamic, through uv.** The setup action installs a
copy of vergil-tooling (the *driver*) to run `vrg-package`. It is deliberately
not baked into the image and not installed from the vergil package repository
(human decision):

- vergil-tooling releases far more often than the images should be rebuilt.
  The driver tracks the checkout (in vergil-tooling itself) or the consumer's
  pinned release without an image rebuild.
- In vergil-tooling's own CI, the driver must be the checkout under test, which
  no released package can be.
- Install-test needs the driver invisible to the package under test. It lives
  in uv's tool directory, outside the system-only `PATH` the smoke checks use.
  An apt-installed driver would already own `/opt/vergil/vergil-tooling` and
  `/usr/bin/vrg-*` when the vergil-tooling `.deb` is being tested. For any
  `python`-builder product, it would also pre-install `vergil-python<X.Y>` and
  mask the product's runtime dependency resolution.

**The driver's Python comes from one place, not hard-coded constants.** The
images bake in no interpreter; uv bootstraps the driver's Python at run time.
Today the action hard-codes 3.14 in two places:

- `action.yml`: `uv tool install --python 3.14 "$spec"` (the driver install);
- `detect.sh`: `uv run --no-config --no-project --python 3.14 …` (reading
  `[dependencies].vergil`).

3.14 still works (vergil-tooling is tested on every Python from 3.12 to 3.15),
but it is drift. vergil-tooling has run on 3.15 for weeks, and every package
job downloads a separate CPython 3.14 (about 35 MiB) just for the driver.

- **Tactical, outside this epic:** vergil-project/vergil-actions#1087 bumps
  both sites to 3.15 now (human decision). It doesn't depend on the images.
- **In this epic:** the driver's Python must come from a single place, so a
  runtime change is one edit or none, and the two sites can't disagree. A
  related mechanism already exists: since vergil-tooling#3458 (commit
  `7dee9d7`), vergil-tooling's packaged runtime is derived from the published
  Python version window (the newest version with a published `vergil-python`
  runtime), and `vrg-package matrix` records it for build jobs. That
  derivation describes the product being packaged, though, not the driver: in
  a consumer repo it is the consumer's runtime, and install-test and staged
  builds carry none. The plan decides whether the driver follows the derived
  runtime or a single constant both sites read, and records why. It must not
  add a third, independent definition. The documented host install
  (`--python 3.14` in vergil-tooling's `vergil.toml`, `CLAUDE.md`,
  `docs/site/docs/getting-started.md` and
  `docs/site/docs/guides/consuming-repo-setup.md`) is aligned in the epic's
  docs-review sweep once the driver version is settled, because those files
  live in vergil-tooling and the driver change lands in vergil-actions.
The raw-image path is transitional. `ci-package.yml@v2.1` is shared, and a
consumer whose pinned vergil-tooling predates §5.2 still emits raw images. A
follow-on task removes the raw-image path, `os-prereqs.sh` and the action's
uv/nFPM pins once every consumer's vergil-tooling pin is at or past the §5.2
release. That task is filed and tracked, not left implicit.

### 5.4 Rollout order

Each step is safe on its own:

1. **vergil-actions:** marker-aware setup action (a no-op until images exist).
2. **vergil-containers:** the images, published `dev-` on develop, then `prod-`
   on a containers release.
   - **Human-gated: make the packages public.** New GHCR packages start
     private, and every consuming repo's package jobs must pull them. After the
     first publish of each, set `dev-pkg-ubuntu`, `dev-pkg-rhel`,
     `prod-pkg-ubuntu` and `prod-pkg-rhel` to **Public**, and confirm
     vergil-containers has **Write** under Manage Actions access, as was done
     for the original image packages
     (vergil-containers `docs/plans/2026-05-09-release-workflow-and-image-naming.md`,
     Step 3). Then verify an anonymous pull of each `prod-` image
     (`https://ghcr.io/token?scope=repository:vergil-project/<package>:pull`
     returns a token). Step 3 does not start until this passes. The plan seeds
     this as a `deployment` task, since only a human can change package
     settings.
3. **vergil-tooling:** `targets.py` / `matrix.py` point at `prod-pkg-*`.
   vergil-tooling's own PR CI installs vergil-tooling from the checkout, so
   that PR proves the switch end to end on its own package gate.
4. **Deployment:** a vergil-tooling release, after which consumers pick it up
   through `vrg-update-deps`.
5. **Follow-on:** remove the raw-image path (§5.3).

Steps 1 and 2 can run in parallel. Step 3 needs both released and the packages
public.

### 5.5 Interaction with epic #213

- New image contexts must be visible to T2's dependency map and T5's selector.
  Whichever of this epic and T2/T5 lands second adapts the other. Until then,
  T5's fail-closed rule (an unclassified `docker/` path selects the full set)
  keeps it safe.
- If T7 lands first, the new entries ride native arm64 automatically; if not,
  they ride QEMU and T7 moves them along with everything else.
- The nightly `no-cache` rebuild covers the new images, as it does every other
  image: it pulls the current upstream tag and reapplies the OS upgrade.
- A failed nightly publish leaves the previous image in place, as for the dev
  images. Before step 3 of the rollout, the `prod-` images must have published
  successfully at least once.

## 6. Success

- In every vergil-tooling PR CI run after the switch, "Set up package
  toolchain" takes under 30 s on every package job.
- No package job runs `apt-get install` or `dnf install` for toolchain
  prerequisites. Run-time `apt-get update` and installs of the package under
  test (and its dependencies) stay, by design.
- Over 20 consecutive PR runs, no run's package gate is the long pole by more
  than one minute because of toolchain setup.
- uv and nFPM versions appear once each, in vergil-containers.

A `validation` task records these numbers from real runs after the deployment
step.

## 7. Left to the builder

- The manifest section's exact schema and how `generate.py` emits it.
- Template layout under `docker/` and fragment names.
- The marker file format, provided §5.3's check can be done from it.
- Whether the version check in §5.3 lives in the action's shell or in a small
  script with tests (`actions/package/setup/tests/` already exists).
- Package-cleanup retention for the new images (follow the dev images'
  existing rules).
