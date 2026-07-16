# Cloud memory as a read-only cached projection

**Epic:** vergil-project/.github#156
**Documentation task:** vergil-project/.github#159
**Spark:** vergil-project/.github#154 — Cloud/local memory split: agent memory is
non-persistent in the cloud VMs — need a persistence + sync model.

**Date:** 2026-07-16
**Status:** Design approved; pushback-reviewed; ready for implementation planning.

## Problem

Extending the dev environment into the cloud (off-platform x86 VMs backed by
the `/vergil` data disk) moved agent sessions away from the durable data store
the memory model assumed — the developer's physical host home directory. That
quietly broke the memory model in two ways.

1. **Global `~/.claude/CLAUDE.md` on a cloud VM lives on the ephemeral boot
   disk.** It is copied once at build (`copy_claude_config`), never placed on the
   persistent volume, so a `vrg-vm rebuild` wipes it. Any edit made to it in a
   cloud session is lost on the next rebuild.
2. **The two environments are keyed by different paths and never meet.** Claude
   derives the per-repo memory directory from the session's starting working
   directory. On the physical host that path is under the host home directory
   (e.g. `/Users/<me>/dev/projects/<org>/<repo>`); on the cloud VM the checkout
   is at `/vergil/projects/<org>/<repo>`. The slugs differ, so even though the
   cloud `~/.claude/projects` is symlinked onto the persistent `/vergil/claude`
   volume, the physical-host memory and the cloud memory are different
   directories on different disks. A memory written on the host is invisible in
   the cloud, and vice versa.

The corrosive failure is **silent**: the harness advertises the memory directory
as persistent, an agent writes memory in a cloud session believing it durable,
and the write either lands on a rebuild-doomed disk or under a slug no other
environment reads. We have very likely already lost memory this way without
noticing.

### What is *not* broken

Per-repo memory written in a cloud session is on the persistent volume
(`~/.claude/projects` → `/vergil/claude/projects`), so it survives a plain
`vrg-vm rebuild` — only a nuclear volume wipe loses it. The real problems are the
ephemeral global `CLAUDE.md` and the cross-environment drift, not wholesale
per-rebuild evaporation. Local (Lima) VMs are unaffected: they path-preserve the
project mount (`mountPoint == host location`) and symlink `~/.claude/{projects,
skills}` onto the host's live directory, so a memory write in a local-VM session
flows straight through to the durable host disk. Local memory works precisely
because the working directory — and therefore the slug — matches the host.

## Principle

The physical host is the **single source of truth** for canonical agent data
(memory surfaces and global `CLAUDE.md`). VMs are projections of *one*
developer's environment — this is a solo-developer extension model, not a shared
multi-tenant system; each VM carries that developer's individual GitHub App
identity for its agent roles. The cloud data disk is ephemeral **by design**: we
want to nuke and rebuild it freely, so it must hold only build artifacts and
work-in-progress code, never anything we care about.

Canonical data therefore reaches the cloud as a **read-only cached projection**.
It never flows back automatically. When a cloud session determines that memory or
config must change, it files an issue; the change is implemented on the physical
host and re-projected on the next session or build. Given how infrequently this
data changes, an issue-driven, human-executed writeback is sufficient — no
automated two-way sync.

### The governing rule

> If we copy a file into a VM and do **not** systematically need to modify it,
> lock it read-only. If we currently *do* modify it, establish why — the reason
> may only be to accommodate a path-name mismatch, which we can eliminate by
> making the host path resolve inside the VM.

A file is a lock candidate only if **neither** our provisioning **nor** the
Claude Code runtime writes it during a session. The Claude Code harness writes
into `~/.claude` at runtime (`~/.claude.json`, onboarding state, and possibly
`settings.json` when plugins/MCP change); locking a harness-written file would
break the session with `EACCES`. So "written by the harness at runtime" is an
explicit disqualifier from the locked set, checked by the Component 3 audit
before any file is locked (see Issue-4 resolution). Harness-written files stay
writable — and they are ephemeral/non-canonical anyway, so that is consistent.

## Platforms

A single platform-awareness signal distinguishes three states along two axes —
*sandbox* (in a VM) vs. *host* (not a VM), and *cloud* vs. *local*:

| Platform        | Sandbox? | Credentials        | Canonical store                                   |
| --------------- | -------- | ------------------ | ------------------------------------------------- |
| `physical-host` | no       | human              | live and writable (source of truth)               |
| `local-vm`      | yes      | agent (GitHub App) | shared live via path-preserved mounts (durable)   |
| `cloud-vm`      | yes      | agent (GitHub App) | read-only copy on ephemeral disk                  |

`physical-host` names the property that matters — not a VM. It is macOS today
only because that is the hardware in use; the signal is about physical-vs-virtual,
not the operating system.

The read-only memory control activates **only** on `cloud-vm`. `local-vm` needs
no lockdown because its writes reach the live host store; `physical-host` *is* the
store.

## Components

### 1. Platform resolver (empirical, fail-closed)

A resolver reports the platform, surfaced by extending `vrg-whoami` with a
`--platform` token parallel to the existing `--mode`; `vrg-whoami --explain`
reports the resolving signal and warns on disagreement.

The resolver derives the platform from **empirical heuristics — no written marker
file.** Candidate signals (the exact set and precedence are settled in
`writing-plans`): presence of the `/vergil` mount ⇒ in a VM; a reachable cloud
metadata endpoint ⇒ cloud; host OS and the resolved identity as corroboration.
The platform axis is distinct from the identity axis but correlates with it
(host → human, any VM → agent); the resolver keeps them separate and reports the
correlation rather than conflating them.

The resolver **fails closed** for the memory control: if a VM signal is present
but the box cannot be positively confirmed as `local-vm`, it is treated as
`cloud-vm` (locked), never as `physical-host`. Fail-open — silently deciding
"host" and re-enabling futile writes — is the exact failure this epic exists to
kill and is prohibited. If empirical detection proves insufficiently reliable in
practice, a more robust signal is revisited then; the no-fail-open invariant is
fixed regardless of mechanism.

### 2. Path preservation in the cloud

Two separately-scoped concerns, deliberately not conflated:

**2a — Slug / project-path alignment (required, bounded, feasible).** The cloud
session must *start* at the host-equivalent project path
(`/Users/<me>/dev/projects/<org>/<repo>`), via a symlink or bind-mount from that
path onto the `/vergil` checkout, so Claude derives the same memory slug as the
host and finds the projected memory under the guest's
`~/.claude/projects/<host-slug>/`. This is localized and clearly feasible, and it
is a hard requirement.

**2b — Host-absolute paths embedded inside copied config (bounded, per-file).**
Any copied file that additionally carries arbitrary `/Users/<me>/…` literals is
handled per-file in the Component 3 audit, under a hard constraint: **never
replicate the full macOS HOME into the guest.** The guest's own `~/.claude`,
`~/.local/bin`, `~/.config/vergil`, and injected credentials all key on the
guest's real `$HOME`; forcing `/Users/<me>` to *be* HOME is invasive and fragile.
If a copied file needs such literals to resolve, prefer rewriting or removing the
absolute reference, or leaving that file writable-and-unlocked, over a HOME
rewrite.

### 3. Memory projection — session-coupled sync + copy-then-lock

**This is net-new logic.** Today `copy_claude_config` copies only `CLAUDE.md` +
`settings.json`, and `link_cloud_claude_dirs` symlinks `projects/todos/plugins`
onto the volume; nothing currently projects the per-repo `memory/` directory or
`MEMORY.md` to the cloud.

**Refresh is coupled to `vrg-vm` cloud-session start, not (only) build time.** A
`vrg-vm` session is always tied to one repo, and we already ensure that repo is
checked out. Before SSHing out to a cloud VM to open a session, incrementally
`rsync` *that repo's* memory subset — `~/.claude/projects/<host-slug>/memory/`
and its `MEMORY.md` — plus the global `CLAUDE.md`, from the host to the guest,
then (re-)apply read-only. This is cheap because memory is small and per-repo. A
build seeds the initial state; bulk-copying all memory at build is overkill since
a cloud VM is per-repo anyway. The read-only re-apply happens after each sync.

Hook points: the **cloud-session start path** (primary refresh) and the **build
path** (initial seed).

**Copy-then-lock, applying the governing rule** (including the runtime-writer
disqualifier). Expected locked set: global `CLAUDE.md`, the per-repo `memory/`
directory, and `MEMORY.md`; `settings.json` is locked *only if* the audit
confirms neither provisioning nor the harness writes it at runtime (plugin
installs materialize under `~/.claude/plugins/`, a separate ephemeral directory;
path preservation is expected to remove any path-driven edit).

**Locking must be surgical.** The `~/.claude/projects/<slug>/` directory holds
both the durable `memory/` subtree *and* the session-transcript `.jsonl` files
Claude writes continuously. Lock `memory/` and `MEMORY.md`; leave the transcripts
writable. A blanket recursive `chmod` of `projects/` would break session logging.

### 4. Read-only policy clause in canonical `CLAUDE.md`

Add a universal, condition-gated clause to the canonical (host-authored)
`~/.claude/CLAUDE.md`: on a cloud VM — as reported by the platform resolver —
memory and copied config are a read-only cache; never write memory; when memory
needs to change, file an issue and let it be implemented on the host. The clause
is authored once on the host and rides the copied file, so it is present in every
environment; its condition is simply almost never true outside `cloud-vm`. Because
the clause is part of the canonical file, `CLAUDE.md` stays byte-identical across
environments and remains lockable.

### 5. Issue-driven writeback

A cloud session that needs a memory or config change files it through the existing
`/vergil:triage-capture` flow, producing an intake issue in the org `.github`
repository. The change is later implemented on the physical host against the live
store. No new machinery.

### 6. Verification (session + build)

The one silent-degradation risk is a broken path indirection (Component 2a): if
the host path does not resolve, the slug diverges and memory reads empty with no
error. Both the session-start sync and the build therefore verify, after the
sync/copy, that the host project path resolves in the guest and that the expected
slug memory directory exists and is non-empty, and **fail loudly** otherwise —
turning the silent risk into a loud failure consistent with the
no-silent-failures policy.

## Data flow

- **Cloud-session start (primary refresh):** the host resolves the session's
  repo and its memory slug → incrementally `rsync`s that repo's `memory/` +
  `MEMORY.md` and the global `CLAUDE.md` to the guest → re-applies read-only →
  verifies the host path resolves and the slug memory dir is non-empty (fail
  loudly otherwise) → opens the session.
- **Cloud build (initial seed):** builds the path indirection (Component 2a),
  seeds the canonical set read-only, and the resolver's empirical detection makes
  the box report `cloud-vm`.
- **Cloud read:** Claude starts at the host-equivalent project path → the slug
  matches the host → it reads the read-only memory and `CLAUDE.md` (including the
  cloud clause).
- **Cloud change request:** the policy clause stops the write attempt (soft
  control) and the read-only permissions stop anything that slips through (hard
  control) → the agent files a triage-capture issue → org `.github`.
- **Host:** implements the issue against the live store → the next cloud session
  (or build) re-projects it.

## Error handling and failure modes

- **Write attempt in the cloud** → fails with `EACCES` from the read-only
  permissions. The failure is *loud*, consistent with the no-silent-failures
  policy: a futile write can no longer succeed-then-vanish. Policy prevents the
  attempt; permissions are the defense-in-depth backstop.
- **Broken or missing path indirection** → the slug diverges and memory silently
  reads empty. Mitigated by Component 6 (session/build verification fails loudly).
- **Resolver cannot confirm `local-vm`** → fails closed to `cloud-vm` (locked),
  never fail-open to `physical-host`; `vrg-whoami --explain` surfaces the
  disagreement.
- **Stale cache** — the cloud copy reflects the last projection; an out-of-band
  host edit made *during* a live cloud session is stale only until the next
  session or rebuild. Accepted given how frequently VMs are rebuilt and sessions
  reconnected, and how slowly memory changes.
- **Already-lost writes** — before the first lockdown, audit the current cloud
  volume for memory that diverges from the host canonical (evidence of earlier
  futile writes) and surface it, so nothing already written is discarded unseen
  when read-only is imposed.
- **Data residency** — projecting memory to a cloud VM places developer-private
  memory on the cloud provider's persistent volume, not just the Mac. Under the
  single-tenant, developer-owned trust boundary this is acceptable (it is the
  same boundary as the checked-out code already there), but it is a conscious
  increase in attack surface. Keep genuinely sensitive secrets out of memory, as
  we already do for the world-readable PR-workflow relay ref.

## Testing

- **Unit:** platform resolver (each heuristic; fail-closed on an unconfirmable VM;
  disagreement warning); the governing rule's membership test including the
  runtime-writer disqualifier (copied+unmodified+not-harness-written → locked;
  otherwise excluded).
- **Cloud provisioning / session integration:** after a session-start sync,
  assert that the host project path resolves in the guest; the slug memory
  directory exists, matches the host slug, and is non-empty; `memory/`,
  `MEMORY.md`, and `CLAUDE.md` are read-only; the transcript directory is
  writable; the resolver reports `cloud-vm`; a write to `memory/` fails with
  `EACCES`; and a host-side memory edit is reflected on the *next* session start.
- **Policy:** `CLAUDE.md` carries the cloud clause; `/vergil:triage-capture` is
  reachable from a cloud session.
- **Regression:** the existing "no injected credential on the persistent volume"
  acceptance still holds.

## Validation (cold-rebuild)

The epic carries a cold-rebuild validation task (vergil-project/vergil-tooling#2406):
on a freshly rebuilt cloud VM, open a session and confirm the host path resolves,
the slug memory directory matches the host, the canonical set is read-only,
transcripts stay writable, the resolver reports `cloud-vm`, and a write to
`memory/` fails with `EACCES`. Its human-gated precondition is that the new
tooling version is released and available to the cloud rebuild.

## Out of scope

- Automated two-way memory sync. Explicitly rejected in favor of issue-driven,
  host-executed writeback.
- Multi-tenant / shared-VM semantics. These VMs are a solo-developer extension;
  sharing would be a different design with different constraints.
- Changes to the local-VM (Lima) memory path, which already works.
