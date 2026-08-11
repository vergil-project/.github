# A `build-command` hook for baking non-apt deps into dev containers

- **Epic:** vergil-project/.github#291
- **Status:** Design (2026-08-11)
- **Origin:** promoted from triage issue vergil-project/vergil-tooling#2751.
- **Implemented in:** vergil-tooling (config + local cache-build + speller) +
  vergil-actions (CI setup step). Docs across both.
- **Direct precedent:** vergil-project/.github#272 (`[container].system-packages`,
  the apt layer this generalizes). This spec assumes #272's mechanisms and extends
  them; read §3 of #272's spec first.

## 1. Summary

`#272` gave `[container]` a declarative **apt** layer: a repo names Debian
packages, and they are installed identically in the local cached image and in CI,
without touching the shared base image. That layer is **apt-only** by construction
— its config schema allows exactly `{env-prefixes, system-packages}`
(`config.py`), and the image is built purely by `apt install`-ing those names
(`container_cache.py`).

A real project needs a **non-apt** dependency baked in the same way.
`mnemosys-project/melete`'s alphaTab output port vendors the npm package
`@coderline/alphatab` under `melete-render/`; the renderer and its black-box
integration test need it resolvable in-container. Node is already in the base image
(`node v22`, `npm 12`), so the **runtime** is not the gap — the gap is that there
is no post-apt hook to run `npm ci` (or any build/dependency-install step) at image
build.

This epic adds one key:

```toml
[container]
build-command = "cd melete-render && npm ci"
```

a single shell command run during the container setup step, after the apt layer
and before the language warmup, flowing through the **same single speller** shared
by the local cache build and CI — the invariant `#272` established.

## 2. Motivation & constraints

The LilyPond precedent (`#272`) does not generalize: LilyPond is one apt package,
and `system-packages = ["lilypond"]` installs it into `/usr` with `apt`. An npm
dependency is not an apt package and does not install into `/usr`; it is a
`node_modules` tree produced by `npm ci` against a vendored `package.json` /
`package-lock.json`. There is no apt name to declare and no `apt` step that
produces it. The asymmetry this epic closes is precisely: **apt deps are
declarable; every other kind of build/dependency-install step is not.**

The extension point is the same one `#272` used: `container_cache.py` already runs
a repo-specific `setup` step in a fresh container and commits the result, and
`#272` already prepends an apt fragment to it. The gap is not a new mechanism —
it is "let the repo contribute one more command to that setup step, and make CI
run it too."

The obvious workarounds are all bad and motivate a first-class hook:

- **Commit `node_modules`** — vendoring a build artifact; bloats the repo, rots,
  and is arch/platform-fragile.
- **`npm ci` at every test run** — slow, network-dependent, cache-defeating; the
  same objection `#272` raised against per-run apt.
- **A bespoke image for melete** — abandons the shared base and its maintenance.

## 3. Design

### 3.1 Config surface — a single shell string

A new optional key in the existing `[container]` table:

```toml
[container]
build-command = "cd melete-render && npm ci"
```

- **A single string**, run as `bash -c` shell during the setup step. The repo
  chains multiple steps with `&&`, matching how the apt fragment is already one
  `&&`-joined string. **Not** a list in v1 (YAGNI; a list is a trivial follow-up
  if a real multi-line need appears).
- **General, not npm-specific.** The hook runs whatever the repo commits. The npm
  case is the *driver*, not the *contract*: `make`, `pip install`, a vendored
  build, anything, are equally valid. Special-casing npm would be more machinery
  to solve a problem one general command already covers.
- `config.py`: add `build-command` to `_KNOWN_KEYS["container"]`; add
  `build_command: str | None` to `ContainerConfig` (default `None`); validate it
  is a string when present (mirroring the `env-prefixes` / `system-packages`
  validation). Add a `container_build_command(repo_root)` accessor alongside
  `container_system_packages`, returning `None` when the key or `vergil.toml` is
  absent. This accessor is the **single reader** of the key; both the local bake
  (§3.2) and CI (§3.3) go through it.

### 3.2 Local path — run in the cache-build setup step

`container_cache.py::_build_cached_image` composes `setup` today as
(apt fragment) → (vergil-tooling install) → (language warmup). Insert the
build-command **between the apt fragment and the vergil-tooling install**:

```
<apt fragment> && <build-command> && <uv tool install …> && <warmup>
```

Rationale for the position: "system deps (apt) → repo build deps (build-command)
→ language deps (vergil-tooling + warmup)." The build-command may rely on apt
packages the same repo declared, so it runs after apt; it is independent of
vergil-tooling, so it runs before it. (§7 open question O3 revisits whether a repo
could ever need the language toolchain *before* its build-command; v1 commits to
this order and documents it.)

- **Fail-closed.** A non-zero `build-command` fails the image build hard, exactly
  like a failed apt install — no silent skip, no degraded image (mirrors #272
  §3.5). The build banner already prints provisioning steps; add the build-command
  to it.
- **Empty/absent ⇒ byte-identical to today.** No key ⇒ no fragment ⇒ the `setup`
  string is unchanged.

### 3.3 CI path — a new vergil-actions setup step

CI does not use the local cache layer; each job runs in the **base** image
directly, and `#272` added a composite action
(`actions/shared/setup/system-packages`) whose `install.sh` runs
`vrg-container-system-packages --install-script` wrapped in a **bounded apt-mirror
retry**. The build-command **cannot reuse that step**: it is not an apt operation,
and the apt-mirror-flake retry is the wrong failure model for an arbitrary command
(a failing `npm ci` should not be retried as if it were a transient mirror hiccup).

So CI parity requires a **new composite step** (working name
`actions/shared/setup/build-command`) that:

- reads the command through the single §3.1 accessor via a new speller mode
  (e.g. `vrg-container-build-command --script`), so CI never parses `vergil.toml`
  by hand and never spells the command itself — the same single-speller invariant
  as #272;
- runs it in the **workspace** (the checked-out repo), after the system-packages
  step and after `vergil-tooling` is installed, on the **test jobs only** (the
  same test-runtime contract as #272 §3.3 — a build dep is needed at test time);
- is **fail-closed with no retry** (unlike the apt step): a non-zero command fails
  the job with the command's own output.

**One command, two consumers.** The local cache build and the CI step read the
single `[container].build-command` through the same accessor and run the identical
command. They differ on *when/where* (baked into the local setup step vs run
per-job in CI), never on *what* — the #272 invariant, restated.

### 3.4 Cache correctness — the build-command's inputs must be hashed

`#272` got cache correctness for free: `system-packages` lives in `vergil.toml`,
which is already in `_CACHE_FILES` for every language, so editing the list
rehashes and rebuilds. The **command string** here gets the same free coverage —
it lives in `vergil.toml`.

But a `build-command`'s real inputs are usually **not** in `vergil.toml`: bumping
`@coderline/alphatab` changes `melete-render/package-lock.json`, **not**
`vergil.toml`, so the cache hash would not change and the stale image would be
reused — a **silent stale-dependency cache**, which violates "no silent failures."
This is the one genuinely new cache concern, and the design must close it. Two
candidate mechanisms (decision deferred to pushback/plan — §7 O2):

- **(a) Companion key** — `[container].build-cache-files = ["melete-render/package-lock.json"]`,
  an explicit list folded into `compute_cache_hash`. Explicit, greppable, and it
  makes the sharp edge visible: if you don't list your lockfile, a dep bump won't
  rebuild. Recommended.
- **(b) Document the sharp edge only** — track nothing extra; state that a
  build-command whose inputs live outside `vergil.toml` must bump `vergil.toml`
  (or the operator clears the cache) to force a rebuild. Cheaper, but leaves a
  silent-staleness footgun — weaker on the "no silent failures" bar.

The recommendation is **(a)**; a test asserts editing a listed file changes the
hash (the invariant, mirroring #272 §3.6).

### 3.5 Trust model — broader than #272, stated plainly

This is the **material** difference from `#272` and must not be understated.
`system-packages` is a constrained surface: Debian package *names* only, from
existing sources, installed as root — the worst case is pulling a Debian-packaged
tool. `build-command` is **arbitrary shell run as root at image build**. That is a
strictly larger surface.

- **Trust boundary is unchanged in kind.** The command comes from the repo's own
  committed `vergil.toml`, reviewed in the PR diff, exactly like `system-packages`.
  A contributor who can land a `vergil.toml` change can already land arbitrary code
  in the repo's own test suite, which runs in the same container as root. So
  `build-command` does not grant a capability an attacker did not already have via
  a normal code PR — it is the same trust boundary (repo write access + review),
  applied to a broader action.
- **What is genuinely new** is that the surface is now "any command" rather than "a
  name from Debian main," so the fail-closed backstop (typo ⇒ build fails, §3.2)
  and PR review are the only guards; there is no name allowlist to lean on. The
  docs must state this explicitly at the key.
- **No network policy change.** The build runs with whatever network the cache
  build/CI already has; this epic does not add or remove network access. A
  `build-command` that reaches the network (e.g. `npm ci`) is as
  network-dependent as any other setup step, and that dependence is the repo's to
  own.

## 4. The bind-mount problem — where does a build-command's output go?

This is the **central technical question** and the reason the epic carries a live
validation rather than trusting unit tests. It is stated here as an explicit open
problem, resolved in pushback/plan/validation — not papered over.

**The facts (verified in code).** `workspace_mount_args` (`container.py:179`)
bind-mounts the repo to `/workspace` at **both** build and run time. Therefore:

1. **At build time**, `cd melete-render && npm ci` writes
   `/workspace/melete-render/node_modules` — a bind-mounted path — so the tree
   lands on the **host** (or the CI checkout), **not** in the committed image
   layer. `docker commit` does not capture bind mounts.
2. **At run time**, `vrg-container-run` bind-mounts the host `/workspace` again,
   which **masks** anything the image baked under `/workspace`. So even if
   `node_modules` *were* committed at a workspace path, the runtime mount would
   hide it.

**Consequence.** A naïve `npm ci` into the vendored dir does **not** produce a
dependency "baked into the image" in any runtime-visible sense. What it actually
produces is:

- locally: `node_modules` written into the **host** working tree as a side effect
  of the cache build, visible at the next `vrg-container-run` only because the same
  host tree is re-mounted (fragile: a fresh clone / `git clean` / a different
  worktree has none), plus a warmed npm cache (`~/.npm`) in the image;
- in CI: `node_modules` in the per-job checkout, present for that job's tests,
  rebuilt every run.

**The design fork (to resolve in pushback):**

- **Model A — workspace provisioning.** Accept that `build-command` populates the
  **workspace**, not the image. It runs in the cache build (leaving `node_modules`
  in the local tree) and in the CI step (per job). "Baked into the image" is
  downgraded to "the image warms the caches (`~/.npm`) so the workspace populate is
  fast/offline." Honest, matches how a vendored `node_modules` actually wants to
  live (adjacent to its package), but leaves a host-tree side effect and no true
  image-resident artifact. Likely the pragmatic answer for the melete case.
- **Model B — out-of-workspace install.** Require the `build-command` to install
  **outside** `/workspace` (a global `npm install -g`, or a fixed image path with
  `NODE_PATH`), so the dependency lives in a committed image layer that the
  runtime mount does not mask. Truly "baked," but fights the vendored-package
  layout and pushes complexity onto every repo.

The spec does **not** pre-decide this; the mechanism (§3.1–§3.4) is identical
either way. What differs is the **acceptance criteria of the validation task**,
which must state which model is in effect and prove the dependency resolves under
it (including a fresh-tree check, to catch Model A's host-pollution fragility).

## 5. Scope boundary — what this does not touch

- **Base-image build (`vergil-containers`)** — untouched. A build-command is a
  per-repo layer, never added to the shared base — same principle as #272.
- **`mnemosys-project/melete` adoption** — downstream and **cross-org**. melete's
  declaring `build-command` and unblocking `mnemosys-project/melete#85` is that
  org's concern (epic `mnemosys-project/.github#46`), **not linked here**
  (cross-org linking is disallowed). This epic delivers and validates the
  capability; melete consumes it.
- **A `build-command` list / multi-command form** — deferred (§3.1); `&&` covers
  v1.
- **Per-language auto-detection** (auto-`npm ci` when a `package.json` exists) —
  explicitly not done; the hook is declarative, like `system-packages`.

## 6. Acceptance

- A repo declaring `[container].build-command = "…"` has the command run in its
  local cached-image setup step, after the apt layer and before the
  vergil-tooling/warmup step; a non-zero command fails the build with the command's
  output (fail-closed).
- The same repo's **test** CI jobs run the identical command (via the new
  vergil-actions step, reading the single §3.1 speller), fail-closed with no retry;
  lint/typecheck jobs do not (test-runtime contract).
- CI reads the command through the single accessor, never ad-hoc shell parsing.
- An absent `build-command` produces behaviour byte-identical to today (no extra
  setup fragment, same cache-hash inputs beyond the always-hashed `vergil.toml`).
- Cache correctness (§3.4): editing the command **or** any declared
  build-cache-file changes the cache hash and triggers a rebuild; not tracking a
  build input is a documented sharp edge, not a silent success.
- **Cold-rebuild validation** (operational task): a live cold rebuild of a repo
  declaring a real npm `build-command` proves the dependency **resolves
  in-container** under the chosen model (§4), including a fresh-tree check that it
  does not depend on pre-existing host `node_modules`. This is the acceptance the
  unit tests cannot provide, and the venue where the §4 model is nailed down.
- Docs: the `[container].build-command` reference (site docs) and the new
  vergil-actions step doc describe the key, its position in the setup order, the
  cache-correctness rule, the **broadened trust model** (§3.5), and the
  workspace-vs-image reality (§4).

## 7. Open questions (resolved in pushback / plan / validation)

- **O1 — the §4 model (A vs B).** The single most important decision; drives the
  validation's acceptance criteria. Leaning Model A for the melete driver, but
  pushback should test whether any consumer genuinely needs a runtime-visible,
  image-resident (Model B) artifact.
- **O2 — cache-correctness mechanism (§3.4 a vs b).** Companion `build-cache-files`
  key (recommended) vs documented sharp edge.
- **O3 — setup order.** Is "apt → build-command → vergil-tooling/warmup" ever
  wrong (a build-command needing the language toolchain first)? v1 commits to this
  order; O3 records the reconsideration trigger.
- **O4 — does vergil-actions truly need a *new* step, or can the existing
  system-packages step be generalized** to run a trailing non-apt command without
  the apt retry? Plan resolves the concrete action shape.

## 8. Bookends (this epic)

- **Documentation (first):** this spec + the plan — vergil-project/.github#292.
- **Documentation-review (closing, sweep):** vergil-project/vergil-tooling#2756 —
  the `[container].build-command` site docs + any per-repo siblings (notably the
  vergil-actions step doc).
- **Retrospective (terminal):** vergil-project/.github#293.
- **Implementation + validation:** filed from the plan (§9 of the workflow) — the
  vergil-tooling change, the vergil-actions step, and the cold-rebuild validation
  (`Blocked-by` the implementation), each in the repo where its PR lands.
