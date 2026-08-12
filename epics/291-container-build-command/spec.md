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

This epic adds a key:

```toml
[container]
build-command    = "npm install -g @coderline/alphatab"
build-cache-files = ["melete-render/package-lock.json"]
```

a single shell command run during the container setup step — after the apt layer
and the vergil-tooling install, before the language warmup — flowing through the
**same single-speller** discipline `#272` established. The command must install
**outside** the workspace (§4), and its input files are declared in
`build-cache-files` so a dependency bump rebuilds the image (§3.4).

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
build-command = "npm install -g @coderline/alphatab"
```

- **A single string**, run as `bash -c` shell during the setup step. The repo
  chains multiple steps with `&&`, matching how the apt fragment is already one
  `&&`-joined string. **Not** a list in v1 (YAGNI; a list is a trivial follow-up
  if a real multi-line need appears).
- **General, not npm-specific.** The hook runs whatever the repo commits. The npm
  case is the *driver*, not the *contract*: `make`, `pip install`, a vendored
  build, anything, are equally valid. Special-casing npm would be more machinery
  to solve a problem one general command already covers.
- **Out-of-workspace contract (decided — §4).** The command **must** install its
  artifacts *outside* `/workspace` (a global `npm install -g`, or a fixed image
  path surfaced via `NODE_PATH`), because the runtime bind-mount masks anything at
  a workspace path (§4). This is a documented contract at the key, not something
  the tooling can enforce; the validation task proves it on a clean tree.

  > **Erratum (#2766/#2781):** the global `npm install -g` case above is **not**
  > `NODE_PATH`-free — it is presented here as if only the "fixed image path"
  > alternative needs `NODE_PATH`, which is wrong. A global install lands in
  > `/usr/lib/node_modules`, which is **not** on Node's default `require.resolve`
  > search path, so the global case **also** needs `NODE_PATH` set to resolve
  > (CommonJS `require` only; ESM `import` ignores `NODE_PATH`). The epic's
  > validation (vergil-tooling#2766) caught this and the fix (vergil-tooling#2781,
  > shipped in v2.1.191) wired up `NODE_PATH`. The out-of-workspace / Model B
  > decision itself is correct — only the resolution mechanism was misstated. See
  > the §4 erratum.
- A companion key, **`build-cache-files`** (§3.4), lists the command's real input
  files so a dependency bump rebuilds the image.
- `config.py`: add `build-command` and `build-cache-files` to
  `_KNOWN_KEYS["container"]`; add `build_command: str | None` (default `None`) and
  `build_cache_files: list[str]` (default `[]`) to `ContainerConfig`; validate
  `build-command` is a string when present and `build-cache-files` is a list of
  strings (mirroring the `env-prefixes` / `system-packages` validation). Add
  `container_build_command(repo_root)` and `container_build_cache_files(repo_root)`
  accessors alongside `container_system_packages`, returning `None` / `[]` when the
  key or `vergil.toml` is absent. `container_build_command` is the **single
  reader** of the command; both the local bake (§3.2) and CI (§3.3) go through it.

### 3.2 Local path — run in the cache-build setup step

`container_cache.py::_build_cached_image` composes `setup` today as
(apt fragment) → (vergil-tooling install) → (language warmup). Insert the
build-command **after the vergil-tooling install and before the warmup**:

```
<apt fragment> && <uv tool install …> && <build-command> && <warmup>
```

Rationale for the position: apt first (the command may rely on apt packages the
repo declared), then the vergil-tooling install, then the build-command. Running
it **after** the vergil-tooling install — rather than before — is deliberate: it
makes the local build-command environment identical to CI's, where the
`Install vergil-tooling` step always precedes the setup steps (§3.3). If
build-command ran before the install locally but after it in CI, a command that
touched any `vrg-*` tool would behave differently by path — the "same
declaration, different result" drift the single-speller design exists to prevent.
Aligning both paths to "after the install" removes that divergence.

For the **self-repo** (vergil-tooling itself), `_build_cached_image` skips the
`uv tool install` (the dev tooling is already on PATH); there, build-command runs
after apt and before warmup, in the same relative slot, against the same
tooling-present environment.

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
This is the one genuinely new cache concern, and the design closes it with an
explicit companion key:

```toml
[container]
build-command    = "npm install -g @coderline/alphatab"
build-cache-files = ["melete-render/package-lock.json"]
```

- `build-cache-files` is a list of repo-relative paths folded into
  `compute_cache_hash` alongside the language cache set. Editing any listed file
  changes the hash and forces a rebuild.
- **The sharp edge is made visible in the config itself:** your build inputs are
  exactly what you list. If a repo declares a `build-command` but forgets to list
  its lockfile, a dependency bump will not rebuild — so the docs state the rule at
  the key ("list every file your build-command reads"), and the fail-closed build
  (§3.2) still catches a command that references a missing file.
- A test asserts editing a listed file changes the hash (the invariant, mirroring
  #272 §3.6), and that an absent `build-cache-files` leaves the hash inputs
  unchanged.

The rejected alternative — track nothing and rely on the operator to bump
`vergil.toml` or clear the cache — was declined for leaving the silent-staleness
footgun armed.

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

## 4. The bind-mount problem — and why the output must land outside `/workspace`

This is the **central technical constraint**, and the reason the epic carries a
live validation rather than trusting unit tests.

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
dependency baked into the image in any runtime-visible sense — locally it merely
pollutes the host tree (fragile: a fresh clone / `git clean` / a different
worktree has none), and in CI it is rebuilt per job. A workspace-resident
dependency cannot be an image feature at all; the image could only ever carry the
`~/.npm` *cache*, never the deps.

**Decision — Model B (out-of-workspace install) is the contract.** A
`build-command` **must** install its artifacts *outside* `/workspace`, so they
live in a committed image layer the runtime mount does not mask. Concretely, the
npm case is a **global** install:

```toml
[container]
build-command = "npm install -g @coderline/alphatab"
```

which lands under the image's global `node_modules` (on the default resolution
path), survives `docker commit`, and is visible at every `vrg-container-run` and
in every CI job **without** depending on any host-tree state. Where a package must
live at a fixed non-global path, the repo installs it there and surfaces it via
`NODE_PATH` (or the language's equivalent) — the rule is simply "not under
`/workspace`."

> **Erratum (#2766/#2781):** the parenthetical "(on the default resolution
> path)" is **wrong**. A global `npm install -g` lands in `/usr/lib/node_modules`,
> which is **not** on Node's default `require.resolve` search path; it resolves
> only when `NODE_PATH` points at that directory — which the epic's fix
> (vergil-tooling#2781, shipped in v2.1.191) wired up after the validation
> (vergil-tooling#2766) caught the omission. This applies to CommonJS `require`
> **only** — ESM `import` ignores `NODE_PATH`. The rest of the sentence (survives
> `docker commit`, host-independent) is correct, and the Model B out-of-workspace
> decision stands; only the resolution-mechanism claim needed correcting — the
> global case is no more "default-path" than the fixed-path case, both needing
> `NODE_PATH`.

The **rejected alternative** (Model A — populate `/workspace`, let the image warm
only the cache) was declined because it produces no runtime-visible baked
artifact, makes local behaviour depend on host-tree pollution, and would make the
epic's own framing ("baked the same way apt is") false.

This contract is stated at the key in the docs (§3.1) and is the crux the
**validation task** proves: a cold rebuild of a repo declaring an out-of-workspace
`build-command`, then a **clean-tree** `vrg-container-run` confirming the
dependency resolves with no pre-existing host `node_modules`.

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
  local cached-image setup step, **after** the apt layer and the vergil-tooling
  install and **before** the warmup (§3.2); a non-zero command fails the build with
  the command's output (fail-closed).
- The same repo's **test** CI jobs run the identical command in the same relative
  slot — after the vergil-tooling install — via the new vergil-actions step
  reading the single §3.1 accessor; fail-closed with no retry; lint/typecheck jobs
  do not (test-runtime contract). The local and CI environments at command time
  match (vergil-tooling present in both), so a command cannot behave differently by
  path.
- CI reads the command through the single accessor, never ad-hoc shell parsing.
- An absent `build-command` produces behaviour byte-identical to today (no extra
  setup fragment, same cache-hash inputs beyond the always-hashed `vergil.toml`).
- Cache correctness (§3.4): editing the command **or** any file listed in
  `build-cache-files` changes the cache hash and triggers a rebuild; a build input
  the repo failed to list is a documented sharp edge, not a silent success.
- **Out-of-workspace contract (§4):** a `build-command`'s artifacts install
  outside `/workspace` so they survive `docker commit` and the runtime mount.
- **Cold-rebuild validation** (operational task): a live cold rebuild of a repo
  declaring a real out-of-workspace npm `build-command`, followed by a
  **clean-tree** `vrg-container-run`, proves the dependency **resolves
  in-container with no pre-existing host `node_modules`**. This is the acceptance
  the unit tests cannot provide.
- Docs: the `[container].build-command` / `build-cache-files` reference (site docs)
  and the new vergil-actions step doc describe the keys, the setup-order position,
  the out-of-workspace contract (§4), the cache-correctness rule, and the
  **broadened trust model** (§3.5).

## 7. Decisions locked in pushback, and the one open question

**Resolved in pushback (2026-08-11):**

- **D1 — output location (§4): Model B, out-of-workspace install** is the contract.
  A workspace-resident dependency is masked by the runtime mount and cannot be an
  image feature; artifacts install outside `/workspace` (global, or a fixed path
  via `NODE_PATH`). Drives the validation's clean-tree acceptance.
- **D2 — setup order (§3.2): after the vergil-tooling install in both paths.** The
  local and CI command environments match, eliminating same-declaration /
  different-result drift.
- **D3 — cache correctness (§3.4): the `build-cache-files` companion key.** The
  command's real inputs are declared and folded into the cache hash, closing the
  silent stale-dependency hole; the sharp edge (list your inputs) is visible in the
  config and documented at the key.

**Open (resolved in plan):**

- **O4 — does vergil-actions need a *new* step, or can the existing
  system-packages step be generalized** to run a trailing non-apt command without
  the apt-mirror retry? The plan resolves the concrete action shape; the design
  constraint either way is "fail-closed, no retry, after the vergil-tooling
  install, test jobs only."

## 8. Bookends (this epic)

- **Documentation (first):** this spec + the plan — vergil-project/.github#292.
- **Documentation-review (closing, sweep):** vergil-project/vergil-tooling#2756 —
  the `[container].build-command` site docs + any per-repo siblings (notably the
  vergil-actions step doc).
- **Retrospective (terminal):** vergil-project/.github#293.
- **Implementation + validation:** filed from the plan (§9 of the workflow) — the
  vergil-tooling change, the vergil-actions step, and the cold-rebuild validation
  (`Blocked-by` the implementation), each in the repo where its PR lands.
