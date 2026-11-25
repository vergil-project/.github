# release-lock: a release lock for `develop` — design

- **Epic:** vergil-project/.github#1664
- **Planning issue:** vergil-project/.github#1662 (review history is in its
  comments)
- **Status:** approved

## 1. Intent

Until epic vergil-project/.github#1384, only the human submitted and merged
PRs, and the human stopped merging while running a release. The human was,
in effect, the exclusive lock on `develop` for the length of a release. Now
agents submit and merge their own PRs into `develop` whenever their gates
pass, so `develop` moves while a release runs.

The problem is no longer hypothetical. During the 2.1.348 release of
vergil-tooling, the release's worktree was removed mid-run, after CD
succeeded, while another agent session was active in the same clone
(vergil-project/vergil-tooling#5187). Until a lock exists, the human must
wait by hand for agents' in-flight PRs to settle before every release.

This epic gives a release a quiescent `develop` without slowing the agents:
agents keep submitting PRs and running CI at full speed throughout a
release, and only the **merge** into `develop` is held back while a release
runs.

## 2. Goals, non-goals and anti-goals

### 2.1 Goals

- No PR merges into `develop` between a release's cut and its end, other
  than the release's own back-merge.
- Agents never wait on a release except at the merge; CI runs on their PRs
  throughout.
- Everything merged before a release starts is in that release.
- A crashed or broken release never leaves a lock held forever, and never
  strands PRs with auto-merge off.
- Nothing is overridden silently: every lock break is recorded.
- Cleanup never removes a running release's worktree.
- Recovery is mechanized in the tooling wherever a deterministic algorithm
  exists; skill prose is the fallback only where it cannot be mechanized.

### 2.2 Non-goals

- Coordinating releases across parallel epics (release blocks, PR sets): the
  follow-on planning issue vergil-project/.github#1663.
- A watcher that starts releases automatically once nothing blocks one. It
  is a long way off; this design only keeps the lock's state machine-readable
  so such a watcher can be added later.
- GitHub rulesets, branch protection changes or merge queues. The lock is
  enforced by the tooling.
- Letting `develop` move during a release. The existing code that pins a
  release to a commit (§3.4) is judged unlikely ever to be reliable on its
  own; the lock replaces that intent.
- Cross-repo scope: the lock is per repo.

### 2.3 Anti-goals

- A skill carrying logic the tooling could carry deterministically.
- A lock that can strand PRs unarmed after a crash.
- A lock that silently disappears or is silently overridden.
- Agents blocking without saying what they are waiting on.

## 3. Current state

What the code does today, in vergil-tooling (`src/vergil_tooling/`):

### 3.1 `vrg-release`

- Human only, fail-closed against every agent identity and agent signal
  (`bin/vrg_release.py`).
- Stages in order (`lib/release/orchestrator.py`): audit, preflight,
  prepare, merge-release, confirm-main, back-merge-bump, teardown-worktree,
  confirm-develop, promote, cd-publish, package-index, finalize,
  update-deps, install, release-records, close-tracking, publish-status.
- Preflight creates `.worktrees/release-X.Y.Z` on `release/X.Y.Z` off
  `develop` and changes into it. Resume adopts the worktree.
- One release at a time is guarded by the open, GitHub-locked tracking issue
  (`_check_no_open_release`). The check is not atomic.
- The tracking issue's body carries the release state block
  (`lib/release/state.py`), a JSON document read and written only by
  `vrg-release`.

### 3.2 `vrg-submit-pr` and auto-merge

- Submit runs its gates, pushes, opens or adopts the PR, then turns on
  auto-merge through GraphQL `enablePullRequestAutoMerge`
  (`lib/auto_merge.py`); a PR that can already merge is merged directly.
- Mechanized PRs (automation and update-deps) also turn on auto-merge
  (`lib/mechanized_pr.py`).
- `vrg-submit-pr` then runs one `vrg-finalize-pr`, which watches the PRs.

### 3.3 `vrg-finalize-pr` cleanup

- After the watch, its cleanup stage removes every `.worktrees/` worktree
  whose PR is merged or closed, whose head matches the branch tip and which
  is clean. It does not check what the worktree is for, whether a process is
  using it, or whether a release is running.
- Only `develop`, `main`, `release` and `gh-pages` are protected; `release/*`
  is not. Once the release PR has merged, a release worktree meets every
  condition for removal. This is the likely cause of #5187.
- `vrg-release` holds the per-clone checkout lock only during its preflight
  sync and its update-deps stage, so the checkout lock does not protect it.

### 3.4 The commit pin

- Preflight records `cut`, the head of the new release worktree, and the
  release reports its contents pinned to it. Nothing stops `develop` from
  moving after the cut, and the back-merge lands on whatever `develop` has
  become.

### 3.5 Refs and locks

- The report-ready relay records live at `refs/vergil/pr-workflow/<branch>`,
  written by a force push of a one-file commit and read with a
  trusted-author check (`lib/pr_workflow/github_transport.py`).
- Nothing creates or compare-and-swaps a `refs/vergil/*` ref atomically
  today. A push with `--force-with-lease=<ref>:` (an empty expected value)
  is a true create-only push, and a lease naming the current value is a
  compare-and-swap.
- The agent identity cannot read repository variables, which rules out a
  repo variable as the lock's home.

## 4. Approaches

Four approaches were considered.

1. **Refuse while PRs are in flight.** The release exits while any PR into
   `develop` is armed. Simplest, but the human is back to waiting by hand
   for a quiet moment, which agents running continuously may never give.
   Rejected.
2. **Lock, then wait for armed PRs to drain.** The release takes the lock so
   no new PR is armed, then waits for the armed ones to merge. Rejected:
   GitHub's auto-merge stays armed after its CI fails, so a fix pushed to an
   armed PR during the release merges after the drain, and a failed armed PR
   that nobody fixes never drains.
3. **Merge at green, never arm.** The tooling never turns on auto-merge; it
   waits for green CI, checks the lock and merges. The lock check sits
   exactly at the merge, but a PR whose agent session dies never merges.
   Rejected.
4. **Lock, disarm the in-flight PRs, re-arm at the end.** The release takes
   the lock and turns off auto-merge on every PR into `develop`, so in-flight
   PRs stop at their CI gates; at the end it re-arms them. Every merged PR is
   in the release, nothing merges during it, and GitHub's auto-merge keeps
   its resilience to a dead session the rest of the time. Chosen (the
   human's design). It needs no record of which PRs were disarmed, because
   of the invariant in §5.1.

## 5. Design

### 5.1 The invariant

**An open, non-draft PR into `develop` always has auto-merge on, except
while a release lock holds it.**

Once the PR workflow declares a PR ready, the PR's CI gates are the only
thing that decides whether it merges. Quiescing `develop` for a release is
the only reason auto-merge is ever off. In the normal workflow every PR is
opened either by `vrg-submit-pr` (agents) or by the mechanized automation
in the tooling; a PR opened by a human by hand is an out-of-band
break-glass exception. A PR opened with auto-merge off and left that way is
not a valid state.

Because of the invariant, every re-arm sweep re-arms **every** open,
non-draft PR into `develop` with auto-merge off, with no record of which PRs
a release disarmed. Drafts cannot be auto-merged and are not used in this
workflow: sweeps skip them, and the audit (§5.9) flags them as invalid.

### 5.2 The lock ref

- One ref per repo, `refs/vergil/release-lock`, on the repo's origin. All
  lock metadata lives in refs, never in issues or repo variables.
- It points at a one-file commit, built the way the relay records are, whose
  file is a JSON document carrying at least: the holder (the identity and
  host running the release), the version, the tracking issue ref, the
  current stage, the health (`running`, `broken` with the stage it broke at,
  or `done`), the start time and a heartbeat time.
- **Taking it** is a create-only push (`--force-with-lease=<ref>:`). A failed
  push means another holder has it: read it.
- **Updating it** (stage, health, heartbeat) is a compare-and-swap push
  against the value the holder last wrote, so a holder whose lock was broken
  or reclaimed finds out at its next update and stops (§5.6).
- **Reading it** applies the trusted-author check the relay records use. An
  untrusted lock commit is treated as held and reported, never as absent.
- The release writes the health as it goes: `running` while stages run,
  `broken` with the stage when the pipeline stops on a failure, `running`
  again on `--resume`, and refreshes the heartbeat every few minutes while
  it runs.

### 5.3 Releasing

`vrg-release` gains two steps around its existing stages:

1. **Lock and quiesce** (before the cut):
   1. Take the lock. If a lock is held and is not stale (§5.6), refuse,
      naming the holder; this replaces the non-atomic one-release-at-a-time
      check. If it is stale, reclaim it (§5.6).
   2. Wait for every arming ref (§5.4) to clear, deleting any that has
      expired.
   3. Turn off auto-merge on every open PR into `develop`.
   4. Fetch `develop` and cut the release from it, as preflight does today.
      Everything merged before this point is in the release; everything
      still in flight stays at its CI gates, CI still running.
2. **Unlock** (the release's last step, after its other stages succeed):
   1. Delete the lock ref (compare-and-swap against its last value).
   2. Run the re-arm sweep: turn on auto-merge, through the arming path
      (§5.4), on every open, non-draft PR into `develop` with it off.

The release's own back-merge PR into `develop` is the one PR that merges
while the lock is held: it is the holder's, and the release merges it
itself. On `--resume`, the release adopts the lock it holds (it is the
holder named in it) rather than taking a new one.

The lock and the quiesce run before `_release_contents` records `cut`, so
the commit pin of §3.4 becomes harmless: `develop` cannot move between the
cut and the end of the release.

### 5.4 Arming: the only gate

Every path that turns on auto-merge goes through one arming function:
`vrg-submit-pr`, a fix pushed by pr-watch, `vrg-finalize-pr`, the mechanized
automation and update-deps PRs, and the re-arm sweeps. No other code calls
`enablePullRequestAutoMerge` or merges a PR into `develop` directly; the
release's back-merge is the single exception, made by the lock's holder.

The arming function closes the race between checking the lock and arming
with a two-sided handshake:

- **The arming side** writes a short-lived **arming ref** for the PR,
  `refs/vergil/release-lock-arming/<pr>`, by a create-only push, *then*
  reads the lock.
  - Lock free: arm (or merge, for a PR that can already merge), then delete
    the arming ref.
  - Lock held by another holder: delete the arming ref and leave the PR
    unarmed, reporting `held for release <version>`.
- **The release** writes the lock, *then* lists the arming refs, and waits
  until each is gone before it disarms (§5.3).

Each side writes before it reads, so at least one sees the other. A PR is
therefore either armed before the release's disarm sweep (and disarmed by
it, or merged first, which puts it in the release) or never armed while the
lock is held. A read-after-arm recheck alone would not be enough: arming a
PR that is already green merges it at once.

An arming ref carries its creation time and expires after a few minutes, so
a crashed arming side never blocks a release; the release deletes an expired
one. The release holding the lock arms its own back-merge PR without the
handshake.

### 5.5 Agents while a release holds the lock

- `vrg-submit-pr` runs its gates, pushes and opens the PR as today. If the
  lock is held, the arming function leaves the PR unarmed, and submit
  reports `held for release <version>; the release re-arms it`, and goes on
  to its finalize watch. CI runs on the PR throughout.
- The merge watch (`vrg-finalize-pr` and `vrg-pr-await`) does the waiting.
  It waits with no limit while a release holds the lock, because the agent's
  work is blocked on the merge anyway, and it always reports what it is
  waiting on: the release, its version, its stage and its health.
  - While the release's health is `running` and its heartbeat is fresh, it
    polls at the normal pace.
  - While the health is `broken`, or the heartbeat is stale (a release
    killed too hard to record `broken`), it reports the broken release and
    switches to a slower watch of the lock. A broken release is the human's
    outage to fix; the agents do not need restarting.
  - Once the lock is released (or reclaimed), the PR is re-armed by the
    release's sweep. If the watch finds its PR open, unarmed and no lock
    held, it re-arms the PR itself through the arming function.
- A tool call in an agent session times out (ten minutes in Claude Code), so
  the watch exits with a defined "still waiting on release" exit code before
  its deadline, as the finalize watch does with its deadline today. The only
  skill rule is that the agent re-runs the watch on that exit; everything
  else is in the tooling.

### 5.6 Stale locks, reclaiming and the escape hatch

- **Stale.** A lock whose tracking issue is closed is stale. The check is
  cheap: one read of the issue's state, made only when a lock is found held.
- **Reclaim.** Any caller that finds a stale lock may reclaim it: a
  compare-and-swap delete against the stale value, then the re-arm sweep. A
  new release that finds a stale lock reclaims it before taking its own.
  Every reclaim is recorded as a comment on the stale lock's tracking issue,
  with the old lock's contents.
- **`vrg-release-lock status`** prints the lock (holder, version, tracking
  issue, stage, health, heartbeat, and whether it is stale) as text, or as
  JSON with `--json`. Any identity may run it. It is the machine-readable
  state a future auto-releaser reads.
- **`vrg-release-lock break --reason "<why>"`** is the human's escape hatch:
  human only, refused under every agent identity and agent signal, as
  `vrg-release` is. It records the reason and the old lock's contents as a
  comment on the tracking issue, then runs the unlock path (delete the lock,
  re-arm sweep).
- A holder whose lock was broken or reclaimed finds out at its next
  compare-and-swap update and stops with a clear error; it never re-takes the
  lock silently.
- There is no "forced release". Because the release disarms in-flight PRs
  instead of waiting for them, it has nothing to force past.

### 5.7 Recovery in layers

The invariant is restored deterministically by the tooling first, with
prose and audit behind it:

1. The release's unlock, or a reclaim, re-arms every PR.
2. The agent's merge watch re-arms its own PR when it finds it unarmed with
   no lock held.
3. The skill re-runs the watch after the tool timeout; this is the only part
   that is prose.
4. The daily audit (§5.9) is the backstop.

### 5.8 Cleanup safety

- `vrg-finalize-pr`'s cleanup, and every other path that removes worktrees,
  skips the worktree the lock names (the lock records it), and skips any
  `release/*` worktree while any lock is held.
- Every worktree removal is logged, naming the worktree, the branch and the
  reason it was judged removable.
- When the release finds its worktree gone, it fails with a clear "release
  worktree vanished" error naming the path, rather than the obscure error a
  later `git fetch` gives today.

### 5.9 The backstop audit

The daily ops audit in `.github` flags any open PR into `develop` with
auto-merge off while no lock is held (a breach of the invariant), and
re-arms it through the arming path. It also flags open draft PRs into
`develop`, and any lock that is stale.

## 6. Constraints

- Git and GitHub access goes through the tooling's sanctioned paths; the
  lock and arming refs use the same transport as the relay records.
- `vrg-release` and `vrg-release-lock break` stay human only. The agent
  identity may read the lock and arm PRs through the gate, never take, break
  or reclaim a lock other than a stale one.
- No silent failures: a lock that cannot be read is held, not absent; a
  failed disarm or re-arm stops the step with an error, never a skip.
- The per-repo scope must not preclude the cross-epic layer of #1663.

## 7. Left to the builder

- Poll intervals for the normal and the broken-release watch, the heartbeat
  interval and its staleness threshold, and the arming-ref expiry.
- Exit codes, including the watch's "still waiting on release" code.
- How the lock JSON and the release state block share or mirror fields.
- How the audit re-arms (in `ops.yml` or through a tooling command it runs).
- How the existing direct-merge paths are routed through the arming
  function.
