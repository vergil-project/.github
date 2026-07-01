# Claude config & plugin delivery for agent VMs

- **Epic:** vergil-project/.github#69
- **Sibling epic:** vergil-project/.github#51 (off-platform cloud VM backend)
- **Status:** Approved design (2026-07-01)
- **Diagnosed live on:** GCP off-platform box `vrg-77241e94d6f9` (rebuilt 2026-06-30)

## 1. Summary

Deliver the operator's Claude configuration — permission mode
(`bypassPermissions`), `settings.json`, and the enabled plugin/marketplace set —
**reliably and durably** into Vergil agent VMs (local Lima and off-platform
cloud), so a freshly-rebuilt cloud box is immediately usable for autonomous work
with no manual repair.

A Vergil VM is a personal analog of the operator's laptop, keyed to identity;
mirroring subsets of the operator's `~/.claude` is the intended model. This epic
is the *config / permission / plugin delivery layer*; sibling #51 is the backend
*infrastructure* it rides on. One fix (C) is backend-agnostic and also corrects
Lima.

## 2. Background — current model

- **Local (Lima):** host `~/.claude` subsets are shared via symlinks
  (`projects`, `skills`); `settings.json` + `CLAUDE.md` are copied into the VM on
  **every session** (`_cmd_session` → `copy_claude_config` + `link_claude_dirs`).
  This self-heals.
- **Cloud (off-platform):** no shared filesystem. A persistent `/vergil` volume
  exists; only `projects`/`todos` are symlinked onto it. `settings.json` and
  `plugins/` live on the **ephemeral boot disk**. Config is seeded only at
  `create`/`rebuild` (`_cs_link_claude`).
- **Marketplace model (#45 / #1974):** single released channel `main`; one ref
  per marketplace (shallow clone). Prior bypass work #1825/#1826 added a config
  copy to the cloud *create* path.

## 3. Evidence (read-only diagnosis of the live box)

- `~/.claude/settings.json` and `CLAUDE.md` **absent** → no
  `permissions.defaultMode: bypassPermissions` → the box prompts for permission
  on everything.
- `~/.claude/projects` is a **real directory on the ephemeral disk** (today's
  transcripts); `~/.claude/todos` missing; `/vergil/claude/projects` mtime stale
  → session history is **not** landing on the volume and is lost on rebuild.
- `~/.claude/plugins/installed_plugins.json` = `{"plugins": {}}` → **zero
  plugins installed**, though `vergil-marketplace` + `claude-plugins-official`
  are cloned. `paad`/`diogenes`/`claude-hud` are not even registered.
- Manual re-seed (copy `settings.json`+`CLAUDE.md`, merge + relink
  `projects`/`todos`) **restored bypass and persistence** → the copy/link
  mechanism works when it is actually invoked.

## 4. Root causes

1. **No self-heal on cloud.** `_cloud_session` launches Claude without
   `copy_claude_config`/link (Lima's `_cmd_session` does both). Create-time
   seeding is the only delivery path; nothing repairs it afterward.
2. **Create/rebuild seeding did not persist** on the diagnosed rebuild — the
   `.credentials.json` from that rebuild survived while every `link-claude`
   effect was absent. Exact mechanism unconfirmed; Fix A makes it operationally
   moot, but it must be instrumented on a real rebuild.
3. **Plugins never installed.** `update_plugins` runs `claude plugin marketplace
   update` + `claude plugin update <enabled>` only — never `plugin install` —
   and post-v2.1.195 nothing auto-installs. Host-only marketplaces
   (`paad`/`diogenes`) are never delivered.
4. **Durable config on the ephemeral disk.** `settings.json` and `plugins/` are
   boot-disk state → wiped on every rebuild (acute given constant cold-boot
   testing of the lab).
5. **`link_cloud_claude_dirs` footgun.** `ln -sfn` creates a nested broken
   symlink when a real dir already exists, and performs no merge.

## 5. Design — phased fixes

### Phase 1 — Fix A: cloud session self-heals

`_cloud_session` calls `copy_claude_config` + `link_cloud_claude_dirs` before
`exec_session`, matching the Lima session path. Idempotent, so a botched rebuild
self-corrects on the next `session`.

**Fix A′:** make `link_cloud_claude_dirs` idempotent and safe over an existing
real directory (merge-then-relink), eliminating the `ln -sfn` footgun. This is
exactly the merge case handled by hand during diagnosis.

### Phase 2 — Fix B: persist durable config on the volume (cloud only)

Add `plugins` to the cloud share set — symlink `~/.claude/plugins →
/vergil/claude/plugins` alongside `projects`/`todos`. Safe on `/vergil` ext4 —
the `EXDEV`/virtiofs atomic-rename problem that forced Lima to keep plugins
VM-local (vergil-tooling#1301/#1603) does not exist on a real block device.
Marketplace clones and installed plugins then survive rebuilds; no reinstall on
every cold boot.

`settings.json` stays **copied from the Mac each session** (Mac = source of
truth) rather than persisted on the volume, to avoid staleness.

### Phase 3 — Fix C: install, not just update (backend-agnostic)

Replace the refresh-only logic in `update_plugins` with a reconcile against the
enabled set in the seeded `settings.json`: register each `extraKnownMarketplaces`
entry, `claude plugin install` anything missing, then `update`. Applies to Lima
and cloud, and delivers the full set (`vergil`, `superpowers`, `paad`,
`diogenes`). Correct per the post-v2.1.195 marketplace behavior (declaring a
plugin in `enabledPlugins` no longer auto-installs it).

### Phase 4 — Fix D: host guardrail

`vrg-vm` refuses (or clearly warns) when not run on the macOS host. `vrg-vm` is a
host-only tool: it manages/creates VMs and reads the operator's real
`~/.claude`; run anywhere else it silently seeds the wrong config or nothing.

## 6. Verification

- After each phase, re-run the read-only inspection on a **real rebuild**:
  `settings.json` present with `defaultMode`; `projects`/`todos`/`plugins` are
  volume symlinks; `claude plugin list --json` non-empty with the expected set.
- Instrument the next natural rebuild to pin root cause #2.
- Unit tests: `_cloud_session` seeding; `link_cloud_claude_dirs`
  idempotency/merge; `update_plugins` install/reconcile path.

## 7. Tasks

Filed in member repos (1:1 with a PR) at implementation time and linked via
`vrg-epic-link --epic vergil-project/.github#69 --task <owner>/<repo>#<TASK>`.

- `vergil-tooling` — Fix A + A′ (cloud session self-heal + link idempotency)
- `vergil-tooling` — Fix B (plugins in the cloud volume share-set)
- `vergil-tooling` — Fix C (`update_plugins` install/reconcile)
- `vergil-tooling` — Fix D (`vrg-vm` host guardrail)
- `vergil-vm` — any provisioning / cloud-init changes surfaced by the above

## 8. Open questions

- Exact mechanism of root cause #2 (why the last rebuild left nothing) —
  instrument the next rebuild.
- Deliver personal (non-plugin) `~/.claude/skills` to cloud? (defer)
- `autoUpdate` on marketplaces vs. explicit install+update in Fix C.

## 9. Cross-references

- vergil-project/.github#51 — sibling epic (off-platform backend)
- vergil-vm#199 — anchor off-platform design
- vergil-tooling#1706 — wrapper / credential companion
- vergil-tooling#1603 — `.claude` share-set audit (plugins never shared)
- vergil-tooling#1825 / #1826 — bypass config copy on cloud create
- vergil-project/.github#45 / vergil-tooling#1974 — single-channel marketplace
