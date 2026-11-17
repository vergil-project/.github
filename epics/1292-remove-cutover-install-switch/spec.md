# Remove the cutover vergil-tooling install switch — design

- **Epic:** vergil-project/.github#1292
- **Planning issue:** vergil-project/.github#983 (review history is in its
  comments)
- **Status:** approved
- **Repos:** vergil-tooling (`container_cache.py`, site docs), vergil-actions
  (`shared/setup/vergil`, site docs)
- **Inherited design:** epic vergil-project/.github#649,
  [`epics/649-ubuntu-base-platform/spec.md`](../649-ubuntu-base-platform/spec.md)
  §5.2, §5.5 and §7 (step 5, step 12)

## Intent

Epic #649 moved every Vergil image from Debian to Ubuntu 24.04. So that the
move could be made, and reverted, without breaking any consumer, its step 5
gave the two places that install vergil-tooling in a container an **install
switch**:

- vergil-tooling's cache build, `vergil_tooling_setup_command` in
  `src/vergil_tooling/lib/container_cache.py`;
- vergil-actions' setup action, `actions/shared/setup/vergil/install-tooling.sh`.

The switch probes the image for two things, the vergil apt source
(`/etc/apt/sources.list.d/vergil.sources`) and a core vergil-python
(`/opt/vergil/python/*/bin/python3`), and chooses from the answer:

| Image has | Meant | Install |
| --- | --- | --- |
| both | an Ubuntu image | packaged (released ref); git on core Python (dev) |
| neither | a Debian image | `uv tool install` from git on `--python 3.15` |
| only one | a half-built image | refused |

The cutover release (vergil-containers v2.1.38, 2026-11-11) has held, and the
human has judged that a revert to the Debian images is off the table
(2026-11-17). Only one kind of image remains, so the switch has nothing left
to choose. This epic is #649's step 12: it deletes the switch.

It goes beyond step 12 as written, by the human's decision (2026-11-17).
Step 12 drops the git-install branch for released refs and keeps the dev-ref
path. This epic also drops the container dev-ref branch for an image with
neither, which only the Debian images reached, and the dev-ref path's count
of core Pythons (the "more than one core Python" refusal), since every image
carries exactly one. The dev-ref git install itself stays.

The point is to remove cutover scaffolding, not to trade it for something
else. Nothing replaces the switch: no new check, test or guard.

## Goals

1. In a container, a released ref (`vX.Y` or `vX.Y.Z`) goes straight to the
   packaged install, and a dev ref goes straight to `uv tool install` from git
   on the image's core Python, in both repos.
2. The probes, the half-image refusals, the git-install branch for an image
   with neither and the core-Python count checks are gone from both repos,
   with their tests and every doc that describes them.

## Non-goals

These are untouched:

- the host-runner installs in `install-tooling.sh`: the packaged install for
  a released ref (`host_install`), and `uv tool install --python 3.15` for a
  dev ref;
- vergil-tooling's self-repo `uv sync`, and its cache build, which installs
  no vergil-tooling;
- `install-source: git` (publish-index's index rebuild);
- the package jobs' install in `pkg-*` images (`actions/package/setup`), a
  separate design;
- the post-install check that `vrg-*` runs from `/usr/bin` on vergil-python
  (`_packaged_lines`, `verify_packaged`). It comes from #649's runtime rule
  (§5.2.1), not from the cutover;
- `DOCKER_DEV_IMAGE`, which skips the cache build and installs no
  vergil-tooling.

## Anti-goals

- **Adding a check in place of the switch.** An image without the vergil apt
  source already fails the packaged install on its own, at the existing
  candidate check: "no packaged vergil-tooling matching vX.Y in …". It needs
  no probe in front of it. That message's advice (release first, or pin a git
  ref) does not fit such an image; this is known and accepted, since no
  Vergil image or workflow reaches it.
- **Hardening cases no image reaches.** Every Vergil image carries the vergil
  apt source and exactly one core Python (vergil-containers'
  `core-python.dockerfile` installs one). A container with no core Python
  fails loudly on a dev ref, at an unbound array element under `set -u`. One
  with more than one would install on the first glob match; that is
  accepted, since no image has two. Nothing is added for either case.
- **Over-managing the risk.** The change deletes branches nothing reaches,
  and a break is fixed forward (see Rollout).

## Design

### vergil-tooling: `container_cache.py`

`vergil_tooling_setup_command(tag)` keeps its contract: a `bash -euo pipefail
-c` fragment that installs vergil-tooling at *tag* and logs the path it took.

- **Released ref:** the `_packaged_lines` path as it is today (pin, update,
  candidate check, install, the `vrg-*`-on-vergil-python check), without the
  probes in front of it.
- **Dev ref:** `uv tool install` from git with `--python` set to the image's
  core Python, as `_dev_core_lines` does today, without its "more than one
  core Python" refusal.
- **Removed:** the `has_source` and `has_core` probes, both half-image
  refusals, the git-install branch for an image with neither, the
  `_UV_FROM_GIT` message, and the core-Python count check. `_probe_lines`
  either goes, or shrinks to the shared preamble the two paths still need
  (the `$VRG_CACHE_ROOT` seam and the core-Python glob).
- **Tests** (`tests/vergil_tooling/test_container_cache.py`): the cases for
  the neither, half-image and more-than-one-core paths are deleted. Any
  other test that asserts removed content for a released tag (the
  neither-branch's `uv tool install`, the probes or `_UV_FROM_GIT`) is
  rewritten to assert the packaged install instead, not deleted; a test that
  uses `uv tool install` to mark where the install sits in the setup order
  uses the packaged install's log line as its marker. The other cases for
  the two remaining paths stay.
- **Site docs:** the `docs/site/docs/guides/ci-architecture.md` paragraph that
  describes the two checks ("The image chooses how vergil-tooling is
  installed …") describes the two paths instead.

The cache key does not include the setup fragment, so the change rebuilds no
cached image.

### vergil-actions: `install-tooling.sh`

- **Container, released ref:** `container_install` does the packaged install
  as it does today for an image with both (pin, update, candidate check,
  install, `verify_packaged`, `ensure_uv`), with no environment check.
- **Container, dev ref:** `uv tool install` from git on the image's core
  Python, as `dev_install` does today for an image with both, without its
  "exactly one core Python" check.
- **Host runner, dev ref:** `uv tool install --python 3.15` from git, as today
  (`uv_git_install`). `dev_install` therefore branches on `is_host`, not on
  the environment check.
- **Removed:** `environment_state`, both half-image refusals, the container
  git-install branch for an image with neither, and the core-Python count
  check.
- **Tests:** the `tests/install-tooling.test.sh` cases for the neither,
  half-image and core-count container paths are deleted, and the host
  dev-ref case is kept.
- **Docs:** the header comment, `action.yml`'s description,
  `docs/site/docs/actions/shared-setup-vergil.md` and the header comment of
  `.github/workflows/smoke-setup-vergil.yml` describe the paths that remain.

## Rollout

- The two repos' PRs are independent and land in either order.
- **vergil-actions** ships the change as the only significant change in its
  release, so moving the floating `v2.1` tag back to the previous release
  stays a clean revert. The human holds other vergil-actions changes back
  until that release is out. A deployment task records the release. The
  PR touches `.github/workflows/**`, so GitHub refuses the agent's push and
  the human's `vrg-submit-pr` pushes the branch.
- **vergil-tooling** ships the change in its next normal release, with no
  isolation and no deployment task.
- **Verification of the vergil-actions half:** its own CI, then the next CI
  run of any consumer, since every consumer's CI installs vergil-tooling
  through `install-tooling.sh`. The fix-forward path is reverting `v2.1` to
  the previous release, then a fix.
- **Verification of the vergil-tooling half:** its unit tests of the setup
  fragment, then the first local `vrg-container-run` or `vrg-finalize-pr` on
  a host that has the new release. Consumer CI never runs the cache build,
  and vergil-tooling's own CI skips the install as the self-repo, so a break
  shows up on developer and agent hosts, where it blocks the local
  `vrg-validate` gate. The fix-forward path is a fix release, then a
  reinstall on each host.
- **No fixture runs:** the change deletes code on the same path for every
  language, its risk is low, and a break is fixed forward. This departs on
  purpose from the fixture-run rule for changes to shared CI workflows.

## Left to the builder

- Whether `_probe_lines` is deleted outright or reduced to a shared preamble,
  and any refactoring of `install-tooling.sh`'s functions that keeps the
  behaviour above.
- The wording of the updated log lines and docs.
