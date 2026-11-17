# Narrow vergil-tooling to the newest Python series — design

- **Epic:** vergil-project/.github#1256
- **Planning issue:** vergil-project/.github#958 (review history is in its
  comments)
- **Status:** draft, in design review

## 1. Intent

vergil-tooling supports and tests every Python series in the version window,
3.12 to 3.15 today (`requires-python = ">=3.12"`). Its CI builds a test leg for
each of the four in parallel, which makes its gates slow. That support came from
the dev containers, which used to install vergil-tooling on the image's own
interpreter, so the tooling had to run on whatever series an image's family
tested.

Epic vergil-project/.github#649 removed that reason. Every Vergil image now
carries a core Python (`vergil-python`, under `/opt/vergil/python/<x.y>`) and
installs vergil-tooling on it, as the agent VMs already do. vergil-tooling is a
set of command-line tools that no other repo imports as a library. The Python it
runs on is therefore independent of the Python a project builds and tests with.

This epic narrows vergil-tooling to the newest series, 3.15 today. The aim is a
faster CI gate and less maintenance. It also makes every install path name the
interpreter vergil-tooling runs on, so that narrowing breaks nothing.

## 2. Goals, non-goals, anti-goals

### Goals

- vergil-tooling declares `requires-python = ">=3.15"`. Its CI test matrix, and
  `vrg-validate --all-versions`, then hold one series today.
- Every path that could otherwise land vergil-tooling on an untested
  interpreter names the interpreter it runs on and fails loudly when that
  interpreter is missing; none of them leaves the choice to uv's implicit
  discovery. The deliberate exceptions are paths that run on an image's own
  interpreter under `UV_PYTHON_PREFERENCE=only-system`: the self repo's
  container `uv sync` (§5 row 5, container case), which runs on each test
  leg's series, and the docs builds (row 14), which run on the base image's
  core Python. Each is a tested series by construction, and `only-system`
  makes uv fail rather than fetch anything else.
- The floor is documented, along with the procedure for raising it when a new
  series joins the window.

### Non-goals

- Reclassifying vergil-tooling as an application (`repository-type`,
  `branching-model`). It stays a library. This is follow-up
  vergil-project/.github#1260.
- Detecting a new leading-edge series and prompting a floor raise, or automating
  that raise. This is follow-up vergil-project/.github#1261.
- Changing the Python any other repo builds or tests with, or the images and
  window that serve them.

### Anti-goals

- A new definition of "the core series". Two exist today and this epic adds
  none: vergil-tooling's package derives it (`derived_runtime`,
  vergil-tooling#3458: the newest window series with a published
  `vergil-python` runtime), and the Vergil images pin it by hand
  (vergil-containers `docker/versions/manifest.yml`, `core: python`, which
  `base` installs and links as `/usr/local/bin/python3`). Unifying them is
  out of scope (vergil-project/.github#1267). The host and VM installs in
  §6.2 pick the newest series the vergil apt repository serves. That is a
  selection rule over installable packages, not a third pin, but it tracks
  the same thing, so #1267 folds it in too.
- A gate that turns red on its own when the window changes. A check that fails on
  every unrelated PR the day a new runtime publishes is a broken gate, not a
  notification.
- An install that works only because uv happened to find or download a suitable
  interpreter.

## 3. The model: floor, tested series, core series

Three ideas, kept distinct:

- **Floor:** the lower bound of vergil-tooling's `requires-python`. Today
  3.15.
- **Tested series:** every window series at or above the floor (the
  resolver's existing rule, unchanged). Today 3.15.
- **Core series:** the series the package derives (the newest window series
  with a published `vergil-python` runtime) and the series the images pin in
  `manifest.yml`, both unchanged. Today both are 3.15.

vergil-tooling stays a library, so its matrix keeps following the window above
the floor. When 3.16 joins the window, CI tests 3.15 and 3.16 side by side with
no change to vergil-tooling. Once both are reliably green, a deliberate PR raises
the floor to `>=3.16` and the matrix returns to one series. The epic writes no
code for that transition; it only documents it (§6.4).

**The invariant:** vergil-tooling runs only on a tested series. An installed
tool runs on the core series in an image with a core Python, and on the
newest packaged `vergil-python` series on a host runner or a VM dev install
(§6.2). The self repo's container test legs run on each leg's own series.
These are all tested series only while the floor is at or below both core
definitions and the newest packaged series. Nothing enforces that
automatically, so the floor-raise procedure checks it before every raise
(§6.4).

## 4. Approaches considered

- **A. Raise the floor and keep the window-following matrix (chosen).** This
  needs no new mechanism, because the resolver already intersects the window with
  `requires-python`. A new series gets an overlap period in which it is tested
  next to the floor before anyone commits to it, which is the transition the
  human wants.
- **B. A core-only test matrix** (a `vergil.toml` setting that makes CI test only
  the derived core series). Rejected: on the day a new runtime publishes, CI
  would jump to the new series with no period testing both.
- **C. Let the nightly upstream refresh raise `requires-python` automatically.**
  Rejected for this epic: it widens the refresh job's reach into vergil-tooling's
  packaging. Any automation of the raise belongs to follow-up #1261.

## 5. Current state (audit)

The audit of every place a Python version is specified for vergil-tooling,
against `origin/develop` of each repo. The plan's first task confirms and
completes it (§7).

Each row names the place, how the interpreter is chosen today, and what
changes.

1. **`requires-python`** (vergil-tooling `pyproject.toml`): `>=3.12`. After:
   `>=3.15`.
2. **ruff `target-version`, mypy `python_version`** (vergil-tooling): `py312`,
   `3.12`. After: `py315`, `3.15`.
3. **CI test matrix** (vergil-actions `ci-test.yml` through `vrg-ci-resolve`):
   the window intersected with `requires-python`, four legs. After: one leg,
   through row 1 alone.
4. **`vrg-validate --all-versions`** (vergil-tooling): the same resolver, four
   series. After: one series, through row 1 alone.
5. **Self-repo `uv sync`** (vergil-actions `install-tooling.sh`
   `self_repo_install`, which `main()` takes before any host or container
   check), with `UV_PYTHON_PREFERENCE=only-system`. It has two cases:
   - in a container job, the image's interpreter on each leg's image.
     Unchanged; the legs shrink through row 1;
   - on an `ubuntu-24.04` host runner (vergil-tooling's CI runs several host
     jobs through the reusable workflows, among them `ci-test.yml`'s `matrix`
     job, which feeds every test leg, and the host jobs of `ci-quality.yml`,
     `ci-audit.yml`, `ci-security.yml`, `ci-version-bump.yml` and
     `cd-release.yml`): the runner's system Python, 3.12. After: §6.2.
6. **Packaged install** (`apt`, in containers and on host runners; vergil-actions
   `install-tooling.sh`): `vergil-python` from the package's derived runtime.
   Unchanged.
7. **Dev-tag install in an image with a core Python** (vergil-actions
   `install-tooling.sh`, vergil-tooling `container_cache.py`): explicit
   `--python <core Python>`. Unchanged.
8. **The `neither` path**: an environment with neither the vergil apt source
   nor a core Python. It is reached in two ways, both with `uv tool install`
   and **no `--python`**:
   - a **host runner with a dev pin**. `main()` sends a dev pin to
     `dev_install` before it checks for a host, and a host runner never has
     either, so every consumer's host job (each reusable workflow's matrix or
     setup job, `smoke-cd-release.yml`) pinned to a dev ref lands here
     (vergil-actions `install-tooling.sh`);
   - a **container image without the vergil setup** (vergil-actions
     `install-tooling.sh`, in `dev_install` and the line and exact-tag
     container paths; vergil-tooling `container_cache.py`).

   After: §6.2.
9. **`install-source: git`** (publish-index's `build-index` job, on an
   `ubuntu-24.04` host runner; vergil-actions `install-tooling.sh`
   `git_source_install`): `uv tool install` with **no `--python`**, where the
   runner's system Python is 3.12. After: §6.2.
10. **The package jobs' driver install** (vergil-actions `package/setup`): its
    `--python` input, default `"3.15"`, the one place epic #607 gave it.
    Unchanged; listed in the §6.4 floor-raise procedure.
11. **The agent VMs** (vergil-tooling `vm_packages.py`). It has two cases:
    - a release ref: the packaged install on `vergil-python`. Unchanged;
    - a dev ref (`vrg-vm update <dev ref>`, `dev_install`): `uv tool install
      --reinstall` (and its `--force --reinstall` retry) with **no
      `--python`**, on a VM whose system `python3` is Ubuntu 24.04's. After:
      §6.2.
12. **The vergil-vm install test** (vergil-vm `tests/test_vergil.sh`): `uv tool
    install` with **no `--python`**. After: §6.2.
13. **Documented host installs** (vergil-tooling `getting-started.md`,
    `consuming-repo-setup.md`, `releasing.md`) **and the `vergil.toml`
    consumer-refresh:** a literal `--python 3.15`. Unchanged; guarded by §6.3.
    Two documented commands name no interpreter: the dev-tree override's
    `uv sync --group dev` in `consuming-repo-setup.md`, and the "Developer
    host" row's `uv tool install --reinstall …` in `releasing.md`'s consumer
    steps. After: both gain `--python <floor>`, and §6.3 guards them.
14. **Docs builds** (vergil-actions `ci-docs.yml` and `cd-docs.yml`;
    vergil-tooling `vrg-container-docs`): `prod-base:latest`'s
    `/usr/local/bin/python3` under `only-system`, which is the core series
    (the base image's `VERGIL_CORE_PYTHON`). Unchanged; it already satisfies
    `>=3.15`.
15. **VM initialisation** (vergil-vm `scripts/vrg-vm-init.sh`, described as
    the vergil-tooling install step in vergil-vm's
    `docs/site/docs/architecture/index.md`): `uv tool install` inside the VM
    with **no `--python`**. After: §6.2.

Rows 5 (its host case), 8, 9, 11 (its dev case), 12 and 15 are the risk.
Today `>=3.12` lets them run on almost any interpreter. Under `>=3.15` they
either find a 3.15, have uv download a managed interpreter, or fail. A job
that exports `UV_PYTHON` for an older family series, or
`UV_PYTHON_PREFERENCE=only-system`, makes them fail.

## 6. Design

### 6.1 The floor (vergil-tooling)

- `requires-python = ">=3.15"`, ruff `target-version = "py315"`, mypy
  `python_version = "3.15"`.
- `uv.lock` is re-locked for the new bound, in the same change.
- The ruff `UP` fixes the new target turns up are applied in the same change.
  They are mechanical, and leaving them out would fail `vrg-validate`.
- No other change is needed for the matrix: rows 3 and 4 shrink through the
  resolver.

### 6.2 Every install names its interpreter

The rule: an install of vergil-tooling passes an explicit interpreter, and a
missing interpreter fails the job loudly. Applied to the rows that break it:

- **The packaged-Python rule.** Where no core Python is present (a host
  runner, a VM before its first packaged install), vergil-tooling runs on the
  **newest packaged series**: the highest `vergil-python<x.y>` the vergil apt
  repository serves. The job sets the repository up the way `host_install`
  already does (`bootstrap_vergil_repository`, with its key checked against
  the pinned fingerprint; `repo_setup.bootstrap` on a VM), lists the
  `vergil-python<x.y>` packages it serves, installs the highest, and passes
  `/opt/vergil/python/<x.y>/bin/python3` explicitly. No list, or no
  `vergil-python` package in it, fails the job. The job then checks that the
  installed tool's (or the synced venv's) interpreter is that series.
  - **Why the newest.** It satisfies every ref's `requires-python`, before
    and after the floor raise and for old dev refs, as long as the floor
    never rises above it, which §6.4 checks. uv checks the ref's
    `requires-python` against the interpreter itself and fails loudly when
    it does not fit, so the job never reads or parses `pyproject.toml`. When
    a new series publishes, these paths move to it at once; it is a tested
    series, because a published runtime is in the window.
  - **Where it applies:** the self repo on a host runner (row 5, host case);
    a dev pin on a host runner (row 8, host case); VM dev installs (row 11,
    dev case, below). `self_repo_install` keeps
    `UV_PYTHON_PREFERENCE=only-system` (issue #1002): the interpreter is a
    system install, not a uv-managed one.
- **The index rebuild (row 9).** `install-source: git` is the one
  uv-managed interpreter. The job rebuilds the vergil apt index, so it
  cannot depend on that index for its interpreter. It installs vergil-tooling
  with an explicit `--python` of a uv-managed interpreter of the floor series
  at the installed tag, and checks the tool's interpreter afterwards. It
  reads the floor with a `git` fetch of `pyproject.toml` at the resolved tag
  (the script already resolves tags with `git ls-remote`). It accepts only a
  bare `>=X.Y` `requires-python` and fails loudly on any other specifier or a
  failed fetch. It runs no project `uv sync`, so the #1002 pin does not
  apply.
- **Container `neither` path (row 8, container case).** Since #649 every
  Vergil image carries both the apt source and a core Python. The audit
  establishes whether any live caller still runs a container job in an image
  without them. If none does, the container `neither` path becomes a refusal
  ("image has neither the vergil apt source nor a core Python") in both
  `install-tooling.sh` and `container_cache.py`, instead of an install on
  whatever uv finds. If one does, the plan names that caller, and the path
  follows the packaged-Python rule.
- **VM dev installs (row 11, dev case).** `vm_packages.dev_install` follows
  the packaged-Python rule. A fresh VM has no `vergil-python` yet, and a VM's
  existing core Python can be older than a later dev ref's floor, so the
  install never relies on whatever is already there.
- **VM initialisation and its tests (rows 12 and 15).** VM initialisation
  stops writing its own install. The plan's audit establishes whether
  anyone runs `vrg-vm-init.sh` outside `vrg-vm create`. If no one does, the
  script's install step is removed, and the install is `vrg-vm create`'s own
  `install_tooling`. If someone does, the script calls `vrg-vm update` for
  the install instead of `uv tool install`. Either way no bash copy of the
  packaged install is written. A uv copy in `~/.local/bin` would shadow the
  packaged `/usr/bin` install without being recorded as a dev ref, and the
  next packaged install would silently replace it.
  - `tests/test_vergil.sh` asserts the packaged install: `vrg-*` resolves
    from `/usr/bin` on a venv whose interpreter lives under
    `/opt/vergil/python/`. Its cleanup removes what that install adds (the
    package, its apt pin and source, and `vergil-python`), not a
    `uv tool uninstall`.
  - vergil-vm `tests/check-vm-init-series.sh`, which asserts the exact
    install command, changes with the script. So does
    `docs/site/docs/operations/build-and-test.md`'s description of
    `test_vergil.sh`, and the architecture doc's description of the install
    step.

### 6.3 Drift guard on the literal host-install commands

The documented host-install commands (row 13) keep a literal `--python 3.15`,
because a human types them. A vergil-tooling test fails when any of these
literals differs from the floor in `requires-python`, and when any
`uv tool install … vergil-tooling` or documented `uv sync` line in these
files has no `--python <floor>` at all:

- the `vergil.toml` consumer-refresh;
- `getting-started.md`, `consuming-repo-setup.md` and `releasing.md`.

The test checks against the **floor**, not the derived core series. The floor
is a value in the repo, so the test stays deterministic. A test against the core
would start failing on every PR the day a 3.16 runtime published, which is the
anti-goal in §2. Raising the floor then forces the documentation update into
the same PR. The plan's audit confirms the exact file list. `package/setup`'s
default (row 10) lives in vergil-actions, so it cannot be checked from
vergil-tooling. It is listed in the floor-raise procedure (§6.4) instead.

### 6.4 Documentation

vergil-tooling's site docs gain a short section, near the existing install and
validation-matrix guides:

- the floor, the tested series and the core series (§3);
- the overlap period when a new series joins the window;
- the floor-raise procedure, in order:
  1. confirm the new series is green;
  2. confirm that the new floor is at or below both core definitions (the
     package's derived runtime, and `core.python` in vergil-containers
     `manifest.yml` with the images republished) and the newest packaged
     `vergil-python` series;
  3. bump `package/setup`'s `python` default in vergil-actions, merge it and
     release vergil-actions so `v2.1` moves. This comes before the
     vergil-tooling release, because a vergil-tooling that declares the new
     floor fails every consumer's package build under the old default;
  4. raise `requires-python` together with the ruff and mypy targets, and
     update the literal host-install commands (the drift guard enforces
     this);
  5. release vergil-tooling.

The documentation-review bookend (vergil-project/vergil-tooling#4578) sweeps
the other repos.

## 7. Delivery order and constraints

- **The install-path fixes land and are released first.** The floor-raise
  PR's own CI runs row 5's host case, and a vergil-tooling release with
  `>=3.15` would break rows 8, 9, 11 (dev case), 12 and 15 wherever they
  still install it. The vergil-actions change (rows 5, 8 and 9) reaches
  vergil-tooling's CI and every consumer only once a vergil-actions release
  moves the `v2.1` tag. That release is a human-gated precondition of the
  floor raise. So are the vergil-tooling changes (`container_cache.py`,
  `vm_packages.dev_install`) and the vergil-vm changes (rows 12 and 15),
  merged and, where they ship, released.
- **The floor raise is one vergil-tooling PR:** §6.1, the §6.3 test and the
  §6.4 docs.
- **Releasing the floor raise** is the human's normal vergil-tooling release.
  Consumers pick it up through the `v2.1` line.
- **The audit is the first implementation task.** It confirms §5, decides §6.2's
  open point (live container `neither` callers) and records them in the plan
  before any code changes.
- **A live check of the changed install paths before the floor raise.** The
  fixtures pin the `v2.1` line, which takes the unchanged packaged path, so
  no fixture run reaches the rewritten host dev-pin path (row 8) or the index
  rebuild (row 9). After the vergil-actions release and before the floor
  raise, a validation task runs both live: a fixture run with a dev
  `[dependencies].vergil` pin, and a dispatched `smoke-publish-index` /
  `build-index` run against the released vergil-actions.
- **Fixture runs.** The narrowed tooling is installed in every fixture's CI. The
  Python fixtures are where a family's `UV_PYTHON` names an older series. The
  plan seeds a fixture run on `python-lib` and `python-app`, after the
  vergil-tooling release that carries the floor raise. The other fixtures are
  seeded only if the audit finds that they reach a changed path.

## 8. Left to the builder

- The exact wording of the `neither` refusal and its tests.
- Where in the site docs the §6.4 section sits, and its title.
- How the §6.3 test finds the literal commands: a parser of the fenced command
  lines or a targeted pattern. Either is acceptable if it fails loudly on a
  missing file or a missing command, never passing on zero matches.
