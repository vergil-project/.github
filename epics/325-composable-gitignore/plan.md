# Composable `.gitignore` + Fleet Sync — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: implement this plan task-by-task via `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Each GitHub task below is one PR. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Replace the monolithic baseline `.gitignore` with a composed `base + <primary_language>` managed block, one code path shared by audit / sync / repo-init, plus a fleet driver that propagates changes across repos from one command.

**Architecture:** A pure-logic library (`lib/gitignore.py`) owns composition, parse, render, check, and sync. A thin applicator CLI (`vrg-gitignore-sync`) wraps it for one repo. `_check_gitignore` and `repo_init.render_gitignore` both call into the library. A generic fleet driver runs the per-repo git/PR work-chain, invoking the applicator. Rollout is release-gated and sweeps with the released tool.

**Tech Stack:** Python 3.12–3.14, `importlib.resources` (packaged data), pytest (100% branch coverage), the existing `vrg-git`/`vrg-gh`/`vrg-pr-workflow` wrappers.

**Spec:** `epics/325-composable-gitignore/spec.md` (travels with this plan; executors read both).

## Global Constraints

- **100% branch coverage** on the single-interpreter local gate; PR-CI re-runs the matrix (3.12/3.13/3.14). Verbatim from repo policy.
- **Portability:** all logic must run on macOS and Linux; no shell-only assumptions in Python.
- **Packaged data:** fragments load via `importlib.resources.files("vergil_tooling.data")` — never a filesystem-relative path.
- **No repo-specific logic** in the library; repo identity enters only via `vergil.toml`.
- **Agents never run `vrg-submit-pr`/merge.** Tasks hand off via `vrg-pr-workflow report-ready`.
- **Wrappers only:** `vrg-git`, `vrg-gh` — never raw `git`/`gh`.
- **Lossless-split invariant:** `base ∪ all-fragments` must equal the 55 legacy monolith patterns exactly, enforced by test, until the monolith is deleted.

---

### Task 1: Fragment data files + `compose` / `managed_vocabulary`

**Files:**

- Create: `src/vergil_tooling/data/gitignore/base`, `.../python`, `.../cpp`, `.../go`, `.../ruby`, `.../rust`, `.../java`, `.../typescript`
- Create: `src/vergil_tooling/lib/gitignore.py`
- Test: `tests/vergil_tooling/test_gitignore.py`
- Reference: `src/vergil_tooling/data/gitignore.baseline` (the 55-line source, unchanged this task)

**Interfaces — Produces:**

- `load_base() -> list[str]`, `load_fragment(lang: str) -> list[str]` (empty list for unknown/empty lang)
- `FRAGMENT_LANGS: tuple[str, ...]` (`python,cpp,go,ruby,rust,java,typescript`)
- `compose(lang: str | None) -> list[str]` — `load_base()` + `load_fragment(lang)`, de-duplicated, base-first, order-stable; base-only when `lang` is None/empty/unknown
- `managed_vocabulary() -> set[str]` — `set(load_base()) | union(load_fragment(l) for l in FRAGMENT_LANGS)`

**Content split** (exact assignment — copy from spec §5): base=24, python=10, cpp=16, go=2, ruby=2, typescript=1, rust/java empty.

- [ ] **Step 1: Write the split fragment files.** Populate each `data/gitignore/<name>` with the spec §5 lines (one pattern per line, no comments in fragments — comments live only in the composed fence header). `rust`/`java` are empty files.

- [ ] **Step 2: Write the lossless-split invariant test (failing).**

```python
def test_split_is_lossless_against_legacy_monolith():
    from vergil_tooling.lib import gitignore, repo_config
    legacy = set(repo_config._gitignore_patterns(repo_config._load_gitignore_baseline()))
    covered = set(gitignore.load_base())
    for lang in gitignore.FRAGMENT_LANGS:
        covered |= set(gitignore.load_fragment(lang))
    assert covered == legacy, {"orphaned": legacy - covered, "invented": covered - legacy}
```

- [ ] **Step 3: Run it — fails** (`gitignore` module absent). `pytest tests/vergil_tooling/test_gitignore.py -k lossless -v`

- [ ] **Step 4: Implement `gitignore.py` loaders + `compose` + `managed_vocabulary`.**

```python
from __future__ import annotations
import importlib.resources

FRAGMENT_LANGS = ("python", "cpp", "go", "ruby", "rust", "java", "typescript")

def _read(name: str) -> list[str]:
    data = importlib.resources.files("vergil_tooling.data").joinpath("gitignore", name)
    if not data.is_file():
        return []
    return [ln.rstrip() for ln in data.read_text(encoding="utf-8").splitlines() if ln.strip()]

def load_base() -> list[str]:
    return _read("base")

def load_fragment(lang: str | None) -> list[str]:
    return _read(lang) if lang in FRAGMENT_LANGS else []

def compose(lang: str | None) -> list[str]:
    out, seen = [], set()
    for line in [*load_base(), *load_fragment(lang)]:
        if line not in seen:
            seen.add(line); out.append(line)
    return out

def managed_vocabulary() -> set[str]:
    vocab = set(load_base())
    for lang in FRAGMENT_LANGS:
        vocab |= set(load_fragment(lang))
    return vocab
```

- [ ] **Step 5: Add compose tests** — python composes 34 lines (24+10), cpp composes 40 (24+16), a `None`/`"shell"` lang composes base-only (24), order is base-then-fragment, no dupes.

- [ ] **Step 6: Run all Task-1 tests — pass.** Confirm the invariant test is green (proves the split lost nothing).

- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope gitignore --message "add composable base + language fragment data and compose()"`

---

### Task 2: Managed-block `render` / `parse` / `check`

**Files:**

- Modify: `src/vergil_tooling/lib/gitignore.py`
- Test: `tests/vergil_tooling/test_gitignore.py`

**Interfaces — Consumes:** `compose`, `FRAGMENT_LANGS`. **Produces:**

- `MANAGED_BEGIN_PREFIX = "# >>> vergil-managed:"`, `MANAGED_END = "# <<< vergil-managed <<<"`
- `render_block(lang: str | None) -> str` — begin marker (descriptor `base + <lang>` or `base`), composed lines, end marker; trailing newline
- `parse(text: str) -> tuple[list[str], str | None]` — returns `(repo_local_lines, fence_text_or_None)`; the fence is the exact substring between and including the markers
- `Compliance` dataclass: `compliant: bool`, `reasons: list[str]`
- `check(text: str, lang: str | None) -> Compliance` — fence present, equals `render_block(lang)`, and no `managed_vocabulary()` line stray outside the fence

- [ ] **Step 1: Failing tests** — `render_block("python")` starts with the begin prefix + `base + python` and ends with `MANAGED_END`; `render_block(None)` descriptor is `base`; `parse` round-trips a file with a fence into `(repo_local, fence)`; `parse` of a no-fence file returns `(all_lines, None)`; `check` returns compliant for a freshly-rendered file, non-compliant with a reason for a mangled fence and for a stray `*.pyc` outside the fence.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement** `render_block`, `parse` (scan for begin/end marker lines; malformed = begin without end → treat as no fence + reason), `check`.

- [ ] **Step 4: Run — pass.**

- [ ] **Step 5: Commit.** `--message "add managed-block render/parse/check"`

---

### Task 3: `sync` — bootstrap vs update

**Files:** Modify `lib/gitignore.py`; Test `test_gitignore.py`.

**Interfaces — Consumes:** all of Task 1–2. **Produces:**

- `SyncResult` dataclass: `text: str`, `changed: bool`, `removed: list[str]` (lines dropped, for logging)
- `sync(text: str, lang: str | None) -> SyncResult`

Behavior (spec §9): if `parse` finds **no fence** → *bootstrap*: repo_local = existing lines **not** in `managed_vocabulary()`; new text = `render_block(lang)` + repo_local. If a fence **is** present → *update*: keep parsed repo_local as-is, replace fence with `render_block(lang)`. `changed = new_text != text`.

- [ ] **Step 1: Failing tests:**
  - bootstrap of a python repo whose loose `.gitignore` holds base+python+**cpp** lines + a repo-local `secrets.json` → result fence is `base+python`, `secrets.json` preserved, all cpp lines in `removed`, `changed True`.
  - bootstrap of a `.github` (lang `None`) file holding the full monolith → fence is base-only, all 31 language lines removed.
  - idempotency: `sync(sync(text).text).changed is False`.
  - update: a file already fenced (base+python) with a hand-added stale line inside the fence → fence rewritten to canonical, repo_local untouched.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement `sync`** using `parse` + `managed_vocabulary` + `render_block`.

- [ ] **Step 4: Run — pass** (incl. idempotency).

- [ ] **Step 5: Commit.** `--message "add sync() with bootstrap/update semantics"`

---

### Task 4: `vrg-gitignore-sync` applicator CLI

**Files:**

- Create: `src/vergil_tooling/bin/vrg_gitignore_sync.py`
- Modify: `pyproject.toml` (`[project.scripts]` → `vrg-gitignore-sync = "vergil_tooling.bin.vrg_gitignore_sync:main"`)
- Test: `tests/vergil_tooling/test_vrg_gitignore_sync.py`

**Interfaces — Consumes:** `gitignore.sync`, `gitignore.check`, `config` primary-language resolution (`resolve_language`/`vergil.toml`). **Produces:** `main(argv: list[str] | None = None) -> int`.

CLI: `--check` (exit 0 compliant, 1 + printed reasons otherwise) / `--write` (apply `sync`, write file, print `removed` with reason line `dropped N line(s) matching other-language fragments; this repo is <lang>`), `--repo <path>` (default cwd). `--check` and `--write` are mutually exclusive; default `--check`.

- [ ] **Step 1: Failing tests** — `--check` on a compliant tmp repo exits 0; on a monolith repo exits 1 with a reason; `--write` on a python monolith repo rewrites to base+python and logs removed cpp lines; second `--write` is a no-op ("already in sync"), exit 0.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement `main`** (resolve lang → read `.gitignore` → check/sync → write/report).

- [ ] **Step 4: Run — pass.**

- [ ] **Step 5: Commit.** `--message "add vrg-gitignore-sync applicator CLI"`

---

### Task 5: Audit swap (transitional)

**Files:**

- Modify: `src/vergil_tooling/lib/repo_config.py` (`_check_gitignore`)
- Test: `tests/vergil_tooling/test_repo_config.py`

**Interfaces — Consumes:** `gitignore.check`, `gitignore.managed_vocabulary`, existing `_load_gitignore_baseline`/`_gitignore_patterns`. **Produces:** unchanged `_check_gitignore(repo_root, items)` signature.

Transitional acceptance (spec §10.1): a repo **passes** if it is the legacy monolith superset (today's check) **OR** `gitignore.check(text, lang).compliant`. A module constant `_GITIGNORE_FENCED_ONLY = False` gates the legacy arm; Task 10 flips it to `True` and deletes the legacy arm.

- [ ] **Step 1: Failing tests** — a legacy-superset repo passes; a correctly-fenced repo passes; a repo that is neither fails with a `DiffItem`; base-only fence on a shell repo passes.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement** the OR of the two acceptance paths behind `_GITIGNORE_FENCED_ONLY`.

- [ ] **Step 4: Run — pass.**

- [ ] **Step 5: Commit.** `--message "make _check_gitignore accept fenced form (transitional)"`

---

### Task 6: `repo-init` composes through `gitignore.py`

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py` (`render_gitignore`, repo_init.py:435)
- Test: `tests/vergil_tooling/test_repo_init.py`, `tests/vergil_tooling/test_repo_config.py`

**Interfaces — Consumes:** `gitignore.render_block`. **Produces:** `render_gitignore(lang: str | None) -> str` (signature gains `lang`; callers pass the repo's resolved language).

`render_gitignore` emits `render_block(lang)` (base-only when no fragment).

**Existing monolith-coupled tests to update (do not leave the suite red):**

- `test_repo_init.py::TestRenderGitignore::test_contains_baseline_patterns` and `::test_contains_vergil_workflow_patterns` — re-point at the composed fence (base patterns like `.worktrees/`, `.venv/`, `build/` still present, now inside the managed block).
- `test_repo_init.py::TestRenderGitignore::test_baseline_is_subset_of_flagship_gitignore` — **replace** with the fenced-form drift guard: vergil-tooling's own `.gitignore` managed block equals `gitignore.render_block("python")`.
- `test_repo_init.py::TestRenderGitignore::test_render_gitignore_returns_packaged_baseline` — re-assert against `render_block("python")` (no longer the byte-for-byte monolith).
- `test_repo_config.py::test_baseline_has_required_patterns` — re-express against `gitignore.compose(...)` / `managed_vocabulary()` rather than `_load_gitignore_baseline()`.

- [ ] **Step 1: Failing tests** — `render_gitignore("python")` contains the fence markers + base+python; `render_gitignore(None)` is base-only; the new flagship-fence test.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement**; update the flagship `.gitignore` to the fenced form so its own drift test passes; update callers of `render_gitignore` to pass `lang`.

- [ ] **Step 4: Run — pass** (incl. full `vrg-container-run -- vrg-validate`).

- [ ] **Step 5: Commit.** `--message "compose repo-init .gitignore through gitignore.render_block"`

---

### Task 7: Fleet driver

**Files:**

- Create: `src/vergil_tooling/lib/fleet_sweep.py` (generic per-repo work-chain), `src/vergil_tooling/bin/vrg_fleet_sync.py` (gitignore-configured entry)
- Modify: `pyproject.toml` (`vrg-fleet-sync` script)
- Test: `tests/vergil_tooling/test_fleet_sweep.py`

**Interfaces — Produces (the seam):**

- `Applicator = Callable[[Path], AppResult]` where `AppResult(changed: bool, summary: str)` — the bespoke file change (gitignore's is `vrg-gitignore-sync --write`)
- `SweepSpec` dataclass: `repos: list[str]`, `branch_slug: str`, `title: str`, `body: str`, `commit_type: str`, `commit_scope: str`, `epic: str | None`
- `run_sweep(spec: SweepSpec, applicator: Applicator, *, dry_run: bool) -> list[RepoResult]` — per repo: ensure ad-hoc/epic issue → worktree/branch → applicator → if changed: `vrg-commit` + `vrg-pr-workflow report-ready`, else skip → collect `RepoResult(repo, status, detail)`. One repo's failure is caught and recorded, never aborts the sweep.

Driver uses `lib/git.py` + `lib/github.py` wrappers; **never** submits/merges.

**Scope boundary (spec §2 non-goal):** gitignore (`vrg-fleet-sync`) is the **sole** consumer of the seam in this epic. Do **not** build a plugin registry, config-driven applicator discovery, a second consumer, or a general CLI for arbitrary change-scripts — that is follow-on #328. The seam is only the `Applicator`/`SweepSpec` types + `run_sweep`, with one hard-wired consumer.

- [ ] **Step 1: Failing tests** (mock git/github/subprocess): dry-run touches nothing and reports intended actions; a repo the applicator reports `changed=False` is skipped (no branch, no issue); a repo that raises is recorded `status="error"` and the sweep continues to the next; a changed repo produces `report-ready` with the templated title/body.

- [ ] **Step 2: Run — fail.**

- [ ] **Step 3: Implement** `run_sweep` + `vrg_fleet_sync.main` (builds a gitignore `SweepSpec`, applicator = shell out to `vrg-gitignore-sync --write`).

- [ ] **Step 4: Run — pass.**

- [ ] **Step 5: Commit.** `--message "add generic fleet-sweep driver + vrg-fleet-sync"`

---

### Task 8 (operational — `deployment`): Release + deploy gate

Not a code PR. Blocked-by Tasks 1–7. Run via `issue-deploy` after the human releases.

- **Precondition (human-attested):** vergil-tooling released with Tasks 1–7 (a `v2.x` tag).
- **Procedure:** install the released tag; `vrg-gitignore-sync --check` a known repo against it; confirm the transitional audit accepts both forms and the composed fence matches `render_block`.
- **Acceptance:** `Outcome: SUCCESS` comment recording the released tag + verification. Gates the sweep (Task 9).

---

### Task 9 (operational — sweep): Migration sweep

Not a hand sweep — **run the fleet driver** with the released tool. Blocked-by Task 8.

- `vrg-fleet-sync --repos <both-orgs list>` (dry-run first, then live) → one migration PR per repo, each **removing** foreign-language lines. Human submits/merges (per policy). Record completion (which repos, PRs) on the task.

---

### Task 10: Tighten audit + delete monolith

**Files:** Modify `repo_config.py` (flip `_GITIGNORE_FENCED_ONLY = True`, delete legacy arm), delete `src/vergil_tooling/data/gitignore.baseline`; Test updates.

Blocked-by Task 9 (every repo fenced). The lossless-split invariant test (Task 1) must still be green at deletion — keep a frozen snapshot of the 55 legacy patterns *in the test* so deletion provably orphans nothing; the test then stands as a regression guard against future fragment edits dropping a pattern.

- [ ] **Step 1:** Freeze the 55 patterns as a literal in the invariant test (was read from the monolith).
- [ ] **Step 2:** Flip `_GITIGNORE_FENCED_ONLY = True`, delete the legacy acceptance arm in `_check_gitignore`, and remove the transitional-acceptance tests added in Task 5 (the "legacy-superset repo passes" case in `test_repo_config.py`); keep the fenced-form and base-only cases.
- [ ] **Step 3:** Delete `gitignore.baseline`; remove `_load_gitignore_baseline` reads that are now dead.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate` green.
- [ ] **Step 5: Commit.** `--message "tighten gitignore audit to fenced-only; delete monolith"`

---

## Self-Review

**Spec coverage:** §3 composition + base-only → Task 1; §5 content split + invariant → Task 1; §6 managed block → Task 2; §7.1 lib → Tasks 1–3; §7.2 CLI → Task 4; §7.3 repo-init → Task 6; §7.4 fleet driver → Task 7; §8 audit → Task 5; §9 migration → Task 3 (logic) + Task 9 (sweep); §10 rollout → Tasks 8–10; §11 testing → per-task tests + Task 1 invariant; §12 risks → transitional audit (Task 5), release gate (Task 8). No gaps.

**Placeholders:** none — each code step carries real code or an exact spec-referenced list.

**Type consistency:** `compose(lang)`, `render_block(lang)`, `parse(text)->(list,str|None)`, `check(text,lang)->Compliance`, `sync(text,lang)->SyncResult`, `run_sweep(spec,applicator)->list[RepoResult]` used consistently across tasks.

## Task → GitHub issue mapping (for epic filing)

Tasks 1–7, 10 are **code** tasks (vergil-tooling, one PR each). Task 8 is a **deployment** operational task; Task 9 is the **sweep** (uses the driver). Ordering: 1→2→3→4; 5, 6, 7 depend on 1–3; 8 blocked-by 1–7; 9 blocked-by 8; 10 blocked-by 9.
