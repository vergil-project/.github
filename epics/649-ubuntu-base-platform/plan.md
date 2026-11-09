# Ubuntu 24.04 base platform — Implementation Plan

- **Epic:** vergil-project/.github#649
- **Planning issue:** vergil-project/.github#605 (review history is in its
  comments)
- **Status:** draft, in design review

> **For epic workers:** These tasks are filed as **GitHub issues under epic
> vergil-project/.github#649** and implemented one PR each via
> `vergil:issue-implement` (which runs its own TDD cycle). Operational tasks
> (`deployment`, `validation`) run via `vergil:issue-deploy` /
> `vergil:issue-validate`. Steps use checkbox (`- [ ]`) syntax. Each code
> task lists its failing tests and its implementation; the refactor step is
> `issue-implement`'s own and is not repeated per task.

**Goal:** every image in vergil-containers builds on `pkg-ubuntu:24.04`, every
agent VM and GitHub-hosted runner runs Ubuntu 24.04, and Debian is gone.

**Architecture:** `base` becomes a child of #607's `pkg-ubuntu:24.04` through
epic #213's `parent:` field, and every language image a child of `base`. `base`
carries the core runtimes (vergil-python from the vergil apt repository and a
nodejs.org Node under `/opt/vergil/`) and runs its Python tools as uv tools on
them. Each language image installs its own toolchain from a version and hash
pinned in source, which the daily upstream refresh moves. vergil-tooling is
installed from its apt package at cache build. One vergil-containers release
moves the whole tree at once.

**Tech Stack:** Dockerfile templates and fragments, Python 3 (stdlib +
`pyyaml`, run through PEP 723) in vergil-containers; Python in vergil-tooling;
Bash composite actions and workflows in vergil-actions; Lima YAML in
vergil-vm.

**Spec:** `epics/649-ubuntu-base-platform/spec.md`

## Global Constraints

- **The tree:** `ubuntu:24.04` → `pkg-ubuntu:24.04` → `base` → `python`, `go`,
  `rust`, `cpp-clang`, `cpp-gcc`, `ts-node`. No image builds from a Debian or
  upstream language image. Image names and tags are unchanged:
  `ghcr.io/vergil-project/{prefix}-{lang}:{version}`.
- **No package toolchain marker** in `base` or any language image:
  `/etc/vergil/package-toolchain` is removed in `base`'s first layer and
  asserted absent.
- **No `pip install`** in any image, and nothing pip-installed into any
  runtime.
- **Core runtimes** live under `/opt/vergil/`: vergil-python at
  `/opt/vergil/python/<core>/` (`<core>` today 3.15), Node at
  `/opt/vergil/node/<core-major>/` (today 22). Their series are entries in the
  version manifest and move only by reviewed PR. No language family uses a
  path under `/opt/vergil/`.
- **Family runtimes** live at `/opt/python/<v>` and `/opt/node/<major>`. Only a
  family image sets a default `python3` or `node`; `base` sets
  `/usr/local/bin/python3` → the core Python, which promises the standard
  library only.
- **The apt signing key** is checked against primary fingerprint
  `B3A1D804DC03AB036566A4E367861822166ECABE` (vergil-tooling
  `lib/package/orgs.py`) before any vergil apt source is written; a mismatch
  fails the build or job. The keyring package is `vergil-archive-keyring`; the
  permanent source is `/etc/apt/sources.list.d/vergil.sources`.
- **Downloaded toolchains** (python-build-standalone, Go, nodejs.org, Rust,
  `rustup-init`) are pinned in `docker/pins/toolchains.yml` by exact version
  and, where a download is verified, a SHA-256 per architecture (`amd64`,
  `arm64`). The build resolves nothing over the network itself.
- **No silent fallback.** An install path is chosen by an explicit check and
  logged; a failure on the chosen path fails. Nothing falls back after a
  failure.
- **Release freeze:** no vergil-containers release from the `base` rebase
  (Task 7) until the cutover release (D2). Merges to `develop` continue.
- **Cutover PRs stay revertible:** Tasks 7–13 carry no change unrelated to the
  cutover, and each PR description says what reverting it restores, so the
  revert release (spec §4) is a clean revert.
- **Generated files stay generated:** edit `generate.py` or the manifest and
  regenerate; never hand-edit `matrix.json` or a `BEGIN generated` block.
- **Workflow pushes are human:** a delta touching `.github/workflows/**` makes
  the agent push fail. `report-ready` records the state anyway.
- **Validation:** `vrg-container-run -- vrg-validate`, from inside the worktree.

## Review Focus

1. **The install check sees half an environment.** An image with the vergil
   apt source but no vergil-python (or the reverse) must fail with a clear
   message, never take the uv path. Tasks 4 and 5 test both half-states.
2. **A matrix leg tests on the core Python.** In `dev-python:3.12`, `python3`
   must be 3.12, not 3.15. Task 8's build check asserts `python3 --version`
   per series, and that `uv` with `only-system` resolves the family Python.
3. **One architecture's pin is missing or wrong.** A toolchain entry with only
   an `amd64` hash would build on amd64 and fail on arm64 at publish. Task 6's
   `check_pins.py` refuses an entry without both hashes, and the refresh hook
   writes both or fails.
4. **`markdownlint` in an image with no `node` on `PATH`.** npm's bin links
   start with `#!/usr/bin/env node`. Task 7's build check runs `markdownlint
   --version` in `base`, and Task 13's in `ts-node`, so the wrapper is proven
   in both.
5. **A one-liner expects a package in `python3`.** Callers that ran
   `python3 -c 'import yaml'` worked only because today's `base` pip-installs
   pyyaml. Task 3 moves vergil-containers' own; V1's consumer CI run is the
   check for the rest.

## Delivery table

| Task | Repo | What | Blocked-by |
|---|---|---|---|
| Docs | .github | This spec and plan | — (#650, filed) |
| Docs-213 | .github | #213's amendment notes (spec §6.1) | — (#974, filed) |
| Docs-607 | .github | #607's amendment notes (spec §6.2) | — (#975, filed) |
| R1 | vergil-actions | `ubuntu-latest` → `ubuntu-24.04` | — |
| R2 | vergil-containers | `ubuntu-latest` → `ubuntu-24.04` | — |
| R4 | each other repo that uses the label | `ubuntu-latest` → `ubuntu-24.04` | — |
| 1 | vergil-tooling | The `ubuntu-latest` guard | R1, R2, every R4 |
| 2 | .github | Spike and its report | — |
| 3 | vergil-containers | Scripts declare pyyaml (PEP 723) | — |
| 4 | vergil-tooling | Cache build chooses the vergil-tooling install | — |
| 5 | vergil-actions | Setup action chooses the install; packaged on host runners; no pip | — |
| D1 | vergil-tooling, vergil-actions | Release Tasks 4 and 5 | 4, 5 |
| 6 | vergil-containers | Toolchain pins, the manifest `core:` section, and the refresh | #213 T7b, 3 |
| 7 | vergil-containers | `base` on `pkg-ubuntu:24.04` | #607 done, #213 T7b, 2, 3, 6, D1 |
| 8 | vergil-containers | python family | 7 |
| 9 | vergil-containers | go family | 7 |
| 10 | vergil-containers | rust family | 7 |
| 11 | vergil-containers | cpp-clang family | 7 |
| 12 | vergil-containers | cpp-gcc family | 7 |
| 13 | vergil-containers | ts-node family | 7 |
| 14 | vergil-containers | OS-usage table and triage docs | 8–13 |
| V1 | vergil-containers | Pre-release consumer check (validation) | 8–14 |
| D2 | vergil-containers | Cutover release (deployment) | V1 |
| V2 | vergil-containers | Cold-rebuild validation | D2, D3 |
| 15 | vergil-vm | Lima pinned to 24.04 | — |
| D3 | vergil-vm | Release and `vrg-vm rebuild` (deployment) | 15 |

Epic #213's T7 (vergil-project/vergil-containers#515) is outside this table and
must merge before #213 resumes (spec §7 step 3); T7b is filed under #213 as
vergil-project/vergil-containers#773, Blocked-by T7. T7b also gives each matrix entry a `build-args` map
(`{NAME: value}`), which the publish workflow, the PR build and
`docker/build.sh` pass in full, `base` included; today each passes the single
`build-arg=version` pair. Task numbers here are plan-local; spec §7's numbered
steps map to them as: step 1 Docs, step 2 #607, step 3 #213, step 4 Task 2,
step 5 Tasks 3–5 and D1, step 6 Tasks 7–13, step 7 (the consumer fixes)
Tasks 3–5, 13 and the D2 notes with no task of its own, step 8 V1, step 9 D2,
step 10 V2, step 11 Task 15 and D3; Task 14 serves §5.9. Step 12, removing the cutover install
branch, is a follow-on epic (see Bookends). The runner labels (R1–R4, Task 1)
run outside that order.

## Task dependency graph

```text
R1, R2, R4… ──────────────────────────────────→ T1 (guard)
T15 (Lima) → D3 ──────────────────────→ V2 (also after D2, below)
T3 (scripts) ─────────────────────────┐
T4 (tooling install) ─┐               │
T5 (setup action) ────┴→ D1 (release) ┤
#213 T7b, T3 → T6 (toolchain pins) ───┤
T2 (spike) ───────────────────────────┤
#607 done, #213 T7b ──────────────────┴→ T7 (base) → T8…T13 (families) → T14 → V1 → D2 → V2
```

R1, R2, R4, Tasks 2–5 and Task 15 can start at once; Task 6 once #213's T7b
and Task 3 have merged.

---

### Task R1–R4: runner labels

One task per repo that uses the label, filed now: R1 vergil-actions, R2
vergil-containers, and one R4 per other repo. The recount on 2026-11-09 counted
non-comment `runs-on:` lines naming `ubuntu-latest` under `.github/`, on each
repo's default branch, across every repo with a `vergil.toml`:
vergil-actions (42 lines in 19 files), vergil-containers (9 in 5),
`mq-rest-admin-project/mq-rest-admin-dev-environment` (2 in 2),
`mq-rest-admin-{go,java,python}` (1 each), `mq-rest-admin-rust` (2 in 1) and
`wphillipmoore/the-infrastructure-mindset` (1). vergil-tooling,
`logical-minds-foundry/mq-resiliency-lab-for-linux` and the other vergil-project
repos have none, so they get no R task; Task 1's own run on vergil-tooling
proves it. A repo outside vergil-project that cannot be linked under #649 files
its R4 under its own ad-hoc epic; Task 1 lists it as Blocked-by either way.

**Files:** every `.github/workflows/*.yml` and doc page in the repo that names
`ubuntu-latest`.

- [ ] **Step 1: Recount** in the repo: `grep -rnE 'runs-on:.*ubuntu-latest' .github`
  and the current doc examples that show a `runs-on` value, on a fresh
  checkout of the default branch. The task's file list is that output.
- [ ] **Step 2: Replace** every `runs-on` value `ubuntu-latest` with
  `ubuntu-24.04`, and every current doc example the same way. Leave
  `ubuntu-24.04-arm` and any other explicit label alone. Completed plans,
  specs, release notes and comments that mention `ubuntu-latest` stay as
  written.
- [ ] **Step 3: Check** `grep -rnE 'runs-on:.*ubuntu-latest' .github` prints
  nothing (the same rule as Task 1's guard).
- [ ] **Step 4: Validate, commit, report-ready.**
  `vrg-commit --type ci --scope runners --message "pin GitHub-hosted runners to ubuntu-24.04"`.
  The workflow edits mean the human pushes.

---

### Task 1 (vergil-tooling): the `ubuntu-latest` guard

Blocked-by R1, R2 and every R4.

**Files:**

- Modify: `src/vergil_tooling/bin/validate_common.py`
- Test: `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**

- Produces `_check_runner_labels(repo_root: Path) -> int` in
  `validate_common.py`, run by `main` with the other common checks. It scans
  every tracked `.github/workflows/*.yml` and `*.yaml`, and for each scalar
  value anywhere in the parsed YAML that is exactly `ubuntu-latest` (a
  `runs-on` value or list item, a `strategy.matrix` value, a `with:` input)
  prints `<path>:<line>: ubuntu-latest is not allowed; use ubuntu-24.04` and
  returns 1. Comments are ignored, because they are not in the parsed YAML. No
  workflows directory returns 0.

- [ ] **Step 1: Write the failing tests:**
  `test_runner_label_scalar_refused` (`runs-on: ubuntu-latest` → 1, message
  names path and line), `test_runner_label_list_refused`
  (`runs-on: [self-hosted, ubuntu-latest]` → 1),
  `test_runner_label_pinned_passes` (`ubuntu-24.04`, `ubuntu-24.04-arm` → 0),
  `test_runner_label_comment_ignored` (`# runs-on: ubuntu-latest` → 0),
  `test_runner_label_expression_passes` (`runs-on: ${{ matrix.os }}` with
  `os: [ubuntu-24.04]` → 0), `test_runner_label_matrix_value_refused`
  (`os: [ubuntu-latest]` with `runs-on: ${{ matrix.os }}` → 1),
  `test_runner_label_with_input_refused` (`with: {runner: ubuntu-latest}` → 1),
  `test_runner_label_no_workflows` (→ 0).
- [ ] **Step 2: Run them and expect failures.**
- [ ] **Step 3: Implement** with a YAML loader that keeps line numbers (for
  example `yaml.compose` nodes) to walk every scalar.
- [ ] **Step 4: Run them and expect passes;** run `vrg-validate` on
  vergil-tooling itself and expect the check to pass (it has no
  `ubuntu-latest` today).
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope validate --message "refuse ubuntu-latest runner labels"`.
  The release that ships it is the human's, after every R task has merged.

---

### Task 2 (.github): spike

**Files:**

- Create: `epics/649-ubuntu-base-platform/spike.md` (the report)
- Modify: `epics/649-ubuntu-base-platform/plan.md` (one line under this task
  linking the report)

Trial Dockerfiles are built locally and not committed. Each item of spec §8's
spike list is one section of the report, with the command run, the result on
amd64 and on arm64, and a verdict: `as planned`, or `decision needed: <what>`.
The arm64 run is on a Lima VM; the amd64 run on one of the amd64 cloud VMs.

- [ ] **Step 1: Run every spec §8 spike item** on both architectures:
  - the GCC 14 source on noble and the libstdc++ the C++ images use;
  - python-build-standalone for every window series at `/opt/python/<series>`,
    found by uv with `UV_PYTHON_PREFERENCE=only-system`, with
    `/usr/local/bin/python3` resolving, and `base`'s uv tools and vergil-tooling
    still running on the core Python in a `dev-python:3.12` trial;
  - the fingerprint check, `vergil-python3.15` and a pinned vergil-tooling from
    the vergil apt repository, and a dev-ref install with `uv tool install` on
    the core Python; and, by hand, that the two probe paths Task 4 checks
    (`/etc/apt/sources.list.d/vergil.sources`,
    `/opt/vergil/python/*/bin/python3`) are both present in an Ubuntu trial and
    both absent in today's `prod-base` (the check itself is proven by Task 4's
    tests and D1);
  - markdownlint on the core Node with no default `node`, and in a
    `ts-node:24` trial with `npm root -g` = `/usr/lib/node_modules`;
  - the Go, Rust and LLVM sources and the nodejs.org tarballs for every series
    in the manifest;
  - noble's cmake and cppcheck against what the C++ images ship today;
  - one `vrg-validate` run of vergil-tooling in a trial `dev-python`;
  - whether a vergil-tooling package venv needs pip inside vergil-python
    (for spec §5.8 and vergil-project/.github#965);
  - a Trivy scan of the trial core Node, `ts-node` and python-build-standalone
    layers, recording each fixable HIGH or CRITICAL finding in the bundled
    `npm` or `pip` (today `base` upgrades npm, pip and setuptools for such
    findings), so that Tasks 7, 8 and 13 each state their remedy. A bundled
    `npm` or `pip` changes only with its runtime (spec §5.2), so the remedy is
    a newer runtime pin, or a `.trivyignore` entry under that file's rules;
    never an in-place upgrade.
- [ ] **Step 2: Write `spike.md`.** Any `decision needed` stops the task for
  the human before the report merges; the decision is recorded in the report.
- [ ] **Step 3: Validate, commit, report-ready.**
  `vrg-commit --type docs --scope epic-649 --message "record the Ubuntu 24.04 spike"`.

---

### Task 3 (vergil-containers): scripts declare pyyaml

**Files:**

- Modify the entry points, the scripts a caller runs:
  `docker/pins/check_pins.py`, `docker/pins/generate_catalog.py`,
  `docker/pins/report_exposure.py`, `docker/versions/generate.py`,
  `docker/versions/lifecycle/check.py`, `docker/versions/lifecycle/publish.py`.
  Library modules they import (`manifest.py`, `adapters/go.py`, …) get no
  block: PEP 723 applies only to the script `uv run` is given.
- Modify the callers: `scripts/bin/validate-custom`,
  `docker/pins/upstream-refresh.sh`, `.github/workflows/ops.yml` (including
  line 42, `python3 report_exposure.py` under `working-directory: docker/pins`),
  and any other caller of an entry point.
- Create: `docker/versions/test_entry_points.py` (run by `validate-custom`'s
  `docker/versions` pytest)

**Interfaces:**

- Produces: each entry point starts with a PEP 723 block naming every
  third-party module in its full import closure, including what it reaches
  through local modules: `pyyaml` for each, plus `packaging` for
  `report_exposure.py` (it imports `packaging.version`). Each is invoked as
  `uv run <script>` with no `--python` and no `--with`.

- [ ] **Step 1: List** the entry points above and every caller of each, by
  script name rather than path prefix (`grep -rnE 'check_pins|generate_catalog|report_exposure|generate\.py|lifecycle/check|publish\.py' scripts docker .github`).
  For each entry point, record its third-party import closure in the PR
  description.
- [ ] **Step 2: Write the failing tests** in `test_entry_points.py`, over an
  `ENTRY_POINTS` table (entry point → its declared dependencies, from Step 1):
  `test_each_entry_point_has_a_pep723_block` (the block parses, and its
  `dependencies` equal the table's), `test_declared_deps_cover_imports` (every
  third-party top-level module the entry point imports, directly or through a
  local module under `docker/`, found with `ast`, maps to a declared
  dependency; `yaml` → `pyyaml`), and `test_no_caller_runs_an_entry_point_with_python3`
  (no `python3 <entry point>` in `scripts/`, `docker/` or `.github/`). Run
  them and expect failures.
- [ ] **Step 3: Add the PEP 723 block** to each entry point and change each
  caller to `uv run`; run the tests and expect passes.
- [ ] **Step 4: Check** in today's `prod-base`, with pyyaml and packaging
  absent from the interpreter, that `scripts/bin/validate-custom`,
  `uv run docker/pins/report_exposure.py --no-probe` (from `docker/pins`, as
  `ops.yml` runs it), `uv run docker/versions/lifecycle/check.py` and
  `uv run docker/versions/lifecycle/publish.py --help` all pass:
  `docker run --rm -v "$PWD":/w -w /w ghcr.io/vergil-project/prod-base:latest bash -c 'python3 -m pip uninstall -y pyyaml packaging && …'`.
  The nightly `pin-exposure` job is `continue-on-error` (vergil-project/.github#972),
  so this step is the only check that `report_exposure.py` still runs end to
  end; Step 2's tests guard its declared dependencies in PR CI.
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type refactor --scope scripts --message "declare pyyaml in each script and run them with uv run"`.
  `ops.yml` means the human pushes.

---

### Task 4 (vergil-tooling): the cache build chooses the vergil-tooling install

**Files:**

- Modify: `src/vergil_tooling/lib/container_cache.py` (the `uv_install` lines
  in the setup-command builder)
- Modify: `src/vergil_tooling/lib/vm_packages.py` (expose the pin helpers it
  already has, without changing their behaviour)
- Test: `tests/vergil_tooling/test_container_cache.py`

**Interfaces:**

- Consumes `vm_packages.classify_ref(ref) -> Literal["line", "exact", "dev"]`,
  `vm_packages.apt_pin(ref) -> str`, the deb version helper and
  `_candidate_matches(ref, candidate)` (both renamed without their
  underscore), and the candidate check `vm_packages.packaged_install` does
  with `apt-cache policy` as the reference behaviour.
- Produces `vergil_tooling_setup_command(tag: str) -> str` in
  `container_cache.py`, replacing the inline `uv_install`. For a `dev` ref it
  returns today's `uv tool install --quiet 'vergil-tooling @ git+…@<tag>'`
  unchanged. For a line or exact ref it returns a shell snippet that:
  1. if `/etc/apt/sources.list.d/vergil.sources` exists **and** a
     `/opt/vergil/python/*/bin/python3` exists: prints
     `vergil-tooling: packaged install (<tag>)` to stderr, writes
     `apt_pin(tag)` to `/etc/apt/preferences.d/vergil-tooling`, runs
     `apt-get update`, reads the candidate from `apt-cache policy
     vergil-tooling` and checks it against the tag as `packaged_install` does
     (prefix `X.Y.` for a line, exactly `X.Y.Z-1` for an exact ref), exiting 1
     with the VM path's message on a mismatch (a pin that matches nothing
     leaves apt's normal candidate in place); then
     `apt-get install -y vergil-tooling[=<deb version>]`, then
     `vrg-whoami --version` (the cache-build check: `vrg-*` runs from the
     package);
  2. if neither exists: prints `vergil-tooling: uv tool install from git
     (image without the vergil apt repository)` and runs today's `uv tool
     install`;
  3. if exactly one exists: prints `vergil-tooling: the image has <one> but not
     <other>; refusing to guess` and exits 1.
  `_compose_setup` joins fragments with `&&` into one `bash -c` and sets no
  `-e`, so the snippet runs as its own `bash -euo pipefail -c '…'`, and a
  failure on the chosen path fails the build.

- [ ] **Step 1: Write the failing tests:**
  `test_setup_dev_ref_is_uv_install` (exact string as today),
  `test_setup_line_ref_branches_on_environment` (the snippet contains both
  probes, both log lines and `apt-get install -y vergil-tooling`),
  `test_setup_exact_ref_pins_version` (`vergil-tooling=<deb version>`),
  `test_setup_half_environment_refuses` (run the snippet with `bash` against a
  temp root via a `VRG_CACHE_ROOT` prefix seam: source present, no runtime →
  exit 1 and the refusing message; the reverse → exit 1),
  `test_setup_line_candidate_mismatch_refuses` (stub `apt-cache policy`
  reports a `2.2.0-1` candidate for tag `v2.1` → exit 1, nothing installed),
  `test_setup_exact_candidate_mismatch_refuses` (candidate `2.1.4-1` for tag
  `v2.1.3` → exit 1), `test_setup_apt_update_failure_fails` (a failing
  `apt-get update` stub → non-zero exit),
  `test_setup_self_repo_unchanged` (vergil-tooling's own repo installs
  nothing).
- [ ] **Step 2: Run them and expect failures.**
- [ ] **Step 3: Implement** `vergil_tooling_setup_command`, and use it in place
  of `uv_install`.
- [ ] **Step 4: Run them and expect passes.**
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope container --message "install vergil-tooling from its package where the image has the vergil apt repository"`.

---

### Task 5 (vergil-actions): the setup action chooses the install

**Files:**

- Modify: `actions/shared/setup/vergil/action.yml`
- Create: `actions/shared/setup/vergil/install-tooling.sh`
- Create: `actions/shared/setup/vergil/tests/install-tooling.test.sh`
- Modify: `.github/workflows/smoke-setup-vergil.yml` (run the new harness)
- Modify: `.github/workflows/publish-index.yml` (`build-index` sets
  `install-source: git`)

**Interfaces:**

- Consumes `read-pin.sh` (unchanged) for the tag.
- Produces `install-tooling.sh <tag>`, called by the "Install vergil-tooling"
  step, replacing the "Check for uv" and "Install uv" steps (today a
  `pip install` of uv) and the inline `uv tool install`.
  - **Pinned uv, wherever a path needs uv and it is absent:** install uv from
    its release tarball for `UV_VERSION`, checked against `UV_SHA256_AMD64` /
    `UV_SHA256_ARM64`, all three pinned at the top of `install-tooling.sh`,
    into `$RUNNER_TEMP/uv/bin` added to `GITHUB_PATH`. Never `pip`. The
    GitHub-hosted runner image ships no uv (ubuntu-24.04 image
    `20261101.360.1`).
  - **Self repo (vergil-tooling's own CI):** the pinned uv if absent, then
    today's `UV_PYTHON_PREFERENCE=only-system uv sync --frozen --group dev`.
    Its host jobs (`ci-test.yml` and `ci-audit.yml` `matrix`, `cd-release.yml`
    `setup`, `package-matrix` and `package-sign`, `publish-index.yml`
    `build-index`) run with no container, because they choose the image later
    jobs use.
  - **Index publisher:** `publish-index.yml`'s `build-index` never installs
    from the vergil apt repository it rebuilds. It sets the action's new input
    `install-source: git` (default `auto`), which makes `install-tooling.sh`
    take the git path with the pinned uv whatever the tag, and log why; a
    container is not an option, since every image is built with that
    repository.
  - **Container job, line or exact tag:** the same three-way check as Task 4,
    with the same log lines, the same refusal and the same `apt-cache policy`
    candidate check.
  - **Host job** (`[ ! -f /.dockerenv ]`, the
    plan's test for "not in a container"), line or exact tag: set up the vergil
    apt repository as vergil-tooling's `repo_setup.bootstrap` does (the
    reference behaviour; this shell copy cannot call it), then install the pin,
    check the candidate as Task 4 does, and install `vergil-tooling` with
    `sudo apt-get`. The bootstrap steps, in order:
    1. remove any `vergil-bootstrap.sources` and bootstrap key a broken earlier
       run left, before any apt call;
    2. if `vergil-archive-keyring` is installed with its source
       (`/etc/apt/sources.list.d/vergil.sources`) and key in place, skip to
       step 7, after checking the installed key as in step 4;
    3. read `VERSION_CODENAME` from `/etc/os-release`, and refuse one that is
       not a registered deb target (vergil-tooling `lib/package/targets.py`:
       `noble` and `resolute` today);
    4. download the key from `https://vergil-project.github.io/packages/keys/vergil.asc`
       and read it with `gpg --show-keys --with-colons`. Refuse unless it holds
       exactly one primary key (one `pub` record), then compare that key's
       fingerprint (the `fpr` line after `pub`) with
       `B3A1D804DC03AB036566A4E367861822166ECABE`, and fail on a mismatch (the
       rule of `repo_setup.parse_primary_fingerprint`);
    5. write the key to `/etc/apt/keyrings/vergil-bootstrap.asc` and
       `/etc/apt/sources.list.d/vergil-bootstrap.sources` (`Types: deb`,
       `URIs: https://vergil-project.github.io/packages/deb`,
       `Suites: <codename>`, `Components: main`, `Signed-By:` that key);
    6. `apt-get update`, install `vergil-archive-keyring`, then remove the
       bootstrap source and key: left next to the keyring's own source, they
       make apt reject the source list (conflicting `Signed-By`);
    7. `apt-get update`, which must pass.
  - **Dev tag, host or container:** the pinned uv if absent, then today's
    `uv tool install`.
  - Every path prints which one it took.

- [ ] **Step 1: Write the harness** in the framework-free style of the action's
  existing tests (stub `apt-get`, `gpg`, `curl`, `uv` first on `PATH`;
  `VRG_ROOT` prefix seam for the probed paths). Cases: `container_packaged`,
  `container_uv`, `container_half_source_only` (exit 1),
  `container_half_runtime_only` (exit 1), `host_packaged`,
  `host_fingerprint_mismatch` (exit 1, nothing installed),
  `host_two_primary_keys_first_matches` (exit 1, nothing installed),
  `host_no_primary_key` (exit 1), `host_candidate_mismatch` (exit 1),
  `host_packaged_leaves_no_bootstrap_source` (after `host_packaged`, no
  `vergil-bootstrap.sources` or bootstrap key remains),
  `host_stale_bootstrap_source_removed` (one left by an earlier run is gone
  before the first `apt-get`), `host_keyring_already_installed` (no bootstrap;
  the installed key is checked), `host_unregistered_codename` (exit 1),
  `container_candidate_mismatch` (exit 1),
  `self_repo_host_installs_pinned_uv`, `install_source_git_line_tag` (a line
  tag with `install-source: git` takes the git path and never runs
  `apt-get`), `dev_tag_installs_pinned_uv`,
  `dev_tag_uv_hash_mismatch` (exit 1).
- [ ] **Step 2: Run** `bash actions/shared/setup/vergil/tests/install-tooling.test.sh`
  and expect failures.
- [ ] **Step 3: Implement** `install-tooling.sh` (`set -euo pipefail`,
  shellcheck-clean) and wire `action.yml`.
- [ ] **Step 4: Run the harness and expect all cases to pass.**
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope setup --message "install vergil-tooling from its package, and never install uv with pip"`.
  The workflow edit means the human pushes.

---

### Task D1 (deployment, vergil-tooling and vergil-actions): release the install paths

`--kind deployment`, Blocked-by Task 4 and Task 5.

**Preconditions (human-attested):** vergil-tooling released with Task 4 and
vergil-actions released with Task 5 (the `v2.1` line moved).

**Procedure:**

1. In a consumer repo's next CI run, the container job's setup log shows
   `vergil-tooling: uv tool install from git (image without the vergil apt repository)`,
   and a host job's shows the packaged path.
2. In a consumer repo on the host, remove its cached image first (the cache
   key covers neither the setup command nor the vergil-tooling version, so an
   older cached image would skip the new code), then
   `vrg-container-run -- vrg-validate` passes, and its cache-build log shows
   the uv path.

**Acceptance:** both pass. Record `Outcome: SUCCESS`.

---

### Task 6 (vergil-containers): toolchain pins and their refresh

**Files:**

- Create: `docker/pins/toolchains.yml`
- Create: `docker/pins/resolve_toolchains.py`
- Create: `docker/pins/test_toolchains.py`
- Modify: `docker/pins/check_pins.py`, `docker/pins/generate_catalog.py`
  (`CATALOG.md` gains a toolchains table), `docker/pins/upstream-refresh.sh`
  (runs `uv run docker/pins/resolve_toolchains.py --write`, then
  `uv run docker/versions/generate.py`, then `generate_catalog.py`: the
  matrix embeds the pins, and PR CI's `generate.py --check` fails on a stale
  `matrix.json`)
- Modify: `docker/versions/generate.py` (emits the pins as build args)
- Modify: `docker/versions/manifest.yml` (a new `core:` section
  `{python: "3.15", node: "22"}`), `docker/versions/manifest.py` (`_TOP_KEYS`,
  the loader, and `dump_manifest`, which emits `core:`),
  `docker/versions/lifecycle/apply.py` (`apply_change` keeps `core`; today it
  rebuilds the manifest from `families` alone)
- Test: `docker/versions/test_manifest.py`, `docker/versions/test_apply.py`

Blocked-by #213's T7b for the `build-args` map it adds to each matrix entry,
and Task 3, whose PEP 723 blocks make the refresh hook's plain `uv run` work
(both tasks edit `upstream-refresh.sh`). `resolve_toolchains.py` is a new
entry point, so it starts with its own PEP 723 block (`pyyaml`) and joins
Task 3's `ENTRY_POINTS` table.

**Interfaces:**

- Produces the manifest's `core:` section and its loader: `core.python` and
  `core.node` are the core series. They have their own pins, separate from
  the family windows (spec §5.2.1): neither has to be a family series, and a
  family dropping a series never forces a core move. The loader checks only
  their shape (`X.Y` for Python, a major for Node). Each is checked against
  its own source: `check_pins.py` requires a `toolchains.node` entry for the
  core Node major, and the core Python needs no pin, because Task 7's build
  installs `vergil-python<series>` from the vergil apt repository and fails if
  vergil-python does not publish it. A core move is a reviewed PR that edits `core:`;
  in this epic no lifecycle automation opens it (vergil-project/.github#978
  covers that follow-on). `check_pins.py` here and Task 7's generator both
  read the section.
- **One resolver per upstream.** `resolve_toolchains.py` holds one fetcher
  per upstream (python-build-standalone, go.dev, nodejs.org, the Rust channel
  manifest, rustup), each taking an injectable `fetch`. Tasks 8–13's adapter
  probes import these fetchers rather than writing their own, so each
  upstream is fetched and parsed in one place (spec §5.4).

- `toolchains.yml` (schema 1):

  ```yaml
  schema: 1
  toolchains:
    python:                       # family key in manifest.yml
      "3.15": {version: "3.15.1", release: "20261101", sha256: {amd64: "…", arm64: "…"}}
    go:
      "1.27": {version: "1.27.3", sha256: {amd64: "…", arm64: "…"}}
    node:                         # serves ts-node and the core Node
      "24": {version: "24.11.1", sha256: {amd64: "…", arm64: "…"}}
    rust:
      "1.93": {version: "1.93.1"}
  tools:
    rustup-init: {version: "1.28.2", sha256: {amd64: "…", arm64: "…"}}
  ```

  `python.release` is the python-build-standalone release tag; the other
  values are examples of shape, not pins.
- `resolve_toolchains.py --write` rewrites `toolchains.yml` to the newest
  patch of each listed series (python-build-standalone's newest release with an
  install-only build of the series for both architectures, with hashes from
  its `SHA256SUMS`; go.dev's JSON release list with its `sha256`; nodejs.org's
  `SHASUMS256.txt`; the Rust channel manifest for the newest `x.y.z` of the
  series; `rustup-init.sha256` per architecture). Any lookup that fails exits
  non-zero, so the refresh pushes nothing.
- `check_pins.py` fails when a `python`, `go`, `rust` or `ts-node` manifest
  series, or the core Node major, has no entry; when an entry exists for a
  series that is neither a manifest family series nor the core Node major; or
  when a `python`, `go`, `node` or `rustup-init` entry lacks either
  architecture's hash. The core Node major counts as a listed series in its
  own right, so its entry is never an orphan.
- `generate.py` emits, into each matrix entry's `build-args` map (T7b),
  `TOOLCHAIN_VERSION`, `TOOLCHAIN_SHA256_AMD64`, `TOOLCHAIN_SHA256_ARM64` (and
  `PBS_RELEASE` for python; `RUSTUP_INIT_VERSION` and its two hashes for rust).
  No template reads them until Tasks 7–13.

- [ ] **Step 1: Write the failing tests** in `test_toolchains.py`:
  `test_entry_needs_both_arches`, `test_missing_series_fails`,
  `test_orphan_series_fails`, `test_rust_needs_no_hash`,
  `test_resolve_python_picks_install_only_both_arches` (fixture release list
  where the newest release lacks arm64 → the older one is chosen),
  `test_resolve_failure_exits_nonzero`, `test_write_round_trips` (load, write,
  load is identical), `test_generate_emits_build_args` (each family entry's
  `build-args` map holds its toolchain keys and values),
  `test_core_node_pin_not_orphan` (core Node 22 with no family series 22: its
  entry passes), `test_core_node_major_needs_pin`; in `test_manifest.py`, `test_manifest_core_section` (`core.python` and
  `core.node` load; a core series outside every family window is accepted) and
  `test_dump_round_trips_core` (`load(dump(m)) == m` with `core` set); and in
  `test_apply.py`, `test_apply_change_keeps_core`.
- [ ] **Step 2: Run them and expect failures.**
- [ ] **Step 3: Implement** the schema loader, the resolvers (each with an
  injectable `fetch`, as `resolve_latest.py` does), the checks and the
  build-arg emission; seed `toolchains.yml` by running the resolver.
- [ ] **Step 4: Run them and expect passes;** `uv run docker/pins/resolve_toolchains.py`
  (no `--write`) prints no change right after seeding. Then, on a scratch
  branch, hand-edit one pin back a patch release and run
  `docker/pins/upstream-refresh.sh`: afterwards `generate.py --check` and
  `generate_catalog.py --check` both pass.
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope pins --message "pin the language toolchains by version and hash, refreshed daily"`.

---

### Task 7 (vergil-containers): `base` on `pkg-ubuntu:24.04`

Blocked-by #607's completion (`prod-pkg-ubuntu:24.04` published), #213's T7b
(`parent:` and the tiered publish), Task 2, Task 3, Task 6 and D1. Starts the
release freeze.

**Files:**

- Modify: `docker/base/Dockerfile.template`
- Create: `docker/common/vergil-apt.dockerfile` (key check, keyring, source)
- Create: `docker/common/core-python.dockerfile`
- Create: `docker/common/core-node.dockerfile` (replaces
  `node-markdownlint.dockerfile` and `nodesource-node.dockerfile` for `base`)
- Create: `docker/common/core-tools.dockerfile` (uv tools)
- Create: `docker/base/checks-invariants.sh` (the checks every image keeps),
  `docker/base/checks-defaults.sh` (one function per `base` default, so a
  family reruns the ones it does not change), both copied into the image, and
  `docker/base/checks.sh` (runs the invariants and every default; the last
  `RUN`)
- Modify: `docker/versions/manifest.yml` (`base` gets `parent: pkg-ubuntu
  24.04`), `docker/versions/generate.py` (emit `CORE_PYTHON`,
  `CORE_NODE_VERSION`, `CORE_NODE_SHA256_AMD64`, `CORE_NODE_SHA256_ARM64`
  into `base`'s `build-args` map, from Task 6's `core:` section)
- Test: `docker/versions/test_manifest.py`, `docker/versions/test_generate.py`

**Interfaces:**

- `base` is `FROM ${PARENT_IMAGE}` (the build arg T7b sets) and its first
  layer is `rm -f /etc/vergil/package-toolchain`.
- `vergil-apt.dockerfile`: runs Task 5's host bootstrap steps 1–7 in one
  layer, without `sudo` (reference: vergil-tooling's `repo_setup.bootstrap`),
  with the pin as
  `ARG VERGIL_APT_FINGERPRINT=B3A1D804DC03AB036566A4E367861822166ECABE` and
  the codename from the image's `/etc/os-release` (`noble`). If the parent
  already ships `vergil-archive-keyring`, step 2's installed-keyring case
  applies. It leaves `/etc/apt/sources.list.d/vergil.sources` and no
  bootstrap source or key.
- `core-python.dockerfile`: `apt-get install vergil-python${CORE_PYTHON}`;
  `ln -s /opt/vergil/python/${CORE_PYTHON}/bin/python${CORE_PYTHON} /usr/local/bin/python3`.
- `core-node.dockerfile`: unpacks the pinned nodejs.org tarball for
  `TARGETARCH` at `/opt/vergil/node/<major>` after checking its hash; installs
  markdownlint-cli with that `npm` into `/opt/vergil/node/<major>`; writes
  `/usr/local/bin/markdownlint` as
  `exec /opt/vergil/node/<major>/bin/node /opt/vergil/node/<major>/lib/node_modules/markdownlint-cli/markdownlint.js "$@"`.
  No `node`, `npm` or `npx` on `PATH`. If the spike (Task 2) found a fixable
  HIGH or CRITICAL finding in the tarball's bundled `npm`, the remedy is the
  one the spike report decided: a newer core Node pin, or a `.trivyignore`
  entry under that file's rules. The bundled `npm` is never upgraded in
  place; the PR names which remedy it took.
- `core-tools.dockerfile`: `uv tool install --python /opt/vergil/python/${CORE_PYTHON}/bin/python${CORE_PYTHON}`
  for `yamllint`, `ansible-lint`, `semgrep` (with build-essential installed and
  purged in the same layer, as today), and
  `mkdocs --with mkdocs-material --with-executables-from mike` (`--with` alone
  links no `mike` executable; the pinned uv must support the flag, as uv
  0.13.7 does), with `UV_TOOL_BIN_DIR=/usr/local/bin`. No `pip install`
  anywhere in the template or its fragments.
- `checks-invariants.sh` asserts what every image keeps:
  - marker absent; `/etc/os-release` `VERSION_ID="24.04"`;
  - `/opt/vergil/python/${CORE_PYTHON}/bin/python${CORE_PYTHON}` is the core
    series and imports `tomllib`;
  - `/usr/local/bin/python3` exists and imports `tomllib` (docs deploy and
    consumers call it in every image, spec §5.5);
  - each uv tool's `--version`, `mike --help`, a material-themed fixture
    `mkdocs build`, and `markdownlint --version`;
  - each core tool runs on its core runtime: every uv tool environment's
    interpreter (its `pyvenv.cfg` `home`, or the shebang of its
    `/usr/local/bin` entry) is under `/opt/vergil/python/${CORE_PYTHON}/`, and
    the `markdownlint` wrapper names `/opt/vergil/node/<core-major>/bin/node`.
    This holds in `dev-python:3.15` too, where the family series equals the
    core.
- `checks-defaults.sh` holds `base`'s defaults, which a family may change:
  `check_python3_is_core` (`/usr/local/bin/python3` is `CORE_PYTHON`'s
  series), `check_no_node` (`! command -v node`) and `check_no_pip`
  (`! command -v pip`).
- `checks.sh` runs `checks-invariants.sh` and all three defaults.

- [ ] **Step 1: Write the failing tests:** `test_generate_emits_core_args`
  (`base`'s `build-args` map holds the four core keys), and
  `test_base_parent_is_pkg_ubuntu`.
- [ ] **Step 2: Run them and expect failures; implement the manifest and
  generator changes; run them and expect passes.**
- [ ] **Step 3: Write the fragments, template and `checks.sh`;** drop uv's pip
  install (uv comes from the parent) and `nfpm.dockerfile` from `base`.
- [ ] **Step 4: Build** `docker/build.sh base` on amd64 and arm64 and expect
  `checks.sh` to pass on both. Then build the `vergil-apt.dockerfile` stage
  against two local key fixtures, one holding the pinned key plus a second
  primary key and one with no primary key, and expect both to fail.
  `checks-invariants.sh` also asserts that no
  `/etc/apt/sources.list.d/vergil-bootstrap.sources` or
  `/etc/apt/keyrings/vergil-bootstrap.asc` exists and that `apt-get update`
  passes.
- [ ] **Step 5: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope base --message "build base on pkg-ubuntu 24.04 with vergil-python and a side-by-side Node"`.

---

### Tasks 8–13 (vergil-containers): the language families

Each Blocked-by Task 7, one PR per family. Each family's image becomes
`FROM ${PARENT_IMAGE}` with `parent: base` in the manifest, drops every
fragment `base` already installs and `python-support.dockerfile`, installs its
toolchain from Task 6's build args (verifying the hash for `TARGETARCH`), moves
its lifecycle adapter's probe (spec §5.4) with its tests and fixtures, and ends
with a `checks.sh` that runs `base`'s `checks-invariants.sh`, reruns each
`checks-defaults.sh` default the family does not change, and asserts its
toolchain's version and the family-specific check below. Only python changes
`python3` and `pip`, and only ts-node changes `node`, so:

| Family | Defaults rerun |
|---|---|
| python | `check_no_node` |
| ts-node | `check_python3_is_core`, `check_no_pip` |
| go, rust, cpp-clang, cpp-gcc | all three |

Tasks 8 and 13 also apply the remedy the spike report decided for any fixable
HIGH or CRITICAL finding in the bundled `pip` (python-build-standalone) or
`npm` (nodejs.org): a newer series pin or a `.trivyignore` entry under that
file's rules, never an in-place upgrade. Or they record that none was needed.

| Task | Family | Toolchain install | Family-specific check | Adapter probe after |
|---|---|---|---|---|
| 8 | python | python-build-standalone install-only tarball at `/opt/python/<v>`; `python3`, `python`, `pip` in `/usr/local/bin` → it | `python3 --version` is the series; `UV_PYTHON_PREFERENCE=only-system uv python find <v>` is `/opt/python/<v>/bin/python3`; no pytest-xdist in the image | python-build-standalone release with both arches; `runtime_published` unchanged |
| 9 | go | `go<version>.linux-<arch>.tar.gz` at `/usr/local/go` | `go version` | go.dev listing has `go<v>.x` |
| 10 | rust | pinned `rustup-init` (hash checked), then `rustup toolchain install <version> --profile minimal` plus today's components | `rustc --version` is the pinned release | Rust channel manifest for the series |
| 11 | cpp-clang | apt.llvm.org `llvm-toolchain-noble-<v>` | `clang-<v> --version`; the libstdc++ the spike chose; `clang-format-<t>` and `clang-tidy-<t>` at `CLANG_TOOLS_VERSION` | apt.llvm.org `noble` listing |
| 12 | cpp-gcc | the spike's GCC 14 source | `gcc-14 --version` is a released 14.x; `clang-format-<t>` and `clang-tidy-<t>` at `CLANG_TOOLS_VERSION` | the spike's source has `gcc-<n>` for noble |
| 13 | ts-node | nodejs.org tarball at `/opt/node/<major>`; `node`, `npm`, `npx` in `/usr/local/bin`; npm prefix `/usr` | `npm root -g` = `/usr/lib/node_modules`; `markdownlint --version` still runs on the core Node | nodejs.org index has the major with both tarballs |

Both C++ families include `common/cpp-analysis.dockerfile`, whose apt.llvm.org
source is hardcoded to Debian (`deb http://apt.llvm.org/trixie/
llvm-toolchain-trixie-${CLANG_TOOLS_VERSION} main`). Whichever of Tasks 11 and
12 merges first moves it to `apt.llvm.org/noble/`,
`llvm-toolchain-noble-${CLANG_TOOLS_VERSION}`; the other rebases onto it.

Each adapter probe that reads an upstream Task 6 pins (python-build-standalone,
go.dev, the Rust channel manifest, nodejs.org) imports that upstream's fetcher
from `docker/pins/resolve_toolchains.py` instead of fetching it itself.

Each task's steps:

- [ ] **Step 1: Update the adapter's tests and fixtures** (spec §5.4 names
  them: `test_adapters_report_only.py`, `test_adapters_python_go.py`,
  `test_sources.py`, the trixie fixtures, `test_generate.py`'s base-image doc
  rows) to the new probe, and expect failures.
- [ ] **Step 2: Implement the adapter change** and expect passes.
- [ ] **Step 3: Rewrite the template and its `checks.sh`;** build every
  manifest series of the family on amd64 and arm64 and expect the checks to
  pass.
- [ ] **Step 4: Validate, commit, report-ready.**
  `vrg-commit --type feat --scope <family> --message "build <family> images on base, with a pinned toolchain"`.

Task 8 also confirms, before dropping pytest-xdist, that every Python consumer
declares it in its dev dependencies (`grep -l pytest-xdist */pyproject.toml`
over the consuming repos), and lists any that does not in the PR.

---

### Task 14 (vergil-containers): OS-usage table and triage docs

Blocked-by Tasks 8–13.

**Files:**

- Create: `docs/site/docs/operations/os-usage.md` (spec §5.9), added to the
  site nav
- Modify: `docs/site/docs/operations/trivy-triage.md` (drop "Debian fix";
  add spec §5.8's ownership rule)
- Modify: `docker/cpp-clang/README.md`, `docker/ts-node/README.md` (the
  image-structure sections the old fragment convention described)

- [ ] **Step 1: Write the table** with the rows spec §5.9 lists, each marked
  `required` or `choice`, and the runners row as `ubuntu-24.04` /
  `ubuntu-24.04-arm`.
- [ ] **Step 2: Update the triage page and READMEs.**
- [ ] **Step 3: Validate, commit, report-ready.**
  `vrg-commit --type docs --scope os --message "publish the OS-usage table"`.

---

### Task V1 (validation, vergil-containers): pre-release consumer check

`--kind validation`, Blocked-by Tasks 8–14. Spec §8 is the procedure.

**Procedure:**

1. In vergil-tooling and one consumer per family, including one that declares
   `[container].system-packages`, push a throwaway branch setting
   `container-prefix: dev` on the reusable workflow calls, including one docs
   deploy dry run. Their CI is green.
2. `DOCKER_DEV_IMAGE=ghcr.io/vergil-project/dev-<family>:<v> vrg-container-run -- vrg-validate`
   in each of those repos passes.
3. In `dev-base`, `scripts/bin/validate-custom` and
   `uv run docker/versions/lifecycle/check.py` pass for vergil-containers.
4. On an amd64 cloud VM, `apt-get install` of one package in `dev-base`
   succeeds through the inherited mirror config.
5. List the CI paths that do not pass `container-prefix` through (jobs that
   name a `prod-` image directly), and run each one's commands by hand in a
   `dev-base` container (`docker run --rm -v "$PWD":/w -w /w ghcr.io/vergil-project/dev-base:latest …`).
   Record the list and each result.
6. Delete the throwaway branches.

**Acceptance:** every step passes. Record `Outcome: SUCCESS`.

---

### Task D2 (deployment, vergil-containers): the cutover release

`--kind deployment`, Blocked-by V1. Ends the freeze.

**Preconditions (human-attested):** the vergil-containers release is cut, its
notes naming the revert path (spec §4), the fail-forward path, the accepted
break of `npm install -g` build-commands outside `ts-node`
(`mnemosys-project/melete`) and the `vrg-vm rebuild` for existing VMs.

**Procedure:**

1. `cd-docker-publish` published every `prod-` image in tier order.
2. `docker run --rm ghcr.io/vergil-project/prod-<image> cat /etc/os-release`
   shows `VERSION_ID="24.04"` for every image.

**Acceptance:** both pass. Record `Outcome: SUCCESS`.

---

### Task V2 (validation, vergil-containers): cold rebuild

`--kind validation`, Blocked-by D2 and D3 (its VM check needs the rebuilt
VMs). The procedure is spec §8's validation list:
a no-cache rebuild of the full tree in tier order on both architectures; every
image on Ubuntu 24.04 without the marker; every consuming repo's PR CI green on
the new `prod-` images; a fresh `vrg-vm create` passing `tests/test_base.sh`
(after D3); the OS-usage table matching what was observed. Record
`Outcome: SUCCESS`.

---

### Task 15 (vergil-vm): Lima pinned to 24.04

**Files:**

- Modify: `templates/agent.yaml.skel` (`template:_images/ubuntu-lts` →
  `template:_images/ubuntu-24.04`, header comment made accurate)
- Modify: `templates/provision/40-profile.sh` (a comment on the racc seed:
  it matters again on 26.04, #651)
- Regenerate: `templates/agent.yaml`, which `scripts/build-template.sh` builds
  from the skel and `templates/provision/*.sh`; `tests/check-template-generation.sh`
  fails a hand edit

- [ ] **Step 1: Edit the skel and the comment, then run
  `scripts/build-template.sh`.**
- [ ] **Step 2: `vrg-vm create`** a scratch VM from the branch and run
  `tests/test_base.sh`; expect `VERSION_ID="24.04"` to pass.
- [ ] **Step 3: Validate, commit, report-ready.**
  `vrg-commit --type fix --scope lima --message "pin agent VMs to Ubuntu 24.04"`.

### Task D3 (deployment, vergil-vm): release and rebuild

`--kind deployment`, Blocked-by Task 15. **Precondition (human-attested):** the
vergil-vm release is cut, with notes saying `vrg-vm update` does not change the
OS. **Procedure:** `vrg-vm rebuild` each existing VM; `tests/test_base.sh`
passes on each. Record `Outcome: SUCCESS`.

---

### Bookends (already filed, bar the scaffolding brainstorm)

- Docs: vergil-project/.github#650 (this spec and plan).
- Amendment docs: vergil-project/.github#974 (#213) and
  vergil-project/.github#975 (#607), each its own PR, with the affected task
  bodies pointing at them; #213's T7b is vergil-project/vergil-containers#773.
- Docs review: vergil-project/vergil-containers#739. Its sweep spawns the
  per-repo doc tasks for the site pages that still describe the Debian images,
  and adds a link to Task 14's OS-usage table from each repo's site docs (spec
  §5.9: link, never copy). The pages known today:
  - vergil-containers `docs/site/docs/`: `images/index.md`, `index.md`,
    `configuration.md`, `languages/cpp-clang.md`, `languages/cpp-gcc.md`,
    `languages/ts-node.md`;
  - vergil-tooling `docs/site/docs/`: `guides/ci-architecture.md`,
    `reference/container-config.md`, `reference/dependency-update-config.md`,
    `standards/development/cpp/overview.md`;
  - vergil-actions `docs/site/docs/`: `actions/package-setup.md`,
    `actions/shared-setup-system-packages.md`.
- Retrospective: vergil-project/.github#652.
- Follow-on brainstorms: vergil-project/.github#651 (Ubuntu 26.04) and
  vergil-project/.github#959 (Node runtime and TypeScript applications).
- Follow-on brainstorm, filed with the tasks: remove the cutover install
  branch (spec §7 step 12). It is a known enabling chain: once the human
  judges that the cutover has held and a revert to the Debian images is off
  the table, a follow-on epic removes the uv path for released refs from
  vergil-tooling's cache build and vergil-actions' setup action. A line or
  exact ref then always takes the packaged path, and an image without the
  vergil apt repository fails with `vergil-tooling: the image has no vergil
  apt repository`. Until then the branch stays, and so does the revert path.

## Self-Review

**Spec coverage:**

| Spec section | Task(s) |
|---|---|
| §2 goal 1, §5.1 the tree | 7, 8–13 |
| §2 goal 2, §6.2 #607 amendments | Docs-607 |
| §2 goal 3, §5.6, §5.7 | R1–R4, 1, 15, D3 |
| §2 goal 4, §6.1 #213 amendments | Docs-213, T7b (vergil-containers#773), 7 |
| §2 goal 5, §5.4 adapters | 8–13 (adapter probes, on Task 6's fetchers) |
| §5.2.1 core entries, their own pins | 6 (`core:` section, `check_pins.py`), 7 (core args) |
| §2 goal 6, §5.9 table | 14 |
| §4 revert and fail-forward | D2 notes; the install branch stays until the follow-on epic |
| §4 silent toolchain drift, §5.3 pins | 6, 8–13 |
| §5.2 marker, vergil-python, key check, core tools, no pip, Node | 7 |
| §5.2 packaged vergil-tooling, host jobs | 4, 5, D1 |
| §5.2 scripts declare dependencies, §5.5 scripts | 3 |
| §5.2 python family owns `python3` | 8 |
| §5.2 apt-mirror check on amd64 | V1 step 4 |
| §5.2.1 build checks | 7, 8–13 `checks.sh`; 4 (cache build) |
| §5.3.1 system Python, not `uv python install` | 8 |
| §5.5 docs deploy `python3`, `npm root -g`, system-packages, melete | 7, 13, V1, D2 notes |
| §5.8 CVE triage and ownership | 14; remedies in 2, 7, 8, 13 |
| §5.9 links from other repos' docs, Debian pages | Docs review bookend (#739) |
| §4 revertible cutover PRs | Global Constraints |
| §7 freeze, steps 1–11 | Delivery table mapping |
| §7 step 12 | Follow-on brainstorm bookend |
| §8 spike | 2 |
| §8 pre-release check, cold rebuild | V1, V2 |

**Type consistency:** the build args Task 6 emits (`TOOLCHAIN_VERSION`,
`TOOLCHAIN_SHA256_AMD64`, `TOOLCHAIN_SHA256_ARM64`, `PBS_RELEASE`,
`RUSTUP_INIT_*`) and Task 7's (`CORE_PYTHON`, `CORE_NODE_*`) are the names
Tasks 7–13 read, all carried in T7b's `build-args` map. The probes in Tasks 4
and 5 are the same two paths:
`/etc/apt/sources.list.d/vergil.sources` and `/opt/vergil/python/*/bin/python3`.
The log lines are the same in Tasks 4, 5 and D1.
