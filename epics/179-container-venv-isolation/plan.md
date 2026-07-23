# Container venv isolation — Implementation Plan

> **For agentic workers:** Each implementation task below (Task 1, Task 2) is
> filed as a GitHub task under epic vergil-project/.github#179 and implemented
> end-to-end via `vergil:issue-implement` (USER agent). Operational tasks (Task 3
> validation, Task 4 deployment) are run via `vergil:issue-validate` /
> `vergil:issue-deploy`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop `vrg-container-run` from corrupting the host's `.venv`, by masking
the workspace venv with a fresh anonymous volume inside the container, and retire
the `.venv-host` dual-venv workaround.

**Architecture:** A single structural barrier — an anonymous Docker/nerdctl volume
mounted over `/workspace/.venv` for Python repos — makes the default `.venv` path
safe inside the container, so no in-container `uv` code path can reach the host
venv. `vrg-validate`'s PATH-add is corrected to tolerate the now-empty-at-startup
venv dir. With the host venv structurally protected, the `.venv-host` split is
removed and the host dev tree uses plain `.venv`.

**Tech Stack:** Python 3.14, `uv`, Docker/nerdctl, pytest (100% branch coverage
gate).

## Global Constraints

- **Validation is one command:** `vrg-container-run -- vrg-validate` (expands to
  `uv run vrg-validate` here via the `[validation]` override). Per-test fast loop
  during development: `uv run pytest <path> -v` against the dev-tree venv.
- **Git/GitHub via wrappers only:** `vrg-git`, `vrg-gh`, `vrg-commit`. Raw
  `git`/`gh` are denied.
- **Coverage gate is 100% branch** (`--cov-fail-under=100`). Every new branch
  needs a test.
- **Portability:** the container flag must work under both `docker` and `nerdctl`
  (the runtime is selected at call time).
- **Sequencing hazard (read before Task 2):** Task 2 switches vergil-tooling's own
  host dev tree from `.venv-host` to `.venv`. That is only safe once the **fixed**
  `vrg-container-run` (Task 1) is installed on the host (Task 4 deployment) —
  otherwise an *old* host tool, run against a `.venv`-based dev tree, would corrupt
  it exactly as before. Order of adoption: Task 1 merges → release → host reinstall
  (Task 4) → then rely on `.venv` in the dev tree. Task 2 may merge earlier, but do
  not `rm -rf .venv-host && uv sync` into `.venv` on a host still running the old
  tool.

---

### Task 1 (T2): Core isolation fix — anonymous venv mask + PATH-add

Files:
- Modify: `src/vergil_tooling/lib/container.py` — `build_container_args`
  (after the `/workspace` mount block, ~line 152-159)
- Modify: `src/vergil_tooling/bin/vrg_validate.py:154-156` — the PATH-add
- Test: `tests/vergil_tooling/test_container.py`
- Test: `tests/vergil_tooling/test_vrg_validate.py`

Interfaces:
- Consumes: `detect_language(repo_root: Path) -> str` (already in `container.py`).
- Produces: no new public signatures; `build_container_args` gains an anonymous
  `-v /workspace/.venv` for Python repos; `vrg_validate`'s PATH-add is
  unconditional.

- [ ] **Step 1: Write the failing test — Python repo gets the mask, non-Python does not**

In `tests/vergil_tooling/test_container.py`, add (reuse the existing
`patch.dict("os.environ", {}, clear=True)` harness):

```python
def test_build_container_args_masks_venv_for_python(tmp_path: Path) -> None:
    # A Python repo: the host .venv must be masked by a fresh anonymous volume
    # so in-container `uv sync` cannot rewrite the bind-mounted host venv (#2473).
    (tmp_path / "pyproject.toml").write_text("[project]\nname = 'x'\n")
    with patch.dict("os.environ", {}, clear=True):
        args = build_container_args(tmp_path, "img:1", ["cmd"], runtime="docker")
    assert "/workspace/.venv" in args
    idx = args.index("/workspace/.venv")
    assert args[idx - 1] == "-v"  # anonymous volume: no host source


def test_build_container_args_no_venv_mask_for_non_python(tmp_path: Path) -> None:
    # Non-Python repos have no .venv corruption vector; no stray mask.
    (tmp_path / "go.mod").write_text("module x\n")
    with patch.dict("os.environ", {}, clear=True):
        args = build_container_args(tmp_path, "img:1", ["cmd"], runtime="docker")
    assert "/workspace/.venv" not in args
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `uv run pytest tests/vergil_tooling/test_container.py -k "masks_venv or no_venv_mask" -v`
Expected: FAIL — `/workspace/.venv` not in args.

- [ ] **Step 3: Implement the mask in `container.py`**

In `build_container_args`, immediately after the `/workspace` bind-mount block
(the `container_args.extend([... "-v", f"{repo_root}:/workspace", "-w",
"/workspace"])`), add:

```python
    # Isolate the Python venv. `/workspace/.venv` is the bind-mounted host venv;
    # in-container `uv sync` would rewrite it with container-only interpreter
    # symlinks and shebangs, corrupting host-side `uv run` / console scripts
    # (#2473). Mount a fresh, empty anonymous volume over it so the container
    # builds a throwaway venv at the default path and never touches the host's.
    # Gated to Python repos — other ecosystems have no `.venv`.
    if detect_language(repo_root) == "python":
        container_args.extend(["-v", "/workspace/.venv"])
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `uv run pytest tests/vergil_tooling/test_container.py -k "masks_venv or no_venv_mask" -v`
Expected: PASS.

- [ ] **Step 5: Write the failing test — validate's PATH-add is unconditional**

In `tests/vergil_tooling/test_vrg_validate.py`, add a test that the venv `bin` is
prepended to PATH **even when the directory does not yet exist** (the masking
volume is empty at container startup). Model it on the existing validate tests
(check how they invoke `main`/the PATH logic; if the PATH-add is only reachable
via `main`, test through `main` with `_in_dev_container` monkeypatched True and a
`cwd` with no `.venv`). Minimal shape:

```python
def test_path_add_is_unconditional_when_venv_missing(
    tmp_path: Path, monkeypatch: pytest.MonkeyPatch
) -> None:
    # A fresh masking volume means .venv/bin does not exist at startup; the
    # PATH-add must still prepend it so post-install console scripts resolve.
    monkeypatch.chdir(tmp_path)                      # no .venv here
    monkeypatch.setattr(vrg_validate, "_in_dev_container", lambda: True)
    monkeypatch.setattr(vrg_validate.config, "read_config",
                        lambda _r: (_ for _ in ()).throw(FileNotFoundError))
    monkeypatch.setenv("PATH", "/usr/bin")
    vrg_validate.main([])
    assert str(tmp_path / ".venv" / "bin") in os.environ["PATH"].split(os.pathsep)
```

Adjust the monkeypatches to match the module's actual seams (e.g. patch
`git.repo_root` if `main` calls it before the stages). The single assertion that
matters: `str(cwd/.venv/bin)` is on `PATH` despite the dir not existing.

- [ ] **Step 6: Run the test to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_vrg_validate.py -k unconditional -v`
Expected: FAIL — the `is_dir()` guard skips the prepend.

- [ ] **Step 7: Drop the `is_dir()` guard in `vrg_validate.py`**

Change lines 154-156 from:

```python
    venv_bin = Path.cwd() / ".venv" / "bin"
    if venv_bin.is_dir() and str(venv_bin) not in os.environ.get("PATH", "").split(os.pathsep):
        os.environ["PATH"] = f"{venv_bin}{os.pathsep}{os.environ.get('PATH', '')}"
```

to:

```python
    # Prepend the project venv's bin unconditionally. Under the container's
    # anonymous .venv mask the directory is empty at startup and is populated by
    # the install stage; PATH is resolved at exec time, so a not-yet-existent dir
    # is harmless here and required for later bare `ruff`/`pytest` to resolve
    # (#2473).
    venv_bin = Path.cwd() / ".venv" / "bin"
    if str(venv_bin) not in os.environ.get("PATH", "").split(os.pathsep):
        os.environ["PATH"] = f"{venv_bin}{os.pathsep}{os.environ.get('PATH', '')}"
```

- [ ] **Step 8: Run the validate tests to verify they pass**

Run: `uv run pytest tests/vergil_tooling/test_vrg_validate.py -v`
Expected: PASS (including the new test; existing tests unaffected).

- [ ] **Step 9: Confirm the `UV_LINK_MODE=copy` pin is untouched**

Run: `uv run pytest tests/vergil_tooling/test_container.py -k uv_link_mode -v`
Expected: PASS — the copy default and operator override are retained (the mask is
on a different filesystem from the uv cache, so the cross-fs copy stays correct).

- [ ] **Step 10: Full validation + commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type fix --scope container \
  --message "isolate container venv with an anonymous mask over .venv" \
  --body "Mount a fresh anonymous volume over /workspace/.venv for Python repos so in-container uv sync builds a throwaway venv instead of corrupting the bind-mounted host venv (#2473). Drop the is_dir() guard on vrg-validate's PATH-add so the venv bin resolves even though the masking volume is empty at container startup. Keep UV_LINK_MODE=copy (mask is cross-fs from the uv cache)."
```

---

### Task 2 (T3): Excise the `.venv-host` dual-venv model

**Prerequisite:** Task 1 merged. **Read the Sequencing hazard in Global
Constraints before adopting `.venv` on any host still running the old tool.**

Files:
- Modify: `src/vergil_tooling/lib/repo_init.py:391` — gitignore template
- Modify: `tests/vergil_tooling/test_repo_init.py:347` — gitignore test
- Modify: `src/vergil_tooling/bin/validate_common.py:85` — docstring comment
- Modify: `tests/vergil_tooling/test_validate_common.py:422` — skip-dirs test
- Modify: `.gitignore` (vergil-tooling's own) — add `.venv/` and `.superpowers/`
- Modify: `CLAUDE.md` — "Environment Setup" section

- [ ] **Step 1: Write the failing test — template ignores `.venv/`, not `.venv-host/`**

Update `tests/vergil_tooling/test_repo_init.py` `test_contains_baseline_patterns`:

```python
    def test_contains_baseline_patterns(self) -> None:
        content = render_gitignore()
        assert ".DS_Store" in content
        assert ".worktrees/" in content
        assert ".venv/" in content            # the host venv every consumer must ignore
        assert ".venv-host/" not in content   # dual-venv model retired (#2473)
```

- [ ] **Step 2: Run it to verify it fails**

Run: `uv run pytest tests/vergil_tooling/test_repo_init.py -k baseline_patterns -v`
Expected: FAIL — template still emits `.venv-host/` and no `.venv/`.

- [ ] **Step 3: Fix the gitignore template**

In `src/vergil_tooling/lib/repo_init.py`, replace the line `".venv-host/\n"` with
`".venv/\n"`. (Replace, not delete: consuming repos must ignore their host `.venv`,
which the baseline previously failed to cover.)

- [ ] **Step 4: Run it to verify it passes**

Run: `uv run pytest tests/vergil_tooling/test_repo_init.py -k baseline_patterns -v`
Expected: PASS.

- [ ] **Step 5: Update the skip-dirs test (drop `.venv-host`)**

In `tests/vergil_tooling/test_validate_common.py`,
`test_find_yaml_files_skips_worktrees_and_venv`, change the skip tuple from
`(".worktrees", ".venv", ".venv-host", "node_modules")` to
`(".worktrees", ".venv", "node_modules")`.

- [ ] **Step 6: Run it to verify it still passes**

Run: `uv run pytest tests/vergil_tooling/test_validate_common.py -k skips_worktrees -v`
Expected: PASS (discovery is by-construction; the tuple only enumerates decoy dirs).

- [ ] **Step 7: Update the `validate_common.py` docstring**

In `src/vergil_tooling/bin/validate_common.py`, change the docstring
`` Vendored paths (`.worktrees`, `.venv`, `.venv-host`, `node_modules`) `` to
drop `` `.venv-host` `` → `` Vendored paths (`.worktrees`, `.venv`,
`node_modules`) ``.

- [ ] **Step 8: Add `.venv/` and `.superpowers/` to vergil-tooling's own `.gitignore`**

vergil-tooling's `.gitignore` currently ignores `.worktrees/` but **not** `.venv/`
(the host venv this epic protects) or `.superpowers/` (throwaway superpowers-skill
scratch that has been polluting branches). Add both under the existing "# Vergil"
(or equivalent) stanza. This is a plain-text hygiene edit — no test. Verify with:

```bash
uv run git check-ignore .venv .superpowers   # both should print, i.e. ignored
```

(Use `uv run git` only for this read-only local check if `vrg-git` lacks
`check-ignore`; the edit itself is a normal file change.)

- [ ] **Step 9: Update `CLAUDE.md` "Environment Setup"**

Replace the dual-venv explanation (the `.venv` vs `.venv-host` bullets and the
`UV_PROJECT_ENVIRONMENT=.venv-host uv sync --group dev` bootstrap) with the plain
model: the host dev tree uses `.venv`:

```bash
# Dev-tree override (vergil-tooling development only)
uv sync --group dev
export PATH="$(pwd)/.venv/bin:$PATH"
```

Add a one-line note that the container never touches the host `.venv` (it is
masked by an anonymous volume, #2473), which is why a single `.venv` is now safe.

- [ ] **Step 10: Full validation + commit**

Run: `vrg-container-run -- vrg-validate`
Then:
```bash
vrg-commit --type refactor --scope venv \
  --message "retire the .venv-host dual-venv model" \
  --body "With the container's .venv now masked by an anonymous volume (#2473), the host dev tree is safe using plain .venv, so the .venv-host split is removed. Fix the repo_init gitignore template to ignore .venv/ (previously it ignored only .venv-host/, leaving consumers' host venv untracked), and add .venv/ + .superpowers/ to vergil-tooling's own .gitignore. Drop .venv-host from the validate_common docstring and the skip-dirs decoy list, and collapse CLAUDE.md Environment Setup to a single-.venv model."
```

> Note: `docs/specs/host-level-tool.md`'s "Why `.venv-host`" section and the
> versioned site docs are **not** edited here — they are handled by the
> documentation-review gate (#2477), which sweeps all human-facing docs. Historical
> plan/spec/release records that merely date past decisions are left as-is.

---

### Task 3 (T4): Validation — host `.venv` survives a container run

Operational task (`--kind validation`, blocked-by Task 1 & Task 2). Not
PR-workable; run via `vergil:issue-validate`, close only on `Outcome: SUCCESS`.

**Precondition self-check:** the fixed `vrg-container-run` (Task 1) is installed on
the host under test (`vrg-container-run --version` or equivalent shows the release
that carries the mask). If not met, comment "blocked: preconditions not met" and
stop.

**Procedure (on a Python repo host, e.g. a lab box or vergil-tooling itself):**
1. Record the host venv fingerprint:
   `readlink .venv/bin/python` and `head -1 .venv/bin/<some-console-script>`.
2. Run the full gate: `vrg-container-run -- vrg-validate`.
3. Re-read the same two values.

**Acceptance (SUCCESS):**
- The interpreter symlink target and the console-script shebang are **byte-for-byte
  unchanged**.
- Host `uv run <entrypoint>` and host console scripts still execute.
- No anonymous volume is leaked: `docker volume ls` (and `nerdctl volume ls`)
  count is unchanged after the run — i.e. `--rm` reclaimed it. If a runtime leaks
  the volume, record FAILURE and note the tmpfs fallback (§9 of the spec) for a
  follow-up.

**FAILURE:** any of the above not met; the task stays open and gates the epic.

---

### Task 4 (T5): Deployment — release + host reinstall

Operational task (`--kind deployment`, blocked-by Task 1, Task 2, Task 4). Not
PR-workable; run via `vergil:issue-deploy`, close only on `Outcome: SUCCESS`.

**Human-gated precondition (attested, never performed by the agent):** a
vergil-tooling release carrying Task 1 + Task 2 has been cut (bump/tag/publish).
The agent records the released version; it does not cut the release.

**Agent-safe deploy steps:**
1. Upgrade the host tool on each affected host:
   `uv tool install --python 3.14 'vergil-tooling @ git+https://github.com/vergil-project/vergil-tooling@<tag>'`.
2. Confirm `vrg-container-run` resolves to the new version.
3. Where a host's `.venv` was corrupted pre-fix, run the one-time reset:
   `rm -rf .venv && uv sync`.

**Acceptance (SUCCESS):** the fixed tool is installed on the target host(s) and a
`vrg-container-run -- vrg-validate` leaves the host `.venv` intact (ties back to
Task 3). Record the deployed version and hosts as a comment.

---

## Task filing (epic-create step 9)

After this plan's docs PR merges (closing the documentation task #180), file the
implementation and operational tasks under the epic so their linkage and
`Blocked-by` resolve:

```bash
# Impl
vrg-issue-create --epic vergil-project/.github#179 --repo vergil-project/vergil-tooling \
  --title "Isolate container venv with an anonymous mask over .venv"        # Task 1 (T2)
vrg-issue-create --epic vergil-project/.github#179 --repo vergil-project/vergil-tooling \
  --title "Retire the .venv-host dual-venv model"                            # Task 2 (T3)
# Operational (fill <T2>,<T3> with the numbers from above)
vrg-issue-create --epic vergil-project/.github#179 --repo vergil-project/vergil-tooling \
  --kind validation --title "Validate host .venv survives a container run" \
  --blocked-by vergil-project/vergil-tooling#<T2> --blocked-by vergil-project/vergil-tooling#<T3>
vrg-issue-create --epic vergil-project/.github#179 --repo vergil-project/vergil-tooling \
  --kind deployment --title "Release + host reinstall of the venv-isolation fix" \
  --blocked-by vergil-project/vergil-tooling#<T2> --blocked-by vergil-project/vergil-tooling#<T3> \
  --blocked-by vergil-project/vergil-tooling#<T4>
```

## Self-review

- **Spec coverage:** §4.1 mask → Task 1 Steps 1-4; §4.2 PATH-add → Task 1 Steps
  5-8; §4.3 copy pin retained → Task 1 Step 9; §4.4 excise `.venv-host` → Task 2;
  §5 migration → Task 4 Step 3; §7.2 validation/deployment → Tasks 3-4; §9
  volume-cleanup check → Task 3 acceptance. Covered.
- **Extra fix surfaced:** the gitignore template ignored `.venv-host/` but not
  `.venv/`; Task 2 Step 3 corrects that (replace, not delete). vergil-tooling's own
  `.gitignore` also lacked `.venv/` and `.superpowers/`; Task 2 Step 8 adds them.
  The systemic `.gitignore`-sprawl reevaluation is spun out as a closing
  brainstorm task (spec §7.4), not done here.
- **Placeholders:** none — each step carries concrete code/commands.
- **Type consistency:** `detect_language(repo_root) -> str` is the only consumed
  signature; used exactly as defined in `container.py`.
