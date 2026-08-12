# Cloud memory read-only projection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Epic:** vergil-project/.github#156 · **Spec:** `epics/156-cloud-memory-projection/spec.md`

**Goal:** Make the physical host the single source of truth for agent memory and project it into cloud VMs as a read-only cache, refreshed at every `vrg-vm` cloud-session start, so cloud sessions read canonical memory and can never silently lose a write.

**Architecture:** A new empirical, fail-closed platform resolver (`vrg-whoami --platform`) tells code which of `physical-host` / `local-vm` / `cloud-vm` it is on. On a cloud session, `vrg-vm` aligns the guest working directory to the host project path (so Claude derives the host's memory slug), copies that repo's memory subset plus the global `CLAUDE.md` from the host over the established `transport.pipe` idiom, verifies the projection resolved, then applies read-only permissions surgically. Cloud change requests are filed via `triage-capture`; there is no automated write-back.

**Tech Stack:** Python 3.12+ (`vergil_tooling` package), `pytest`, bash-over-transport (`Transport.run`/`pipe`) as used throughout `vm_cloud.py`/`vm_guest.py`; file projection uses the `transport.pipe` (`cat > <dest>`) idiom (no `rsync` over the IAP/SSH transport).

## Global Constraints

- **Portability:** all guest-side shell must work on the cloud guest (Ubuntu) and be shellcheck-clean; host-side Python must run on macOS and Linux. (spec + repo CLAUDE.md)
- **No silent failures:** a failed projection must fail *loudly*; a futile memory write must fail with `EACCES`, never succeed-then-vanish. (spec §Principle, §Error handling)
- **Read-only control is cloud-only:** it activates on `cloud-vm` and must never activate on `local-vm` or `physical-host`. (spec §Platforms)
- **Fail closed:** an unconfirmable VM is treated as `cloud-vm` (locked), never as `physical-host`. (spec §Component 1)
- **Never replicate the full macOS HOME** into the guest. (spec §Component 2b)
- **Surgical locking:** lock `memory/` + `MEMORY.md`, never blanket-`chmod` `projects/<slug>/` (transcripts must stay writable). (spec §Component 3)
- **Runtime-writer disqualifier:** lock a file only if neither provisioning nor the Claude Code runtime writes it during a session. (spec §Governing rule)
- **Validation command:** `vrg-container-run -- vrg-validate` is the only validation entry point (expands to `uv run vrg-validate` here). Do not run individual linters.
- **Git/GitHub:** use `vrg-git` / `vrg-gh`; commit via `vrg-commit`. All work on a feature-branch worktree; agents stop at `vrg-pr-workflow report-ready` (never `vrg-submit-pr`).

## File Structure

| File | Responsibility |
|------|----------------|
| `src/vergil_tooling/lib/platform_env.py` *(new)* | Pure resolver: derive `physical-host`/`local-vm`/`cloud-vm` from empirical signals; fail-closed; expose a `Resolution`-style dataclass mirroring `identity_mode.py`. |
| `src/vergil_tooling/bin/vrg_whoami.py` *(modify)* | Add `--platform` token + `--explain` platform section, reusing the existing signal-disagreement idiom. |
| `src/vergil_tooling/lib/vm_memory.py` *(new)* | Host→guest memory projection: compute host slug + host memory path, copy the per-repo `memory/`+`MEMORY.md` and global `CLAUDE.md` file-by-file via `transport.pipe`, apply surgical read-only, and verify (fail loudly). Pure-ish helpers + transport-driven ops. |
| `src/vergil_tooling/lib/vm_cloud.py` *(modify)* | Add the host-path indirection helper (Component 2a) used by the cloud session/build. |
| `src/vergil_tooling/bin/vrg_vm.py` *(modify)* | In `_cloud_session`: create the host-path symlink, set `workdir` to the host path, call the memory projection + verification before `exec_session`. |
| `docs/site/docs/guides/agent-vm-claude-share-set.md` *(modify)* | Document the read-only cloud-memory projection model (feeds the doc-review bookend #2405). |
| `tests/vergil_tooling/test_platform_env.py` *(new)* | Resolver unit tests incl. fail-closed + disagreement. |
| `tests/vergil_tooling/test_vrg_whoami.py` *(modify)* | `--platform` token + explain output. |
| `tests/vergil_tooling/test_vm_memory.py` *(new)* | Slug/path computation, pipe-copy command shape, surgical-lock command shape, verification-fails-loudly. |
| `tests/vergil_tooling/test_vm_cloud.py` *(modify)* | Host-path indirection command shape; `_cloud_session` wiring (workdir + projection call order). |

## Task dependency graph

```text
Task 1 (audit) ─┬─▶ Task 4 (memory sync) ─▶ Task 5 (lock) ─▶ Task 6 (verify)
                └─▶ Task 3 (path align) ───▶ Task 4
Task 2 (resolver) ── independent, consumed by Task 5 (gate) + Task 7
Task 7 (policy clause) ── independent (soft control)
```

Each task below becomes a GitHub implementation task filed under the epic (plan step 9).

---

### Task 1: Copied-config & runtime-writer audit (gating investigation)

**Why:** the locked set (Component 3) and the path-preservation breadth (Component 2b) both depend on facts we must establish empirically before writing lock code: exactly what `vrg-vm` copies into the guest, which of those files the Claude Code runtime writes during a session, and whether any copied file embeds `/Users/<me>/…` literals that would otherwise force a HOME rewrite. This task produces a written finding, not code.

**Files:**

- Create: `epics/156-cloud-memory-projection/audit-findings.md` (in `.github`, committed with this task)

**Steps:**

- [ ] **Step 1: Enumerate what is copied into the guest.** Read `copy_claude_config` (`vm_guest.py:427`) and `_cloud_session` (`vrg_vm.py:2396`); list every file/dir written into the guest `~/.claude` and `~/.config/vergil`. Record the list.
- [ ] **Step 2: Classify each copied file by writer.** For each, determine: does *provisioning* rewrite it after copy? does the *Claude Code runtime* write it during a session? Check `settings.json` specifically — inspect the host `~/.claude/settings.json` and confirm whether `claude plugin install/update` or a session mutates it (cross-ref `update_plugins`, `vm_guest.py:339`). Record a table: file → {provisioning-writes, runtime-writes} → lockable?
- [ ] **Step 3: Scan copied config for host-absolute literals.** Grep the host `~/.claude/CLAUDE.md` and `settings.json` for `/Users/` literals. For each hit, decide per Component 2b: rewrite/remove the literal, or leave that file unlocked. Record the decision.
- [ ] **Step 4: Inspect existing cloud volume for divergent memory.** On a current cloud box (or from `/vergil/claude/projects` if reachable), check for any `memory/` content that diverges from the host canonical — evidence of already-lost futile writes (spec §Error handling). Record findings so nothing is discarded unseen.
- [ ] **Step 5: Write `audit-findings.md`** with the locked set, the 2b decisions, and the divergent-memory findings. This is the input to Tasks 3–5.
- [ ] **Step 6: Commit.**

```bash
vrg-commit --type docs --scope epic-156 --message "copied-config and runtime-writer audit findings"
```

---

### Task 2: Platform resolver (`platform_env.py` + `vrg-whoami --platform`)

**Files:**

- Create: `src/vergil_tooling/lib/platform_env.py`
- Modify: `src/vergil_tooling/bin/vrg_whoami.py`
- Test: `tests/vergil_tooling/test_platform_env.py`, `tests/vergil_tooling/test_vrg_whoami.py`

**Interfaces:**

- Produces:
  - `class Platform(enum.Enum): PHYSICAL_HOST="physical-host"; LOCAL_VM="local-vm"; CLOUD_VM="cloud-vm"`
  - `@dataclass(frozen=True) class PlatformResolution: platform: Platform; resolved_from: str; signals: dict[str, str]; disagreement: bool`
  - `def resolve_platform() -> PlatformResolution` — empirical, fail-closed.
  - `def current_platform() -> Platform`
  - `def is_cloud() -> bool`
- Consumes: `vergil_tooling.lib.identity_mode.resolve` (identity corroboration).

**Design (empirical signals, precedence):** derive without any written marker file —

1. macOS (`platform.system() == "Darwin"`) and no `/vergil` mount ⇒ `PHYSICAL_HOST`.
2. `/vergil` mount present ⇒ in a VM. Distinguish cloud vs local: a reachable cloud metadata endpoint (GCP `metadata.google.internal` / Azure IMDS, probed with a short timeout) ⇒ `CLOUD_VM`; else Lima markers (e.g. `/mnt/lima` or the lima serial device) ⇒ `LOCAL_VM`.
3. **Fail closed:** any VM signal present but not positively confirmed `LOCAL_VM` ⇒ `CLOUD_VM`. Never fall through to `PHYSICAL_HOST` from inside a VM.
4. Corroborate with identity (`is_agent()`): agent identity on a box resolved as `PHYSICAL_HOST` ⇒ set `disagreement=True`.

- [ ] **Step 1: Write failing resolver tests.**

```python
# tests/vergil_tooling/test_platform_env.py
from vergil_tooling.lib.platform_env import Platform, resolve_platform

def test_darwin_no_vergil_is_physical_host(monkeypatch):
    monkeypatch.setattr("platform.system", lambda: "Darwin")
    monkeypatch.setattr("vergil_tooling.lib.platform_env._vergil_mount_present", lambda: False)
    assert resolve_platform().platform is Platform.PHYSICAL_HOST

def test_vergil_plus_cloud_metadata_is_cloud_vm(monkeypatch):
    monkeypatch.setattr("vergil_tooling.lib.platform_env._vergil_mount_present", lambda: True)
    monkeypatch.setattr("vergil_tooling.lib.platform_env._cloud_metadata_reachable", lambda: True)
    assert resolve_platform().platform is Platform.CLOUD_VM

def test_vm_without_local_confirmation_fails_closed_to_cloud(monkeypatch):
    monkeypatch.setattr("vergil_tooling.lib.platform_env._vergil_mount_present", lambda: True)
    monkeypatch.setattr("vergil_tooling.lib.platform_env._cloud_metadata_reachable", lambda: False)
    monkeypatch.setattr("vergil_tooling.lib.platform_env._lima_marker_present", lambda: False)
    assert resolve_platform().platform is Platform.CLOUD_VM  # never PHYSICAL_HOST

def test_agent_identity_on_physical_host_flags_disagreement(monkeypatch):
    monkeypatch.setattr("platform.system", lambda: "Darwin")
    monkeypatch.setattr("vergil_tooling.lib.platform_env._vergil_mount_present", lambda: False)
    monkeypatch.setattr("vergil_tooling.lib.platform_env._identity_is_agent", lambda: True)
    assert resolve_platform().disagreement is True
```

- [ ] **Step 2: Run to confirm they fail** — `uv run pytest tests/vergil_tooling/test_platform_env.py -v` → FAIL (module missing).
- [ ] **Step 3: Implement `platform_env.py`** with the enum, dataclass, the four private signal helpers (`_vergil_mount_present`, `_cloud_metadata_reachable`, `_lima_marker_present`, `_identity_is_agent`), and `resolve_platform()` applying the precedence above. Model the file on `identity_mode.py` (same dataclass/`disagreement` shape).
- [ ] **Step 4: Run resolver tests** → PASS.
- [ ] **Step 5: Add `--platform` to `vrg_whoami.py`** — extend the mutually-exclusive group; `--platform` prints the token; `--explain` gains a platform block listing each signal and warning on `disagreement`. Add tests to `test_vrg_whoami.py` asserting the token output and the explain warning.
- [ ] **Step 6: Run whoami tests** → PASS.
- [ ] **Step 7: `vrg-container-run -- vrg-validate`** → PASS.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope whoami --message "add empirical fail-closed platform resolver (--platform)"`

---

### Task 3: Host-path alignment on cloud sessions (Component 2a)

**Files:**

- Modify: `src/vergil_tooling/lib/vm_cloud.py` (new `ensure_host_path` helper)
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`_cloud_session`: create indirection, set `workdir`)
- Test: `tests/vergil_tooling/test_vm_cloud.py`

**Interfaces:**

- Produces: `def ensure_host_path(transport, host_projects_dir: str, org: str, repo: str) -> str` — creates `<host_projects_dir>/<org>/<repo>` in the guest as a symlink onto `/vergil/projects/<org>/<repo>` (mkdir -p the parent first), returns the host-equivalent absolute path. Idempotent.
- Consumes: `target.identity.projects_dir` (the host projects dir, e.g. `/Users/<me>/dev/projects`), `target.org`, `target.repo`.

**Why:** today `_cloud_session` sets `workdir = /vergil/projects/<org>/<repo>` (`vrg_vm.py:2436`), so Claude derives slug `-vergil-projects-…`. Pointing the session at the host path makes the slug match the host, which is the precondition for the memory projection to be *found*.

- [ ] **Step 1: Write failing test for `ensure_host_path`.**

```python
# tests/vergil_tooling/test_vm_cloud.py
from unittest.mock import MagicMock
from vergil_tooling.lib import vm_cloud

def test_ensure_host_path_symlinks_onto_volume():
    transport = MagicMock()
    path = vm_cloud.ensure_host_path(transport, "/Users/me/dev/projects", "vergil-project", "vergil-tooling")
    assert path == "/Users/me/dev/projects/vergil-project/vergil-tooling"
    # transport.run("bash", "-c", <script>) — the script is the last positional arg
    script = transport.run.call_args.args[-1]
    assert "mkdir -p /Users/me/dev/projects/vergil-project" in script
    assert "ln -sfn /vergil/projects/vergil-project/vergil-tooling /Users/me/dev/projects/vergil-project/vergil-tooling" in script
```

- [ ] **Step 2: Run** → FAIL (no `ensure_host_path`).
- [ ] **Step 3: Implement `ensure_host_path`** in `vm_cloud.py` following the `link_cloud_claude_dirs` bash-over-transport idiom (single `bash -c`, `mkdir -p` parent then `ln -sfn` the leaf onto the `/vergil` checkout).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Wire into `_cloud_session`** — after `link_cloud_claude_dirs`, call `host_workdir = vm_cloud.ensure_host_path(transport, identity.projects_dir, target.org, target.repo)` and pass `workdir=host_workdir` to `exec_session` (replacing the `/vergil/projects/...` literal at `vrg_vm.py:2436`). Add a `test_vm_cloud.py` test asserting `_cloud_session` passes the host path as workdir (mock transport + `exec_session`).
- [ ] **Step 6: Run wiring test** → PASS.
- [ ] **Step 7: `vrg-container-run -- vrg-validate`** → PASS.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope vm-cloud --message "align cloud session workdir to host project path for memory-slug parity"`

---

### Task 4: Memory projection sync (`vm_memory.py`) — per cloud-session refresh

**Files:**

- Create: `src/vergil_tooling/lib/vm_memory.py`
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`_cloud_session`: call the projection before `exec_session`)
- Test: `tests/vergil_tooling/test_vm_memory.py`

**Interfaces:**

- Produces:
  - `def host_slug(host_workdir: str) -> str` — the Claude memory slug for an absolute path (leading `-`, `/`→`-`), matching how Claude derives `~/.claude/projects/<slug>`.
  - `def host_memory_dir(claude_dir: Path, slug: str) -> Path` — `claude_dir/"projects"/slug/"memory"`.
  - `def project_memory(transport, *, claude_dir: Path, host_workdir: str) -> None` — copy the per-repo `memory/`+`MEMORY.md` and the global `CLAUDE.md` from host to guest at the matching slug, file-by-file over `transport.pipe` (the `copy_claude_config` idiom). Read-only is applied by Task 5's `lock_projection`, called at the end here.
- Consumes: `ensure_host_path`'s returned `host_workdir` (Task 3); `Path.home()/".claude"`.

**Why:** net-new logic — nothing copies the per-repo `memory/` today. Refresh is per cloud-session (spec §Component 3).

- [ ] **Step 1: Write failing tests for pure helpers.**

```python
# tests/vergil_tooling/test_vm_memory.py
from pathlib import Path
from vergil_tooling.lib import vm_memory

def test_host_slug_matches_claude_convention():
    assert vm_memory.host_slug("/Users/me/dev/projects/org/repo") == "-Users-me-dev-projects-org-repo"

def test_host_memory_dir():
    d = vm_memory.host_memory_dir(Path("/Users/me/.claude"), "-Users-me-dev-projects-org-repo")
    assert d == Path("/Users/me/.claude/projects/-Users-me-dev-projects-org-repo/memory")
```

- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement `host_slug` / `host_memory_dir`.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Write failing test for `project_memory` command shape.**

```python
def test_project_memory_copies_memory_and_claude_md(tmp_path):
    transport = MagicMock()
    claude = tmp_path / ".claude"
    (claude / "projects/-Users-me-dev-projects-org-repo/memory").mkdir(parents=True)
    (claude / "projects/-Users-me-dev-projects-org-repo/memory/MEMORY.md").write_text("m")
    (claude / "CLAUDE.md").write_text("global")
    vm_memory.project_memory(transport, claude_dir=claude, host_workdir="/Users/me/dev/projects/org/repo")
    # copy_claude_config idiom: transport.pipe("cat > <dest>", content)
    piped = [c.args[0] for c in transport.pipe.call_args_list]
    assert any("cat > " in cmd and "/memory/" in cmd for cmd in piped)
    assert any("cat > " in cmd and "CLAUDE.md" in cmd for cmd in piped)
```

- [ ] **Step 6: Run** → FAIL.
- [ ] **Step 7: Implement `project_memory`** — resolve slug from `host_workdir`; `transport.run("mkdir", "-p", <guest memory dir>)`; then for each file under the host `memory/` dir (including `MEMORY.md`) and the global `CLAUDE.md`, `transport.pipe(f"chmod u+w <dest> 2>/dev/null; cat > <dest>", content)` to the matching guest path under `~/.claude/projects/<slug>/memory/` and `~/.claude/CLAUDE.md` (guest paths are volume-linked) — the `chmod u+w` clears a prior read-only lock before overwrite. Memory files are text, so the `cat >` idiom applies directly. Then call `lock_projection` (Task 5). If the host memory dir is absent, create the slug dir and skip the memory copy (no error — a repo may have no memory yet).
- [ ] **Step 8: Run** → PASS.
- [ ] **Step 9: Wire into `_cloud_session`** — after `ensure_host_path`, call `vm_memory.project_memory(transport, claude_dir=claude_dir, host_workdir=host_workdir)` before `exec_session`. Add a wiring test.
- [ ] **Step 10: Run** → PASS.
- [ ] **Step 11: `vrg-container-run -- vrg-validate`** → PASS.
- [ ] **Step 12: Commit.** `vrg-commit --type feat --scope vm-memory --message "project host memory into cloud session at matching slug (per-session pipe copy)"`

---

### Task 5: Surgical read-only lock (`lock_projection`)

**Files:**

- Modify: `src/vergil_tooling/lib/vm_memory.py` (add `lock_projection`)
- Test: `tests/vergil_tooling/test_vm_memory.py`

**Interfaces:**

- Produces: `def lock_projection(transport, *, claude_dir_guest: str, slug: str, locked_set: Sequence[str]) -> None` — `chmod` the projected canonical set read-only: `~/.claude/projects/<slug>/memory` (recursive), `~/.claude/projects/<slug>/memory/MEMORY.md` if present, `~/.claude/CLAUDE.md`, and any additional files in `locked_set` from the Task-1 audit. Never touches the `.jsonl` transcripts in `projects/<slug>/`.
- Consumes: the audit's `locked_set` (Task 1).

- [ ] **Step 1: Write failing test — surgical scope.**

```python
def test_lock_projection_locks_memory_not_transcripts():
    transport = MagicMock()
    vm_memory.lock_projection(transport, claude_dir_guest="$HOME/.claude",
                              slug="-Users-me-dev-projects-org-repo",
                              locked_set=["$HOME/.claude/CLAUDE.md"])
    script = transport.run.call_args.args[-1]
    assert "chmod -R a-w \"$HOME/.claude/projects/-Users-me-dev-projects-org-repo/memory\"" in script
    assert "chmod a-w \"$HOME/.claude/CLAUDE.md\"" in script
    # never a blanket chmod of the projects/<slug> dir itself
    assert "projects/-Users-me-dev-projects-org-repo\"" not in script.replace("/memory", "")
```

- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement `lock_projection`** — a single `bash -c` that chmods exactly the memory subtree, `MEMORY.md`, `CLAUDE.md`, and audited extras read-only; guarded with `[ -e ]` tests so a missing optional file is a no-op, not an error.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Call `lock_projection` from `project_memory`** (end of Task 4's function) with the audited `locked_set`. Update `test_project_memory_*` to assert a lock op follows the pipe copy.
- [ ] **Step 6: Run** → PASS.
- [ ] **Step 7: `vrg-container-run -- vrg-validate`** → PASS.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope vm-memory --message "apply surgical read-only lock to projected memory and CLAUDE.md"`

---

### Task 6: Projection verification — fail loudly (`verify_projection`)

**Files:**

- Modify: `src/vergil_tooling/lib/vm_memory.py` (add `verify_projection`)
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (call verification after `project_memory`)
- Test: `tests/vergil_tooling/test_vm_memory.py`

**Interfaces:**

- Produces: `class ProjectionError(RuntimeError)`; `def verify_projection(transport, *, host_workdir: str, slug: str) -> None` — asserts, in-guest, that (a) `host_workdir` resolves (the Task-3 symlink exists and points at the `/vergil` checkout) and (b) `~/.claude/projects/<slug>/memory` exists; raises `ProjectionError` with an actionable message otherwise.

**Why:** a broken indirection would otherwise degrade *silently* to empty memory (spec §Component 6).

- [ ] **Step 1: Write failing test.**

```python
import subprocess
import pytest
from unittest.mock import MagicMock

def test_verify_projection_raises_when_host_path_missing():
    transport = MagicMock()
    transport.run.side_effect = subprocess.CalledProcessError(1, "test")
    with pytest.raises(vm_memory.ProjectionError):
        vm_memory.verify_projection(transport, host_workdir="/Users/me/dev/projects/org/repo",
                                    slug="-Users-me-dev-projects-org-repo")
```

- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement `verify_projection`** — run the two `test` checks over the transport; on `CalledProcessError` raise `ProjectionError` naming the failed check and the fix (re-run session / rebuild).
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Wire into `_cloud_session`** — call `vm_memory.verify_projection(...)` immediately after `project_memory`, before `exec_session`; let `ProjectionError` abort the session with a non-zero exit and the message (no silent continue).
- [ ] **Step 6: Run wiring test** → PASS.
- [ ] **Step 7: `vrg-container-run -- vrg-validate`** → PASS.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope vm-memory --message "verify projection resolved and fail loudly on a broken indirection"`

---

### Task 7: Read-only policy clause in canonical `CLAUDE.md` (soft control)

**Files:**

- Modify: the developer's canonical `~/.claude/CLAUDE.md` (a **human/config action**, not a repo PR — this file is personal/global)
- Modify: `docs/site/docs/guides/agent-vm-claude-share-set.md` (document the clause + the platform resolver so the convention is discoverable)

**Why:** the soft control that stops an agent *attempting* a cloud memory write; the read-only permissions (Task 5) are the hard backstop. Because the file is personal, the deliverable is (a) the exact clause text and (b) repo docs describing it; the human applies the clause to their global file.

- [ ] **Step 1: Draft the exact clause** to add to `~/.claude/CLAUDE.md`:

  > **Cloud VM memory is a read-only cache.** When `vrg-whoami --platform` reports `cloud-vm`, agent memory and copied config are a read-only projection of the physical host. Never write memory in a cloud session — the write will fail (`EACCES`) and would be futile anyway. When memory needs to change, file it via `/vergil:triage-capture` and let it be implemented on the physical host, which re-projects on the next session.

- [ ] **Step 2: Document it in `agent-vm-claude-share-set.md`** — a section describing the three platforms, the resolver, the read-only projection, and the writeback path; link the epic/spec.
- [ ] **Step 3: Human applies the clause** to `~/.claude/CLAUDE.md` on the host (manual; the agent cannot PR a personal file). Record completion in the task.
- [ ] **Step 4: `vrg-container-run -- vrg-validate`** (docs markdownlint) → PASS.
- [ ] **Step 5: Commit the docs change.** `vrg-commit --type docs --scope vm --message "document read-only cloud-memory projection and platform resolver"`

---

## Operational tasks (already seeded under the epic)

- **Cold-rebuild validation** — vergil-project/vergil-tooling#2406. Blocked-by Tasks 3–6 (and the release that ships them). On a freshly rebuilt cloud VM: host path resolves; slug memory dir matches host and is non-empty; `memory/`+`MEMORY.md`+`CLAUDE.md` read-only; transcripts writable; `vrg-whoami --platform` → `cloud-vm`; write to `memory/` fails `EACCES`; a host-side edit shows up on the next session start.
- **Documentation-review sweep** — vergil-project/vergil-tooling#2405 (closing gate; may spawn a `vergil-vm` doc task if provisioning templates there need updating).

## Self-Review

**Spec coverage:**

- Component 1 (resolver) → Task 2. ✅
- Component 2a (slug/path alignment) → Task 3. ✅
- Component 2b (config-internal paths, no full HOME) → Task 1 (audit decides per file). ✅
- Component 3 (session-coupled sync + surgical copy-then-lock, runtime-writer disqualifier) → Tasks 1, 4, 5. ✅
- Component 4 (policy clause) → Task 7. ✅
- Component 5 (issue-driven writeback) → Task 7 clause references `triage-capture`; no code needed (reuses existing skill). ✅
- Component 6 (verify, fail loudly) → Task 6. ✅
- Testing §, Validation § → per-task tests + #2406. ✅
- Data-residency, already-lost-writes → Task 1 Step 4 + docs (Task 7). ✅

**Placeholder scan:** the audit (Task 1) is a deliberate investigation, not a placeholder; the exact `locked_set` and 2b rewrites are its typed outputs consumed by Tasks 4–5. The projection has a single hook — cloud-session start — and the first session on a fresh box performs the initial projection (no separate build-seed path). Files are moved via the `transport.pipe` (`cat >`) idiom; there is no `rsync` over the transport.

**Type consistency:** `host_workdir` (str) flows Task 3 → Task 4 → Task 6; `slug` (str) from `host_slug` used in Tasks 4/5/6; `locked_set` (Sequence[str]) from Task 1 into `lock_projection`; `Platform`/`is_cloud` from Task 2 consumed by the Task-7 clause condition and any gate. Names consistent across tasks.
