# Install vergil-tooling with uv everywhere — design

- **Epic:** vergil-project/.github#1350
- **Planning issue:** vergil-project/.github#466 (review history is in its
  comments)
- **Status:** draft, in design review
- **Repos:** vergil-tooling, vergil-actions, vergil-containers, vergil-vm
- **Related:** epic vergil-project/.github#356 (binary package publishing,
  decision D14), epic vergil-project/.github#649 (Ubuntu base platform, §5.2
  and §5.3.1), epic vergil-project/.github#1256 (vergil-tooling on Python
  3.15), epic vergil-project/.github#1292 (the removed install switch)

## 1. Intent

A vergil-tooling release must be usable everywhere the moment its tag
exists.

Today it is not. Container cache builds, GitHub-hosted runner jobs and the
agent VMs install vergil-tooling as a package from the vergil apt repository
on GitHub Pages. GitHub Pages serves every file with `Cache-Control:
max-age=600`, cached per file at each CDN edge, and the site cannot change
it. So for up to 10 minutes after each release, a consumer can see an index
without the new version. The fleet has paid for that window repeatedly:

- vergil-project/vergil-tooling#4706: dev-image provisioning fails after
  every release, because the tag exists before apt serves the package.
- vergil-project/vergil-tooling#4293: `vX.Y` had to wait for the index, so
  that git and packaged installs agree; the release's `package-index` stage
  gates `promote` and `close-finalize`, and the index publish takes 20 to 27
  minutes (vergil-project/.github#993).
- vergil-project/packages#54: a cached index pointing at a deleted file broke
  fleet CI with 404s until it expired; `keep = 2` fixed it, at a cost in site
  size that pushes the site toward Pages' 1 GB cap.
- vergil-project/vergil-tooling#4663: dnf's `repomd.xml` and its signature
  can come from different publishes.

The only general workaround is to wait out 600 seconds, and work is moving
too fast for that. The window cannot be tuned away on GitHub Pages, and the
fleet's own tooling does not need a package repository to be installed: uv
already installs it from git for every dev ref, on every install path.

So vergil-tooling goes back to `uv tool install` from git on every install
path, on a uv-managed Python 3.15. The images' core Python goes to a
uv-managed 3.15 too, so the fleet's development path stops reading the
package repository altogether.

This undoes a trade-off, not a mistake. The packages were built for the IBM
MQ lab (epic vergil-project/.github#356 §2.1), and Vergil then adopted them
for its own installs. They stay a product Vergil builds and publishes for
outside consumers; Vergil itself stops consuming them, as it already
consumes its GitHub Actions and its Claude plugin straight from git.

### 1.1 Why the move to packages was made, and how each reason is met

The package solved real problems. Each must stay solved, or be given up on
purpose:

- **Its own interpreter.** The package runs vergil-tooling apart from the
  3.12–3.15 Pythons a repo tests (#649 §5.2). With uv: its own tool
  environment on a uv-managed 3.15, named on every install (§4.1).
- **One interpreter choice, immune to a job's `UV_PYTHON`** (#649 alignment
  finding A2.2). With uv: every install names `--python 3.15
  --managed-python` and runs with the project's uv interpreter variables
  cleared (§4.1).
- **The dependency set the release was tested with** (the package's venv is
  built from `uv.lock`). With uv: the install constrains dependencies to the
  tag's `uv.lock` (§4.2).
- **`vrg-*` on `PATH` from a fixed place** (`/usr/bin`). With uv: a fixed tool
  bin directory per install path, checked after install (§4.3).
- **No uv bootstrap on host runners** (vergil-actions stopped pip-installing
  uv). With uv: `install-tooling.sh` already downloads uv with pinned hashes
  (`ensure_uv`).
- **A signed index checked against a pinned fingerprint.** Given up for
  vergil-tooling installs: trust is the git tag over HTTPS from github.com
  (§3, anti-goals), resolved once to one commit (§4.2). Only the release
  pipeline and the repository Admin role may create, update or delete a
  `v*.*.*` or `v*.*` tag. vergil-tooling's enforced "Tag protection" ruleset
  (`github_config.desired_tag_protection_ruleset`, re-checked by the daily
  config audit) carries that rule, with this epic adding **creation** to its
  update and deletion rules for both patterns. Its bypass is limited to the
  Admin role and the one identity the release pipeline uses to create tags,
  named precisely so no other workflow passes. The rule is live, and the audit enforces it, before rollout step 1
  (§5).

## 2. Goals

1. Every released vergil-tooling install (`vX.Y` or `vX.Y.Z`) on every path
   (container cache build, GitHub-hosted runner job, agent VM, macOS host) is
   `uv tool install` from the exact `vX.Y.Z` git tag, on a uv-managed
   Python 3.15.
2. A release is installable as soon as its `vX.Y.Z` tag exists. Nothing on
   any install path reads the vergil package repository or waits on it.
3. The images' core Python is a uv-managed 3.15. `base` no longer installs
   vergil-python and carries no vergil apt source, so image builds do not read
   the package repository either.
4. The existing agent VMs move over through `vrg-vm update`, and new VMs are
   created that way. Old and new VMs end the same: no vergil-tooling package,
   no vergil-python package, no vergil apt source.
5. The waits and guards whose only reason was the package install path are
   removed, each after checking that nothing else needs it (§4.6).

## 3. Non-goals and anti-goals

### Non-goals

- **Package publishing stays.** vergil-tooling and vergil-python keep being
  built, signed, attached to their releases, indexed and install-tested
  exactly as today, as a product for outside consumers.
- **`vrg-release`'s `package-index` stage.** A release is the only point at
  which packages are built and published (no merge to `develop` publishes
  one), so a release keeps watching its package publish, as it watches every
  output it publishes. The stage, and `promote`'s ordering after it, stay
  exactly as they are.
- **The package images and install tests.** The `pkg-*` images and their
  install tests are unchanged, including vergil-project/vergil-tooling#4663
  (dnf retries), which stays a separate task. Their driver vergil-tooling
  install does change (§4.5).
- **The package site.** Its hosting, `keep = 2` retention and size guards are
  unchanged. Replacing GitHub Pages as the package host, the original question
  of vergil-project/.github#466, is a follow-on (§7).
- **The family Pythons.** The python family's images keep their system
  python-build-standalone interpreters at `/opt/python/<series>`, and
  `UV_PYTHON` with `UV_PYTHON_PREFERENCE=only-system` keep pinning a project's
  own environment to them (#649 §5.3.1, issue vergil-project/.github#1002).
- **vergil-tooling's own repo.** It still syncs its checkout as the project
  under test, and `publish-index.yml` still installs from git.

### Anti-goals

- **A wheel or other release artifact for vergil-tooling installs.** Ruled
  out by the human: it buys nothing significant over the exact tag.
- **An install switch or package fallback.** No path chooses between uv and
  the package, and none falls back to the package on failure. A failed uv
  install fails loudly. The rollout (§5) is ordered so no switch is needed;
  vergil-project/.github#1292 just removed one.
- **Coupling the tooling path to vergil-python.** No install path names
  `/opt/vergil/python/…`, or any other interpreter path.
- **Any fetch of mutable content through a CDN-cached URL on an install
  path.** Resolving and reading the tag goes through git over github.com
  (`git ls-remote`, clone), never `raw.githubusercontent.com` or the package
  site, which carry their own cache windows. Immutable, versioned downloads
  are not stale-able and are allowed: PyPI files, uv's own pinned download,
  and the python-build-standalone files uv fetches and checks for a managed
  interpreter.
- **Silent downgrades.** A line pin installs the newest release of that line,
  never an older one because something lagged.

## 4. Design

### 4.1 The interpreter

Every vergil-tooling install runs `uv tool install` with `--python 3.15
--managed-python`, in an environment with `UV_PYTHON`, `UV_PYTHON_PREFERENCE`
and `UV_NO_MANAGED_PYTHON` unset, and with `--no-config`, so no uv config
file in the working directory (a consuming repo's `[tool.uv]` or `uv.toml`)
or the user's config can change or block it, as `actions/package/setup`
already does (issue #1003). uv then uses, or downloads once, its own
python-build-standalone 3.15 and never a system interpreter. The series stays
3.15 until vergil-tooling moves series (vergil-project/.github#1261 tracks
detecting that).

The images pre-install the managed 3.15 (§4.4), so a cache build or a
container job never downloads an interpreter. Runners, VMs and the macOS
host download it on first install and keep it in uv's Python directory.

### 4.2 What a released install installs

- **Exact pin `vX.Y.Z`:** that tag.
- **Line pin `vX.Y`:** the newest `vX.Y.Z` tag, resolved with `git ls-remote`
  as `install-tooling.sh`'s `newest_release` does today.
- **One resolution, one commit.** A released install resolves its tag to a
  commit SHA exactly once, with that same `git ls-remote`. Everything after
  it uses the SHA, never the tag name: the code (`git+…@<sha>`), the
  `uv.lock` the constraints come from, and, for a line pin, the cache key,
  which records the SHA beside the tag. The install logs both the tag and
  the SHA. So the code, the lock, the log and the cache key always name one
  commit.
- **An exact pin needs no network to find its cache.** For an exact
  `vX.Y.Z` pin the cache key uses the tag itself, which tag protection (§1.1)
  keeps immutable, and the SHA is resolved only inside the build. So a
  cached image is found offline, and the documented remedy for working
  without the network (pin an exact release with `VRG_DOCKER_INSTALL_TAG`)
  and its message stay valid.
- **Dev ref:** the ref, as today, with the interpreter rule of §4.1.
- **Locked dependencies (released refs):** the install constrains
  dependencies to the resolved commit's own `uv.lock`, read over git (not a
  raw-content URL), for example by exporting it from a shallow fetch of that
  commit with
  `uv export --frozen --no-dev` and passing it as `--constraints`. A
  resolution that the lock cannot satisfy fails the install. Dev refs install
  unconstrained, as today.

These rules hold on **every** install path, including the ones that are
already uv today (§4.5): none installs from the moving `vX.Y` tag or
upgrades in place with `uv tool upgrade`, which re-resolves the moving tag
without a lock.

One implementation of the resolve-and-install sequence lives in
vergil-tooling (Python), used by the cache build, `vrg-vm` and the macOS
host's install (§4.5).
The shell setup action carries its own, kept equivalent, as today. A shell
script that runs before vergil-tooling exists, which is only vergil-tooling's
first install on a fresh host, uses the same rules.

The cache key's line resolution (`tooling_release.resolve_tooling_tag`,
called from `container_cache._resolve_tooling`) reads the package index
today. It moves to the same single `git ls-remote` resolution, so the key and
the install always name the same commit.

### 4.3 Where `vrg-*` lands, and the post-install check

- **Container cache build:** a fixed directory on the image's `PATH`
  (builder's choice of `UV_TOOL_DIR` and `UV_TOOL_BIN_DIR`, following
  `core-tools.dockerfile`'s precedent).
- **GitHub-hosted runner:** uv's default (`uv tool dir --bin`), appended to
  `GITHUB_PATH`.
- **GitHub container job** (a job with `container:` on a Vergil image, where
  the setup action installs): uv's default (`uv tool dir --bin`), appended to
  `GITHUB_PATH`, as on a host runner. The install passes the image's managed
  Python directory (§4.4) as `UV_PYTHON_INSTALL_DIR`, so it uses the
  pre-installed 3.15 and no container job downloads an interpreter.
- **macOS host:** uv's default, `~/.local/bin`, already on `PATH`.
- **Agent VM:** uv's default, `~/.local/bin`. Every VM, Lima or cloud, new or
  existing, puts `~/.local/bin` on `PATH` for **non-login** remote commands
  too (a `gcloud compute ssh --command=…`, an `ssh <host> <cmd>`, a Lima
  shell command), through `/etc/environment` as vergil-vm's
  `15-environment.sh` writes it, applied to existing VMs by `vrg-vm update`.
  Today the package's `/usr/bin/vrg-*` hides any gap in that guarantee; once
  it is removed, a bare in-guest call such as `vrg_vm.py`'s
  `transport.run("vrg-vm-resolve-session", …)` depends on it. The plan
  inventories every bare in-guest `vrg-*` call on each transport.

After each install, a check confirms `vrg-whoami` resolves to that directory
and runs, replacing today's `/usr/bin/vrg-whoami` checks
(`install-tooling.sh` `verify_packaged`, the cache build's packaged-install
assertion). A stale `/usr/bin/vrg-*` shim from an old package install that
shadows the new one fails the check. On a VM the check runs as a bare,
non-login remote command with no `PATH` prepend, so it tests the guarantee
above rather than working around it.

### 4.4 Images (vergil-containers)

- `base` drops `core-python.dockerfile`'s vergil-python install and the vergil
  apt source and keyring it set up. It runs `uv python install 3.15` into a
  fixed Python directory (builder's choice of path, not under
  `/opt/vergil/python`).
- **The image publishes that path, and consumers read it.** `base` sets it
  in an image environment variable that is not a `UV_*` variable (next to
  the existing `VERGIL_CORE_PYTHON`; builder's choice of name). The cache
  build and the setup action in container jobs (§4.3) read the path from
  that variable and keep no copy of their own. When it is set, they install
  with `UV_PYTHON_DOWNLOADS=never`, so a missing or mismatched interpreter
  fails the install loudly instead of downloading. When it is not set (an
  image older than step 4, or a non-Vergil image), uv may download 3.15, and
  the install logs that it did.
- **That directory is not set image-wide.** `UV_PYTHON_INSTALL_DIR` is passed
  only to the commands that use the core Python: the vergil-tooling install
  and the core tools' creation. Project uv commands never see a managed
  interpreter, so the python images' check that none is installed
  (`docker/python/checks.sh`, from #649 §5.3.1) stays exactly as it is, and
  still passes.
- `/usr/local/bin/python3` links to that managed 3.15 and still promises the
  standard library only (#649 §5.2).
- `base`'s core tools (yamllint, ansible-lint, semgrep, the mkdocs/mike
  environment) are created on the managed 3.15.
- **The image build checks follow the change, not the old layout.** The
  checks every image runs (`docker/base/checks-invariants.sh`,
  `docker/base/checks-defaults.sh`) assert each core tool's interpreter and
  `/usr/local/bin/python3` against the documented managed-Python directory
  instead of `/opt/vergil/python/<core>/`. The vergil apt source check flips:
  a build fails if any vergil apt source or keyring **is** present, so every
  image build enforces Goal 3.
- The version manifest's `core.python` entry still pins the series. The patch
  is the one the image's pinned uv resolves for 3.15, recorded in the image
  build log.
- Nothing else in the image tree may depend on the vergil apt source; the
  plan inventories it (including any `[container].system-packages` user)
  before the source is removed.

### 4.5 Install paths (vergil-tooling, vergil-actions, vergil-vm)

- **Container cache build** (`container_cache.vergil_tooling_setup_command`):
  the packaged branch (`_packaged_lines`, the apt pin file, the candidate
  check) is deleted. Released and dev refs take the uv path of §4.1–§4.3.
- **Setup action** (`shared/setup/vergil/install-tooling.sh`):
  `container_install`, `host_install`, the repository bootstrap and the
  packaged verification are deleted. A released ref takes today's
  `git_source_install` path, with the §4.1 interpreter rule and §4.2
  constraints, in containers and on host runners alike. `install-source`
  keeps its `auto`/`git` input; both now install from git.
- **Package setup action** (`actions/package/setup`, `detect.sh`): its driver
  vergil-tooling install in the `pkg-*` images follows §4.2 (newest `vX.Y.Z`
  for a line pin, one resolved SHA, that commit's lock) instead of
  installing the moving `vX.Y` tag. It keeps its own `--python` input for the
  driver interpreter and its `--no-config`. In vergil-tooling itself it still
  installs the checkout under test.
- **VMs** (`vm_packages`, `vrg-vm update` and create, `vm_guest`,
  `vm_cloud`): a released ref installs with uv, then runs the §4.3 check.
  `_uv_tool_install`'s cache self-heal stays. A released install clears the
  dev-ref marker (`~/.config/vergil/tooling-dev-ref`), which only the
  packaged install clears today, and the dev banner reads "dev ref, not a
  release" instead of naming the packaged install.
- **VM cleanup** (`vrg-vm update`, existing VMs): after the uv install passes
  its check, `vrg-vm update` removes the vergil-tooling package, every
  `vergil-python*` package, the vergil apt source and the
  `vergil-archive-keyring`, and the apt pin file, then confirms none is left.
  Each removal names its packages; `apt-get autoremove` is never run, as
  today. A VM created after the change never gets any of them: `vrg-vm
  create` no longer bootstraps the vergil apt repository.
- **vergil-vm** (`vrg-vm-init.sh`, `tests/test_vergil.sh`, the Lima
  template, cloud-init): already uv, but on the moving `vX.Y` tag with no
  lock. Each moves to the §4.1 and §4.2 rules. Where `vrg-vm create` already
  installs vergil-tooling, the duplicate install in a script goes, rather
  than becoming a third copy.
- **macOS host: the release owns the install.** vergil-tooling's own release
  installs the just-released version on the Mac through its
  `[publish].consumer-refresh` (`vergil.toml`), as it already does. That
  command moves from a bare `uv tool install … @v<VERSION>` to the shared
  resolve-and-install code, so the Mac gets the exact release, one resolved
  SHA and that commit's lock (§4.1, §4.2), the same way `vrg-vm update`
  updates the VMs. The documented first install on a fresh Mac follows the
  same rules.
- **`vrg-host` never installs or upgrades vergil-tooling.** It reports the
  installed version and says the release owns its install. Today's code still
  runs `uv tool upgrade vergil-tooling` (the manifest entry's `upgrade =
  "auto"`, `host_methods.upgrade`); vergil-project/vergil-tooling#4603
  removes that, as a task of this epic.

### 4.6 Waits and guards that lose their reason

- **vergil-project/vergil-tooling#4706** (wait for apt to catch up) is
  superseded: the cache build no longer reads apt. It closes as superseded by
  this epic, with nothing implemented.
- **The consumer-refresh commands** use the shared resolve-and-install code
  (§4.5, macOS host).
- **`keep = 2`** stays (non-goals): the package install-tests still read the
  index.

## 5. Rollout

Ordered, one repo at a time, with no install switch, while the rest of the
fleet's work is frozen and this epic owns it. The freeze is expected to last
one working session (a morning); during it, only security fixes land in
other repos. Each step is released and checked before the next starts.

0. **Precondition:** the Tag protection ruleset's creation rule (§1.1) is
   released in vergil-tooling's `github_config`, applied, and confirmed by
   the config audit, before step 1.

1. **vergil-tooling** releases the uv install in the cache build, `vrg-vm`
   and its consumer refresh, and `vrg-host`'s report-only handling of
   vergil-tooling.
2. **vergil-actions** releases the setup action and package setup action
   changes, moving `v2.1`.
3. **The macOS host** gets the new release from step 1's consumer refresh, and **every
   existing VM** runs `vrg-vm update` to the new release, removing every vergil
   package and the vergil apt source (§4.5). vergil-vm's script changes land here.
4. **vergil-containers** releases `base` on the managed 3.15, without
   vergil-python or the vergil apt source. This is last because any
   vergil-tooling still doing a packaged install in a container needs that
   source; steps 1–3 remove every such install first. Before this step, the
   plan recounts every consuming repo's vergil-tooling pin; an exact pin to a
   release older than step 1 does not matter, since the cache-build code that
   runs is the host's, not the pin's.

## 6. Verification

- Unit tests for every changed install path, including: the resolved tag
  for a line pin; the constraints applied for a released ref; the cleared
  interpreter variables; a failed uv install failing loudly; a shadowing
  `/usr/bin` shim failing the post-install check; `vrg-vm update` removing the
  package; one SHA used for code, lock and cache key; an exact pin finding
  its cached image with no network call; a released install clearing the
  dev-ref marker; a hostile `uv.toml` in the working directory changing
  nothing about the install.
- A cold rebuild: a new Lima VM created from scratch has vergil-tooling from
  uv and no vergil package or apt source; a cache build on the new `base` installs
  vergil-tooling on the managed 3.15 with no interpreter download.
- The new `dev-` images, as published by vergil-containers' develop CD build,
  carry the managed 3.15 and no vergil apt source, checked before step 4's
  release.
- A fixture run on each language fixture after step 4, against the released
  vergil-tooling, vergil-actions and vergil-containers.
- The headline check: release vergil-tooling, then immediately provision a
  container and update a VM to the new tag, with no wait, and both get it.

## 7. Follow-on

- **Package hosting.** Replacing GitHub Pages as the package host (object
  storage, a hosted service, or the planned home hosting platform in
  February 2027) goes back to the deferred queue as its own item, with
  vergil-project/.github#466's survey list and the owned-domain item
  vergil-project/.github#365.

## 8. Left to the builder

- The tool and Python directories in the images (§4.3, §4.4), off
  `/opt/vergil/python`.
- The exact mechanics of reading the tag's `uv.lock` over git and passing it
  as constraints (§4.2).
- How the Python resolve-and-install code is shared between the cache build,
  `vrg-vm` and the consumer refresh, and the command the consumer refresh
  calls.
- Whether `--managed-python` together with the cleared variables needs any
  further override in the cache-build environment; settled by a real cache
  build, not assumed.
