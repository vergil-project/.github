# Vergil Observability Platform — Brick 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `vrg-fleet` — a CLI that renders a recency-sorted, correlated map of all in-flight work (Claude sessions ↔ git worktrees ↔ GitHub epics) across the Mac and local Lima VMs, backed by a versioned JSON contract.

**Architecture:** A new `vergil_tooling.lib.fleet` package with a **host-as-data-source** collector feeding a **correlator** that emits an immutable, versioned **contract** dataclass tree; `bin/vrg_fleet.py` renders it as a human tree or raw `--json`. It reuses existing infra (`github.graphql`, `worktrees.list_worktrees`, `platform_env.resolve_platform`, `session.parse_name`, `vm_transport`) rather than reinventing.

**Tech Stack:** Python 3.12/3.14, stdlib only for the core (json, pathlib, dataclasses); `gh` via `lib.github`; `limactl` via `lib.vm_lima`/`vm_transport`. Tests: pytest. Validation: `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here).

## Global Constraints

- **Repo:** all code lands in `vergil-tooling` (`src/vergil_tooling/…`, `tests/vergil_tooling/…`). The epic/spec/plan live in `.github`; this plan's tasks are filed as `vergil-tooling` issues under epic `vergil-project/.github#171`.
- **Validation is one command:** `vrg-container-run -- vrg-validate`. Do not run linters individually.
- **Git/GH via wrappers only:** `vrg-git`, `vrg-gh` (raw `git`/`gh` are denied).
- **Portability:** macOS + Linux; shellcheck-clean for any shell; Python is the primary surface here.
- **No silent failures** (repo policy, and the *product* here): every unreachable host / unparseable file / failed query becomes an explicit fault in the contract; `vrg-fleet` exits non-zero on a partial snapshot.
- **Read-only:** the collector never mutates sessions, worktrees, or GitHub.
- **Contract is versioned:** `CONTRACT_VERSION` starts at `"1.0"`; later planes add keys, never reshape.

---

## Decisions to resolve first (the spec's 4 open questions)

Resolve these as **Task 0** spikes before building on them. Each has a guaranteed-correct fallback so the MVP is never blocked.

- **D1 — positive VM signal for host attribution.** `~/.claude` is a **shared virtiofs mount**: sessions run *inside* a local Lima VM write to the same `~/.claude/projects` the Mac reads, with identical `cwd` (`/Users/pmoore/…`). So `cwd` cannot attribute a session to a host. **Spike:** inspect session JSONL records for a per-session host discriminator (hostname, `origin`, `userType`, a platform axis, or an env marker). **Fallback if none exists:** every shared-home session is `host: "ambiguous"` — never defaulted to the Mac. (Optionally, a follow-up teaches the VM to stamp a marker; out of scope for the MVP.)
- **D2 — Lima collection mechanism.** Because local VMs share `~/.claude`, the Mac-side read **already sees their sessions** — no `limactl shell` exec needed for shared-home hosts. **Resolution:** the collector reads the shared `~/.claude` locally and uses a transport (`vm_transport.LimaTransport`) only for a host that does **not** share home. This makes "Phase 2 local Lima" mostly an *attribution + enumeration* problem, not a remote-exec one.
- **D3 — cloud relay producer + registry.** Phase 3 is droppable. Cloud fleet-state rides the epic-#148 relay-ref transport; cloud VMs enumerate via `lib.vm_cloud`. Confirm both exist/are-ready at Phase 3 start; if not, cut Phase 3 to a follow-on.
- **D4 — metrics transport (Prometheus vs OTLP).** Explicitly **out of scope**; belongs to the brick-2 epic. Do not build metrics here.

### Task 0: Resolve D1/D2/D3 and record the decisions

**Files:**
- Create: `epics/171-observability-platform/decisions.md` (in the `.github` epic dir) — short record of what each spike found.

- [ ] **Step 1:** Dump the distinct top-level keys and a sample record from three known session files — one Mac-only session, one known to have run inside the shared-home VM (e.g. an `epic-79`/`epic-91` file under the `mq-resiliency-lab-for-linux` slug) — and diff their record shapes for a host discriminator.

Run (read-only inspection):
```bash
python3 - <<'PY'
import json, glob, os, collections
root=os.path.expanduser('~/.claude/projects')
for f in sorted(glob.glob(root+'/*/*.jsonl'))[:0]: pass
PY
```
(Use a scratch script; do not commit it. Record findings, not the script.)

- [ ] **Step 2:** Record in `decisions.md`: D1 result (discriminator field name, or "none → ambiguous fallback"), D2 confirmation (which local VMs share `~/.claude`), D3 readiness (relay producer + `vm_cloud` enumeration present? yes/no → keep or cut Phase 3).
- [ ] **Step 3:** Commit `decisions.md`.

```bash
vrg-git add epics/171-observability-platform/decisions.md
vrg-commit --type docs --scope epic-171 --message "record brick-1 D1/D2/D3 spike decisions"
```

---

## File structure (locked)

New package `src/vergil_tooling/lib/fleet/`:

| File | Responsibility |
|---|---|
| `__init__.py` | Package marker; re-export `CONTRACT_VERSION`, `collect_snapshot`. |
| `contract.py` | Versioned dataclasses (`Snapshot`, `Host`, `SessionRec`, `WorktreeRec`, `GitHubRollup`, `EpicRec`, `TaskRec`, `PrRec`) + `to_dict()`/`to_json()`. Pure data. |
| `sessions.py` | Read `~/.claude/projects`; **tail/reverse-scan** for the last `custom-title`; extract lineage, `cwd`, `gitBranch`, `uuid`, `mtime`. Pure-ish (I/O isolated in one function). |
| `hosts.py` | `Host` enumeration (Mac + local Lima via `vm_lima`); the **host-attribution rule** (D1) → host id or `"ambiguous"`. |
| `github_rollup.py` | Epics/tasks/PRs/CI pull; **one batched GraphQL** `feature/<N> → issue N → parent epic`; short-TTL cache; offline degradation to `epic=null`. |
| `correlate.py` | Join sessions ↔ worktrees ↔ GitHub; resolve `epic`; `uuid` de-dup; produce `Snapshot`. |
| `collect.py` | Orchestrate per-host collection; capture faults; assemble `Snapshot`; compute `partial` flag. |
| `render.py` | Tree renderer + JSON emitter; fail-loud rows for faulted hosts. |

New CLI: `src/vergil_tooling/bin/vrg_fleet.py`.
Tests: `tests/vergil_tooling/lib/fleet/test_*.py` + `tests/vergil_tooling/test_vrg_fleet.py`; fixtures in `tests/vergil_tooling/lib/fleet/fixtures/`.
Register script in `pyproject.toml`: `vrg-fleet = "vergil_tooling.bin.vrg_fleet:main"`.

---

## Phase 0 — Contract + Mac collector + tree/`--json`

### Task 1: The versioned contract dataclasses

**Files:**
- Create: `src/vergil_tooling/lib/fleet/__init__.py`, `src/vergil_tooling/lib/fleet/contract.py`
- Test: `tests/vergil_tooling/lib/fleet/__init__.py`, `tests/vergil_tooling/lib/fleet/test_contract.py`

**Interfaces:**
- Produces: `CONTRACT_VERSION: str`; frozen dataclasses `Host`, `SessionRec`, `WorktreeRec`, `PrRec`, `TaskRec`, `EpicRec`, `GitHubRollup`, `Snapshot`; `Snapshot.to_dict() -> dict`, `Snapshot.to_json(indent=2) -> str`.

- [ ] **Step 1: Write the failing test**
```python
# tests/vergil_tooling/lib/fleet/test_contract.py
import json
from vergil_tooling.lib.fleet.contract import (
    CONTRACT_VERSION, Snapshot, Host, SessionRec,
)

def test_snapshot_serialises_versioned_json():
    snap = Snapshot(
        generated_at="2026-07-19T00:00:00Z",
        generator="vrg-fleet/test",
        hosts=[Host(id="mac", kind="physical-host", name="mac",
                    reachable=True, claude_projects_root="/x", collected_at="t", error=None)],
        sessions=[SessionRec(host="mac", uuid="u1", project_slug="s", cwd="/x",
                             repo="o/r", branch="develop", title="epic-79-x",
                             title_lineage=[("vergil-user:01:x", 1), ("epic-79-x", 689)],
                             last_active="t", age_seconds=10, epic=79)],
        worktrees=[], github=None, partial=False,
    )
    doc = json.loads(snap.to_json())
    assert doc["contract_version"] == CONTRACT_VERSION == "1.0"
    assert doc["sessions"][0]["epic"] == 79
    assert doc["sessions"][0]["title_lineage"][1] == ["epic-79-x", 689]
    assert doc["partial"] is False
```
- [ ] **Step 2: Run it — expect ImportError/fail.** `uv run pytest tests/vergil_tooling/lib/fleet/test_contract.py -v`
- [ ] **Step 3: Implement `contract.py`.**
```python
# src/vergil_tooling/lib/fleet/contract.py
from __future__ import annotations
import dataclasses, json
from dataclasses import dataclass, field

CONTRACT_VERSION = "1.0"

@dataclass(frozen=True)
class Host:
    id: str
    kind: str            # physical-host | local-vm | cloud-vm
    name: str
    reachable: bool
    claude_projects_root: str | None
    collected_at: str | None
    error: str | None

@dataclass(frozen=True)
class SessionRec:
    host: str            # a Host.id or the sentinel "ambiguous"
    uuid: str
    project_slug: str
    cwd: str | None
    repo: str | None
    branch: str | None
    title: str | None
    title_lineage: list[tuple[str, int]]  # (title, first_seen_line), in order
    last_active: str | None
    age_seconds: int | None
    epic: int | None

@dataclass(frozen=True)
class WorktreeRec:
    host: str
    repo: str
    path: str
    branch: str | None
    issue: int | None
    epic: int | None
    dirty: bool
    ahead: int | None
    behind: int | None

@dataclass(frozen=True)
class PrRec:
    number: int
    state: str
    ci: str | None

@dataclass(frozen=True)
class TaskRec:
    number: int
    repo: str
    state: str
    pr: PrRec | None

@dataclass(frozen=True)
class EpicRec:
    org: str
    number: int
    title: str
    state: str
    tasks: list[TaskRec]
    open_prs: list[PrRec]

@dataclass(frozen=True)
class GitHubRollup:
    epics: list[EpicRec]
    reachable: bool
    error: str | None

@dataclass(frozen=True)
class Snapshot:
    generated_at: str
    generator: str
    hosts: list[Host]
    sessions: list[SessionRec]
    worktrees: list[WorktreeRec]
    github: GitHubRollup | None
    partial: bool

    def to_dict(self) -> dict:
        d = dataclasses.asdict(self)
        return {"contract_version": CONTRACT_VERSION, **d}

    def to_json(self, indent: int = 2) -> str:
        return json.dumps(self.to_dict(), indent=indent, sort_keys=False)
```
- [ ] **Step 4: Run test — expect PASS.**
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope fleet --message "add versioned fleet-state contract dataclasses"`

### Task 2: Tail-read of the authoritative session title (+ lineage)

**Files:**
- Create: `src/vergil_tooling/lib/fleet/sessions.py`
- Test: `tests/vergil_tooling/lib/fleet/test_sessions.py`
- Fixtures: `tests/vergil_tooling/lib/fleet/fixtures/renamed_session.jsonl`, `.../plain_session.jsonl`

**Interfaces:**
- Produces:
  - `read_session_title(path: Path) -> tuple[str | None, list[tuple[str,int]]]` — returns `(authoritative_title, lineage)` where `authoritative_title` is the **last** `custom-title` seen and `lineage` is the ordered list of `(title, first_seen_line)` transitions.
  - `read_session(path: Path) -> SessionRaw` (dataclass: `uuid, project_slug, cwd, gitBranch, last_active_epoch, title, lineage`).
  - `iter_session_files(projects_root: Path) -> Iterator[Path]`.

- [ ] **Step 1: Build the fixture** `renamed_session.jsonl`: 60+ lines where lines 1..40 carry `{"type":"custom-title","customTitle":"vergil-user:01:x","sessionId":"u1"}` (interleaved with dummy message records that carry `cwd`/`gitBranch`), then from line ~45 onward `customTitle` becomes `"epic-79-observability-extraction"`. This encodes the real bug: a naive head-read returns the stale title.
- [ ] **Step 2: Write the failing test.**
```python
# tests/vergil_tooling/lib/fleet/test_sessions.py
from pathlib import Path
from vergil_tooling.lib.fleet.sessions import read_session_title

FIX = Path(__file__).parent / "fixtures"

def test_authoritative_title_is_the_last_record():
    title, lineage = read_session_title(FIX / "renamed_session.jsonl")
    assert title == "epic-79-observability-extraction"          # NOT the stale first title
    assert lineage[0][0] == "vergil-user:01:x"
    assert lineage[-1][0] == "epic-79-observability-extraction"
    assert lineage[0][1] < lineage[-1][1]                        # first-seen line order

def test_plain_session_single_title():
    title, lineage = read_session_title(FIX / "plain_session.jsonl")
    assert title is not None
    assert len(lineage) == 1
```
- [ ] **Step 3: Run — expect fail.**
- [ ] **Step 4: Implement `read_session_title` with a reverse/tail scan** (efficient for multi-MB files). Read the file once, but derive the authoritative title from the *last* matching record; build lineage by tracking title changes in line order.
```python
# src/vergil_tooling/lib/fleet/sessions.py  (excerpt)
from __future__ import annotations
import json
from pathlib import Path

def read_session_title(path: Path) -> tuple[str | None, list[tuple[str, int]]]:
    authoritative: str | None = None
    lineage: list[tuple[str, int]] = []
    last_title: str | None = None
    with path.open("r", encoding="utf-8", errors="replace") as fh:
        for lineno, line in enumerate(fh, start=1):
            line = line.strip()
            if not line or '"custom-title"' not in line:
                continue
            try:
                rec = json.loads(line)
            except json.JSONDecodeError:
                continue                      # skip a torn line, do not abort the file
            if rec.get("type") != "custom-title":
                continue
            t = rec.get("customTitle")
            if not t:
                continue
            authoritative = t
            if t != last_title:               # a transition → lineage entry
                lineage.append((t, lineno))
                last_title = t
    return authoritative, lineage
```
- [ ] **Step 5: Run test — expect PASS.**
- [ ] **Step 6: Add the multi-MB tail case** — generate (in the test, not committed) a ~3MB file whose only `epic-*` title is on the final line; assert `read_session_title` returns it and completes under a soft time bound. (Documents the "must reach EOF" property.)
- [ ] **Step 7: Implement `read_session` + `iter_session_files`** (parse `cwd`/`gitBranch`/`uuid`, `path.stat().st_mtime` for `last_active`), with a test over the fixtures.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope fleet --message "read authoritative (last) session title + lineage via tail scan"`

### Task 3: Map a session `cwd`/`project_slug` to a repo `owner/name`

**Files:** Modify `src/vergil_tooling/lib/fleet/sessions.py`; Test `.../test_sessions.py`
**Interfaces:** Produces `repo_from_cwd(cwd: str | None, dev_root: Path) -> str | None` (returns `owner/name` from a `dev/projects/<org>/<repo>` path, else `None`).

- [ ] **Step 1:** Failing test: `repo_from_cwd("/Users/pmoore/dev/projects/vergil-project/vergil-tooling", Path("/Users/pmoore/dev/projects")) == "vergil-project/vergil-tooling"`; a path outside the tree → `None`.
- [ ] **Step 2:** Run — fail. **Step 3:** Implement (path-relative split; also strip a trailing `/.worktrees/…` segment back to the repo root). **Step 4:** Pass. **Step 5:** Commit.

### Task 4: Mac worktree enumeration (reuse `lib.worktrees`)

**Files:** Modify `src/vergil_tooling/lib/fleet/hosts.py` (create); Test `.../test_hosts.py`
**Interfaces:** Produces `enumerate_worktrees(dev_root: Path, host_id: str) -> list[WorktreeRec]`.

- [ ] **Step 1:** Failing test using a temp git repo with a `.worktrees/issue-687-x` worktree fixture; assert one `WorktreeRec` with `issue == 687`, and that enumeration **does not descend** into `.git/` or a `build/` dir seeded with a fake worktree-looking path.
- [ ] **Step 2:** Run — fail.
- [ ] **Step 3:** Implement over `lib.worktrees.list_worktrees(repo_root)`; parse `issue` from the `issue-<N>-slug` branch/dir name via a compiled regex `^issue-(\d+)-`; bound enumeration to `<repo>/.worktrees/` only.
- [ ] **Step 4:** Pass. **Step 5:** Commit `feat(fleet): enumerate worktrees bounded to .worktrees/ with issue parse`.

### Task 5: Mac host record + assemble a Mac-only `Snapshot`

**Files:** Modify `hosts.py`, create `collect.py`, `correlate.py`; Tests `.../test_collect.py`
**Interfaces:**
- `mac_host(projects_root: Path) -> Host`
- `collect_snapshot(*, dev_root, projects_root, github=False, hosts="mac") -> Snapshot` (github/VMs added in later phases)
- `correlate(sessions, worktrees, github) -> Snapshot` (Phase-0 form: no GitHub join yet; `epic` from `epic-<N>` title only)

- [ ] **Step 1:** Failing test: point `collect_snapshot` at a temp `projects_root` (2 fixture sessions, one `epic-79`) + temp `dev_root`; assert `Snapshot.partial is False`, both sessions present, the `epic-79` session resolves `epic == 79` from its title, the other `epic is None`.
- [ ] **Step 2:** Run — fail.
- [ ] **Step 3:** Implement: read sessions (Task 2), map repos (Task 3), enumerate worktrees (Task 4), and a Phase-0 `correlate` that sets `epic` only when the **title** matches `^epic-(\d+)-` (GitHub join is Phase 1). Assemble `Snapshot`.
- [ ] **Step 4:** Pass. **Step 5:** Commit.

### Task 6: `vrg-fleet` CLI — tree + `--json`

**Files:** Create `src/vergil_tooling/bin/vrg_fleet.py`, `src/vergil_tooling/lib/fleet/render.py`; Modify `pyproject.toml`; Test `tests/vergil_tooling/test_vrg_fleet.py`
**Interfaces:** `render.tree(snapshot) -> str`, `render.emit_json(snapshot) -> str`; CLI flags `--json`, `--dev-root`, `--projects-root` (test seams).

- [ ] **Step 1:** Failing test: invoke `main(["--json", "--projects-root", <fixtures>, "--dev-root", <tmp>])`, capture stdout, `json.loads` it, assert `contract_version == "1.0"` and the `epic-79` session appears. A second test asserts `main([...])` (no `--json`) prints a tree containing `epic-79` and the repo name.
- [ ] **Step 2:** Run — fail.
- [ ] **Step 3:** Implement the CLI (argparse in the `vrg_whoami` module style: docstring + `parse_args` + `main`), calling `collect_snapshot` then `render`. Tree groups `org/repo → epic/issue → session (age, host)`, recency-sorted by `last_active`. Register `vrg-fleet` in `pyproject.toml [project.scripts]`.
- [ ] **Step 4:** Pass. **Step 5:** Commit `feat(fleet): vrg-fleet CLI with tree and --json renderers`.

### Task 7: Phase-0 acceptance — reproduce the corrected live board

- [ ] **Step 1:** Run `uv run vrg-fleet` against the real `~/.claude/projects`; **confirm the three live epic sessions** (`epic-79`, `epic-91`, `epic-74`) render with their **correct** titles (the tail-read win), not stale `vergil-user:NN`.
- [ ] **Step 2:** Run `uv run vrg-fleet --json | python3 -m json.tool` and eyeball the contract.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate` green. **Step 4:** Commit any fixes.

---

## Phase 1 — GitHub correlation + batched epic-join

### Task 8: Batched `feature/<N> → issue → parent epic` resolver

**Files:** Create `src/vergil_tooling/lib/fleet/github_rollup.py`; Test `.../test_github_rollup.py`
**Interfaces:** `resolve_epics(issue_numbers: list[int], repo: str) -> dict[int, int|None]` — **one** GraphQL call resolving many issues→parent-epic; `EpicRollup` fetch of open epics+tasks+PRs+CI.

- [ ] **Step 1:** Failing test with a **stub** `github.graphql` (monkeypatch) returning a canned multi-issue payload; assert `resolve_epics([687, 688], "vergil-project/vergil-tooling")` issues **exactly one** `graphql` call and maps `687→79`, an unlinked issue → `None`.
- [ ] **Step 2:** Run — fail.
- [ ] **Step 3:** Implement over `lib.github.graphql(...)` using aliased sub-queries (batch), plus a process-lifetime short-TTL cache dict. On any GitHub error, raise a typed `GitHubUnreachable` the collector catches (degradation, not abort).
- [ ] **Step 4:** Pass. **Step 5:** Commit.

### Task 9: Wire GitHub into `correlate` + offline degradation

**Files:** Modify `correlate.py`, `collect.py`; Test `.../test_correlate.py`
- [ ] **Step 1:** Failing tests: (a) with a stubbed rollup, a worktree on `feature/687-x` gets `epic == 79` and its session inherits it; (b) with `github=True` but the stub raising `GitHubUnreachable`, `Snapshot.github.reachable is False`, `github.error` is set, every `epic` falls back to title-only (or `None`) and is rendered "unresolved" — **not dropped** — and `Snapshot.partial is True`.
- [ ] **Step 2:** Run — fail. **Step 3:** Implement join + degradation. **Step 4:** Pass. **Step 5:** Commit.

### Task 10: Render idle-but-open epics and orphan branches

**Files:** Modify `render.py`, `github_rollup.py`; Test `test_vrg_fleet.py`
- [ ] **Step 1:** Failing test: a GitHub rollup with an open epic that has **no** local worktree/session renders as an "idle" row; a local `feature/x` branch with **no** resolvable epic renders under an "orphan / ad-hoc" heading. **Step 2:** fail. **Step 3:** implement. **Step 4:** pass. **Step 5:** commit.

- [ ] **Step 6 (GitHub-as-fallback-truth acceptance — spec finding ③):** Add a failing test asserting the reconstruction promise: given a **host that is down** (a `reachable:false` Host, so its sessions/worktrees are absent) whose branch/PR nonetheless exists in the GitHub rollup, that work still renders as an **"idle — pushed work"** row attributed to the down host, proving an off VM's pushed branches remain visible via GitHub. Assert it appears **and** that `Snapshot.partial` is still `True` (the host itself was not reachable). **Step 7:** run — fail. **Step 8:** implement the join (a down host + a GitHub task/PR on its branch → idle-pushed row). **Step 9:** pass. **Step 10:** commit `feat(fleet): show pushed work for an unreachable host via the GitHub fallback`.

---

## Phase 2 — Local Lima hosts = MVP ceiling

### Task 11: Host enumeration + attribution rule (D1)

**Files:** Modify `hosts.py`; Test `.../test_hosts.py`
**Interfaces:** `enumerate_hosts() -> list[Host]` (Mac + local Lima via `lib.vm_lima`); `attribute_host(session_rec, hosts) -> str` → a host id or `"ambiguous"`.

- [ ] **Step 1:** Failing tests: (a) `enumerate_hosts` includes the Mac and each running Lima instance from a stubbed `vm_lima` list; (b) `attribute_host` returns `"ambiguous"` for a shared-home session with **no** discriminator (per D1 fallback), and the specific host id when the D1 discriminator (whatever the spike found) **is** present.
- [ ] **Step 2:** Run — fail.
- [ ] **Step 3:** Implement using the D1 result from `decisions.md`. If D1 found no discriminator, `attribute_host` returns `"ambiguous"` for every shared-home session — **never** the Mac by default.
- [ ] **Step 4:** Pass. **Step 5:** Commit `feat(fleet): host enumeration + shared-home attribution (ambiguous fallback)`.

### Task 12: `uuid` de-dup across hosts

**Files:** Modify `correlate.py`; Test `.../test_correlate.py`
- [ ] **Step 1:** Failing test: two `SessionRec` with the same `uuid` (one seen via Mac read, one via a VM path) collapse to **one** record, attributed per Task 11. **Step 2:** fail. **Step 3:** implement de-dup keyed on `uuid`. **Step 4:** pass. **Step 5:** commit.

### Task 13: Non-shared local VM reach via transport (only if D2 requires)

**Files:** Modify `hosts.py`, `collect.py`; Test `.../test_hosts.py`
- [ ] **Step 1:** *Guard:* if D2 confirmed **all** local VMs share `~/.claude`, mark this task **N/A** in `decisions.md` and skip. Otherwise: failing test with a stubbed `vm_transport.LimaTransport` returning a session listing from a non-shared VM; assert those sessions appear as that host's, and an unreachable VM becomes a `reachable:false` Host with an error and sets `Snapshot.partial=True`. **Step 2–5:** implement over `vm_transport`, test, commit.

### Task 14: Phase-2 acceptance (MVP)

- [ ] **Step 1:** `uv run vrg-fleet` shows Mac + local Lima work; the mq-resiliency lab sessions attribute correctly (or `ambiguous` per D1). **Step 2:** a stopped Lima instance renders as a fault; `vrg-fleet` exits non-zero. **Step 3:** `vrg-validate` green. **Step 4:** commit. **This is the shippable MVP.**

---

## Phase 3 — Cloud hosts via relay (DROPPABLE)

> Gate on D3. If the relay producer / `vm_cloud` enumeration are not ready, **cut this phase** — file a follow-on task and ship the MVP. Record the cut in `decisions.md`.

### Task 15: Read cloud fleet-state from the relay ref

**Files:** Create `src/vergil_tooling/lib/fleet/relay.py`; Modify `collect.py`; Test `.../test_relay.py`
- [ ] **Step 1:** Failing test with a stubbed relay reader returning a canned per-VM contract fragment; assert cloud sessions/worktrees merge into the `Snapshot` tagged to a `cloud-vm` Host, and that a cloud VM with **no** relay data but present in the registry renders as `reachable:false` (idle-with-pushed-work if GitHub covers it). **Step 2–5:** implement over the #148 relay-ref transport + `vm_cloud` enumeration; test; commit.

---

## Phase 4 — Polish

### Task 16: Scoping flags + live-only view + active-recency threshold

**Files:** Modify `bin/vrg_fleet.py`, `render.py`; Test `test_vrg_fleet.py`
- [ ] **Step 1:** Failing tests: `--org vergil-project` filters; `--host mac` filters; `--live` shows only sessions with `age_seconds <= ACTIVE_WINDOW` (default constant, e.g. 24h) — assert the default and that the flag can override it. **Step 2–5:** implement; test; commit.

### Task 17: Non-zero-exit-on-partial + fail-loud rows

**Files:** Modify `bin/vrg_fleet.py`, `render.py`; Test `test_vrg_fleet.py`
- [ ] **Step 1:** Failing test: a `Snapshot.partial=True` (a faulted host) makes `main(...)` return a non-zero exit code **and** the tree shows an explicit red/fault row for the host; a fully-collected snapshot exits 0. **Step 2–5:** implement; test; commit.

### Task 18: Docs (site docs) — deferred to the epic docs-review bookend

- [ ] **Step 1:** Draft the `vrg-fleet` reference under `docs/site/…` (what it shows, the `--json` contract shape, host attribution, the two-plane context). Note: the **epic's documentation-review task `vergil-tooling#2458` is the final gate** and may spawn sibling doc tasks; this step seeds that surface. **Step 2:** `vrg-validate` (markdownlint) green. **Step 3:** commit.

---

## Self-review — spec coverage map

| Spec requirement | Task(s) |
|---|---|
| Versioned JSON contract | 1 |
| Last-record session title (tail-read) + multi-MB case | 2 |
| Session lineage captured | 2 |
| repo/branch/issue → epic join | 3, 8, 9 |
| Worktree enumeration bounded to `.worktrees/` | 4 |
| Mac collector + tree/`--json` | 5, 6, 7 |
| Batched GraphQL epic-join + cache + offline degrade | 8, 9 |
| Idle-but-open epics / orphan branches shown | 10 |
| GitHub as fallback truth (off host's pushed work visible) | 10 (Steps 6–10) |
| Host attribution (positive signal else `ambiguous`) | 0 (D1), 11 |
| `uuid` de-dup | 12 |
| Local Lima hosts (MVP) | 11–14 |
| Cloud via relay (droppable) | 15 |
| Scoping flags / live-only / recency threshold | 16 |
| No silent failures / non-zero exit on partial | 9, 13, 17 |
| Site docs | 18 (+ epic bookend #2458) |
| Metrics transport (Prometheus vs OTLP) | **Deferred to brick-2 epic — not in this plan (D4)** |

## Implementation-task filing (after alignment)

File these under epic `vergil-project/.github#171`, each in `vergil-tooling`, born linked:
- **T-A** Contract + Mac collector + `vrg-fleet` (Tasks 1–7) — Phase 0.
- **T-B** GitHub correlation + batched join (Tasks 8–10) — Phase 1.
- **T-C** Local Lima hosts + attribution + de-dup (Tasks 0/D1, 11–14) — Phase 2 (MVP ceiling).
- **T-D** *(droppable)* Cloud via relay (Task 15) — Phase 3.
- **T-E** Polish: flags, live view, non-zero-exit (Tasks 16–17) — Phase 4.
- **T-F** *(operational, `validation`)* Live cross-host acceptance: `vrg-fleet` correctly correlates sessions↔worktrees↔epics across Mac + a running Lima VM, with correct titles and `ambiguous`/fault handling — a check the unit suite cannot fully make. Blocked-by T-A…T-C.
- Docs land via the epic's docs-review bookend `vergil-tooling#2458` (Task 18 seeds it).
