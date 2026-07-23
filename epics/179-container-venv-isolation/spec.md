# Isolate the container venv off the workspace mount

- **Epic:** vergil-project/.github#179
- **Status:** Design drafted (2026-07-23) via superpowers brainstorming; hardened
  via paad pushback the same day (design pivoted from an env-var guard to a
  structural volume overlay — see §4). Pending human review.
- **Repo:** vergil-project/vergil-tooling (member); epic homed in `.github`.
- **Tracking issue:** vergil-project/vergil-tooling#2473 — "vrg-container-run
  corrupts the host venv" (Critical). Surfaced during
  `logical-minds-foundry/mq-resiliency-lab-for-linux` epic #122 T4 cold rebuilds.
- **Brainstorm source:** superpowers brainstorming session, 2026-07-23.

## 1. Summary

`vrg-container-run` bind-mounts the whole repo — **including `.venv`** — into the
dev container at `/workspace`, and the container's `uv sync` rewrites that shared
venv with container-only interpreter symlinks and shebangs. When the container
exits, the host's venv is corrupted, so every host-side `uv run` / console script
fails with cryptic errors (`bad interpreter`, exit 127, `No such file or
directory: …/.venv…`). This has been the hidden variable behind roughly two weeks
of intermittent "lab reliability" failures and is currently **blocking cold
rebuilds in the lab**.

This epic mounts a **fresh, empty anonymous volume over `/workspace/.venv`** inside
the container. The volume physically masks the host's bind-mounted venv, so the
container cannot see or touch it; in-container `uv` builds a throwaway venv on the
volume using the same default `.venv` path. This removes the shared-venv coupling
entirely and lets us **retire the `.venv-host` dual-venv model**. A fleet-wide
read-only sweep catalogs stale references so the change does not leave dangling
documentation across the org.

**Core principle: a host and a container never share a venv.** We enforce it
*structurally* — by making the default `.venv` path safe — not *behaviorally*, so
correctness does not depend on catching every code path that could target `.venv`.

## 2. Problem / motivation

The corruption is deterministic and confirmed (issue #2473):

1. `vrg-container-run` mounts `{repo_root}:/workspace`
   (`vergil_tooling/lib/container.py`), so `/workspace/.venv` **is** the host's
   `.venv`.
2. The Python install stage runs `uv sync --frozen --group dev` inside the
   container (`vergil_tooling/lib/languages.py:119`). With no isolation, uv targets
   the default `.venv` — i.e. the bind-mounted host venv — and **removes and
   recreates it** with container-only paths:
   - interpreter symlink → `/root/.local/share/uv/python/cpython-3.12-…/python3.12`
   - console-script shebangs → `#!/workspace/.venv/bin/python`
3. The container exits; on the host both paths are invalid, so host-side `uv run`
   and console scripts break.

These errors masquerade as unrelated lab/ansible bugs, which is why they were so
expensive to chase. Each was previously "fixed" with a workaround (`uv sync`,
retry, venv rebuild) rather than the root cause.

## 3. Root cause

**One venv used in two places** — the host and the container — with no isolation.
The container writes container-valid paths into the shared, bind-mounted venv,
corrupting it for the host.

`container.py` already pinned `UV_LINK_MODE=copy` to suppress the hardlink warning
that arises because the venv and the uv cache live on different filesystems
(#2461) — but that addressed only the cosmetic warning, not the corruption. That
pin **stays** under this design (see §4.3).

The `.venv-host` dual-venv model was a deliberate but partial workaround: it gave
vergil-tooling's **own** dev tree a separate host venv (`.venv-host`) so container
`uv sync` (targeting `.venv`) would not overwrite it. It never helped **consuming**
repos — which have no such split — and it never addressed the underlying "shared
venv" defect. Once the container can no longer touch any in-workspace venv, the
reason for `.venv-host` evaporates.

## 4. Design

### 4.1 The structural barrier — an anonymous volume over `.venv`

In `build_container_args` (`container.py`), when the repo is a Python project
(`detect_language(repo_root) == "python"`), add an **anonymous volume** mounted
over the workspace venv path:

```
-v /workspace/.venv          # anonymous volume: no host source, container path only
```

- A `-v <container-path>` with no host source creates a **fresh, empty anonymous
  volume**. Mounted at `/workspace/.venv`, it **masks** the host's bind-mounted
  `.venv` for the life of the container — the container sees an empty directory
  there, never the host's files. This is the standard "mask a subdirectory of a
  bind mount with an empty volume" pattern (the `node_modules` trick).
- The container's `uv sync` uses the **default** `.venv` path (`/workspace/.venv`)
  and writes into the volume. **No `UV_PROJECT_ENVIRONMENT`, no env-var
  propagation, no hunting down uv call sites** — every present and future code
  path that targets the default `.venv` is automatically isolated. That is the
  point: the guarantee is structural, so nothing can be "missed."
- The host's real `/workspace/.venv` on disk is never exposed inside the container
  and is left byte-for-byte untouched.
- **Gate to Python.** Non-Python containers (Ruby/Go/Rust/Java) have no `.venv`
  corruption vector; masking a nonexistent path would only create a stray empty
  `.venv` in their workspace view, so the volume is added only for Python repos.

Why an env-var guard was rejected (it was the first-draft design): setting
`UV_PROJECT_ENVIRONMENT=/root/.venv` leaves the host venv **visible** at
`/workspace/.venv` inside the container, so any missed reference — or any tool that
bootstraps the default venv — would silently corrupt the host again. It is a
fail-open, convention-based guard. The volume overlay is fail-safe.

### 4.2 Keep `vrg-validate`'s PATH-add correct — `bin/vrg_validate.py:154`

`vrg-validate` prepends the project venv's `bin` to `PATH` so the bare check
commands in the registry (`ruff`, `mypy`, `ty`, `pytest`, `pip-audit`,
`pip-licenses` — invoked by name, not via `uv run`) resolve. Today it guards the
prepend with `if venv_bin.is_dir()`, and it runs at startup, **before** the INSTALL
stage that creates the venv.

That guard is now wrong. The masking volume is **empty at container startup**, so
`/workspace/.venv/bin` does not exist when validate runs — the guard would fail,
the PATH-add would be skipped, and every bare check command would become "command
not found." (Today it only works by accident: the bind-mounted host venv already
exists at startup.)

**Fix:** drop the `is_dir()` guard and prepend `.venv/bin` **unconditionally**. PATH
is resolved dynamically at exec time, so a directory that does not exist when PATH
is set but does exist once INSTALL runs resolves correctly. A missing directory on
PATH is harmless — not worth guarding. The venv path itself is unchanged
(`Path.cwd() / ".venv" / "bin"`); only the guard is removed.

### 4.3 `UV_LINK_MODE=copy` pin stays — `container.py`

The anonymous volume lives on Docker's volume storage, a different filesystem from
the uv cache (`/root/.cache/uv`, on the container overlay fs), so uv still cannot
hardlink and falls back to a copy. The existing `UV_LINK_MODE=copy` pin (#2461)
therefore remains correct and stays, silencing the warning. (An operator override
continues to win.) We forgo the hardlink speedup the pin's removal would have
allowed under an overlay-local venv — an acceptable trade for the stronger,
structural isolation.

### 4.4 Excise the `.venv-host` model

Once the container can no longer touch an in-workspace venv, vergil-tooling's host
dev tree uses plain `.venv` like every consumer. Remove the dual-venv concept:

- **`CLAUDE.md`** "Environment Setup": collapse the `.venv` vs `.venv-host`
  explanation to a plain `uv sync --group dev` against `.venv`.
- **`repo_init` `.gitignore` template**: drop the `.venv-host/` line injected into
  consuming repos; update the corresponding test
  (`tests/vergil_tooling/test_repo_init.py`, which asserts `.venv-host/`).
- **validate skip-dirs**: remove `.venv-host` from the skip list
  (`tests/vergil_tooling/test_validate_common.py:422` and its source constant).
- **docs**: remove the "Why `.venv-host`" rationale from
  `docs/specs/host-level-tool.md` and any live site docs describing the dual-venv
  model (bulk handled by the documentation-review gate, task #2477).

Historical plan/spec/release documents that merely record past decisions are left
as-is (they are dated records, not live guidance).

## 5. Migration

The change is forward-only. A host or lab whose `.venv` was **already corrupted**
before the fix needs a one-time reset:

```bash
rm -rf .venv && uv sync
```

This goes in the release notes accompanying the deployment (§7.2). No automated
migration is warranted — the corruption is self-evident once host tooling fails,
and the reset is a single command.

## 6. Where the fix runs, and why deployment matters

`build_container_args` runs **host-side** (it constructs the container `run`
arguments). So the fix ships in the **host** `vrg-container-run`. A lab host only
benefits after a vergil-tooling **release** and a host `uv tool install` upgrade.
This is why the epic carries an explicit deployment task (§7.2): merging to
develop is necessary but not sufficient to unblock the lab.

## 7. Task breakdown

Bookend and research tasks are seeded at epic creation; implementation and
operational tasks are filed from the plan so their `Blocked-by` links resolve.

### 7.1 Implementation

- **T2 — core isolation fix** (`vergil-project/vergil-tooling`): add the anonymous
  `.venv` masking volume for Python repos in `container.py`; drop the `is_dir()`
  guard in `vrg_validate.py` (prepend unconditionally); keep the
  `UV_LINK_MODE=copy` pin; tests. One PR.
- **T3 — excise `.venv-host`** (`vergil-project/vergil-tooling`): CLAUDE.md,
  `repo_init` `.gitignore` template, validate skip-dirs, supporting docs; tests.
  One PR. Sequenced after T2 (only safe once the container can no longer touch an
  in-workspace venv).

### 7.2 Operational

- **T4 — validation (cold rebuild)** (`vergil-project/vergil-tooling`,
  `--kind validation`, blocked-by T2, T3): prove the host `.venv` survives a
  container run. **Acceptance (observable on the host):** capture the host
  `.venv/bin/python` symlink target and one console-script shebang; run
  `vrg-container-run -- vrg-validate`; assert both are **byte-for-byte unchanged**
  afterward, and that host `uv run` / console scripts still work. Also verify
  `--rm` reclaims the anonymous volume on **both** `docker` and `nerdctl` (no
  volume leak per run; tmpfs is the fallback if a runtime does not — see §9).
  Closes only on `Outcome: SUCCESS`.
- **T5 — deployment (release + host reinstall)**
  (`vergil-project/vergil-tooling`, `--kind deployment`, blocked-by T2, T3, T4):
  make the fix usable on lab hosts. The **release itself is a human-gated
  precondition** (bump/tag/publish is never performed by the agent); the
  deployment task owns the agent-safe host `uv tool install` upgrade and records
  the deployed version. Closes only on `Outcome: SUCCESS`.

### 7.3 Research

- **T6 — fleet reference sweep** (`vergil-project/.github#182`): read-only sweep
  across all orgs for stale venv / dual-venv / host-vs-container venv setup
  references; deliverable is a catalog PR into this epic's docs dir
  (`reference-sweep.md`). Scope is **venv only** — Lima agent-VM setup docs are
  out of scope and filed separately if found. Remediation splits by org: in-org
  hits become doc tasks under #179 (folded into the documentation-review gate);
  other-org hits are handed off via triage (cross-org linkage is out of scope).

### 7.4 Bookends

- **T1 — documentation** (`vergil-project/.github#180`): publishes this
  `spec.md` + `plan.md`.
- **Follow-on brainstorm** (`vergil-project/.github#181`): review outcomes,
  brainstorm follow-on epic(s) — including whether a persistent container
  uv-cache/venv volume is worth adding for install speed (deferred here).
- **Documentation review** (`vergil-project/vergil-tooling#2477`): the closing
  gate; sweep human-facing docs (especially versioned site docs) and spawn
  per-repo doc tasks.

## 8. Out of scope

- **Persistent container venv/cache volume** for install speed. Each container run
  builds a fresh venv on a throwaway anonymous volume; the uv cache already makes
  this cheap. A persistent (named) volume is a possible follow-on (§7.4), not part
  of this epic.
- **Lima agent-VM setup docs** — a distinct concern from the Python venv.
- **How consuming repos declare or install vergil-tooling** — unchanged.

## 9. Risks and edge cases

- **Anonymous-volume cleanup.** `docker run --rm` reclaims a container's anonymous
  volumes on exit; T4 must confirm `nerdctl --rm` does the same. If a runtime does
  not, mount a **tmpfs** over `/workspace/.venv` instead (`--mount
  type=tmpfs,dst=/workspace/.venv`), which is always fresh and needs no cleanup, at
  the cost of holding the venv in RAM. Anonymous volume is the default because a
  built venv can be large.
- **Non-Python repos.** The masking volume is gated to Python projects, so
  Ruby/Go/Rust/Java containers are unaffected and gain no stray `.venv`.
- **Hardlink fallback.** Retained by design (§4.3); `UV_LINK_MODE=copy` keeps the
  install correct and quiet on the cross-filesystem copy.
- **Stale references left behind.** Addressed by the fleet sweep (T6) and the
  documentation-review gate — the change does not silently orphan docs.
- **Transition window.** Hosts corrupted before the fix still need the one-time
  reset (§5); documented in release notes so it is not a silent surprise.

## 10. Testing

- Unit: `build_container_args` adds `-v /workspace/.venv` (anonymous volume) for a
  Python repo and **omits** it for non-Python repos.
- Unit: the `UV_LINK_MODE=copy` default and its operator override are retained.
- Unit: `vrg_validate`'s PATH-add prepends `.venv/bin` **unconditionally** (guard
  removed) — verified to prepend even when the directory does not yet exist.
- Unit: `repo_init`'s `.gitignore` template no longer emits `.venv-host/`; skip
  lists no longer reference it.
- Validation (T4): the live cold-rebuild reproduction — the definitive proof,
  including the anonymous-volume-cleanup check across runtimes.
- Full gate: `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate`
  here), run from inside each task's worktree.
