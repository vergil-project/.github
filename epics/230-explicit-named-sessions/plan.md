# Explicit, purpose-named sessions — Implementation Plan

> **For epic workers:** These tasks are filed as **GitHub issues under epic
> vergil-project/.github#230** and implemented one PR each via
> `vergil:issue-implement` (its own TDD cycle). Steps use checkbox (`- [ ]`) syntax.
> Code lands in **vergil-project/vergil-tooling** unless noted; docs land via the
> doc-review bookend (`vergil-vm#291`).

**Goal:** Make `vrg-vm` sessions explicitly named by purpose (`label:workspace`) and
reconnectable by exact name, delete the slot/archive machinery, and move the only
unsupported dependency (session enumeration + name→id resolution) behind one seam so
it can ride a verified Agent SDK backend instead of transcript-scraping.

**Architecture:** The exec layer already uses supported Claude flags (`-n`,
`--resume <id>`, `--fork-session`). The unsupported part is *enumeration/resolution*
(`vrg_vm_resolve.name_by_session` scrapes transcripts; `read_roster` reads the
internal roster). We introduce a `SessionStore` seam (`lib/session_store.py`) with a
`ScrapeStore` backend wrapping today's scrape, plus an optional SDK backend gated by a
verification spike. The pure planning in `lib/session.py` is adapted from slot-based to
name-based; the CLI in `bin/vrg_vm.py` gets `--label` (create) and a promoted
`--resume` (attach), and loses `--slot` + auto-resume-most-recent + the archive/staleness
machinery.

**Tech Stack:** Python 3 (stdlib; `claude-agent-sdk` only if Stage 0 verifies it),
pytest (`tests/vergil_tooling/`, existing `test_session.py` / `test_vrg_vm_resolve.py`
idiom). Validation: `vrg-container-run -- vrg-validate`. Git via `vrg-git`; PR handoff
via `report-ready` (humans submit).

## Global Constraints

- **Supported interfaces only in the general code path.** Naming via `-n`/`/rename`;
  resume/fork via `--resume <id>`/`--continue`/`--fork-session`. **Never hand-write
  transcript naming events**; never branch on `agent-name` vs `custom-title` outside the
  seam's scrape backend.
- **Transcript JSONL and `~/.claude/sessions/<pid>.json` are internal/unstable** — only
  the seam's `ScrapeStore` may touch them.
- **Name = `label:workspace`**, colon-delimited; identity dropped. `label` is a clean
  slug; warn (never reject) on a non-`epic-`/`adhoc-` prefix.
- **No auto-resume-most-recent, no `--slot`.** Create = `--label` (must not collide with
  a visible session); attach = `--resume` (must resolve to exactly one visible session).
- **Workspace derived from the resolved session on resume** (memory-slug parity;
  `_project_slug(cwd)`), never a divergent positional.
- **Fail loud** on a name resolving to zero or multiple co-equal live sessions.
- **Never delete a transcript.** `--fresh` retires a name via supported rename, never
  deletion.
- **Portable, shellcheck/hadolint-clean where applicable; `vrg-validate` is the only
  validation entry point.**

**Task dependency graph:**

```text
T0 (SDK spike, BLOCKING) ─gates→ T1 (seam) → T2 (--label create)
                                        └────→ T3 (--resume attach)
T2,T3 → T4 (delete archive + recency list) → T5 (--fresh retire-rename)
T3,T4 → T6 (selection-correctness / #2602)
T4    → T7 (optional cosmetic archived@ strip)
Validation (live-lab) ──Blocked-by── T2,T3,T4,T6  (+ human-gated deploy of new vrg-vm)
```

---

### Task 0 (Stage 0): Agent SDK verification spike — BLOCKING

**Files:**
- Create: `scripts/dev/probe-claude-sessions.py`
- Create: `docs/claude-session-surface.md` (findings + go/no-go)

**Interfaces:**
- Produces: a written **go/no-go** on an SDK backend for the seam, answering: does an
  installed `claude-agent-sdk` expose functions to (a) **enumerate** sessions with
  id/name/cwd/last-activity, (b) **resolve a name → id**, and (c) **see
  interactively-launched sessions** (not just SDK `query()` ones)? Consumed by T1's
  backend choice.

- [ ] **Step 1: Probe the installed surface.** In `probe-claude-sessions.py`: print
  `claude --version`; attempt `import claude_agent_sdk` and enumerate its public
  attributes (`dir()`), recording which of `list_sessions`/`get_session_info`/
  `rename_session`/`tag_session` (or their real names) actually exist. If present, call
  the listing function and print whether the **current interactive session** (this VM's
  live `claude`) appears, with its name/cwd/last-activity fields.
- [ ] **Step 2: Run it in the dev container / VM.**
  `vrg-container-run -- python scripts/dev/probe-claude-sessions.py` (and, because the
  SDK's view of *interactive* sessions is the open question, also run it in a real VM
  session per the issue notes).
- [ ] **Step 3: Record findings + decision** in `docs/claude-session-surface.md`:
  the verified function names (or "absent"), whether name→id resolution is possible,
  whether interactive sessions are visible, and a **GO (SDK backend)** or **NO-GO
  (scrape backend)** verdict for T1. Separate *observed* from *documented*.
- [ ] **Step 4: Validate + commit.** `vrg-container-run -- vrg-validate`;
  `vrg-commit --type docs --scope vrg-vm --message "verify Claude session-management surface; choose seam backend"`.

---

### Task 1 (Stage A): the `SessionStore` seam + `ScrapeStore` backend

**Files:**
- Create: `src/vergil_tooling/lib/session_store.py`
- Create: `tests/vergil_tooling/test_session_store.py`
- Modify: `src/vergil_tooling/bin/vrg_vm_resolve.py` (route enumeration through the seam)

**Interfaces:**
- Produces:
  - `@dataclass(frozen=True) class SessionInfo(session_id: str, name: str | None, cwd: str, active: bool, last_active: float | None)`
  - `class AmbiguousSessionError(Exception)`
  - `class SessionStore(Protocol)` with `list_sessions(self) -> list[SessionInfo]` and
    `resolve_name(self, name: str) -> SessionInfo | None`.
  - `class ScrapeStore` implementing it over the *existing* `name_by_session()` +
    `read_roster()` (the isolated hack). `resolve_name`: filter `list_sessions()` to
    exact `name` matches; **0 → None**; **≥2 active → raise AmbiguousSessionError**;
    else the single active, else the most-recent by `last_active`.
- Consumes: nothing new (wraps existing scrape functions).

- [ ] **Step 1: Write failing tests** with a fake store list:

```python
from vergil_tooling.lib.session_store import SessionInfo, resolve_over, AmbiguousSessionError
import pytest

def _s(sid, name, active, last): return SessionInfo(sid, name, "/w", active, last)

def test_resolve_picks_active_over_idle():
    rows = [_s("a","epic-1:w",False,10.0), _s("b","epic-1:w",True,5.0)]
    assert resolve_over(rows, "epic-1:w").session_id == "b"

def test_resolve_idle_by_most_recent():
    rows = [_s("a","epic-1:w",False,10.0), _s("b","epic-1:w",False,20.0)]
    assert resolve_over(rows, "epic-1:w").session_id == "b"

def test_resolve_none_when_absent():
    assert resolve_over([_s("a","other:w",True,1.0)], "epic-1:w") is None

def test_resolve_raises_on_two_active():
    rows = [_s("a","epic-1:w",True,10.0), _s("b","epic-1:w",True,20.0)]
    with pytest.raises(AmbiguousSessionError):
        resolve_over(rows, "epic-1:w")
```

- [ ] **Step 2: Run, expect FAIL.**
  `vrg-container-run -- pytest tests/vergil_tooling/test_session_store.py -v`
- [ ] **Step 3: Implement** `SessionInfo`, `AmbiguousSessionError`, the pure
  `resolve_over(rows, name)` helper (the rule above), the `SessionStore` Protocol, and
  `ScrapeStore` (whose `resolve_name` = `resolve_over(self.list_sessions(), name)`;
  `list_sessions` adapts `name_by_session()` + `read_roster()` into `SessionInfo` rows).
- [ ] **Step 4: Route enumeration through the seam.** In `vrg_vm_resolve.py`, replace
  direct `name_by_session`/`read_roster` reads used for listing/resolution with a
  `ScrapeStore` instance (leave `_exec_claude` and the supported-flag exec untouched).
- [ ] **Step 5: Run tests, expect PASS** (plus existing `test_vrg_vm_resolve.py` green).
- [ ] **Step 6: Validate + commit.** `vrg-commit --type refactor --scope vrg-vm --message "introduce SessionStore seam (scrape backend) for session enumeration/resolution"`.

---

### Task 2 (Stage A): `--label` — named creation

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (add `--label`; compose `label:workspace`)
- Modify: `src/vergil_tooling/lib/session.py` (name composition + slug validation)
- Modify: `tests/vergil_tooling/test_session.py`

**Interfaces:**
- Produces: `make_label_name(label: str, workspace: str) -> str` → `f"{label}:{workspace}"`;
  `validate_label(label: str) -> list[str]` returning warning strings (empty = clean;
  a non-`epic-`/`adhoc-` prefix yields one warning; a `:`/whitespace/empty label raises
  `ValueError`). Create goes through the seam's uniqueness check then `claude -n <name>`.
- Consumes: `SessionStore.resolve_name` (T1).

- [ ] **Step 1: Write failing tests:**

```python
from vergil_tooling.lib.session import make_label_name, validate_label
import pytest

def test_make_label_name_composes():
    assert make_label_name("epic-213-x","vergil-project/vergil-tooling") == "epic-213-x:vergil-project/vergil-tooling"

def test_validate_label_clean_epic():
    assert validate_label("epic-213-x") == []

def test_validate_label_warns_non_convention():
    assert validate_label("scratch") == [w for w in validate_label("scratch") if "epic-" in w or "adhoc-" in w] and validate_label("scratch")

def test_validate_label_rejects_colon():
    with pytest.raises(ValueError):
        validate_label("bad:name")
```

- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement** `make_label_name` and `validate_label` in `session.py`.
- [ ] **Step 4: Wire `--label` in `vrg_vm.py`:** add `p_session.add_argument("--label")`;
  when set, `validate_label` (print warnings to stderr, don't reject on convention),
  compose `make_label_name(label, workspace)`, call `store.resolve_name(name)` — if it
  returns a **visible** session, **error** ("a session named X already exists; --resume
  it or pick another label"); else exec `claude -n <name>` (the existing `Create` path).
- [ ] **Step 5: Drop identity from the name path** — `Create`/`_execute` use the composed
  `label:workspace`, not `make_name(identity, slot, path)`.
- [ ] **Step 6: Run tests, expect PASS.**
- [ ] **Step 7: Validate + commit.** `vrg-commit --type feat --scope vrg-vm --message "add --label for named session creation (label:workspace); soft-warn convention"`.

---

### Task 3 (Stage A): `--resume` promotion + remove slots/auto-resume

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (remove `--slot`; `--resume` primary; no-arg guide)
- Modify: `src/vergil_tooling/bin/vrg_vm_resolve.py` (resolve name→id via seam; derive workspace)
- Modify: `tests/vergil_tooling/test_vrg_vm_resolve.py`

**Interfaces:**
- Consumes: `SessionStore.resolve_name` (T1). Produces: a resolve path that maps
  `--resume <name>` → `SessionInfo` → `claude --resume <session_id> -n <name>`, with the
  bootstrap **cwd derived from `SessionInfo.cwd`**, not a positional.

- [ ] **Step 1: Write failing test** (fake store) asserting resume resolves name→id and
  errors on absent/ambiguous:

```python
# with a fake SessionStore returning known rows:
def test_resume_resolves_name_to_id(fake_store):
    action = plan_resume(fake_store, "epic-1:w")
    assert action.session_id == "b"        # the active one
def test_resume_errors_when_absent(fake_store_empty):
    with pytest.raises(SystemExit):
        plan_resume(fake_store_empty, "nope:w")
```

- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement** a `plan_resume(store, name)` that calls
  `store.resolve_name(name)`; `None` → error+exit ("no session named X — create it with
  --label"); `AmbiguousSessionError` → error+exit (fail loud); else a `Resume(session_id)`
  whose exec derives cwd from `SessionInfo.cwd`.
- [ ] **Step 4: Remove `--slot`** and the auto-resume-most-recent default; make the
  **no-arg** `vrg-vm session <workspace>` print the recency `list` + the two verbs
  (`--label` / `--resume`) and exit non-zero (list-and-guide).
- [ ] **Step 5: Derive workspace on resume** — do not require/allow a workspace that
  diverges from the resolved session; the cwd for the memory slug comes from
  `SessionInfo.cwd`.
- [ ] **Step 6: Run tests, expect PASS.**
- [ ] **Step 7: Validate + commit.** `vrg-commit --type feat --scope vrg-vm --message "promote --resume to exact-name attach via seam; remove --slot and auto-resume-most-recent"`.

---

### Task 4 (Stage B): delete archive; recency display-filter

**Files:**
- Modify: `src/vergil_tooling/lib/session.py` (remove archive/staleness types + `make_archived_name`/`parse_archived`/`classify_age`)
- Modify: `src/vergil_tooling/bin/vrg_vm_resolve.py` (remove `_archive_session`, `_run_sweep`, `_prompt_stale`, `PromptStale`)
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`list` flags; `session_recent_days`)
- Modify: `tests/vergil_tooling/test_session.py`, `test_vrg_vm_resolve.py`

**Interfaces:**
- Produces: a `list --sessions` that filters `SessionStore.list_sessions()` to
  `last_active` within `session_recent_days` by default; `--all` returns everything.
  `--active`/`--idle` retained; `--archived` removed.

- [ ] **Step 1: Write failing test** for the recency filter (pure function over
  `SessionInfo` rows + a `now` + `recent_days`), asserting old rows are excluded unless
  `all=True`, and active rows always included.
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement** `filter_recent(rows, now, recent_days, all=False)` and wire it
  into the `list` command; add `session_recent_days` config (cascading like the old
  thresholds); remove the `--archived` flag; re-point `--all`.
- [ ] **Step 4: Delete the archive/staleness machinery** — `_archived_prefix`,
  `make_archived_name`, `parse_archived`, `classify_age`, `AgeBand`, `PromptStale`,
  `_archive_session`, `_run_sweep`, `_prompt_stale`, and the `session_stale_days`/
  `session_archive_days` config — and every reference. (Legacy `archived@` *names* remain
  opaque strings the seam still lists/resolves; only the *behavior* is removed.)
- [ ] **Step 5: Run tests, expect PASS** (update tests that asserted archive behavior to
  assert its absence).
- [ ] **Step 6: Validate + commit.** `vrg-commit --type feat --scope vrg-vm --message "delete session archive + staleness; add recency display-filter"`.

---

### Task 5 (Stage B): `--fresh` = supported retire-rename

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_vm_resolve.py`, `bin/vrg_vm.py`
- Modify: `tests/vergil_tooling/test_vrg_vm_resolve.py`

**Interfaces:**
- Produces: `--fresh --label <name>` renames the prior visible session of that name to a
  retired suffix (`<name>~<datestamp>`) **via the supported rename path**, then creates a
  new session with the clean `name`. `retired_name(name, stamp) -> f"{name}~{stamp}"`.

- [ ] **Step 1: Write failing test** for `retired_name` and for the plan: given a visible
  session named `epic-1:w`, `--fresh` yields (rename-old-to-`epic-1:w~<stamp>`, then
  create `epic-1:w`).
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement** `retired_name` + the `--fresh` plan. The rename uses the
  supported mechanism the T0 spike selected (SDK `rename` if GO; otherwise Claude's
  `/rename` semantics through the seam) — **not** a raw transcript write. Timestamp is
  passed in (no `Date.now()` in pure logic; the CLI supplies it).
- [ ] **Step 4: Run tests, expect PASS.**
- [ ] **Step 5: Validate + commit.** `vrg-commit --type feat --scope vrg-vm --message "reconceive --fresh as supported retire-rename (never delete)"`.

---

### Task 6 (Stage C): selection-correctness regression (#2602)

**Files:**
- Modify: `tests/vergil_tooling/test_vrg_vm_resolve.py`
- Modify: source only if a defect surfaces

**Interfaces:** none new — proves the requirement.

- [ ] **Step 1: Write a regression test** encoding #2602: given rows including a
  legacy/`archived@`-named session and a requested exact name, `plan_resume` selects the
  **requested** session (never the archived one), and the exec re-asserts that name via
  `-n` (title equals requested name). Also assert the no-arg path never auto-selects.
- [ ] **Step 2: Run** — expect PASS if Stages A/B already made selection explicit;
  if it fails, fix the offending path (the point of the test is to prove selection is
  structurally explicit).
- [ ] **Step 3: Validate + commit.** `vrg-commit --type test --scope vrg-vm --message "regression: reconnect selects only the exactly-named session (#2602)"`.

---

### Task 7 (Stage D): optional cosmetic `archived@` strip

**Files:**
- Create: `scripts/dev/strip-archived-markers.py`
- Modify: `docs/` note as needed

**Interfaces:** one-time, idempotent; renames any session whose current name begins
`archived@<ts>@<orig>` back to `<orig>` **via the supported rename**, so the list renders
uniformly. Purely cosmetic; never deletes.

- [ ] **Step 1: Write the script** using the seam + supported rename; dry-run by default,
  `--apply` to execute; print each rename.
- [ ] **Step 2: Verify** on a copy/dry-run that only `archived@`-prefixed names are
  touched and originals are restored; nothing deleted.
- [ ] **Step 3: Validate + commit.** `vrg-commit --type chore --scope vrg-vm --message "one-time cosmetic strip of legacy archived@ markers"`.

---

### Operational task (filed at step 9): live-lab validation

`--kind validation`, `Blocked-by` T2, T3, T4, T6 **and** a human-gated deploy of the new
`vrg-vm` (release + `uv tool install`). Runs the §8 checks on a real VM: named create,
exact-name resume with correct cwd/memory slug, typo/collision errors, no auto-archive,
recency list + `--all`, `--fresh` retire, legacy-name reconnect, and that the seam backend
matches T0's decision. Records `Outcome: SUCCESS`.

## Self-Review

**Spec coverage:** §3 seam + §6 Stage 0 → T0/T1; §4.1 name → T2; §4.2 create/resume +
workspace-derived → T2/T3; §4.3 uniqueness/fail-loud + slot removal → T1/T3; §4.4 archive
delete + recency → T4; §4.5 `--fresh`/`--fork`/rename → T5 (fork already supported in
exec); §4.6 migration → T4 (opaque legacy) + T7 (cosmetic); §4.7 selection → T6; §8
validation → operational task. Fork guardrail (no-double-attach) is existing behavior,
untouched. No spec requirement is unmapped.

**Placeholder scan:** none — pure-logic tasks carry real tests/signatures; CLI/exec tasks
carry concrete wiring + fake-store tests. The one *deliberately* deferred choice (SDK vs
scrape backend) is isolated in T0 and consumed by T1, not a placeholder.

**Type consistency:** `SessionInfo(session_id,name,cwd,active,last_active)` and
`resolve_over`/`SessionStore.resolve_name`/`AmbiguousSessionError` are used consistently
across T1/T3/T4; `make_label_name`/`validate_label` (T2), `retired_name` (T5), and
`filter_recent` (T4) match their definitions.
