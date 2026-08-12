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

- `write(state)` — serialize the `WorkflowState`, build a **single-file commit
  object out-of-band via git plumbing** (`hash-object` → `mktree`/`commit-tree`),
  and push it to `refs/vergil/pr-workflow/<branch>` on origin (force-update: the
  ref is a mailbox, last write wins, mirroring `report-ready`'s idempotency).
  **Invariant: `write()` never mutates HEAD, the index, or the working tree** — it
  must not `git commit` on the branch. A normal commit would advance the feature
  branch just as the #146 freeze is arming (both happen inside `report-ready`);
  the out-of-band build makes the push a pure ref write, freeze-neutral. A test
  asserts the branch tip and working tree are unchanged after a `write()`.
- `read()` — fetch `refs/vergil/pr-workflow/<branch>` from origin and return the
  `WorkflowState`; `None` when the ref does not exist.

**Feasibility (load-bearing).** The design assumes `refs/vergil/*` — neither
`refs/heads/*` nor `refs/tags/*` — is pushable with the `contents: write`
permission the agent already uses for the branch, unblocked by branch/tag
protections or org rulesets. This is **verified up front by a feasibility spike
(§6) before the transport is built**, not asserted blind: if an org ruleset
restricts `refs/*` creation or the cloud identity's token cannot push a non-branch
ref, the whole design must change, so it is proven first.

### 3.2 `report-ready` — always push

After writing the local file (unchanged), `report-ready` **always** also calls
`GitHubTransport.write()`. The trigger is unconditional — not off-platform
detection — deliberately: a detection-dependent push reintroduces the exact
failure mode this epic exists to remove (state stranded because *something wasn't
set*). The redundant push on a Lima-local `report-ready` is negligible and
harmless. The local write remains the source of truth on the Mac; the ref is the
fallback carrier.

### 3.3 `vrg-submit-pr` — branch-list relay mode

`vrg-submit-pr` gains an optional positional **branch list**. This is a real
refactor, not a small add: today the command is built around *"I am running inside
the worktree of the branch I am submitting"* — it reads `git.current_branch()`,
enumerates `worktrees.list_worktrees(root)`, checks `is_main_worktree()`, and
**force-pushes the branch itself**. The relay path submits a branch with **no
local worktree and nothing to push** (the cloud already pushed it).

The change factors out a **shared PR-open core** — "open the PR from
`(branch, base, metadata)`" — that both paths call:

- **No args** → unchanged: iterate local ready worktrees
  (`worktrees.list_worktrees`), keep the existing worktree/branch-push behavior,
  and submit each (the Lima local batch).
- **`<branch> …`** → the **relay path**: for each branch, resolve the ready-state
  — prefer a local worktree's `pr-workflow.json` if one exists, else
  `GitHubTransport.read()` — then drive the shared core from **origin + the GitHub
  API only**: no `current_branch`, no worktree lookup, **no branch push** (the
  branch is already on origin). Provenance and standards checks run against GitHub
  data, so they need no local checkout. Branches are **batched** exactly like the
  local path.

Branches, not issue numbers, are the identifier: deterministic, and they sidestep
the corner case where a discarded-then-redone issue owns two branch names. A
passed branch that happens to have a local worktree flows through the local-file
resolution (one unified resolution seam feeding the shared core).

This replaces "`/issue-localize <issue-numbers>` then `vrg-submit-pr`" with
"`vrg-submit-pr <branch> …`". The batch power (2–3+ at a time) is preserved.

### 3.4 `vrg-finalize-pr` — relay-ref cleanup

Relay refs must not become the next class of stranded cruft. Cleanup covers every
way a branch ends, not just merge:

- **Idempotent overwrite** — `report-ready` force-overwrites the ref, so a reused
  branch of the same name self-heals (already in §3.2).
- **Finalize** — `vrg-finalize-pr` deletes `refs/vergil/pr-workflow/<branch>` on
  origin when it cleans up a branch on **both** the merged and the closed-PR
  paths. Deleting a nonexistent ref is a no-op, never an error.
- **Swept safety net** — finalize's existing straggler sweep also prunes a relay
  ref whose branch no longer exists (an abandoned branch, or one deleted
  out-of-band), so a truly orphaned ref cannot linger.

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
- `report-ready` gains one call (the always-push); `vrg-finalize-pr` gains ref
  deletion on its existing cleanup paths — both small, local changes with a clear
  seam.
- `vrg-submit-pr` is the substantial change: extracting a shared PR-open core and
  adding a worktree-free, push-free relay path around it (§3.3). It is its own
  task with its own tests, not a one-line add.

## 4. Data flow

```text
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
- **Data exposure.** On a public repo the relay ref is world-readable.
  `pr-workflow.json` carries only title, summary, notes, issue number, branch, and
  commit SHAs — no secrets — so this is acceptable. The `--notes` field must not
  carry secrets (already true for the PR body it becomes); note it in the docs.

## 6. Testing

- **Feasibility spike (first task, before any implementation).** A throwaway
  probe from a cloud VM: push a dummy `refs/vergil/probe/x`, fetch it from the
  Mac, delete it. Proves the load-bearing `refs/vergil/*` push assumption (§3.1)
  before real code depends on it. A `validation`-kind task, sequenced first; if it
  fails, the design is revisited before building.
- Unit: `GitHubTransport` round-trip (write→read), missing ref → `None`, force
  re-write, ref delete; against a throwaway git repo. Includes the **out-of-band
  invariant** test: HEAD, index, and working tree unchanged after `write()`.
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
  `Outcome: SUCCESS`. It is gated on the impl tasks being **released** (a
  human-gated release, never agent-performed) and **deployed** onto the cloud VM
  and the Mac — the canonical impl → deploy → validate shape.

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
