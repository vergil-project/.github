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

- One PR path: every PR the tooling opens into `develop` goes through one
  submit path, which always arms auto-merge. Submitting a PR asserts it is
  ready for `develop`; from there its merge is automatic, and no tool merges
  a PR into `develop` itself.
- No PR merges into `develop` between a release's cut and its end, other
  than the lock holder's own back-merge and dependency-update PRs.
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
- Making the agent gates or the provenance check a required CI check. The
  security architecture as a whole is to be re-evaluated with the planned
  move off public GitHub to a self-hosted forge; until then the daily audit
  is the check after submit.
- Agent-run releases. They are a later milestone; when they come,
  `--supersede` and `vrg-release-lock break` stay human-only administrative
  operations.

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

### 3.2 How PRs reach `develop` today

The tooling gets a PR merged in several different ways:

| Path | How it merges | Run by |
|---|---|---|
| `vrg-submit-pr` | `auto_merge.enable_auto_merge` through GraphQL `enablePullRequestAutoMerge`; a PR that can already merge is merged directly | agents and the human |
| `mechanized_pr` (used by `vrg-github-repo-config` and `vrg-upstream-refresh`) | `enable_auto_merge` with squash, then `pr_merge.wait_until_merged` | automation |
| `vrg-update-deps` | `update_deps/pr.merge_pr`, which merges directly with `pr_merge.wait_and_merge`; no auto-merge | the human, at any time |
| the release's `update-deps` stage | the same `run_update` and `merge_pr` as `vrg-update-deps` | `vrg-release` only |
| the release's back-merge (`lib/release/bump.py`) | `release/merge.wait_and_merge`, a direct merge | `vrg-release` only |
| the release PR into `main` | `wait_and_merge` with `--admin`, pinned to the head | `vrg-release` only |

- `vrg-submit-pr` then runs one `vrg-finalize-pr`, which watches the PRs.
- `wait_and_merge` predates general auto-merge: the tool waits for green
  checks and merges the PR itself.

### 3.3 `vrg-finalize-pr`

- Its watch re-runs the provenance check and the agent gates on each new
  head of an open PR, and turns auto-merge off on a failure
  (`lib/finalize_watch.py`). These re-checks date from when the client
  merged the PR; GitHub now merges it, and the watch is not guaranteed to run
  at all, so they cannot be what decides a merge.
- A watch that finds a PR's auto-merge off, at the start or mid-watch, ends
  it at once as `failed: auto-merge off`, an exception handed to the human.
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

The epic is built in two phases, after one standalone fix:

0. **Cleanup safety first** (§5.8), for the incident in #5187: protect every
   `release/*` worktree while an open release tracking issue exists, and log
   every removal. It needs neither phase, and phase 2 moves it onto the lock.
1. **Rationalize the PR path** (§5.4): every tool that gets a PR into
   `develop` converges on the one PR path, which is made stable before
   anything is built on it. `vrg-finalize-pr` is reduced to its primary job:
   it watches and reports each PR's outcome, syncs the local `develop` and
   cleans up. It never turns auto-merge off, and its provenance and gate
   re-checks are removed (§5.1).
2. **The release lock** (the rest of this section), built on that path.

### 5.1 The invariant

**An open, non-draft PR into `develop` always has auto-merge on, except
while a release lock holds it.**

Once the PR workflow declares a PR ready, the PR's CI gates are the only
thing that decides whether it merges. Quiescing `develop` for a release is
the only reason auto-merge is ever off: only the release lock's disarm sweep
turns it off, and no other code does.

The tooling's own judgment of a PR (the provenance check and the agent
gates) runs in the one PR path, at the PR's current head, every time it
arms, a re-arm sweep's included; a PR it refuses stays unarmed and is
reported. The path arms only a same-repo PR whose branch has a trusted
author (the watch's existing `confirmed_head` rule) and which passes the
provenance check. Any other PR into `develop` is reported and never armed,
by any path. Once a PR is armed, the merge is GitHub's. A check that must stop a merge after the PR is armed is a
required CI check in the ruleset, or it is not a merge gate at all: the
client cannot gate a merge it does not make. The daily agent audit is the
backstop after the merge. In the normal workflow every PR is
opened either by `vrg-submit-pr` (agents) or by the mechanized automation
in the tooling; a PR opened by a human by hand is an out-of-band
break-glass exception. A PR opened with auto-merge off and left that way is
not a valid state.

Because of the invariant, every re-arm sweep offers **every** open,
non-draft PR into `develop` with auto-merge off to the one PR path, with no
record of which PRs a release disarmed; the path arms those it accepts and
reports the rest. Drafts cannot be auto-merged and are not used in this
workflow: sweeps skip them, and the audit (§5.9) flags them as invalid.

### 5.2 The lock ref

- One ref per repo, `refs/vergil/release-lock`, on the repo's origin. All
  lock metadata lives in refs, never in issues or repo variables.
- It points at a one-file commit, built the way the relay records are, whose
  file is a JSON document carrying at least: the holder (the identity and
  host running the release), the version, the tracking issue ref (empty
  until `prepare` creates the issue, then written by the holder's next
  update), the current stage, the health (`running`, `broken` with the stage it broke at,
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
- **The lock history** is a second ref, `refs/vergil/release-lock-history`:
  one commit per break, reclaim or supersede takeover, each carrying the old
  lock's contents,
  who acted, when, and the reason. It is appended by compare-and-swap, read
  with the same trusted-author check, and is the record of every override
  whether or not the old lock named a tracking issue.

### 5.3 Releasing

`vrg-release` gains two steps around its existing stages:

1. **Lock and quiesce** (before the cut):
   1. Take the lock. If a lock is held and is not stale (§5.6), refuse,
      naming the holder; this replaces the non-atomic one-release-at-a-time
      check. If it is stale, reclaim it (§5.6). From here until the
      tracking issue exists, every refusal and every failure deletes the
      lock (compare-and-swap against the value this run wrote) before the
      release exits.
   2. Wait for every arming ref (§5.4) to clear, deleting any that has
      expired.
   3. Turn off auto-merge on every open PR into `develop`.
   4. Fetch `develop`, then run every preflight gate that judges its head
      (the pending-deps gate, the develop CD gate, which may wait on an
      in-progress CD run) against the fetched head, and cut the release from
      it. The lock comes first so the gates judge exactly what is cut; merges
      wait meanwhile, CI does not. Everything merged before this point is in
      the release; everything still in flight stays at its CI gates, CI
      still running.
2. **Unlock** (the release's last step, after its other stages succeed):
   1. Delete the lock ref (compare-and-swap against its last value).
   2. Run the re-arm sweep: turn on auto-merge, through the arming path
      (§5.4), on every open, non-draft PR into `develop` with it off.

The lock is held for the whole release, deliberately. A release that breaks
at any stage, a stage after publication included, leaves `develop` frozen
until it is finished with `--resume` (or the human breaks the lock). A
broken release is a stop-the-line event: the workflow stops, the release is
fixed and finished, and the failure is analysed so the tooling can be made
to avoid it or recover from it. Unlocking earlier would let `develop` move
under stages that still work against it, and is not done until those
failure modes are well understood.

The release's own back-merge and dependency-update PRs into `develop` are
the only PRs that merge while the lock is held: they are the holder's, and
the release arms them through the one PR path with the holder's pass
(§5.4). On `--resume`, the release adopts the lock it holds rather than
taking a new one. It is the holder when the lock names its identity and its
tracking issue; the host is recorded for information only, so a resume on
another machine adopts the lock too.

On `--supersede X.Y.Z`, the new release takes over the superseded release's
lock instead of taking a new one: a compare-and-swap against the lock's
current value, so `develop` stays locked throughout and no re-arm sweep
runs in between. The takeover is recorded in the lock history (§5.2) and on
both tracking issues. It goes ahead only when the superseded release's lock
shows health `broken` or a heartbeat older than the staleness threshold; a
release that is `running` with a fresh heartbeat is refused, naming it.

The lock and the quiesce run before `_release_contents` records `cut`, so
the commit pin of §3.4 becomes harmless: `develop` cannot move between the
cut and the end of the release.

### 5.4 The one PR path and its arming gate

**One path.** Every PR into `develop` that the tooling opens or arms goes
through one submit path: a shared library function, with `vrg-submit-pr` as
its command line. That covers agents' PRs, a fix pushed by pr-watch,
`vrg-finalize-pr`, the mechanized automation (`vrg-github-repo-config`,
`vrg-upstream-refresh`), `vrg-update-deps` run on its own, the release's
`update-deps` stage, the release's back-merge, and the re-arm sweeps.
`mechanized_pr` and the update-deps merge converge on it rather than
keeping paths of their own.

- The path always arms auto-merge. It never merges a PR into `develop`
  itself, and nothing else does either: no code outside it calls
  `enablePullRequestAutoMerge`, `gh pr merge` or a wait-and-merge against
  `develop`. A new tool that opens PRs into `develop` uses the path.
- The path works out the merge method itself every time it arms, from the
  PR's trusted linkage, as submit does today: rebase for a batch PR (one
  commit per issue), a merge commit for the holder's back-merge and
  dependency-update PRs, and squash otherwise. It refuses to arm a PR whose
  method it cannot work out, saying why. So every re-arm (the sweeps, the
  watch's own re-arm, the audit) arms with the right method and needs no
  record of the one a disarm discarded.
- `wait_and_merge` is retired for `develop`. A caller that must block until
  its PR lands (the release, a scheduled job) waits until the PR is merged,
  and never merges it.
- The release PR into `main` is the one direct merge left in the tooling. It
  needs the ruleset bypass (`--admin`, pinned to the head), which auto-merge
  cannot use, and `main` is outside the lock.

**The holder's pass.** The path takes the lock value its caller holds. The
running `vrg-release` passes the value it wrote, and the path checks it
against the lock ref; a matching caller arms through the lock without the
handshake below. Only the release's back-merge and its `update-deps` stage
use it. Because the lock admits one release at a time, there is at most
one of each. Without a matching value a call is an ordinary one:
`vrg-update-deps` run on its own while a release holds the lock is held like
any other PR (§5.5), and the unlock sweep arms it.

**The arming gate.** Inside the path, the arming gate closes the race
between checking the lock and arming with a two-sided handshake:

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

An arming ref expires a few minutes after the release first sees it, so a
crashed arming side never blocks a release. Expiry is measured from the
release's own first sighting, never from a time the ref carries. Arming
refs are read with the same trusted-author check as the lock: an untrusted
one is reported and deleted, never waited on. The release's total wait on
arming refs is bounded; past the bound it fails, naming the refs. Expiry
must not reopen the race:

- The arming side renews its ref by compare-and-swap immediately before the
  arm call, and aborts, leaving the PR unarmed, if the ref is gone.
- The release deletes an expired arming ref only by compare-and-swap against
  the value it read, and after deleting any, runs the disarm sweep again.
- A PR into `develop` that merged after the cut, other than the holder's own,
  is a hard error: the release reports it, naming the PR, and stops.

### 5.5 Agents while a release holds the lock

- `vrg-submit-pr` runs its gates, pushes and opens the PR as today. If the
  lock is held, the one PR path leaves the PR unarmed, and submit
  reports `held for release <version>; the release re-arms it`, and goes on
  to its finalize watch. CI runs on the PR throughout.
- A scheduled job that opens a PR through the one PR path (the mechanized
  automation) treats `held for release` as a successful outcome and leaves
  the PR to the unlock sweep, rather than failing or replacing it.
- The merge watch (`vrg-finalize-pr` and `vrg-pr-await`) does the waiting.
  It checks for a held lock before it judges auto-merge off, so a PR a
  release holds is reported as held, never as `failed: auto-merge off`.
  It waits while a release holds the lock, because the agent's work is
  blocked on the merge anyway, and it always reports what it is
  waiting on: the release, its version, its stage and its health.
  - While the release's health is `running` and its heartbeat is fresh, it
    polls at the normal pace.
  - While the health is `broken`, or the heartbeat is stale (a release
    killed too hard to record `broken`), it reports the broken release and
    switches to a slower watch of the lock. A broken release is the human's
    outage to fix; the agents do not need restarting.
  - Once the lock is released (or reclaimed), the PR is re-armed by the
    release's sweep. If the watch finds its PR open, unarmed and no lock
    held, it re-arms the PR itself through the one PR path.
- The watch runs as a background command and keeps its existing deadline
  (`VRG_FINALIZE_DEADLINE`) while a lock holds the PR. A PR still held by a
  release when the deadline passes ends with a distinct `held by release`
  outcome and exit code, apart from an ordinary timeout. The only skill rule
  is that the agent re-runs the watch on that exit; everything else is in
  the tooling.

### 5.6 Stale locks, reclaiming and the escape hatch

- **Stale.** A lock whose tracking issue is closed is stale. The check is
  cheap: one read of the issue's state, made only when a lock is found held.
  A lock that names no tracking issue (a release that died before
  `prepare`, too hard to delete its lock) is stale once its heartbeat is
  older than the staleness threshold.
- **Reclaim.** Any caller that finds a stale lock may reclaim it: a
  compare-and-swap delete against the stale value, then the re-arm sweep. A
  new release that finds a stale lock reclaims it before taking its own.
  Every reclaim is recorded in the lock history (§5.2), and as a comment on
  the stale lock's tracking issue when it names one.
- **`vrg-release-lock status`** prints the lock (holder, version, tracking
  issue, stage, health, heartbeat, and whether it is stale) as text, or as
  JSON with `--json`. Any identity may run it. It is the machine-readable
  state a future auto-releaser reads.
- **`vrg-release-lock break --reason "<why>"`** is the human's escape hatch:
  human only, refused under every agent identity and agent signal, as
  `vrg-release` is. It records the reason and the old lock's contents in the
  lock history (§5.2), and as a comment on the tracking issue when the lock
  names one, then runs the unlock path (delete the lock, re-arm sweep).
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
3. The skill re-runs the watch on its `held by release` exit; this is the
   only part that is prose.
4. The daily audit (§5.9) is the backstop.

### 5.8 Cleanup safety

- `vrg-finalize-pr`'s cleanup, and every other path that removes worktrees,
  skips the worktree the lock names (the lock records it), and skips any
  `release/*` worktree while any lock is held.
- Every worktree removal is logged, naming the worktree, the branch and the
  reason it was judged removable.
- Before the lock exists (§5, the standalone fix), the same protection is
  keyed on an open release tracking issue: no `release/*` worktree is removed
  while one is open.
- When the release finds its worktree gone, it fails with a clear "release
  worktree vanished" error naming the path, rather than the obscure error a
  later `git fetch` gives today.

### 5.9 The backstop audit

The daily ops audit in `.github` flags any open PR into `develop` with
auto-merge off while no lock is held (a breach of the invariant), and
offers it to the one PR path to re-arm; a PR the path refuses (one that is
not same-repo with a trusted author on its branch, or fails provenance) is
only flagged, never armed. It also flags open draft PRs into `develop`, and
any lock that is stale.

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
- Exit codes, including the watch's `held by release` code.
- How the lock JSON and the release state block share or mirror fields.
- How the audit re-arms (in `ops.yml` or through a tooling command it runs).
- The shape of the shared submit function and how each existing tool is
  moved onto it (§5.4).
