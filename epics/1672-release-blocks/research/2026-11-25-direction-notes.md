# Direction notes — 2026-11-25

- **Epic:** vergil-project/.github#1672
- **Planning issue:** vergil-project/.github#1663
- **Status:** the human's working direction, recorded before the brainstorm
  restarts; not an approved design

The brainstorm for this epic restarts once the release-lock spec
(vergil-project/.github#1664) is approved. These notes record the direction
the human set while it was paused, so the restart begins from it. The
evidence behind it is in
[2026-11-25-batch-prs-in-practice.md](2026-11-25-batch-prs-in-practice.md).

## 1. Three layers

1. **The release lock** (epic vergil-project/.github#1664). A running
   release holds back merges into `develop`.
2. **Batching as the unit of release** (this epic). It replaces release
   blocks and PR sets.
3. **Epic ownership of repos** (a follow-on epic). It coordinates work
   that must ship together across repos.

## 2. Batching as the unit of release

- **One invariant:** a submitted PR that goes green merges into `develop`,
  and from that point it is releasable. There are no release blocks and no
  PR sets.
- **The gate moves before submission.** Work that cannot be released on its
  own is not submitted until it can be released at once.
- **Tasks that must ship together form one batch:** one PR with one commit
  per task.
- **Batching becomes an essential building block,** not the low-risk
  optimization epic vergil-project/.github#1019 introduced it as. The rules
  to rethink:
  - Drop "batched issues must be small".
  - Drop "never batch security issues".
  - Drop "batch only independent tasks". A dependency chain is the best case
    for a batch: where A builds on B and B on C, implement C, then B, then A
    on one branch and merge them together, instead of three merge-and-wait
    round trips.
- **Harden the mechanism before scaling it up.** Identify its edge
  conditions and corner cases and make sure each is tested.

## 3. Epic ownership of repos (follow-on)

Designed as if agents can run `vrg-release`, which is the next major
milestone. Until then the human runs every release, and so is the lock in
practice.

- **The ownership tool.** A new tool takes, shows and releases
  ownership of a repo for an epic. It uses its own ref per repo, separate
  from the release lock, holding the owning epic's ref.
- **The agent's sequence:**
  1. Prepare every batch the epic needs, in every repo, up to report-ready.
     Nothing is submitted yet.
  2. Take ownership of every repo those batches touch.
  3. Submit the batches and watch them merge.
  4. Release each repo: for now by handing the human the `vrg-release`
     commands, later by running them itself.
  5. Release ownership.
- **Blocking.** An epic that cannot take ownership of every repo it needs
  blocks and says which epic holds which repo. The human resolves it,
  since every epic runs in an interactive session the human drives. A
  candidate for the corner case: an epic blocked on one repo gives up the
  repos it already holds, so others can use them.
- **`vrg-release` checks ownership; it never takes it.**
  - With an epic argument, the release runs only if that epic owns the
    repo, and aborts otherwise, telling the human and the agents to
    reconcile.
  - Without one, the release runs as today when no epic owns the repo, and
    fails, naming the owner, when one does.
- **Ownership ends only when the owning agent releases it.** A release does
  not end it.
- **The human's escape hatch.** The human can break an epic's ownership,
  for example when its session is lost and the work abandoned. Every
  break is recorded.
- **Releasing ad-hoc work (subtle; a candidate, not settled).** Ad-hoc
  tasks are standalone and normally reach `main` whenever the next release
  happens. When an ad-hoc task needs another to be released first, and the
  fleet is quiet, the agent may ask the human for permission to run a
  release, and run it once confirmed.

## 4. Open for the brainstorm

- Acquiring ownership all at once (holding nothing while waiting) or in a
  fixed order (holding what is already held). The first cannot deadlock but
  can starve a large epic; the second avoids both.
- The full list of batch edge cases to test, from the research's §6.
- Whether the epic's slug and title, chosen when it was planned as release
  blocks, still fit.
