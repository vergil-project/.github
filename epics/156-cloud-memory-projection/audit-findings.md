# Copied-config and runtime-writer audit findings

**Epic:** vergil-project/.github#156 · **Task:** vergil-project/.github#161
(plan Task 1) · **Spec:** `epics/156-cloud-memory-projection/spec.md`

**Date:** 2026-07-16

This is the gating investigation for the cloud-memory-projection epic. It
establishes, empirically from the `vergil-tooling` source, exactly what
`vrg-vm` copies into a cloud guest, which of those files a writer (provisioning
or the Claude Code runtime) mutates, and whether any copied file embeds
host-absolute `/Users/` literals. Its outputs — the **locked set** and the
**Component 2b decisions** — are consumed by the memory-sync and read-only-lock
tasks (#2411 / #2412) and by the surgical-lock task in the plan (Tasks 4–5).

Source inspected (sibling checkout `vergil-tooling`, as of this audit):

- `src/vergil_tooling/lib/vm_guest.py` — `copy_claude_config` (line 427),
  `inject_credentials` (line 55), `update_plugins` (line 339),
  `install_tooling` (line 228).
- `src/vergil_tooling/lib/vm_cloud.py` — `link_cloud_claude_dirs` (line 531),
  `bootstrap_volume`.
- `src/vergil_tooling/bin/vrg_vm.py` — `_cloud_session` (line 2396) and the
  cloud-create stages (`_cs_credentials`, `_cs_tooling`, `_cs_link_claude`,
  `_cs_bootstrap_volume`, lines 842–867).

## The governing rule (restated)

A file is a **lock candidate only if neither our provisioning nor the Claude
Code runtime writes it during a session** (spec §Governing rule). "Written by
the harness at runtime" is an explicit disqualifier: locking a harness-written
file would break the session with `EACCES`. Harness-/provisioning-written files
stay writable — and they are ephemeral/non-canonical anyway, so that is
consistent.

## Step 1 — Everything `vrg-vm` copies into a cloud guest

Two write paths reach a cloud guest, on different triggers:

- **Cloud create/rebuild** (`_cloud_create_stages`, run once per box):
  `inject_credentials` → `install_tooling` → `bootstrap_volume` →
  `copy_claude_config` + `link_cloud_claude_dirs` (via `_cs_link_claude`) →
  `update_plugins`.
- **Every cloud session** (`_cloud_session`, run on each `vrg-vm session`): only
  `copy_claude_config` + `link_cloud_claude_dirs` re-run (a self-healing re-seed;
  idempotent). Credential injection, tooling install and plugin reconcile do
  **not** re-run per session.

Files/objects actually written into the guest:

| Path in guest | Written by | Trigger | Kind |
|---|---|---|---|
| `~/.claude/CLAUDE.md` | `copy_claude_config` (verbatim host copy) | create **and** every session | canonical config |
| `~/.claude/settings.json` | `copy_claude_config` (verbatim host copy) | create **and** every session | config (harness-mutated) |
| `~/.config/vergil/app.pem` | `inject_credentials` (`chmod 600`) | create only | credential |
| `~/.config/vergil/app.env` | `inject_credentials` (`chmod 600`) | create only | credential/config |
| `~/.config/vergil/identity-mode` | `inject_credentials` (`chmod 600`) + `~/.bashrc` line | create only | config |
| `~/.config/vergil/claude.env` | `inject_credentials` (`chmod 600`) + `~/.bashrc` line | create only | credential |
| `~/.config/vergil/tooling-tag` | `install_tooling` | create (and on `update`) | marker |
| `~/.claude/.credentials.json` | `inject_credentials` (`chmod 600`) | create only | credential |
| `~/.claude.json` | `inject_credentials` (onboarding seed) | create only, then harness-owned | harness state |
| git `user.name` / `user.email` / HTTPS rewrite | `inject_credentials` → `git config --global` | create only | git config |
| `~/.claude/{projects,todos,plugins}` | `link_cloud_claude_dirs` — **symlinks** onto `/vergil/claude/...`, not copies | create and every session | volume link (not a copied file) |
| `~/.claude/plugins/*` | `update_plugins` — materialized in-guest from GitHub marketplaces (**not** host-copied) | create/update | plugin cache (on volume via the symlink) |

**Net for the projection question:** the only files `copy_claude_config`
*copies* from the host `~/.claude` are **`CLAUDE.md`** and **`settings.json`**
(`_CLAUDE_CONFIG_FILES` in `vm_guest.py`). The per-repo `memory/` directory and
`MEMORY.md` are **not copied today** — projecting them is the net-new logic of
this epic (spec §Component 3). Everything else above is a credential, a marker,
a git setting, a harness state file, or a volume symlink — none is canonical
memory/config, and the credentials deliberately live on the ephemeral boot disk
and die with the VM (the standing "no injected credential on the persistent
volume" acceptance).

## Step 2 — Writer classification and the locked set

For each file, "provisioning-writes?" = does any `vrg-vm` provisioning step
mutate it after the initial copy; "runtime-writes?" = does the Claude Code
harness write it during an interactive session.

| File | provisioning-writes | runtime-writes | Lockable? |
|---|---|---|---|
| `~/.claude/CLAUDE.md` | No — copied verbatim, nothing rewrites it | No — user-authored config; the harness never writes it | **YES — locked** |
| per-repo `memory/` (net-new projection) | No — projected verbatim from host | No — canonical data, only the host writes it | **YES — locked** |
| `memory/MEMORY.md` (net-new projection) | No — projected verbatim from host | No — canonical data, only the host writes it | **YES — locked** |
| `~/.claude/settings.json` | **Yes** — `update_plugins` runs `claude plugin marketplace add` / `claude plugin install --scope user`, which the `claude` CLI writes back into `settings.json` (`enabledPlugins`, `extraKnownMarketplaces`) | **Yes** — the harness rewrites `settings.json` on in-session plugin/MCP/model/permission changes (`/plugin`, `/model`, MCP edits) | **NO — must stay writable** |
| `~/.claude.json` | seeded once at create | **Yes** — harness writes it continuously (project history, onboarding, state) | NO — not a canonical file; harness-owned |
| `~/.claude/.credentials.json`, `~/.config/vergil/*`, `tooling-tag` | provisioning-owned | n/a | NO — credentials/markers, not canonical memory; ephemeral by design |

### The `settings.json` decision (the one the spec flagged as conditional)

The spec left `settings.json` conditional: "locked *only if* the audit confirms
neither provisioning nor the harness writes it at runtime." **The audit
disqualifies it on both counts:**

1. **Provisioning writes it.** `update_plugins` (`vm_guest.py:339`) reconciles
   plugins by invoking `claude plugin marketplace add <source>` and
   `claude plugin install/update <id> --scope user`. The host `settings.json`
   confirms it carries `enabledPlugins` and `extraKnownMarketplaces` (the very
   keys `_read_guest_settings` / `_enabled_plugin_ids` / `_marketplace_sources`
   read). Those `claude plugin` subcommands write the enabled/known-marketplace
   state back into `~/.claude/settings.json`. If `settings.json` were read-only,
   the plugin reconcile would fail with `EACCES`.
2. **The runtime writes it.** In an interactive session, enabling/disabling a
   plugin (`/plugin`), changing the model (`/model`), or editing MCP servers
   mutates `~/.claude/settings.json`. Locking it would break those flows.

Therefore **`settings.json` is excluded from the locked set** and stays
writable. This matches the spec's expectation that plugin state is
harness-owned and that plugin installs materialize under `~/.claude/plugins/`
(a separate directory, symlinked to the volume) rather than being locked.

### Locked set (the typed output consumed by #2411 / #2412)

```text
locked_set = {
    "~/.claude/CLAUDE.md",                          # global canonical config
    "~/.claude/projects/<host-slug>/memory/",       # per-repo memory subtree (recursive)
    "~/.claude/projects/<host-slug>/memory/MEMORY.md",
}
unlocked (explicitly excluded):
    "~/.claude/settings.json"     # provisioning- AND runtime-written (claude plugin CLI + session)
    "~/.claude.json"              # harness-written runtime state
    credentials + markers         # ephemeral, non-canonical
```

**Locking must be surgical** (spec §Component 3): `~/.claude/projects/<slug>/`
holds both the durable `memory/` subtree *and* the session-transcript `.jsonl`
files the harness writes continuously. Confirmed on the host: the slug directory
co-mingles `memory/` (and `MEMORY.md`) with many live `*.jsonl` transcripts. A
blanket recursive `chmod` of `projects/<slug>/` would break session logging with
`EACCES`. Lock only `memory/` (recursive) + `MEMORY.md` + `CLAUDE.md`; leave the
transcripts and `settings.json` writable.

## Step 3 — Host-absolute (`/Users/`) literals in copied config

Scanned the host `~/.claude/CLAUDE.md` and `~/.claude/settings.json`:

- **`~/.claude/CLAUDE.md` — no `/Users/` literals.** No Component 2b action; the
  file is byte-portable and lockable as-is. (This is the global "Global agent
  instructions" file; the memory-directory reference in it is relative, not an
  absolute `/Users/` path.)
- **`~/.claude/settings.json` — no `/Users/` literals.** No `/Users/` rewrite
  needed.

**Component 2b decision:** **no rewrites required** for the locked set —
`CLAUDE.md` carries no host-absolute literals and is safe to project and lock
verbatim. The memory files are per-repo canonical text and are projected under
the aligned host slug (Component 2a), so they need no literal rewriting either.

**Secondary observation (not a 2b blocker, not in the locked set).**
`settings.json` does carry two host-absolute, macOS-specific paths in its
`statusLine.command` — `/opt/homebrew/bin/node` and a `$HOME/.claude/plugins/...`
glob (the `$HOME` part is guest-correct). On the Ubuntu guest,
`/opt/homebrew/bin/node` does not exist, so the status line is best-effort and
degrades to nothing. This is **pre-existing behavior**, unrelated to memory
projection, and it reinforces the Step-2 decision to leave `settings.json`
**unlocked/writable** rather than lock a file that (a) the harness rewrites and
(b) already contains host-specific paths. No action for this epic; flagged so
the doc-review bookend (#2405) and any future statusLine-portability work have
the finding on record. Because these are not `/Users/` literals and the file is
not locked, they trigger no Component 2b rewrite.

## Step 4 — Existing-cloud-volume divergent-memory check (DEFERRED)

Spec §Error handling ("Already-lost writes") calls for auditing a current cloud
volume for `memory/` content that diverges from the host canonical — evidence of
earlier futile writes — before the first lockdown, so nothing already written is
discarded unseen.

**This check requires a LIVE cloud volume and is deferred.** This audit session
has **no reachable `/vergil` mount** and **no reachable `/vergil/claude/projects`**
(verified: neither path exists in this environment), so there is no cloud volume
to inspect from here. Per the task's guidance, this does **not** block the audit:
the check is a runtime/live-environment residual, not a code fact.

**Disposition:** run the divergent-memory check as part of the **cold-rebuild
validation task, vergil-project/vergil-tooling#2406**, which by definition
operates on a live, freshly-rebuilt cloud VM with the `/vergil` volume attached.
When performed there, before read-only is first imposed:

1. On the cloud box, enumerate `/vergil/claude/projects/*/memory/` (and any
   `MEMORY.md`).
2. Diff each against the host canonical at the matching slug.
3. Surface any divergence (an out-of-band cloud write that never made it back to
   the host) so it is preserved/triaged, not silently overwritten by the first
   projection or frozen by the first lock.

Because no cloud volume was reachable here, this audit records **no divergent
memory found** only in the trivial sense that **none could be inspected**; the
substantive finding is the deferral above. #2406 owns the live result.

## Summary of decisions (inputs to Tasks 4–5 / #2411–#2412)

- **Locked set:** `~/.claude/CLAUDE.md`, per-repo `memory/` (recursive),
  `memory/MEMORY.md`. Surgical — never blanket-`chmod` `projects/<slug>/`.
- **Excluded from lock:** `settings.json` (provisioning- **and**
  runtime-written), `~/.claude.json` (harness runtime state), and all injected
  credentials/markers (ephemeral, non-canonical).
- **Component 2b:** no `/Users/` literals in `CLAUDE.md` or `settings.json`; **no
  rewrites required**. Noted (non-blocking): `settings.json` carries a macOS
  `/opt/homebrew/bin/node` statusLine path — one more reason it stays unlocked.
- **Already-lost-writes check:** **deferred to the cold-rebuild validation
  (#2406)** — it needs a live cloud volume, unavailable in this audit
  environment.
