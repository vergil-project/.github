# Cloud↔Mac PR handoff via GitHub relay — GitHubTransport (#1858 Deliverable B)

## 1. Summary

The PR-ready metadata (`.vergil/pr-workflow.json`) is gitignored and never rides
the push, so off-platform (cloud x86) it is stranded on a volume the Mac cannot
see. This epic moves that metadata onto GitHub via a reserved git ref, so the
human opens the PR on the Mac with **no local checkout and no metadata
regeneration**. `issue-localize` — the workaround that regenerated the stranded
metadata — is retired.

Submission stays a **human-on-Mac** action. Only metadata travels through GitHub;
human GitHub authority never enters a cloud VM. This is Deliverable B of the
design in `vergil-project/vergil-tooling#1858` (closed with only Deliverable A —
the advisory "cloud is triage-only" boundary — shipped).

## 2. The problem in detail

The sandboxing/dual-role workflow assumed VMs co-located on the Mac with a shared
filesystem. That shared FS quietly carried two payloads: the feature
branch/worktree (already on GitHub via the push) and the small PR-handoff
metadata (`pr-workflow.json`). Off-platform there is no shared FS, so the
`report-ready` → `vrg-submit-pr` handoff breaks: the ready-state is stranded on
the cloud volume, which the Mac cannot read and which the stale-session lifecycle
may reap.

`issue-localize` was built to bridge this: it checks the pushed branch out
locally, **re-validates**, and **regenerates** `pr-workflow.json` from the issue
and the branch diff. Two functions were conflated there — regeneration and
re-validation. The re-validation was rationalized post hoc as "a green cloud run
doesn't prove a green local cold rebuild"; in fact it was an artifact of assuming
regeneration required a validate. It was never a real requirement, and PR CI runs
the same pipeline on the PR regardless. Epic `vergil-project/.github#146` then
spent its effort hardening this workaround (freeze-after-`report-ready`, honest
`vrg-worktree-status`, `vrg-finalize-pr` surfacing). The actual fix has been a
closed issue the whole time.

## 3. Design

The branch already rides GitHub; only the metadata is trapped on local disk. Move
the metadata onto GitHub too and the FS coupling dissolves — at zero recurring
cost, with the identity boundary preserved.

### 3.1 `GitHubTransport` — the relay

A new `GitHubTransport` implementing the existing `Transport` ABC
(`lib/pr_workflow/transport.py`), alongside `LocalFileTransport`. It carries the
handoff via a reserved ref `refs/vergil/pr-workflow/<branch>` — a single-file
commit whose tree contains `pr-workflow.json`.

- `write(state)` — serialize the `WorkflowState`, create a single-file commit
  bearing it, and push it to `refs/vergil/pr-workflow/<branch>` on origin
  (force-update: the ref is a mailbox, last write wins, mirroring
  `report-ready`'s idempotency).
- `read()` — fetch `refs/vergil/pr-workflow/<branch>` from origin and return the
  `WorkflowState`; `None` when the ref does not exist.

The ref namespace `refs/vergil/*` is neither `refs/heads/*` nor `refs/tags/*`, so
branch/tag protection rules do not apply and the standard `contents: write` push
permission the agent already uses for the branch covers it.

### 3.2 `report-ready` — always push

After writing the local file (unchanged), `report-ready` **always** also calls
`GitHubTransport.write()`. The trigger is unconditional — not off-platform
detection — deliberately: a detection-dependent push reintroduces the exact
failure mode this epic exists to remove (state stranded because *something wasn't
set*). The redundant push on a Lima-local `report-ready` is negligible and
harmless. The local write remains the source of truth on the Mac; the ref is the
fallback carrier.

### 3.3 `vrg-submit-pr` — branch-list relay mode

`vrg-submit-pr` gains an optional positional **branch list**:

- **No args** → unchanged: iterate local ready worktrees
  (`worktrees.list_worktrees`) and submit each (the Lima local batch).
- **`<branch> …`** → for each branch: resolve the ready-state — prefer a local
  worktree's `pr-workflow.json` if one exists, else `GitHubTransport.read()` —
  then open the PR from the metadata against the branch already on origin,
  **batched** exactly like the local path.

Branches, not issue numbers, are the identifier: deterministic, and they sidestep
the corner case where a discarded-then-redone issue owns two branch names. A
passed branch that happens to have a local worktree flows through the same code as
the no-arg path (one unified resolution).

This replaces "`/issue-localize <issue-numbers>` then `vrg-submit-pr`" with
"`vrg-submit-pr <branch> …`". The batch power (2–3+ at a time) is preserved.

### 3.4 `vrg-finalize-pr` — relay-ref cleanup

When `vrg-finalize-pr` deletes a merged branch, it also deletes
`refs/vergil/pr-workflow/<branch>` on origin, so relay refs do not accumulate.
Deleting a nonexistent ref is a no-op, never an error.

### 3.5 Plugin — retire `issue-localize`, reconcile the cloud boundary

- Remove the `issue-localize` skill entirely.
- Rewrite the cloud-session contract (currently Deliverable A's "cloud is
  triage-only, no PR-development" advisory in `CLAUDE.md`): with the relay, the
  cloud **can** do PR-development (implement → `report-ready` → relay); only
  submission/merge stay human-on-Mac. Update references across the plugin and
  tooling docs.

### 3.6 Component boundaries

- `GitHubTransport` — the only genuinely new unit; a self-contained
  `Transport` implementation testable in isolation against a throwaway repo.
- `report-ready` gains one call; `vrg-submit-pr` gains one argument and one
  resolution branch; `vrg-finalize-pr` gains one cleanup call. Each is a small,
  local change with a clear seam.

## 4. Data flow

```
Cloud VM:  implement → report-ready
             → writes local pr-workflow.json  AND  pushes refs/vergil/pr-workflow/<branch>
Mac human: vrg-submit-pr feature/<N>-a feature/<M>-b …
             → per branch: read local file if present else fetch relay ref
             → verify origin tip == metadata head_sha → open PR → batch
PR CI is the gate. Merge + finalize on Mac (unchanged);
             finalize also deletes refs/vergil/pr-workflow/<branch>.
```

## 5. Safety and error handling

- **head_sha verification.** Before opening a PR in the relay path, verify
  origin's branch tip equals the metadata's recorded `git.head_sha`. A mismatch
  (a branch that advanced or a reused name) is a clear error, never a silent
  submit. With #146's freeze this should not occur; the check is cheap
  belt-and-suspenders.
- **Missing ready-state.** An explicit branch with neither a local file nor a
  relay ref → a clear "no ready-state found for `<branch>`" error. No silent skip.
- **Provenance / standards checks** in `vrg-submit-pr` operate on GitHub data and
  keep working with no local checkout.
- **Freeze interaction.** The relay push happens inside `report-ready`, before the
  freeze applies — no conflict with epic #146.
- **No silent failures.** A failed ref push in `report-ready` surfaces loudly
  (the local write still succeeded, so the Lima path is unaffected, but an
  off-platform run must know the relay did not land).

## 6. Testing

- Unit: `GitHubTransport` round-trip (write→read), missing ref → `None`, force
  re-write, ref delete; against a throwaway git repo.
- Unit: `report-ready` always-push (ref written on both Lima and off-platform
  paths); push-failure surfacing.
- Unit: `vrg-submit-pr` no-arg local batch unchanged; branch-list relay batch;
  local-file-preferred-over-ref; head_sha mismatch error; missing-ready-state
  error.
- Unit: `vrg-finalize-pr` deletes the relay ref on branch delete; no-op when
  absent.
- **Live validation task (operational).** A real cloud→Mac round-trip: a cloud VM
  runs `report-ready` and pushes the ref; the Mac runs `vrg-submit-pr <branch>`,
  fetches the ref, and opens the PR. Unit tests cannot prove the ref traverses
  GitHub with the cloud agent's real permissions; this closes on a recorded
  `Outcome: SUCCESS`.

## 7. Non-goals

- **Branch-sync-on-handoff / the agent↔agent audit loop.** Deferred — it serves
  the parked audit experiment, not the human-submit path, which needs no local
  checkout. Build it when the audit loop is stood up (the follow-on brainstorm,
  #150, revisits this).
- **A human-role cloud VM with a shared disk.** Considered and rejected: it
  requires human GitHub authority in the cloud, the boundary we will not cross.
- **Cloud-side `vrg-submit-pr` / merge.** Submission and merge stay human-on-Mac.
- **NFS / shared-FS rebuild (#1796).** Superseded; stays parked.

## 8. Known tradeoffs

- **Redundant ref push on Lima.** `report-ready` always pushes, even Mac-local
  where the ref is never read. Accepted: one tiny push removes the
  detection-failure class entirely.
- **PR CI as the sole gate.** Dropping Mac re-validation means a cold-local break
  that CI does not catch would surface only in CI. Accepted: CI runs the same
  pipeline, and the re-validation was never load-bearing.

## 9. References

- Design origin: `vergil-project/vergil-tooling#1858` (Deliverable B).
- Umbrella: `vergil-project/.github#51` (Off-platform cloud VM backend).
- Supersedes hardening: `vergil-project/.github#146`; decision record: `#147`.
