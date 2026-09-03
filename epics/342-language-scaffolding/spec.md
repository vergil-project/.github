# Born-green language-skeleton scaffolding for `vrg-github-repo-init` (cpp first)

Epic: `vergil-project/.github#342`.

## 1. Summary

`vrg-github-repo-init` scaffolds a repo's **config** but stamps **no language
skeleton**. This epic gives it a language-skeleton phase, driven by the
`--language` it already takes, so a new repo of a supported language is **born
green** — the tools run and pass immediately, with nothing left to hand-assemble.
**C++ is implemented first** (current pain); the structure is per-language so
python/typescript/… follow the same shape.

The epic's scope is deliberately two-sided: build the mechanism **and** pay down
the defensive workarounds this class of defect forced (§6), since the mechanism
is what makes them unnecessary.

## 2. The problem

For C++, a new repo's `CMakeLists.txt`, `conanfile.txt`, `src/`, `tests/`, and its
committed `conan.lock` are hand-written, one repo at a time. The stretch where the
repo exists but its skeleton is incomplete or inconsistent — the
**hand-bootstrapping window** — is where an entire session of defects lived:

- the container warmup **running-and-failing** on a half-bootstrapped tree;
- Conan generator output **polluting the source root**;
- a **missing committed `conan.lock`** breaking validation (mandatory since
  `vergil-tooling#3021`).

Each was patched defensively in the tooling. The root cause is singular: **no
mechanism produces a complete, correct, green language skeleton in one shot.**

## 3. Design

### 3.1 Flow

`vrg-github-repo-init <name> --language <lang>` gains one phase after its existing
config scaffolding:

```
1. (existing, host)   config: vergil.toml, CLAUDE.md, .gitignore, workflows, …
2. (NEW, host)        render the language SKELETON from per-language templates
3. (NEW, container)   resolve locks: run the language's lock command → commit
4. (NEW, container)   one `vrg-validate` to assert born-green
5. (existing)         initial commit / push
```

Steps 1–2 stay **host-only** (pure templating). Only step 3 (lock resolve) and
step 4 (verify) enter the container — the "needs a container" cost is confined to
exactly the two steps that require it.

**Container is a hard, fail-fast precondition for a language whose init resolves
locks (cpp).** Before writing anything, init checks that a container runtime is
available; if not, it refuses with a clear message and creates nothing. A cpp
repo is therefore **born green or not born at all** — never half-created — which
is the epic's no-partial-states invariant. (repo-init runs where the vergil
workflow runs, so a runtime is normally present; this only makes the rare
absence a clean refusal rather than a mid-init failure.)

**Step 4 runs the full `vrg-validate`, deliberately.** "Born green" means *every*
gate is green at birth, so the repo's first real PR is guaranteed green. This
makes cpp init slow (cold image build + gtest compiled from source, no armv8
prebuilt on ConanCenter). That one-time cost is accepted as the price of the
guarantee; the image is cached for the repo afterward. A `--no-verify` pressure
valve is explicitly a *later* addition if init time becomes painful — not part
of v1, because it would weaken the default guarantee.

### 3.2 Components

- **`vergil_tooling/data/skeletons/<lang>/`** — the static skeleton templates with
  thin placeholders, mirroring the existing `render_*` config templates.
- **`lang_scaffold.py`** — a `scaffold_language(ctx)` step (its own module, since
  it grows per language) that renders the skeleton then invokes the container
  resolve.
- **`languages.py` `lock` command** — the per-language lock command lives in the
  registry that already owns INSTALL/LINT/TEST, **not** a new hardcoded spot.
  `#3021` removed `conan lock create` from INSTALL; this gives it a proper home
  (single source of truth for the scaffold and any future "re-lock").

### 3.3 The cpp skeleton

Built **on top of `#2912`** (Conan output → `build/`) so the template ships clean
rather than baking in today's workarounds. **`#2912` is designed and delivered as
task 1 *of this epic*** (see §6) — not an external prerequisite. It is not a
one-liner: it rewires how every cpp `cmake` invocation finds Conan (toolchain
file instead of the source-root `CMAKE_PREFIX_PATH` hack) across the `build` and
`build-sanitize` configure passes, and changes the INSTALL/typecheck/test
commands in `languages.py`. The plan (writing-plans) details it.

| file | purpose | templating |
|---|---|---|
| `conanfile.txt` | `[requires] gtest/<pinned>`; `[options] gtest/*:build_gmock=False`; `[generators] CMakeDeps CMakeToolchain` | fixed; gtest pinned |
| `CMakeLists.txt` | the `VERGIL_CPP_STD/STDLIB/COVERAGE/SANITIZE` contract; `find_package(GTest)` **via the Conan toolchain file** (no source-root prefix-path hack, given `#2912`); placeholder lib + `gtest_discover_tests` | fixed except a repo-derived name |
| `src/<name>.hpp`/`.cpp` | one trivial unit (`toolchain_ready() → true`), no domain logic, labeled "remove when real code lands" | name derived |
| `tests/<name>_test.cpp` | one GoogleTest smoke test → 100% line coverage of `src/` | name derived |
| `conan.lock` | **resolved** in step 3, pinning gtest | generated |

Born-green by construction: placeholder + test give 100% coverage; gtest matches
the pipeline's `cppcheck --library=googletest`; `build_gmock=False` avoids the
`gmock_main` configure error; step 3 supplies the committed `conan.lock` `#3021`
requires. Templating is thin — the only substitution is a sanitized
project/namespace name derived from the repo name (`mq-protocol-gateway` →
`mq_protocol_gateway`).

### 3.4 Greenfield-only; retrofit is manual

The scaffold is **greenfield-only** — it produces a complete skeleton at repo
creation. Re-running on an existing repo stamps only **missing** required files
and never clobbers existing ones. `--force` is a narrow, clearly-labeled
"regenerate the skeleton, overwriting — safe only on an un-customized repo"
hatch; it is **not** a retrofit mechanism, because `CMakeLists.txt` is exactly
the file real repos customize (added targets, deps), so no automated force-update
of it is safe in general.

**Retrofitting an existing repo is therefore a manual PR, not a scaffold run.**
In particular, bringing `mq-protocol-gateway` onto the clean path (§6 task) is a
deliberate hand-edit — swap in the clean `CMakeLists.txt`, delete the repo's
`.gitignore` Conan blocks — which is safe there precisely because that repo is
still the placeholder skeleton, barely customized.

## 4. Testing

- **Host unit tests (bulk):** template rendering (content + name substitution),
  the `languages.py` `lock` command entry, idempotency (stamp-missing,
  no-clobber, `--force`). Pure functions, no container.
- **One container integration test:** scaffold a cpp repo → resolve →
  `vrg-validate` green, end to end. Needs container + network (conan fetches
  gtest), so it is gated like the existing `test_vrg_container_*` integration
  tests.
- **100% coverage invariant** applies to all new code.

## 5. Scope — paying down the workarounds

The epic is **not done** until the defensive debt this class of defect forced is
undone. Split by what removes each.

### 5.1 Removed by born-green (this scaffold)

- **`_WARMUP_REQUIRES` cpp skip (`#2881`).** Added so an unbootstrapped cpp
  repo's warmup skips instead of crashing. Born-green means no half-bootstrapped
  cpp state exists, so the cpp reliance is dead — **remove the cpp entry.** (Full
  removal of the mechanism awaits every language being scaffolded; until then it
  stays for the not-yet-scaffolded languages and as a guard for manually-created
  repos.)
- **The `conan.lock` warmup one-liner** (the gap surfaced during the fight):
  never added — born-green guarantees the lock exists.

### 5.2 Removed by the `#2912` prerequisite (Conan output → `build/`)

- Baseline `.gitignore` Conan generic block (`#2878`).
- Baseline `.gitignore` Conan CMakeDeps per-package block (`#2908`).
- `mq-protocol-gateway` `CMakeLists.txt` `CMAKE_PREFIX_PATH`/`CMAKE_MODULE_PATH`
  source-root hack — replaced by the toolchain-file path.
- `mq-protocol-gateway` repo `.gitignore` Conan blocks — the repo mirror of the
  baseline blocks.

### 5.3 Kept — correct fixes, orthogonal to bootstrap

`#2859` (image selection from asserted language), `#853` (semgrep `p/c`), `#863`
(version-bump `python3` from PATH), `#2906`/`#2907` (lint worktree-scope +
provisioning), `#2902` (token resolved from file), `#3021` (committed lock is the
desired end state — the scaffold satisfies it), `#3026` (gcov ignores
`source_not_found` on third-party dep headers — supporting fix that lets the
skeleton's gtest headers not break the 100% coverage gate), `#2895` (audit skip
on absent token — graceful degradation for tokenless/fork contexts).

## 6. Sequencing (dependency order)

1. **`#2912` — Conan output → `build/`** — designed and built as task 1 **of this
   epic** (§3.3): the toolchain-file wiring across `build`/`build-sanitize`, the
   `languages.py` command changes. Enables the clean skeleton template and the
   §5.2 reverts.
2. **Scaffold mechanism** (this spec), built on task 1: `languages.py` `lock`
   command, `skeletons/cpp/` templates, `lang_scaffold.py`, repo-init wiring
   (with the fail-fast container precondition + full-`vrg-validate` verify).
3. **Retrofit `mq-protocol-gateway`** — a **manual PR** (§3.4), not a scaffold
   run: drop the `CMakeLists.txt` prefix-path hack, revert the repo's
   `.gitignore` Conan blocks; verify born-green.
4. **Revert the baseline `.gitignore` Conan blocks** (`#2878`, `#2908`).
5. **Remove the cpp `_WARMUP_REQUIRES` entry** (`#2881`).

Steps 4–5 land only after 1–3 are in and mq-protocol-gateway is confirmed green
on the clean path.

**Closing acceptance — live born-green validation.** The epic's headline claim
("a new repo is born green") is proven by an operational **`validation`** bookend
(run via `issue-validate`, `Blocked-by` the scaffold task): actually run
`vrg-github-repo-init --language cpp` end to end and confirm the resulting repo is
green with no hand-assembly. The mechanism-level integration test (§4) proves
`scaffold_language`; this proves the real command. It closes on a recorded
`Outcome: SUCCESS` and gates the epic's rollup.

## 7. Non-goals

- Real application code or real dependencies — the skeleton carries only
  "enough to go green."
- Retrofitting every language at once. cpp is implemented; python (which also
  commits a `uv.lock`) is the obvious next language but a **follow-up epic**.
- Full retirement of `_WARMUP_REQUIRES` — gated on all languages being
  scaffolded; only the cpp entry is removed here.
- Reversing the §5.3 correct fixes.

## 8. Known tradeoffs / risks

- **Network in the resolve step.** `conan lock create` fetches recipes; the
  resolve step and the integration test depend on ConanCenter reachability.
  Acceptable (CI already fetches conan deps), but noted.
- **gtest version pin drift.** The template pins a gtest version and needs the
  usual dependency-bump maintenance.
- **Container dependency in repo-init.** Steps 3–4 make repo-init depend on the
  container for languages that resolve locks. This is accepted deliberately (a
  born-green repo needs its resolved lock) and contained to those two steps.
