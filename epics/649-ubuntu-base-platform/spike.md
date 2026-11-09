# Ubuntu 24.04 image tree — spike report

The report of [plan.md](plan.md) Task 2 (vergil-project/.github#982): every
item of [spec.md](spec.md) §8's spike list, built as throwaway trial images and
checked on each architecture. Each section gives the command, the result per
architecture and a verdict: `as planned`, or `decision needed: <what>`.

**Status: arm64 recorded, amd64 pending.** The arm64 half ran on 2026-11-09.
The amd64 half needs an amd64 machine, which only the human can reach; its
column reads `pending: needs an amd64 machine (human)` until then. The task is
not complete until it is recorded.

## Setup

- **arm64 host:** aarch64 Linux, rootless `nerdctl` 2.4.1 with buildkit
  0.33.1 (no `docker`).
- **Root:** `ubuntu:24.04`
  (`sha256:fe81560990fae86dc4743202b2cb4d3df98d0bcf949173cecef15ce1127ef852`,
  Ubuntu 24.04.5 LTS).
- **Trial tree,** built in tier order, each child `FROM` the local trial parent:
  - `pkg-ubuntu:24.04`: #607 plan Task 4's contents. `ubuntu:24.04`,
    vergil-actions' `os-prereqs.sh` (the amd64-only mirror list and
    `80-vergil-fast-fail`, then `ca-certificates curl git tar gzip gnupg`), the
    OS upgrade, uv 0.13.7 (sha256-checked), nFPM 2.48.0 and a trial marker.
    Its build check (mirror config absent on arm64; `curl`, `git`, `gpg`, `uv`
    and `nfpm` present; no `python3`; no apt lists) passes.
  - `base`: spec §5.2. The marker removed; the vergil apt repository added
    behind the fingerprint check; `vergil-python3.15`;
    `/usr/local/bin/python3` linked to the core Python; the core Node 22 at
    `/opt/vergil/node/22` with a `markdownlint` wrapper; today's fragments
    (`path-defaults`, `github-cli`, `validation-tools`, `security-tools`,
    `opentofu`, `rpm`, `pandoc`); yamllint, ansible-lint, semgrep and
    `mkdocs` (with `mkdocs-material` and `mike`) as uv tools on the core
    Python; no `pip install` anywhere.
  - Families on `base`: `python:3.12`–`3.15`, `go:1.26`, `go:1.27`,
    `rust:1.92`, `rust:1.93`, `cpp-gcc:14`, `cpp-clang:19`, `cpp-clang:20`,
    `ts-node:22`, `ts-node:24`.
- **Fragments** are vergil-containers `develop` at `c4a4b46`.
- **Where the trial lives:** the trial Dockerfiles, scripts and logs are not
  committed (plan Task 2). They are in the main clone's ignored
  `build/spike-982/`: `ctx/` (one directory per image), `scripts/`, `vendor/`
  (the vergil-containers fragments, the C++ smoke project, `.trivyignore`,
  `os-prereqs.sh` and vergil-tooling's `cppcheck-suppressions.txt`, copied so
  the directory is self-contained) and `logs/`.

Every image of the tree reports `VERSION_ID 24.04` and carries no
`/etc/vergil/package-toolchain`. Outside `python:*`, `/usr/local/bin/python3`
resolves to `/opt/vergil/python/3.15/bin/python3.15`. Only `ts-node:*` has a
`node` on `PATH`. `uv` is the parent's 0.13.7 in every image.

## Pins the trial resolved

These are the newest releases of each manifest series on 2026-11-09. They are
the trial's inputs, not the pins Task 6 will write.

| Input | Release | Note |
|---|---|---|
| python-build-standalone | `20261026`: 3.12.16, 3.13.18, 3.14.10, 3.15.2 | `install_only` builds for `aarch64-` and `x86_64-unknown-linux-gnu`; sha256 from the release's asset digests |
| vergil-python (core) | `vergil-python3.15` 3.15.1+20261021.1.0.4-1 | vergil apt index, suite `noble` |
| vergil-tooling | 2.1.293-1 | depends on `vergil-python3.15 (>= 3.15.1+20261021.1.0.4)` |
| Node (core and ts-node) | 22.23.4 (npm 10.9.9), 24.22.2 (npm 11.19.1) | nodejs.org `index.json`; both carry `linux-arm64` and `linux-x64` |
| Go | 1.26.10, 1.27.3 | go.dev `?mode=json&include=all` |
| Rust | 1.92.0, 1.93.1 | `channel-rust-<x.y.z>.toml`; no 1.92.1 or 1.93.2 exists |
| rustup-init | 1.29.3 | sha256 per architecture from `static.rust-lang.org/rustup/archive/1.29.3/` |
| LLVM | 19.1.7, 20.1.8 | apt.llvm.org `llvm-toolchain-noble-<v>` |
| GCC | 14.2.0-4ubuntu2~24.04.1 | noble-updates, `universe` |

## 1. The GCC 14 source and the C++ images' libstdc++

**Command.** In the trial `pkg-ubuntu`:
`apt-get update && apt-cache policy gcc-14 g++-14 g++ libstdc++6 libstdc++-14-dev`;
the Ubuntu Toolchain PPA's `dists/noble/main/binary-arm64/Packages.gz`; the
`cpp-gcc:14` trial build (`gcc-14`, `g++-14` from noble); then the
vergil-containers C++ smoke test (`docker/cpp/smoke-test.sh`) on `cpp-gcc:14`,
`cpp-clang:19` and `cpp-clang:20`. For the libstdc++ question,
`clang++ -v` reports its `Selected GCC installation` with `g++-14` installed,
and again with only the unversioned `g++`.

| | arm64 | amd64 |
|---|---|---|
| Result | noble ships `gcc-14`/`g++-14` 14.2.0-4ubuntu2~24.04.1 in `noble-updates` (`universe`), a released 14.2.0. The `noble` release pocket alone has `14-20240412-0ubuntu1`, a pre-release snapshot. The PPA has 14.3.0-12ubuntu1~24~ppa1. Unversioned `g++` is 13.2.0 (`g++-13` 13.3.0). `libstdc++6` is 14.2.0 (from the gcc-14 source). The `cpp-gcc:14` build gives `gcc (Ubuntu 14.2.0-4ubuntu2~24.04.1) 14.2.0`. Clang 19 and 20 with `g++-14` select `/usr/lib/gcc/aarch64-linux-gnu/14`; with only `g++` they select `.../13`. All three smoke tests pass (`SMOKE PASS`). Today's trixie images ship gcc and libstdc++ 14.2.0-19. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** The source is noble's own `gcc-14` (a released
14.2.0, from `noble-updates`, which `ubuntu:24.04` enables), not the PPA. The
C++ images install `g++-14`, not the unversioned `g++`, for libstdc++. That
keeps libstdc++ 14 in both C++ families, as today. With `g++` alone, Clang
would build against libstdc++ 13.

## 2. python-build-standalone, uv discovery and the core Python

**Command.** For each series `<v>` in 3.12–3.15, build the `python:<v>` trial
on `base`. It downloads
`cpython-<patch>+20261026-<triple>-install_only.tar.gz`, checks its pinned
sha256 and unpacks it at `/opt/python/<v>`. It links `/usr/local/bin/python3`,
`python` and `pip` there, and its build check asserts
`python3 --version` matches `<v>`. Then, in each image,
`scripts/check-python.sh <v>`:

- `UV_PYTHON=<v> UV_PYTHON_PREFERENCE=only-system uv python find`;
- `uv run` in a fresh project with a dependency, under the same environment;
- the interpreter behind each of `base`'s uv tools;
- `python3 -m venv` and its `pip`.

In `python:3.12`, `scripts/check-vergil-tooling.sh 2.1.293-1 3.12` exports
`UV_PYTHON=3.12 UV_PYTHON_PREFERENCE=only-system`, as the cache build does. It
installs `vergil-tooling=2.1.293-1` from the apt repository and runs
`vrg-whoami --help` and `vrg-validate --help`.

| | arm64 | amd64 |
|---|---|---|
| Result | All four series unpack at `/opt/python/<v>`. In each image `uv python find` returns `/usr/local/bin/python`, which resolves to `/opt/python/<v>/bin/python<v>`, and `uv run` reports `<v>` with `sys.base_prefix` `/opt/python/<v>`. No uv-managed interpreter exists. `python3`, `python` and `pip` resolve to the series, and `pip` is 26.2.2 in every series. yamllint, ansible-lint, semgrep, `mkdocs` and `mike` all run on `/opt/vergil/python/3.15` in every series, and `markdownlint` runs with no `node` on `PATH`. In `python:3.12`, the packaged `vrg-*` run from `/opt/vergil/vergil-tooling/venv`, whose `sys.base_prefix` is `/opt/vergil/python/3.15`. | pending: needs an amd64 machine (human) |

**Verdict: as planned.**

A builder note for Task 7, not a deviation: on the core 3.15, semgrep's
dependency `ruamel.yaml.clib` (0.2.15) has no `cp315` wheel on PyPI for any
platform, so `uv tool install semgrep` compiles it. vergil-python's sysconfig
`CC` is `clang -pthread`, so with no compiler the build fails with
`No such file or directory: 'clang'`. The trial keeps today's pattern: a
transient `build-essential` for the semgrep install, purged afterwards, with
`CC=gcc`. That works. It stops being needed once a `cp315` wheel ships.

## 3. vergil-python, vergil-tooling and the install-path probes

**Command.** The `base` trial build:

1. Download `https://vergil-project.github.io/packages/keys/vergil.asc`.
2. Run `gpg --show-keys --with-colons` with a scratch `GNUPGHOME`.
3. Fail unless the file holds exactly one `pub` record and the `fpr` after it
   is `B3A1D804DC03AB036566A4E367861822166ECABE`.
4. Only then write a bootstrap source, `install vergil-archive-keyring` from
   it alone, and remove it.
5. Check the installed key the same way, then `install vergil-python3.15`.

Then `scripts/check-vergil-tooling.sh` (item 2) does the pinned install and a
dev-ref install, `uv tool install --python
/opt/vergil/python/3.15/bin/python3.15 'vergil-tooling @
git+https://github.com/vergil-project/vergil-tooling@develop'`. By hand, the
two probe paths Task 4 checks, `/etc/apt/sources.list.d/vergil.sources` and
`/opt/vergil/python/*/bin/python3`, were looked for in the trial `base` and
in today's `ghcr.io/vergil-project/prod-base:latest`
(`sha256:9a772d1b5275f5e40e2228377f8ec0fc30dd166c001b201e252ea128fb7280c2`).

| | arm64 | amd64 |
|---|---|---|
| Result | The downloaded key holds 1 primary key with fingerprint `B3A1D804DC03AB036566A4E367861822166ECABE`, and so does the installed `/usr/share/keyrings/vergil-archive-keyring.asc`. `vergil-archive-keyring` 1.1.1-1 writes `/etc/apt/sources.list.d/vergil.sources` (suite `noble`). `vergil-python3.15` 3.15.1+20261021.1.0.4-1 installs at `/opt/vergil/python/3.15`. `vergil-tooling` 2.1.293-1 installs. Its `/usr/bin/vrg-*` link into `/opt/vergil/vergil-tooling/venv/bin` (shebang `#!/opt/vergil/vergil-tooling/venv/bin/python`, base prefix `/opt/vergil/python/3.15`), and `vrg-whoami --help` and `vrg-validate --help` run. The dev-ref install puts `vrg-validate` at `/root/.local/bin`, on a uv tool venv whose base prefix is `/opt/vergil/python/3.15`, and it runs. Probes: the trial `base` (Ubuntu 24.04.5) has both paths; `prod-base` (Debian 13 trixie) has neither. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** The environment check itself is Task 4's tests and
D1. This shows only that its two probes tell the generations apart.

## 4. Node side by side

**Command.** `scripts/check-node.sh` in `base`, `ts-node:24` and
`ts-node:22`. It lints a clean and a failing Markdown file with
`markdownlint`, and prints `command -v node` and, where `npm` exists,
`npm root -g` and `npm prefix -g`. The `ts-node:<major>` trials unpack the
nodejs.org tarball at `/opt/node/<major>`, link `node`, `npm` and `npx` into
`/usr/local/bin` and write `prefix=/usr` to `/opt/node/<major>/etc/npmrc`.
Their build checks assert the major and `npm root -g`, and run `tsc`, `eslint`
and `prettier` from the `typescript-analysis` fragment.

| | arm64 | amd64 |
|---|---|---|
| Result | `base`: no `node`; `markdownlint` 0.50.0 (the wrapper, running `/opt/vergil/node/22/bin/node`) passes the clean file and reports `MD022` on the failing one. `ts-node:24`: `node` v24.22.2 from `/usr/local/bin`; `markdownlint` is still the core wrapper, with the same results; `npm root -g` = `/usr/lib/node_modules`, `npm prefix -g` = `/usr`, npm 11.19.1. `ts-node:22`: the same, with `node` v22.23.4 and npm 10.9.9. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** A builder note: the core Node's `npm` is itself an
`#!/usr/bin/env node` script, so the `RUN` that installs `markdownlint-cli`
puts `/opt/vergil/node/22/bin` on `PATH` for that command only.

## 5. Go, Rust, LLVM and nodejs.org sources for every manifest series

**Command.** Build `go:1.26` and `go:1.27`: the go.dev tarball, sha256-pinned,
then `go version`, `go install golang.org/x/vuln/cmd/govulncheck@latest` and a
`go run` with `GOTOOLCHAIN=local`. Build `rust:1.92` and `rust:1.93`: the
pinned `rustup-init` 1.29.3, sha256-checked, installing the exact release
with `--profile minimal --component clippy,rustfmt,llvm-tools`, then
`cargo run` and `cargo clippy` on a new crate. Build `cpp-clang:19` and
`cpp-clang:20` from `apt.llvm.org/noble/ llvm-toolchain-noble-<v>`, and the
two `ts-node` trials (item 4).

| | arm64 | amd64 |
|---|---|---|
| Result | Go: `go1.26.10 linux/arm64` and `go1.27.3 linux/arm64`; govulncheck v1.9.0 builds and the program runs on both. Rust: `rustc 1.92.0` and `rustc 1.93.1`, each with exactly cargo, clippy, llvm-tools, rust-std, rustc and rustfmt; both crates build, run and pass clippy. LLVM: `Ubuntu clang version 19.1.7` and `20.1.8` install with `lld`, `llvm` and `libclang-rt-<v>-dev`, and the smoke test passes (item 1); today's trixie channel ships the same 20.1.8. Node: 22.23.4 and 24.22.2 (item 4). Every amd64 download (go.dev, `rustup-init`, nodejs.org, python-build-standalone) is listed with its own hash, but none was fetched or built here. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** A builder note for Task 10 (rust): the image needs a
C linker. `rust:<v>-slim` ships `gcc`, so the trial installs `gcc`,
`libc6-dev` and `pkg-config` from noble.

## 6. noble's cmake and cppcheck against today's C++ images

**Command.**

- `apt-cache policy cmake cppcheck` in the trial `pkg-ubuntu`.
- `cmake --version` and `cppcheck --version` in today's
  `prod-cpp-gcc:14` and `prod-cpp-clang:20`.
- In a scratch clone of the one C++ consumer found locally
  (`logical-minds-foundry/mq-protocol-gateway`, `[ci] versions =
  ["clang-20"]`, `cmake_minimum_required(VERSION 3.20)`), install
  `vergil-tooling` 2.1.293-1 from the apt repository and run `vrg-validate` in
  the trial `cpp-clang:20`. Its command was
  `DOCKER_DEV_IMAGE=localhost:5000/spike-982/cpp-clang:20 vrg-container-run
  -- bash -c 'apt-get install vergil-tooling && vrg-validate'`, through a
  throwaway local registry, since `vrg-container-run` pulls a
  `DOCKER_DEV_IMAGE`.
- The failing lint command again, with vergil-tooling's packaged
  `cppcheck-suppressions.txt` and with the same file minus its comment lines,
  in the trial and in today's `prod-cpp-clang:20`.

| | arm64 | amd64 |
|---|---|---|
| Result | noble: cmake 3.28.3, cppcheck 2.13.0. Today (trixie): cmake 3.31.6, cppcheck 2.17.1. In the consumer run, `common`, `install`, `typecheck` (cmake configure and build), `test` and `audit` pass on cmake 3.28.3, and `lint` fails. clang-format and run-clang-tidy pass, but cppcheck 2.13.0 exits with `cppcheck: error: Failed to add suppression. No id.` cppcheck 2.13.0 rejects the comment (`#`) lines of vergil-tooling's `configs/cpp/cppcheck-suppressions.txt`: the same file minus its comments passes (exit 0), and cppcheck 2.17.1 accepts it as it is. `googletest.cfg` is present on noble, and `--std=c++20` and `--std=c++23` are accepted. | pending: needs an amd64 machine (human) |

**Verdict.**

- **cmake: as planned.** noble's 3.28.3 is accepted. The consumer and the
  vergil-tooling skeleton both require 3.20, and the consumer configures,
  builds and tests on it, so Kitware's channel is not needed.
- **cppcheck: decision needed:** noble's cppcheck 2.13.0 fails every C++
  consumer's lint on vergil-tooling's packaged suppressions file. The options
  the spike found:
  - (a) A consumer fix in vergil-tooling, ahead of the C++ family PRs as
    spec §5.5 orders: keep the suppressions file free of comment lines,
    moving the rationale elsewhere. 2.13 and 2.17 both accept that file, so
    it also works on the Debian images.
  - (b) A newer cppcheck from another prebuilt source. The spike found none
    and did not search further.

## 7. One `vrg-validate` of vergil-tooling in a trial `dev-python`

**Command.** In a scratch clone of vergil-tooling `develop` (`c82f18938e`),
`DOCKER_DEV_IMAGE=localhost:5000/spike-982/python:3.15 vrg-container-run --
uv run vrg-validate`, the trial `python:3.15` served from a throwaway local
registry. The failing test's subject was then probed with vergil-tooling's
own `lib/package/elf.glibc_requirements` and with
`readelf -V --dyn-syms`.

| | arm64 | amd64 |
|---|---|---|
| Result | `common`, `install`, `lint`, `typecheck` and `audit` pass. `test` fails: 11409 passed, 1 skipped, 1 failed, coverage 100%. The failure is `tests/vergil_tooling/test_package_elf.py::test_real_interpreter_requires_some_glibc`, which asserts every glibc requirement of the running interpreter is at least 2.2. `glibc_requirements` returns `[(2, 0), (2, 17)]` for python-build-standalone 3.15.2 and for vergil-python 3.15.1, against `[(2, 17), (2, 34)]` for today's `prod-python:3.15`. The `GLIBC_2.0` is not a libc requirement: `readelf` shows it as the version need of `libgcc_s.so.1`, for `__register_frame@GLIBC_2.0` and `__deregister_frame@GLIBC_2.0`, which is how aarch64's libgcc_s names those symbols. | pending: needs an amd64 machine (human) |

**Verdict: decision needed:** vergil-tooling's `elf.glibc_requirements`
counts version needs of non-libc libraries (here libgcc_s's `GLIBC_2.0`) as
glibc requirements, so its own test fails on any python-build-standalone or
vergil-python interpreter on arm64. The packaged glibc floor, which takes the
highest requirement, is unaffected here. The decision is whether to fix the
function (count only libc and the dynamic loader's version needs) or the
test, and to land it in vergil-tooling as a consumer fix ahead of the python
family PR (Task 8). Whether amd64 shows it too is the amd64 half's question.

## 8. Does vergil-tooling's package venv need pip inside vergil-python?

**Command.** After the item 2 install:

- list `/opt/vergil/vergil-tooling/venv` and its `site-packages`;
- read `pyvenv.cfg`;
- look for maintainer scripts under `/var/lib/dpkg/info/vergil-*`;
- read how vergil-tooling's builder makes the venv
  (`lib/package/python_builder.py`);
- check `/opt/vergil/python/3.15` for `pip` and `ensurepip`.

| | arm64 | amd64 |
|---|---|---|
| Result | No. The venv holds no `pip` or `setuptools`. `pyvenv.cfg` records `uv = 0.12.23` and `home = /opt/vergil/python/3.15/bin`, since the builder makes it with `uv venv --python <interpreter>` at package build. Neither `vergil-tooling` nor `vergil-python3.15` has a `postinst`, so nothing runs pip at install time either. vergil-python itself bundles pip 26.2.2 in its `site-packages`, and `ensurepip`. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** Nothing in the design needs pip inside
vergil-python, which is vergil-project/.github#965's question 4. The bundled
pip 26.2.2 belongs to vergil-python (spec §5.8). It is the same pip
python-build-standalone bundles in every series, and Trivy finds nothing
fixable in it (item 9).

## 9. Trivy: bundled npm and pip in the core Node, ts-node and python-build-standalone layers

**Command.** In each trial image, the image's own Trivy 0.76.2 (the
`security-tools` pin) runs `trivy rootfs --scanners vuln --severity
CRITICAL,HIGH --ignore-unfixed --skip-files '**/pip/_vendor/bom.cdx.json'` on
the runtime directory, the publish gate's settings. It runs once without and
once with vergil-containers' `.trivyignore` (`scripts/trivy-layers.sh`,
summarized by `scripts/summarize-trivy.py`). The directories:

- `/opt/vergil/node/22` and `/opt/vergil/python/3.15` in `base`;
- `/opt/node/24` and `/opt/node/22` in the `ts-node` images;
- `/opt/python/<v>` in each `python` image.

| | arm64 | amd64 |
|---|---|---|
| Result | Fixable HIGH/CRITICAL findings: core Node 22.23.4 (npm 10.9.9): 10, none in `.trivyignore`; `ts-node:22` (the same Node 22.23.4): the same 10; `ts-node:24` (Node 24.22.2, npm 11.19.1): 6, of which 3 are already in `.trivyignore`; `/opt/vergil/python/3.15` and python-build-standalone 3.12–3.15 (pip 26.2.2): 0. All 16 Node findings are in packages npm vendors under `lib/node_modules/npm/node_modules/`. None is in Node itself or in markdownlint-cli. No CRITICAL. Each node runtime is the newest release of its major, so no newer runtime pin exists. The per-CVE list is below. | pending: needs an amd64 machine (human) |

**Verdict: as planned.** Each task's remedy is a `.trivyignore` entry under
that file's rules (a carrier group with its reason and removal condition).
Spec §5.8 prescribes that while no fixed runtime exists. Never an in-place npm
upgrade.

The arm64 findings, all public CVEs, all HIGH, all in npm's vendored
packages. The Node 22 rows hold for both the core Node and `ts-node:22`:

| CVE | Package (installed) | Fixed in | Carriers |
|---|---|---|---|
| CVE-2026-102276 | brace-expansion 2.0.2 (Node 22), 5.0.9 (Node 24) | 1.1.19, 2.1.5, 3.0.7, 5.0.10 | Node 22, Node 24 |
| CVE-2026-102278 | brace-expansion 2.0.2 (Node 22), 5.0.9 (Node 24) | 1.1.20, 2.1.6, 3.0.8, 5.0.11 | Node 22, Node 24 |
| CVE-2026-13149 | brace-expansion 2.0.2 | 1.1.16, 2.1.2, 5.0.7 | Node 22 |
| CVE-2026-14257 | brace-expansion 2.0.2 | 1.1.17, 2.1.3, 3.0.3, 5.0.8 | Node 22 |
| CVE-2026-69152 | brace-expansion 2.0.2 | 1.1.18, 2.1.4, 3.0.6, 5.0.9 | Node 22 |
| CVE-2026-33671 | picomatch 4.0.3 | 2.3.2, 3.0.2, 4.0.4 | Node 22 |
| CVE-2026-48815 | sigstore 3.1.0 | 4.1.1 | Node 22 |
| CVE-2026-69192 | ip-address 10.1.0 | 10.3.1 | Node 22 |
| CVE-2026-9496 | pacote 19.0.2 and 20.0.1 | 21.5.1 | Node 22 |
| CVE-2026-19534 | undici 6.28.0 | 6.28.1, 7.29.1, 8.10.2 | Node 24 |
| CVE-2026-87018, CVE-2026-109383, CVE-2026-109869 | undici 6.28.0 | 6.29.1, 7.30.1, 8.12.2 | Node 24 (already in `.trivyignore`) |

The 10 Node 22 findings are 9 CVE IDs (pacote appears twice), and
`.trivyignore` is one file keyed by ID, so:

- **Task 7 (`base`):** 9 IDs for the core Node 22's npm, the first nine rows.
  Moving the core Node pin to 24 would cut that to 3 (the two brace-expansion
  IDs and CVE-2026-19534), but a core move is a reviewed PR of its own (spec
  §5.2.1), not this task's.
- **Task 13 (`ts-node`):** one new ID, CVE-2026-19534, for `ts-node:24`; the
  carrier notes of Task 7's entries gain `ts-node:22` and `ts-node:24`.
- **Task 8 (`python`):** none, as for every bundled pip.

## Summary

| # | Item | arm64 | amd64 |
|---|---|---|---|
| 1 | GCC 14 source and libstdc++ | as planned | pending: needs an amd64 machine (human) |
| 2 | python-build-standalone, uv, the core Python | as planned | pending: needs an amd64 machine (human) |
| 3 | vergil-python, vergil-tooling, probes | as planned | pending: needs an amd64 machine (human) |
| 4 | Node side by side | as planned | pending: needs an amd64 machine (human) |
| 5 | Go, Rust, LLVM, nodejs.org sources | as planned | pending: needs an amd64 machine (human) |
| 6 | cmake and cppcheck | cmake as planned; cppcheck decision needed | pending: needs an amd64 machine (human) |
| 7 | `vrg-validate` of vergil-tooling | decision needed | pending: needs an amd64 machine (human) |
| 8 | pip in vergil-python for the package venv | as planned | pending: needs an amd64 machine (human) |
| 9 | Trivy of the bundled npm and pip | as planned | pending: needs an amd64 machine (human) |

### Decisions needed

1. **cppcheck (item 6).** noble's cppcheck 2.13.0 rejects the comment lines of
   vergil-tooling's packaged `cppcheck-suppressions.txt`, failing every C++
   consumer's lint. The choice is a comment-free suppressions file in
   vergil-tooling, as a consumer fix ahead of the C++ family PRs, or another
   cppcheck source.
2. **vergil-tooling's glibc test (item 7).** `elf.glibc_requirements` counts
   libgcc_s's `GLIBC_2.0` version need, so
   `test_real_interpreter_requires_some_glibc` fails on python-build-standalone
   on arm64. The choice is fixing the function or the test, as a consumer fix
   ahead of Task 8.

Human ruling (2026-11-09) on both: fix it in vergil-tooling. The cppcheck fix
is vergil-project/vergil-tooling#4138, which blocks Tasks 11 and 12
(vergil-project/vergil-containers#783, #785). The glibc fix is
vergil-project/vergil-tooling#4139: `elf.glibc_requirements` counts only the
version needs of glibc's own libraries, and it blocks Task 8
(vergil-project/vergil-containers#780).

## amd64: the commands for the human

The amd64 half runs on one of the amd64 cloud VMs. That VM is also where the
amd64-only mirror config is exercised: the trial `pkg-ubuntu` writes it through
`os-prereqs.sh`, and every later `apt-get` in the tree runs through it.

1. Copy the trial to the VM, from this machine's main clone:

   ```bash
   tar -C build -czf /tmp/spike-982.tgz spike-982/ctx spike-982/scripts spike-982/vendor
   ```

   Then on the VM, in any scratch directory:

   ```bash
   tar -xzf spike-982.tgz && cd spike-982
   ```

2. Build the whole tree and run items 1–6, 8 and 9 (needs `nerdctl` with
   buildkit). Results land in `logs/` and `scan/`;
   `logs/trivy-summary.txt` is item 9.

   ```bash
   bash scripts/run-all.sh 2>&1 | tee logs/run-all-amd64.log
   ```

3. Item 7, `vrg-validate` of vergil-tooling in the trial `python:3.15`:

   ```bash
   nerdctl run -d --name spike982-registry -p 127.0.0.1:5000:5000 registry:2
   nerdctl tag spike-982/python:3.15 localhost:5000/spike-982/python:3.15
   nerdctl push --insecure-registry localhost:5000/spike-982/python:3.15
   git clone https://github.com/vergil-project/vergil-tooling && cd vergil-tooling
   DOCKER_DEV_IMAGE=localhost:5000/spike-982/python:3.15 \
     vrg-container-run -- uv run vrg-validate 2>&1 | tee ../logs/vrg-validate-vergil-tooling-py315.log
   ```

4. Item 6's consumer run, in a clone of
   `logical-minds-foundry/mq-protocol-gateway`, after pushing
   `spike-982/cpp-clang:20` to the same registry:

   ```bash
   DOCKER_DEV_IMAGE=localhost:5000/spike-982/cpp-clang:20 vrg-container-run -- \
     bash -c 'apt-get update -qq && apt-get install -y -qq --no-install-recommends vergil-tooling && vrg-validate'
   ```

5. Record each result in this file's amd64 column, replacing `pending`.
