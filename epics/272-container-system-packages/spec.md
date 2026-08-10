# Repo-specific system dependencies in dev containers

- **Epic:** vergil-project/.github#272
- **Status:** Design (2026-08-10)
- **Origin:** promoted from research issue vergil-project/vergil-tooling#2718.
- **Implemented in:** vergil-tooling (config + local cache-build) + vergil-actions
  (CI setup step). Docs across both.

## 1. Summary

Vergil's container model assumes a system dependency is a property of the
**language**, not the **repository**: `dev-python:3.14` means the same thing in
every org, which is what makes the cache sound and builds reproducible. That
assumption has failed for a real project — `mnemosys-project/melete` shells out
to the **LilyPond binary** (`apt-get install lilypond`) for its render
integration test and end-to-end CLI test — and there is no sanctioned way to
express the requirement.

This epic adds a **declarative, per-repo system-package layer** to dev
containers, expressed once in `vergil.toml` and applied identically to the local
cached image and to CI, **without** adding anything repo-specific to the shared
base images.

```toml
[container]
system-packages = ["lilypond"]
```

## 2. Motivation & constraints

LilyPond is not a Python package; it is a Debian package. Nothing in
`dev-python` has it, and nothing should — it is meaningless to every other Python
repo in every other org. The PyPI `lilypond` wheel redistribution is ruled out:
it ships **x86_64 wheels only**, and both the dev container (Debian trixie arm64)
and the developer host (Apple Silicon) are aarch64. Debian trixie carries
`lilypond 2.24.x` for arm64, so an `apt` layer works where the wheel does not.

The obvious workarounds are all bad and motivate a first-class mechanism:

- **Add lilypond to `dev-python`** — every Python repo in every org pays for one
  repo's dependency; does not scale past the first request.
- **A bespoke image for melete** — abandons the shared base and its maintenance.
- **Skip the tests that need it** — the render adapter and the end-to-end path
  are exactly what a unit test cannot cover.
- **Install it inside each test run** — slow, network-dependent, cache-defeating.

The extension point already exists: `vrg-container-run` builds a **per-repo
cached image** over the shared base (`container_cache.py`), whose `setup` step
already does repo-specific work (`uv tool install …`, `uv sync`). The gap is not
"design a new mechanism" — it is "generalize the setup step to also install
declared system packages, and make CI honour the same declaration."

## 3. Design

Five decisions, locked during brainstorming:

### 3.1 Config surface — package names only

A new optional key in the existing `[container]` table:

```toml
[container]
system-packages = ["lilypond"]
```

- A list of Debian package **names** installed with
  `apt-get install -y --no-install-recommends` from the base image's **existing**
  apt sources (Debian main). **No** third-party sources, signing keys,
  `add-apt-repository`, or scripts.
- `config.py`: add `system-packages` to `_KNOWN_KEYS["container"]`; add
  `system_packages: list[str]` to `ContainerConfig` (default `[]`); validate it
  is a list of strings (mirroring `env-prefixes`, including the non-list /
  non-string rejection). Add a `container_system_packages(repo_root)` accessor
  alongside `container_env_prefixes`, returning `[]` when there is no
  `vergil.toml`. This accessor is the **single reader** of the key; both the
  local bake (§3.2) and CI (§3.3) go through it — CI via a thin `vrg-*` entry
  point that prints the list — so there is exactly one TOML parser and one source
  of truth.
- The key is documented as "Debian `apt` package names" right where it is
  defined; see §3.4 on naming.

### 3.2 Local path — bake into the cached image

`container_cache.py::_build_cached_image` already runs a `setup` command in a
fresh container and commits the result. When `system-packages` is non-empty,
prepend an apt fragment to `setup`:

```
apt-get update && apt-get install -y --no-install-recommends <names> && <existing warmup>
```

- Base images run as **root** (no `USER` directive in the `vergil-containers`
  Dockerfiles), so `apt-get install` works in the cache build with no elevation.
- The apt fragment is produced by **one helper** shared with the CI step (§3.3),
  so the two consumers cannot drift on *how* the install is spelled.
- Empty list ⇒ **no** apt step ⇒ byte-identical to today.

### 3.3 CI path — setup step in vergil-actions

CI does not use the local cache layer: `ci-quality.yml` runs each job in
`container: ${{ matrix.image }}` — the **base** image directly. So the same
declaration must take effect in CI by a different path.

- A composite setup step, added to the container jobs that **run the repo's own
  tests**, reads `[container].system-packages` (via the §3.1 accessor — a thin
  `vrg-*` entry point that prints the list; **not** ad-hoc shell parsing of
  `vergil.toml`, which would silently mis-read valid TOML) and runs
  `apt-get install -y --no-install-recommends <names>` inside the already-selected
  base-image container (as root) before the tests.
- **Bounded retry.** The `apt-get update && apt-get install` invocation is wrapped
  in a small bounded retry, because `apt-get update` against live Debian mirrors
  is a known CI flake source and the CI path (unlike the baked-once local image)
  re-fetches per run. The retry keeps a transient mirror hiccup from failing an
  otherwise-green PR.
- **Scope: test jobs only.** The install runs only on the jobs that execute the
  repo's tests, not on lint/typecheck — see the test-runtime contract below.
- **One list, two consumers.** Both the local cache build and the CI step read
  the single `[container].system-packages` key through the same §3.1 accessor and
  the same Debian sources. They may differ on *when* the packages arrive —
  **baked** into the local image vs **installed per-run** in CI — but never on
  *what* is installed. The invariant is "the same packages are present," not "the
  same image bytes."

**Test-runtime dependency contract.** A declared system package is a
**test-runtime** dependency. Locally it is baked into the single cached image
used for *every* check (lint, typecheck, test) as a convenience of the one-image
model; in CI it is installed **only on the jobs that run tests**. This means the
package is visible everywhere locally but test-only in CI — a deliberate, stated
divergence, sound because these binaries (e.g. LilyPond, invoked by render/e2e
tests) are genuinely needed only at test time. **Repos must not rely on a system
package during lint or typecheck.** If a real need for a system package outside
test jobs ever appears, the CI step is widened to those jobs then — not
pre-emptively.

### 3.4 Naming — `system-packages`, Debian coupling stated in docs/errors

The key is the generic `system-packages`, matching the issue sketch and the
`[vm].packages` precedent. The Debian/`apt` coupling is stated in the docs at the
key and in the fail-closed error message (§3.5) — not encoded in the key name. If
the base distro ever changed, the key would not lie; only its implementation
would.

### 3.5 Cross-architecture — fail-closed, loud

The cache layer builds on the host arch (Apple Silicon → arm64; the CI runner's
arch). When a declared package has no installation candidate for the build arch:

- The build (local) and the CI setup step both **fail immediately** with a
  message naming the package and the architecture, e.g.
  `system package 'X' is not installable on linux/arm64 (apt: no installation candidate)`.
- No silent skip, no degraded image. A missing render dependency stops the build
  rather than producing a container that fails mysteriously three steps later
  inside a test.

### 3.6 Cache key — no new code, guarded by a test

`vergil.toml` is already in the cache-sensitive file set (`_CACHE_FILES`) for
every language, so a changed `system-packages` list already changes
`compute_cache_hash` and forces a rebuild of the stale layer. This needs **no
cache-key code change**; a test asserts the invariant (editing the package list
changes the hash) so a future refactor cannot silently break it.

### 3.7 Trust model

- **Constrained surface.** A repo can express Debian package *names* only,
  installed from the base image's existing apt sources, as root — exactly what
  the base image already grants. It cannot fetch arbitrary URLs or run arbitrary
  code. The worst a careless or hostile declaration can do is pull a
  Debian-packaged tool into the dev/CI image.
- **Review = PR review of the diff.** Adding to `system-packages` is a
  `vergil.toml` change, reviewed like any config change; the list is a plain,
  greppable set of names. **No separate allowlist** — a curated set would be
  unmaintainable and adds little over "a human reviewed the diff and it is a real
  Debian-main name." The fail-closed build (§3.5) is the backstop for typos and
  nonexistent names.
- **Same trust in CI** — the setup step installs the identical list from the
  identical sources, as root, with no elevation beyond the base image.

## 4. Scope boundary — what this does not touch

- **Base-image build (`vergil-containers`)** — untouched. System packages are a
  per-repo layer, never added to the shared base. That is the whole point.
- **The `[vm]` stanza** — a separate surface (full VMs, not dev containers). Even
  though `[vm].packages` is also an apt list, the two stay distinct because their
  install / trust / lifecycle contexts differ. The design notes the parallel but
  does not couple them.
- **`mnemosys-project/melete` adoption** — downstream and **cross-org**. melete
  lives in a different org, so its adoption (declaring `system-packages =
  ["lilypond"]` and unblocking its two render tasks) is a follow-on epic in
  `mnemosys-project`, seeded by this epic's follow-on-brainstorm bookend (#274) —
  not linked here (cross-org linking is disallowed).
- **`mnemosys-project/melete#21`** (publishing aarch64 LilyPond wheels) — a
  repo-specific sidestep for that one case; does not address the general problem
  and is not part of this epic.

## 5. Sequencing

`vergil-tooling (accessor + local bake) → vergil-actions (CI step) → adoption → validation`.

The invariant that matters is **no repo may declare `system-packages` until both
the tooling accessor and the CI step are live** — otherwise a repo could declare
packages that CI silently ignores, the divergence the shared containers exist to
prevent. That invariant is preserved by gating **adoption**, not by landing CI
first.

The real dependency order follows from the single-reader decision (§3.1, §3.3):
CI reads the key through a `vergil-tooling` `vrg-*` accessor and installs it in
the CI job, so `vergil-tooling` must land — and be installed in the CI job —
**before** `vergil-actions` can consume it. Hence tooling first, then the CI
step, then adoption. The plan resolves the concrete `vrg-*` entry-point name and
how the composite step installs `vergil-tooling` ahead of the apt step (the
existing `setup/vergil` composite action is the natural seam).

## 6. Acceptance

- A repo declaring `[container].system-packages = ["lilypond"]` gets a local
  cached image with the binary present; `vrg-container-run` finds it on `PATH`.
- The same repo's **test** CI jobs have the binary present before the tests run
  (lint/typecheck jobs deliberately do not — the test-runtime contract, §3.3).
- CI reads the package list through the single §3.1 accessor, not ad-hoc shell
  parsing; a transient apt/mirror failure is absorbed by the bounded retry rather
  than failing the PR.
- An empty or absent `system-packages` produces behaviour byte-identical to
  today (no apt step, same cache hash inputs beyond the always-hashed
  `vergil.toml`).
- Editing the package list changes the cache hash and triggers a rebuild.
- A bogus package name fails the build **and** the CI step with a message naming
  the package and the architecture — no silent skip, no degraded image.
- The `[container].system-packages` docs and the container-model guide describe
  the key, its Debian coupling, the trust model, and the fail-closed behaviour.

## 7. Testing

- **`config.py`** — parse a valid list; reject non-list and non-string entries;
  default `[]` when the key or `vergil.toml` is absent; unknown-key warning path
  unaffected.
- **`container_cache.py`** — apt-fragment construction from a package list; empty
  list ⇒ no apt step (byte-identical `setup`); a `system-packages` edit changes
  `compute_cache_hash` (the §3.6 invariant); fail-closed message format at the
  fragment/build boundary.
- **`vergil-actions`** — the composite step is exercised by its own workflow
  tests / a fixture repo declaring a package; the fail-closed path asserts a
  non-zero exit with the expected message; the step reads the list through the
  `vrg-*` accessor (not shell parsing) and the bounded retry is exercised (a
  simulated transient failure that succeeds on retry).
- **Cold-rebuild validation (#2725)** — after the mechanism merges, a live cold
  rebuild of a repo declaring `system-packages` confirms the package is present
  and that a bogus name fails closed. This is the operational acceptance the unit
  tests cannot provide.
