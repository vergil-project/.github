# Rust toolchain on the leading edge — design

- **Epic:** vergil-project/.github#1095
- **Planning issue:** vergil-project/.github#410 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Rust gets the same lifecycle automation and toolchain enforcement that Python
has. A new Rust series arrives, and an old one leaves, through a reviewed
lifecycle pull request opened by the vergil-containers lifecycle job, not by
hand. A Rust repo cannot pick its own toolchain behind Vergil's back.

The consumer-facing invariant is the same as Python's: **a consuming repo
deals only in `x.y`.** It builds and tests on "Rust 1.99", never on a patch
release. The patch level (`1.99.z`) is owned by vergil-containers and moves
forward each time the containers rebuild, with no change in any consumer.

## 2. Why

- **Rust is the 2027 target.** Vergil is to be rebuilt in Rust in 2027, with
  Vergil itself as the tool that does it; the 2.x series is the proof of
  concept. The first step is to bring Rust up to Python's level in the 2.1
  series.
- **Rust moves fast.** A new stable Rust release ships every six weeks, and
  the policy window is two series (newest and newest-1). A family that changes
  every six weeks must change through automation, or it falls behind: today
  the manifest lists 1.93/1.92 while the window is 1.99/1.98, and the four
  report issues vergil-containers#640–643 stay open.
- **Toolchain selection belongs to Vergil** (epic #407 spec §6.5). A
  `rust-toolchain.toml` lets a repo choose its toolchain, so a CI leg can pass
  on a version it never ran. The Rust standards page still recommends the
  file.

This epic is Epic A of the Rust-parity wave. Epic B (validation, supply-chain
and dependency-update parity) is planned on vergil-project/.github#1087.

## 3. Goals, non-goals, anti-goals

### Goals

1. Lifecycle add and drop PRs write and remove the series' toolchain pin, so
   they pass CI as opened. This holds for every family with automation `pr`,
   so Python gains it too (the pin half of vergil-project/.github#978).
2. Rust's manifest automation is `pr`, and Rust reaches the policy window
   through the automation's own PRs.
3. Inside every Vergil Rust image, rustup uses the image's installed toolchain
   and nothing else. A repo file cannot select another one, and rustup never
   downloads one.
4. `vrg-validate` reports a `rust-toolchain.toml` or `rust-toolchain` file as
   an error, and a `rust-version` floor below the window as a warning.
5. Every Rust CI leg asserts that it runs the `x.y` CI resolved for it.
6. Rust version handling is correct for a three-digit minor (1.100 and on)
   everywhere this epic touches.
7. The docs describe the new behaviour.

### Non-goals

- Rust validation, supply chain and dependency updates: Epic B
  (vergil-project/.github#1087).
- Lifecycle automation for the manifest `core:` section:
  vergil-project/.github#1094.
- C++, TypeScript and Go. They remain supported and strategic; bringing them
  to the same level is deferred, not dropped.
- The Rust beta channel (vergil-project/.github#424), a cargo builder for
  `vrg-package`, and wider Rust docs and test depth.
- Auto-merging lifecycle PRs. They stay human-reviewed, as Python's are.

### Anti-goals (the epic fails if any of these happen)

- A consumer repo has to name, or is exposed to, a Rust patch release.
- A Rust CI leg or dev container runs a toolchain other than the image's,
  silently.
- A lifecycle add or drop PR, for any family, goes red in CI for a reason the
  automation itself caused.
- Rust 1.100 is mis-sorted, mis-parsed, or treated as older than 1.99
  anywhere in the chain.

## 4. Current state (facts the design builds on)

### vergil-containers

- `docker/versions/manifest.yml`: the `rust` family is `automation: "report"`
  with series 1.93 and 1.92. Under `report`, the lifecycle check classifies a
  version that has left the window as `out_of_policy` (reported as an issue);
  under `pr` it is a `drop` (`versions/lifecycle/core.py`, `classify`).
- `docker/versions/lifecycle/adapters/rust.py`: window = the newest two GA
  releases by release order, from endoflife.date `rust`. A series is
  installable when the Rust release channel has a manifest for it, read with
  `resolve_toolchains.rust_release`.
- `docker/versions/lifecycle/apply.py` `render_change` writes the edited
  manifest plus every file `generate.render_all` produces. It never touches
  `docker/pins/toolchains.yml`. For an add, `render_all` has no pin to emit
  and raises `GenerateError`, which becomes `ApplyError` (apply.py:125–128),
  so no PR opens. A drop PR would leave an orphaned pin, which
  `docker/pins/check_pins.py` (run in PR CI by `scripts/bin/validate-custom`)
  rejects.
- `docker/pins/toolchains.yml` is auto-managed: the daily upstream refresh
  (`docker/pins/upstream-refresh.sh`) runs `resolve_toolchains.py --write`,
  which re-resolves the newest patch of every manifest series. A Rust pin is
  an exact release with no hash (rustup checks components against the channel
  manifest); `rustup-init` is pinned separately with a per-arch SHA-256.
- `docker/rust/Dockerfile.template` installs the pinned release with
  `rustup-init --default-toolchain "${TOOLCHAIN_VERSION}"`, then sets
  `ENV RUST_VERSION=${TOOLCHAIN_VERSION}` (the exact release).
  `docker/rust/checks.sh` checks at build time that rustc and cargo are that
  release.
- The publish job rebuilds every open lifecycle PR branch on each run
  (`publish.py`, `_publish_change`), so a PR whose base moved is refreshed on
  the next run.

### vergil-tooling

- `src/vergil_tooling/lib/versions/extensions/__init__.py` registers the
  per-language hooks: `FLOOR_CHECKS` (python, go), `TOOLCHAIN_CHECKS`
  (python), `CONTAINER_ENVS` (python). Rust has none.
- `python.py` `check_toolchain` errors on `.python-version`; `go.py`
  `check_floor` warns on a `go.mod` `go` directive below the window.
- `docs/site/docs/standards/development/rust/overview.md` recommends
  `rust-toolchain.toml` for formatting config and toolchain pinning.

### vergil-actions

- `actions/ci/toolchain/assert/assert.sh` asserts `x.y` for python and go,
  and only posts a notice for any other language.
- The assert step already runs on every leg for every language
  (`ci-quality.yml`, `ci-test.yml`, `ci-audit.yml`), and the Rust legs run in
  the Rust image. So adding a Rust assertion changes only `assert.sh` and its
  tests, not a workflow file.

### Why Python's mechanism does not carry over

uv takes a version *request*: `UV_PYTHON=3.14` finds whichever 3.14.z the
image has, so vergil-tooling can pass `x.y`. rustup matches toolchain names
exactly: `RUSTUP_TOOLCHAIN=1.99` names a toolchain called `1.99`, which is not
the installed `1.99.1`. Only the image knows its exact release, so the Rust
pin is set by the image, one layer below where Python's is.

### Fleet

There is no active Rust consumer. `mq-rest-admin-project/mq-rest-admin-rust`
is decommissioned and out of scope: it runs old 2.1 releases and is not
expected to be revived, so this epic neither changes it nor waits on it. The
first Rust consumer is `vergil-fixture-rust`, which arrives with
vergil-project/.github#1022 and is built to current standards (no toolchain
file). So no fleet sweep is needed, and the vergil-tooling release that turns
a toolchain file into an error has no consumer to break.

## 5. Approaches considered

1. **Pin in the image (chosen).** The Rust image sets `RUSTUP_TOOLCHAIN` to
   its own installed release and turns rustup's auto-install off. Every place
   the image runs (CI legs, `vrg-container-run`, the dev container) is
   covered with no wiring, and the patch level never leaves vergil-containers.
   vergil-tooling keeps the static checks (toolchain file error, floor
   warning), and CI keeps the `x.y` assertion as the backstop.
2. **Python's shape** (a vergil-tooling container hook plus a CI `pin-rust`
   action). Each would have to read the exact release back out of the image to
   hand it to rustup: two components doing what one Dockerfile line does, each
   of which must stay right across every rebuild. Rejected.
3. **Hand-opened catch-up PRs** before switching Rust to `pr`. Rejected: the
   automation's own catch-up PRs are the first real test of goal 1.

## 6. Design

### 6.1 Lifecycle PRs carry their toolchain pin (vergil-containers)

- An **add** change resolves the new series' pin with the
  `resolve_toolchains` resolver for that family and writes it to
  `toolchains.yml` before `generate.render_all` runs, so the rendered files
  carry the new series' build args. The PR's file list includes
  `docker/pins/toolchains.yml`.
- A **drop** change removes the series' pin only when the edited manifest no
  longer needs it (`required_series` of the edited manifest). A pin that also
  serves the `core:` section, such as `node: "22"` for `core.node`, stays.
  The tests include that overlap case.
- The add PR changes **only** the added series' pin. It does not re-resolve
  the patch of any other series; that remains the daily refresh's job, so an
  add PR never smuggles in unrelated patch bumps. Any shared tool pin the
  family needs (for Rust, `rustup-init`) is written only if it is absent.
- A pin that cannot be resolved (upstream unreachable, no release of the
  series) is an `ApplyError` naming the family and series. Per `publish.py`,
  one failed change does not block the others, and the failure is reported,
  never skipped.
- The mechanism lives in shared code and dispatches to the family's existing
  resolver; no family is special-cased in `apply.py`. It covers python, go,
  node (ts-node) and rust: every family that has a toolchain pin.
- `CATALOG.md`, if it lists the pins, is regenerated in the same PR, so the
  PR passes CI as opened.

### 6.2 Rust moves to `pr` and catches up through the automation

- `manifest.yml`: `rust.automation: "report"` → `"pr"`. From then on the
  lifecycle job opens add PRs for 1.99 and 1.98 and drop PRs for 1.93 and
  1.92 in place of vergil-containers#640–643. The report issues close as the
  job stops reporting them.
- **Catch-up merge order:** adds first, then drops. Each PR is built against
  the base it was opened on. After each merge, the next publish run rebuilds
  the remaining branches against the new base before the next one merges.
  `_drop` already refuses to leave a family empty. The order is a human step,
  written into the deployment task.
- If Rust 1.100 is released before the catch-up completes, the automation
  targets the window it then sees (1.100/1.99); no special handling.

### 6.3 The image owns Rust toolchain selection (vergil-containers)

- `docker/rust/Dockerfile.template` sets, after the install:
  - `RUSTUP_TOOLCHAIN=${TOOLCHAIN_VERSION}`: the exact installed release.
    rustup gives this env var priority over a directory override and over
    `rust-toolchain.toml`.
  - rustup auto-install off (`RUSTUP_AUTO_INSTALL=0`), so a request for any
    other toolchain fails loudly instead of downloading one.
- `docker/rust/checks.sh` adds build-time checks:
  - `RUSTUP_TOOLCHAIN` equals the installed release;
  - in a scratch directory holding a `rust-toolchain.toml` that names a
    different release, `rustc --version` still reports the installed
    release, and nothing is downloaded.
- **Verification before building on it:** the first task confirms, against
  the rustup changelog and documentation for the `rustup-init` version
  pinned in `toolchains.yml`, the precedence of `RUSTUP_TOOLCHAIN` and the name
  and semantics of the auto-install switch, and records the URLs in its PR.
  If the switch differs from what is written here, the task uses the
  documented mechanism and the spec is corrected in the same PR.
- The patch level floats: when the daily refresh moves `1.99.1` to `1.99.2`,
  the rebuilt image installs and pins `1.99.2`. No consumer changes.

### 6.4 Rust resolver hooks (vergil-tooling)

A new `extensions/rust.py`, registered in `extensions/__init__.py`:

- **`check_floor`** (`FLOOR_CHECKS["rust"]`): warns when `rust-version` in
  `Cargo.toml` (`[package]`, or `[workspace.package]` for a workspace) is
  below the window's oldest series. A floor above it is a deliberate
  restriction and is allowed, as for Python and Go (epic #407 spec §6.4). An
  unparseable `Cargo.toml` or `rust-version` is a `VersionsError`, never
  ignored.
- **`check_toolchain`** (`TOOLCHAIN_CHECKS["rust"]`): errors when
  `rust-toolchain.toml` or the legacy `rust-toolchain` exists at the repo root,
  with the fix: delete the file. The message says Vergil selects the toolchain
  through the image.
- **No `CONTAINER_ENVS` entry:** the image sets the toolchain (§6.3).

### 6.5 CI asserts the leg's Rust version (vergil-actions)

`actions/ci/toolchain/assert/assert.sh` gains a `rust` branch: run
`rustc --version` in the repo checkout (so any override would be reported),
take its `x.y`, and compare it with the expected `x.y`. A failure to run
rustc is an error. The header comment and the notice for unhandled languages
are updated. Tests cover a match, a mismatch, a rustc failure and
`rustc 1.100.0`.

### 6.6 Three-digit minor versions (all repos)

Every component this epic touches is tested with a `1.100` series alongside
`1.99`: the lifecycle classify and sort order, the branch-name regex and
`parse_branch`, `toolchains.yml` ordering and the `x.y`/series checks, the
Dockerfile's `"${RUST_VERSION}".*` series check, the resolver floor
comparison, and `assert.sh`'s `x_y`. Any defect found is fixed in that
component's task.

### 6.7 Docs

- `rust/overview.md` (vergil-tooling site docs) stops recommending
  `rust-toolchain.toml`. It says that Vergil selects the toolchain, that a
  repo declares only an `x.y` floor through `rust-version`, and that rustfmt
  config belongs in `rustfmt.toml`.
- The vergil-containers Rust policy page (`docs/site/docs/languages/rust.md`)
  and the toolchain-pin docs describe the `pr` automation and the image-level
  pin.
- The docs say what a Rust repo does when it needs a component or target the
  image lacks: the image gains it through a vergil-containers change. A repo
  never selects it through a toolchain file, and is never exempted from the
  check.

### 6.8 Rollout and validation

1. vergil-containers: §6.1, §6.3 and the 1.100 tests merge. The container
   changes publish through the normal image build.
2. vergil-containers: §6.2's switch to `pr` merges. The catch-up PRs then
   merge in order (adds, then drops).
3. vergil-tooling and vergil-actions: §6.4 and §6.5 merge and ship in their
   releases. Releases are human-gated.
4. **Deployment** (operational task): with the releases attested, the fleet
   picks them up through its normal sync.
5. **Validation** (operational task, blocked by #1022's fixture task and the
   deployment):
   - `vergil-fixture-rust` passes CI on both window series;
   - adding a `rust-toolchain.toml` to a branch of the fixture fails
     `vrg-validate` with the toolchain error;
   - the assertion reports the leg's `x.y`;
   - a dev container (`vrg-container-run`) on the fixture reports the
     installed release, with auto-install off.

## 7. Failure model

- Upstream failures (endoflife.date, the Rust channel, rustup archive)
  surface as lifecycle failure issues or `ApplyError`s, never as a silent
  "no change".
- A pin that cannot be resolved blocks only that change's PR.
- A toolchain mismatch in CI fails the leg with
  `CI resolved <x.y> but the job runs <x.y>`.
- A toolchain file in a repo fails `vrg-validate`.

## 8. Constraints

- No agent cuts a release; releases of vergil-tooling, vergil-actions and the
  images are human-gated.
- Lifecycle PRs stay human-merged.
- Fail loudly: no fallback that hides an unresolved pin, an unparsed
  `Cargo.toml` or a failed rustc call.
- The shared lifecycle code stays language-neutral (vergil-containers
  lifecycle principle 7).

## 9. Related work

- Epic vergil-project/.github#407: the version lifecycle and resolver design
  (§6.4 floors, §6.5 toolchain selection).
- Epic vergil-project/.github#649: the toolchain pins this builds on.
- vergil-project/.github#978: the pin half is delivered here; the `core:`
  half moved to vergil-project/.github#1094.
- vergil-project/.github#1022: the fixture repos; `vergil-fixture-rust` is
  the validation target.
- vergil-project/.github#1087: Epic B.

## 10. Left to the builder

- How `apply.py` calls the family resolver (a per-family registry in
  `resolve_toolchains`, or the existing `resolve_*` functions keyed by family),
  as long as the shared code stays language-neutral.
- The exact wording of the toolchain error and floor warning, following the
  Python messages' shape.
- Whether the checks.sh override test uses `rustc` or `cargo` as its probe.
