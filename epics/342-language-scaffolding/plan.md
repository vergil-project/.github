# Born-green language-skeleton scaffolding — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. In the Vergil framework each **Task** below becomes one linked GitHub issue and one PR under epic `vergil-project/.github#342`, implemented via `issue-implement`.

**Goal:** Give `vrg-github-repo-init` a language-skeleton phase so a cpp repo is *born green*, and pay down the defensive workarounds that phase makes unnecessary.

**Architecture:** First relocate Conan's generator output under `build/` and wire cmake through the Conan toolchain file (Task 1), so the skeleton template ships clean. Then add per-language skeleton templates + a containerized lock-resolve step to repo-init (Task 2). Then retrofit the one existing cpp repo and delete the now-dead workarounds (Tasks 3–5).

**Tech Stack:** Python (`vergil_tooling`), Conan 2, CMake, GoogleTest, pytest (100% coverage gate), the `vrg-*` CLI wrappers.

**Spec:** `epics/342-language-scaffolding/spec.md` (read it alongside this plan).

## Global Constraints

- **100% line coverage** on all new/changed Python (`vrg-validate` `test` gate). Copy verbatim.
- **Validation is `vrg-container-run -- vrg-validate` only.** No ad-hoc linters.
- **One task = one branch = one PR**, `feature/<issue>-<slug>`, via `issue-implement`; workflow-file and cross-repo rules per `CLAUDE.md`.
- **Conan output folder is `build/`; the Conan toolchain file is `build/conan_toolchain.cmake`** (verified: `conan install --output-folder=build` writes it there and leaves the source root clean).
- **cpp `[cpp]` pins:** `std = c++20`, `stdlib = libstdc++` (from `config.DEFAULT_CPP_STD`/`DEFAULT_CPP_STDLIB`).
- **gtest pin for the skeleton template:** `gtest/1.15.0`, `build_gmock=False`.

---

## Task 1: `#2912` — Conan output → `build/`, cmake via the toolchain file

Relocate Conan's generator output out of the source root and make cmake find packages through the Conan toolchain file instead of the source-root `CMAKE_PREFIX_PATH` hack. Fleet-wide cpp command change; no repo edits here (existing repos keep working — the toolchain file makes `find_package` resolve, and their in-tree prefix-path hack becomes a harmless no-op, removed in Task 3).

**Files:**
- Modify: `src/vergil_tooling/lib/languages.py` — cpp `INSTALL`, `TYPECHECK`, `TEST` command lists; add a `_CPP_TOOLCHAIN_FILE` constant.
- Test: `tests/vergil_tooling/test_languages.py`

**Interfaces:**
- Produces: cpp `language_commands(...)` output where (a) `conan install` carries `--output-folder=build`, and (b) every `cmake -S . -B <dir>` carries `-DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake`. `_CPP_BUILD_DIR = "build"`, `_CPP_SANITIZE_BUILD_DIR = "build-sanitize"` unchanged.

- [ ] **Step 1: Failing test — INSTALL relocates Conan output**

```python
def test_cpp_install_writes_conan_output_to_build() -> None:
    cmds = language_commands("cpp", CheckKind.INSTALL, cpp_std="c++20", cpp_stdlib="libstdc++")
    conan_install = next(c for c in cmds if c[:2] == ["conan", "install"])
    assert "--output-folder=build" in conan_install
```

- [ ] **Step 2: Run it, verify FAIL** — `uv run pytest tests/vergil_tooling/test_languages.py::test_cpp_install_writes_conan_output_to_build -v` → FAIL (flag absent).

- [ ] **Step 3: Add `--output-folder=build`** to the cpp `INSTALL` `conan install` arg list in `languages.py` (after `--lockfile=conan.lock`).

- [ ] **Step 4: Run it, verify PASS.**

- [ ] **Step 5: Failing test — every cpp cmake configure passes the toolchain file**

```python
def test_cpp_cmake_configures_use_conan_toolchain() -> None:
    for kind in (CheckKind.INSTALL, CheckKind.TYPECHECK, CheckKind.TEST):
        cmds = language_commands("cpp", kind, cpp_std="c++20", cpp_stdlib="libstdc++")
        for c in cmds:
            if c[:1] == ["cmake"] and "-S" in c:  # a configure, not a --build
                assert "-DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake" in c, (kind, c)
```

- [ ] **Step 6: Run it, verify FAIL.**

- [ ] **Step 7: Implement** — add `_CPP_TOOLCHAIN_FILE = "build/conan_toolchain.cmake"` and append `"-DCMAKE_TOOLCHAIN_FILE=" + _CPP_TOOLCHAIN_FILE` to each cpp cmake **configure** invocation (INSTALL configure, TYPECHECK configure, TEST coverage configure, TEST sanitize configure). Do **not** add it to `cmake --build` steps. Note the sanitize configure (`-B build-sanitize`) intentionally references the shared `build/conan_toolchain.cmake` (INSTALL creates it first).

- [ ] **Step 8: Run the full languages test file, verify PASS.**

- [ ] **Step 9: Container smoke check** — in a scratch dir with a `conanfile.txt`/`conan.lock`, run the rendered INSTALL+TYPECHECK+TEST commands and confirm green and that `git status`-equivalent shows **no** Conan files at the source root (only `build/`, `build-sanitize/`). Command: `vrg-container-run -- vrg-validate` against a temporary cpp fixture, or manually against mq-protocol-gateway's tree with a throwaway CMakeLists lacking the prefix-path hack.

- [ ] **Step 10: `vrg-container-run -- vrg-validate` green; commit.**

```bash
vrg-commit --type fix --scope cpp --message "write Conan output under build/, cmake via toolchain file (#2912)"
```

---

## Task 2: The scaffold mechanism

Add a per-language skeleton phase to repo-init: host-render templates, containerized lock-resolve, full-`vrg-validate` verify, fail-fast container precondition. cpp is the first (and only) language implemented.

**Files:**
- Create: `src/vergil_tooling/data/skeletons/cpp/conanfile.txt`
- Create: `src/vergil_tooling/data/skeletons/cpp/CMakeLists.txt.tmpl`
- Create: `src/vergil_tooling/data/skeletons/cpp/src/{name}.hpp.tmpl`, `{name}.cpp.tmpl`
- Create: `src/vergil_tooling/data/skeletons/cpp/tests/{name}_test.cpp.tmpl`
- Create: `src/vergil_tooling/lib/lang_scaffold.py`
- Modify: `src/vergil_tooling/lib/languages.py` — add a per-language **lock command** + accessor.
- Modify: `src/vergil_tooling/lib/repo_init.py` — call `scaffold_language(ctx)` after `step_scaffold_config_files`.
- Modify: `src/vergil_tooling/bin/vrg_github_repo_init.py` — fail-fast container precondition for a lock-resolving language.
- Test: `tests/vergil_tooling/test_lang_scaffold.py`, additions to `tests/vergil_tooling/test_languages.py`.

**Interfaces:**
- Produces:
  - `languages.language_lock_command(lang: str) -> list[str] | None` — cpp → `["conan", "lock", "create", ".", "-s", "build_type=Debug"]`; unknown/lockless → `None`.
  - `lang_scaffold.render_skeleton(lang: str, project: str) -> dict[str, str]` — maps repo-relative path → rendered file content (name-substituted). Empty dict for a language with no skeleton.
  - `lang_scaffold.scaffold_language(ctx: RepoInitContext) -> None` — writes missing skeleton files (never clobbers existing), runs the lock command + one `vrg-validate` in the container, or raises if no runtime.
  - `lang_scaffold.sanitize_project_name(repo_name: str) -> str` — `mq-protocol-gateway` → `mq_protocol_gateway`.

### 2a — lock command in the registry

- [ ] **Step 1: Failing test**

```python
def test_cpp_lock_command() -> None:
    assert languages.language_lock_command("cpp") == ["conan", "lock", "create", ".", "-s", "build_type=Debug"]

def test_lockless_language_has_no_lock_command() -> None:
    assert languages.language_lock_command("go") is None
```

- [ ] **Step 2: Run, verify FAIL.**
- [ ] **Step 3: Implement** — add a `lock` field to the cpp registry entry (the `_REGISTRY` value dataclass) and `language_lock_command(lang)` returning it or `None`.
- [ ] **Step 4: Run, verify PASS.**

### 2b — name sanitizer

- [ ] **Step 5: Failing test**

```python
def test_sanitize_project_name() -> None:
    assert lang_scaffold.sanitize_project_name("mq-protocol-gateway") == "mq_protocol_gateway"
    assert lang_scaffold.sanitize_project_name("Foo.Bar") == "foo_bar"
```

- [ ] **Step 6–8:** run→FAIL; implement (`re.sub(r"[^a-z0-9]+","_", name.lower()).strip("_")`); run→PASS.

### 2c — skeleton templates + render

Author the templates (content below is the **clean, #2912-based** skeleton — no prefix-path hack, no gitignore blocks needed).

`skeletons/cpp/conanfile.txt`:
```
[requires]
gtest/1.15.0

[options]
gtest/*:build_gmock=False

[generators]
CMakeDeps
CMakeToolchain
```

`skeletons/cpp/CMakeLists.txt.tmpl` (key point: no `CMAKE_PREFIX_PATH` hack — the pipeline passes `-DCMAKE_TOOLCHAIN_FILE`):
```cmake
cmake_minimum_required(VERSION 3.20)
project({project} CXX)

set(VERGIL_CPP_STD "" CACHE STRING "")
set(VERGIL_CPP_STDLIB "" CACHE STRING "")
option(VERGIL_CPP_COVERAGE "" OFF)
set(VERGIL_CPP_SANITIZE "" CACHE STRING "")

if(VERGIL_CPP_STD)
    string(REGEX REPLACE "^c\\+\\+" "" _std "${VERGIL_CPP_STD}")
    set(CMAKE_CXX_STANDARD "${_std}")
else()
    set(CMAKE_CXX_STANDARD 20)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_library(vergil_cpp_options INTERFACE)
if(VERGIL_CPP_STDLIB AND CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(vergil_cpp_options INTERFACE "-stdlib=${VERGIL_CPP_STDLIB}")
    target_link_options(vergil_cpp_options INTERFACE "-stdlib=${VERGIL_CPP_STDLIB}")
endif()
if(VERGIL_CPP_COVERAGE)
    target_compile_options(vergil_cpp_options INTERFACE --coverage)
    target_link_options(vergil_cpp_options INTERFACE --coverage)
endif()
if(VERGIL_CPP_SANITIZE)
    target_compile_options(vergil_cpp_options INTERFACE "-fsanitize=${VERGIL_CPP_SANITIZE}" -fno-omit-frame-pointer)
    target_link_options(vergil_cpp_options INTERFACE "-fsanitize=${VERGIL_CPP_SANITIZE}")
endif()

find_package(GTest REQUIRED)

add_library({project} src/{name}.cpp)
target_include_directories({project} PUBLIC src)
target_link_libraries({project} PUBLIC vergil_cpp_options)

enable_testing()
add_executable({project}_tests tests/{name}_test.cpp)
target_link_libraries({project}_tests PRIVATE {project} GTest::gtest GTest::gtest_main)
include(GoogleTest)
gtest_discover_tests({project}_tests)
```

`src/{name}.hpp.tmpl`, `src/{name}.cpp.tmpl`, `tests/{name}_test.cpp.tmpl` — the placeholder unit (`{namespace}::toolchain_ready() → true`) + one `TEST` asserting it, formatted to the packaged `.clang-format` (single space before trailing comments). `{name}` = sanitized project name; `{namespace}` = same.

- [ ] **Step 9: Failing test — render substitutes the name and covers the file set**

```python
def test_render_skeleton_cpp() -> None:
    files = lang_scaffold.render_skeleton("cpp", "mq_protocol_gateway")
    assert set(files) == {
        "conanfile.txt", "CMakeLists.txt",
        "src/mq_protocol_gateway.hpp", "src/mq_protocol_gateway.cpp",
        "tests/mq_protocol_gateway_test.cpp",
    }
    assert "project(mq_protocol_gateway CXX)" in files["CMakeLists.txt"]
    assert "CMAKE_PREFIX_PATH" not in files["CMakeLists.txt"]  # clean, #2912-based

def test_render_skeleton_unknown_language_empty() -> None:
    assert lang_scaffold.render_skeleton("go", "x") == {}
```

- [ ] **Step 10–12:** run→FAIL; implement `render_skeleton` (load `data/skeletons/<lang>/`, substitute `{project}`/`{name}`/`{namespace}`, drop the `.tmpl` suffix); run→PASS.

### 2d — write skeleton (idempotent, greenfield)

- [ ] **Step 13: Failing tests**

```python
def test_write_stamps_missing_only(tmp_path) -> None:
    (tmp_path / "conanfile.txt").write_text("CUSTOM")
    lang_scaffold._write_skeleton(tmp_path, {"conanfile.txt": "TEMPLATE", "CMakeLists.txt": "X"})
    assert (tmp_path / "conanfile.txt").read_text() == "CUSTOM"      # never clobbered
    assert (tmp_path / "CMakeLists.txt").read_text() == "X"          # missing → stamped

def test_write_force_restamps(tmp_path) -> None:
    (tmp_path / "conanfile.txt").write_text("CUSTOM")
    lang_scaffold._write_skeleton(tmp_path, {"conanfile.txt": "TEMPLATE"}, force=True)
    assert (tmp_path / "conanfile.txt").read_text() == "TEMPLATE"
```

- [ ] **Step 14–16:** run→FAIL; implement `_write_skeleton(root, files, *, force=False)` (create parent dirs; skip existing unless `force`); run→PASS.

### 2e — container precondition + orchestration

- [ ] **Step 17: Failing test — no runtime → refuse before writing**

```python
def test_scaffold_refuses_without_container(tmp_path, monkeypatch) -> None:
    ctx = _fake_ctx(tmp_path, language="cpp")
    monkeypatch.setattr(lang_scaffold, "container_runtime_available", lambda: False)
    with pytest.raises(ScaffoldError, match="requires a container runtime"):
        lang_scaffold.scaffold_language(ctx)
    assert not (tmp_path / "CMakeLists.txt").exists()  # nothing written
```

- [ ] **Step 18: Failing test — happy path renders, resolves, verifies (resolve/verify mocked)**

```python
def test_scaffold_cpp_happy_path(tmp_path, monkeypatch) -> None:
    ctx = _fake_ctx(tmp_path, language="cpp", repo_name="mq-protocol-gateway")
    monkeypatch.setattr(lang_scaffold, "container_runtime_available", lambda: True)
    calls = []
    monkeypatch.setattr(lang_scaffold, "_run_in_container", lambda root, cmd: calls.append(cmd) or 0)
    lang_scaffold.scaffold_language(ctx)
    assert (tmp_path / "CMakeLists.txt").exists()
    assert (tmp_path / "conan.lock") is not None  # produced by the mocked resolve; assert resolve+verify invoked
    assert ["conan", "lock", "create", ".", "-s", "build_type=Debug"] in calls
    assert ["vrg-validate"] in calls
```

- [ ] **Step 19–21:** run→FAIL; implement `scaffold_language(ctx)`:
  1. `lang = ctx.primary_language`; if `language_lock_command(lang) is None` **and** `render_skeleton(lang, …)` is empty → return (nothing to do).
  2. If the language resolves locks and `not container_runtime_available()` → `raise ScaffoldError("cpp init requires a container runtime …")` **before writing anything**.
  3. `render_skeleton` → `_write_skeleton`.
  4. `_run_in_container(root, language_lock_command(lang))` → produces `conan.lock`; then `_run_in_container(root, ["vrg-validate"])`; any non-zero → `ScaffoldError`.
  Add `container_runtime_available()` (reuse `container.detect_runtime`) and `_run_in_container(root, cmd)` (shell to `vrg-container-run -- <cmd>`).
  run→PASS.

### 2f — repo-init wiring

- [ ] **Step 22: Failing test — repo-init calls scaffold after config**

```python
def test_repo_init_invokes_scaffold(monkeypatch, tmp_path) -> None:
    called = {}
    monkeypatch.setattr(repo_init.lang_scaffold, "scaffold_language", lambda ctx: called.setdefault("ctx", ctx))
    repo_init.step_scaffold_config_files(_ctx(tmp_path, "cpp"))
    repo_init.step_scaffold_language(_ctx(tmp_path, "cpp"))
    assert "ctx" in called
```

- [ ] **Step 23–25:** run→FAIL; add `step_scaffold_language(ctx)` calling `lang_scaffold.scaffold_language(ctx)`, wired into the repo-init step sequence right after `step_scaffold_config_files`; run→PASS.

### 2g — integration test (container, gated)

- [ ] **Step 26: Add a gated integration test** (`tests/vergil_tooling/test_lang_scaffold_integration.py`, marked like the existing `test_vrg_container_*` gate): scaffold a cpp repo into `tmp_path`, run the **real** resolve + `vrg-validate`, assert exit 0 and a committed `conan.lock` present with no Conan files at the source root.

- [ ] **Step 27: `vrg-container-run -- vrg-validate` green (100% coverage); commit.**

---

## Task 3: Retrofit `mq-protocol-gateway` onto the clean path (manual PR, in that repo)

Not a scaffold run — a deliberate hand-edit, safe because the repo is still the placeholder skeleton.

**Files (in `logical-minds-foundry/mq-protocol-gateway`):**
- Modify: `CMakeLists.txt` — remove the `list(APPEND CMAKE_PREFIX_PATH …)` / `CMAKE_MODULE_PATH` block (the toolchain file from Task 1 now handles `find_package`).
- Modify: `.gitignore` — remove the Conan generic + CMakeDeps blocks (Task 1 keeps output under the already-ignored `build/`).

- [ ] **Step 1:** In the issue-3 worktree, delete the prefix-path/module-path lines from `CMakeLists.txt`.
- [ ] **Step 2:** Delete the `# C/C++ Conan …` blocks from `.gitignore` (keep `build/`, `conan.lock` committed).
- [ ] **Step 3:** `vrg-container-run -- vrg-validate` → green, and confirm no Conan files at the source root after a run (only `build/`).
- [ ] **Step 4:** Commit; `report-ready`.

```bash
vrg-commit --type refactor --scope build --message "drop Conan prefix-path hack + gitignore blocks; use toolchain file (#<issue>)"
```

---

## Task 4: Revert the baseline `.gitignore` Conan blocks (`#2878`/`#2908`)

With Task 1 landed, nothing writes Conan output to the source root, so the baseline blocks are dead.

**Files:**
- Modify: `src/vergil_tooling/data/gitignore.baseline` — remove the two `# C/C++ Conan …` blocks (keep `build/`, and `*.o`/`*.a`/etc.).
- Modify: `.gitignore` (flagship) — mirror the removal (drift guard `test_baseline_is_subset_of_flagship_gitignore`).
- Test: `tests/vergil_tooling/test_repo_init.py`, `test_repo_config.py`.

- [ ] **Step 1: Failing test — baseline no longer lists the Conan CMakeDeps patterns**

```python
def test_baseline_drops_conan_generator_blocks() -> None:
    patterns = repo_config._gitignore_patterns(repo_config._load_gitignore_baseline())
    for gone in ("conan_toolchain.cmake", "Find*.cmake", "module-*.cmake", "CMakePresets.json"):
        assert gone not in patterns
    assert "build/" in patterns  # still ignored
```

- [ ] **Step 2:** run→FAIL.
- [ ] **Step 3:** Remove both Conan blocks from `gitignore.baseline` **and** the flagship `.gitignore`.
- [ ] **Step 4:** run→PASS; run `test_baseline_is_subset_of_flagship_gitignore` → PASS.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate` green; commit.

*(Note: consuming cpp repos reconcile to the smaller baseline automatically — a smaller baseline is still a subset of their `.gitignore`. mq-protocol-gateway's own blocks are removed in Task 3.)*

---

## Task 5: Remove the cpp `_WARMUP_REQUIRES` entry (`#2881`)

Born-green means no half-bootstrapped cpp state, so the cpp warmup-skip is dead. Remove the cpp entry only; the mechanism stays for not-yet-scaffolded languages.

**Files:**
- Modify: `src/vergil_tooling/lib/container_cache.py` — delete the `"cpp": [...]` line from `_WARMUP_REQUIRES`.
- Test: `tests/vergil_tooling/test_container_cache.py`

- [ ] **Step 1: Update the warmup tests** — a bootstrapped cpp repo still warms; there is no longer a cpp *skip* path. Replace `missing_warmup_files("cpp", …)` skip assertions with: cpp has no entry so `missing_warmup_files("cpp", repo) == []` for any tree (the mechanism returns `[]` for unlisted languages).

```python
def test_cpp_has_no_warmup_skip_entry(tmp_path) -> None:
    # cpp is born-green (epic #342); no skip machinery for it
    assert missing_warmup_files("cpp", tmp_path) == []
```

- [ ] **Step 2:** run→FAIL (cpp entry still present → returns unmet groups).
- [ ] **Step 3:** Delete the `"cpp"` entry from `_WARMUP_REQUIRES`; remove/adjust the now-obsolete cpp warmup-skip tests (the `_bootstrap_cpp`-based skip tests) — keep the tests that assert warmup *runs* when files are present.
- [ ] **Step 4:** run→PASS.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate` green; commit.

---

## Self-review

- **Spec coverage:** §3.1 flow → Task 2 (2e/2f); §3.2 components → Task 2 (2a/2c/2e/2f); §3.3 cpp skeleton + #2912 → Task 1 + Task 2c; §3.4 greenfield/retrofit → Task 2d + Task 3; §4 testing → tests in every task + 2g integration; §5.1 (`_WARMUP_REQUIRES`) → Task 5; §5.2 (gitignore/prefix-path) → Tasks 1/3/4; §6 sequencing → task order 1→5. Covered.
- **Container precondition / full-validate verify** (pushback [3]/[4]) → Task 2e steps 17–21.
- **Ordering guard:** Tasks 4–5 must land after 1–3 (dependency noted in each). Task 3 is cross-repo (mq-protocol-gateway), filed there per the placement law.
- **Type consistency:** `render_skeleton`/`_write_skeleton`/`scaffold_language`/`language_lock_command`/`sanitize_project_name` used consistently across 2a–2f.
- **Open empirical check folded in:** Task 1 Step 9 verifies the toolchain-file path end-to-end before the template (Task 2c) depends on it.
