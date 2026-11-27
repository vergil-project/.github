# batch-release-unit: batches as the unit of release — design

- **Epic:** vergil-project/.github#1672
- **Planning issue:** vergil-project/.github#1663 (review history is in its
  comments)
- **Status:** draft

## 1. Intent

An ad-hoc task stands alone: its PR can merge and be released by itself.
An epic's tasks often cannot. When an epic has four PRs coming and the
first is in `develop`, `develop` must not be released until the other
three land. Agents now submit and merge their own PRs, and several epics
run at once, so one epic's partial work can be swept into another epic's
release and break it. Until now the human prevented this by sequencing the
work by hand.

This epic removes the need for that sequencing with one invariant:

> **A submitted PR that goes green merges into `develop`, and from that
> point it is releasable.**

There are no release blocks and no PR sets. The gate moves before
submission: work that cannot be released by itself is not submitted until
it can be released at once. Tasks that must ship together are built as
**one batch PR**, one commit per task, using the batch mechanism of epic
vergil-project/.github#1019. Batching becomes an essential building block
of the fleet instead of the low-risk optimization it was introduced as.

The three layers this belongs to:

1. **The release lock** (epic vergil-project/.github#1664, approved): a
   running release holds back merges into `develop`.
2. **Batches as the unit of release** (this epic).
3. **Epic ownership of repos** (follow-on, planning issue
   vergil-project/.github#1678): work that must ship together across repos.

The supporting documents are in [research/](research/): the batch PR
research of 2026-11-25 and the direction notes recorded before this design.

## 2. Goals, non-goals and anti-goals

### 2.1 Goals

- The plan of an epic declares which of its tasks must ship together, as
  **release groups**, and the tooling enforces it: no PR holds part of a
  group.
- A dependency chain inside a group is built on one branch in dependency
  order and merges as one PR, instead of one merge-and-wait round trip per
  task.
- A group changes only by an explicit edit recorded on the epic with its
  reason; an edit that removes a member needs the human's verified
  approval.
- The batch mechanism is hardened before it carries releases: every edge
  case the research lists has an automated test, the gaps those tests
  expose are fixed, and one live run of a deliberately awkward batch
  proves the skill-driven paths end to end.
- A broken invariant (a group whose members merged through different PRs)
  is detected and reported, never silent.

### 2.2 Non-goals

- **Release blocks and PR sets.** Replaced by this design; the planning
  issue's original question is answered by batching.
- **Groups across repos.** A PR closes only issues in its own repo, and a
  batch is one PR. Work that must ship together across repos is the
  ownership follow-on, vergil-project/.github#1678.
- **Parallel batch builds** (building a batch's tasks in parallel on local
  branches, then combining them into one branch). A throughput
  optimization, captured as planning issue vergil-project/.github#1825.
- **Agents deciding by themselves to take a task out of a group.** For now
  every such case stops for the human, so its reasons can be studied and
  turned into criteria later (§5.5).
- **Changes to the release lock.** A batch is one PR on the lock's one PR
  path; nothing in #1664 changes.
- **Groups in ad-hoc epics.** An ad-hoc epic holds standalone work by
  definition.

### 2.3 Anti-goals

- A group that can be broken without anyone noticing: a task of a group
  shipped alone, by an extraction, an unbatch or a separate PR.
- A rule that lives only in skill prose where the tooling could check it.
- Size limits justified by human review. Vergil takes humans out of code
  review, so a PR's size is not a correctness concern (§4).

## 3. Current state

Data, from the research (§2–§7 of
[research/2026-11-25-batch-prs-in-practice.md](research/2026-11-25-batch-prs-in-practice.md))
and the code:

- **Hard rules** (`vergil-tooling`, `lib/pr_workflow/batch_rules.py`,
  `check_hard_rules`), enforced at `batch-start`, `report-ready`,
  `batch-check` and `vrg-submit-pr`: 2 to 8 issues (`MAX_BATCH = 8`), no
  duplicates, all in the PR's repo, tasks only, a buildable parent, trusted
  authors, not operational, not blocked by an open deployment task, one
  shared parent epic. `check_commit_per_issue` enforces one commit per
  issue in the listed order, each subject ending in its issue's ref.
- **Guidelines** (`vergil-claude-plugin`, `skills/issue-implement/SKILL.md`,
  "What to batch"): batch only small issues (about 150 lines and 6 files);
  never batch a security-labelled issue, design work, a cross-repo contract
  or a documentation review; a cap of 4 (8 when every item is tiny).
- **`epic-implement`** batches only tasks on the runnable frontier, which
  are independent of each other by construction. Nothing records that
  tasks must ship together, and nothing stops one of them being submitted
  alone.
- **Batch stops** let the human extract an issue from a batch
  (`vrg-pr-workflow batch-extract`) or unbatch it.
- **In practice:** 44 real batches merged or open, at most 4 issues, none
  merged wrongly; four fixes to a non-last commit after the PR opened, each
  correct. The hard paths rest on that small sample (research §5, §6).

## 4. The principle behind it

The two ways of getting batching wrong are not symmetric:

- **Under-batching is a correctness failure.** Tasks that must ship
  together, submitted apart, let a release carry partial work.
- **Over-batching is only a throughput cost.** A batch is built serially
  on one branch, so parallel development becomes serial, and it gets one
  CI run instead of several. Nothing breaks.

So a release group is the **minimum** that must ship together, and the
tooling enforces only that minimum. Anything above it is a throughput
judgment, left to `epic-implement` as batching is today.

PR size used to be limited because a human had to read the change. Vergil's
founding question is whether humans can be taken out of code development
and review entirely; under it, size limits no longer guard correctness.
The "small" guideline and the guideline caps go (§5.2).

## 5. Design

### 5.1 Release groups in the plan and on the epic

- **In the plan.** The delivery table gives each task that must ship with
  others a **release group ID** (for example `G1`), and the plan lists each
  group's members in build order: a dependency chain is listed from the
  bottom up, so where A builds on B and B on C, the order is C, B, A. A task
  with no group is **standalone** and ships alone, as every task does today.
  Planning marks only the exceptions.
- **The group rules**, which `epic-create`'s plan stage and its pushback
  rounds hold the plan to, and which the group tool checks on every write
  (§5.5), all but the design-question rule, which stays a planning
  judgment:
  - every member is an open code task in one repo (no deployment or
    validation task, no documentation review, no task in another repo or
    org);
  - no dependency path runs from one member to another through a task
    outside the group, a deployment task or any other: the group would wait
    on work that waits on the group. So a group sits wholly between
    deployments;
  - a group has at least 1 member and at most the hard ceiling (§5.2);
    a one-member group ships as one PR like a standalone task, and exists
    so that the task can be named as a group, as a coordinated release
    names each repo's share (§5.9);
  - no member's plan entry leaves a design question open;
  - each group exists only in a development epic.
- **The group record is the only source of membership.** A group's
  membership lives in one place: a log of **group records** on the epic,
  each a trusted comment the group tool posts. Nothing about a group is
  written into a task's body, so no rule parses task bodies for it, and a
  tool that rewrites a body cannot change a group.
  - **One line, a full snapshot.** A record's first line is the
    machine-readable record in a fixed grammar, for example
    `release-group G1 rev 3: <ref> <ref> <ref>`, giving the group's ID, a
    revision number and its full membership, in build order, as fully
    qualified issue refs. It is a snapshot, never a delta, so no history is
    replayed to read it. The lines after it carry the change's reason and
    its approval, if it needs one (§5.5), and the parser reads none of
    them for membership.
  - **The latest record wins.** A group's membership is its latest trusted
    record, the highest revision. A record comment that has been edited
    since it was posted is refused, so a record can only be superseded by
    a new one, never rewritten.
  - **Safe to retry.** A change is one comment, so there is no partial
    write. The group tool re-reads the log before it posts, so a retry
    after a failure posts nothing twice.
  - A group is referenced as its epic and its group ID; an ID is never
    reused within an epic, a dissolved group's included.
- **The group tool.** A new tool (name left to the builder; §8) is the only
  writer of group records: it lists an epic's groups and members (read
  through the trust layer, failing closed), files each group at
  `file-tasks` and records every change after it (§5.5).
- **A grouped task stays where its group is.** A group belongs to its
  epic, and its records name its members, so a member cannot leave the
  epic or the repo without a recorded change. `vrg-epic-move` and
  `vrg-intake-route` refuse to move a task that is a member of a group,
  and `vrg-gh issue transfer` refuses to transfer one, naming the group,
  until a recorded change (§5.5) has removed it.
- **A split task.** Splitting a grouped task (13 into 13a and 13b) is one
  recorded change: the sub-tasks are added and the original removed, with
  the human's verified approval (§5.5). A split already stops for the
  human, since `issue-implement` proposes it.

### 5.2 The hard rules, updated

The batch hard rules gain the group rules, enforced at every stage that
checks a batch (`batch-start`, `report-ready`, `batch-check`,
`vrg-submit-pr`). The whole-group rule also runs on every PR path that
takes a single issue, since a single-issue PR is where a group is most
easily broken:

- **Whole groups only.** A PR that holds any member of a group holds every
  member of it. A single-issue PR of a grouped task is refused, naming the
  group and its missing members. The check runs whether or not the PR is a
  batch, on each path: `report-ready`, `vrg-submit-pr` in each of its modes
  (template, `--all`/`--select`, named-branch through the local and relay
  paths, the CLI emergency mode, and push) and `batch-check`. No path
  skips it because the PR holds one issue. The rule finds a PR's groups
  by looking up each issue it closes in the group records of that issue's
  epic.
- **The PR's closing lines stay as checked.** Which issues a PR closes is
  read from its body, so `vrg-gh pr edit` refuses any body change that
  adds, removes or changes a `Closes` or `Ref` line of a PR into
  `develop`. The finalize watch re-runs the whole-group rule on the PR's
  current body before the merge.
- **Other tasks may come along.** A batch may hold several whole groups and
  standalone tasks: over-batching is allowed (§4).
- **Order.** Members of a group appear in the batch in the group's build
  order. A `Blocked-by` between two members is satisfied by commit order
  within the batch.
- **Same epic, same repo** as today. A group record on an epic that is not
  a development epic is refused.
- **Reads fail closed,** like every batch rule today: a group record that
  cannot be read, an untrusted or edited one, or one that drops a member
  without its verified approval (§5.5), refuses.
- **Size.** `MAX_BATCH = 8` becomes a high safety ceiling (about 20; the
  value is the builder's), kept only to refuse a runaway list. The skill
  guidelines on size ("small", about 150 lines and 6 files, the caps of 4
  and 8) are removed.
- **The other "never batch" guidelines.** Two more are removed as
  anachronisms of batching as an optional saving: never batch a task whose
  plan entry calls for design work (design is done in the spec and plan,
  and a task that still needs it is not ready to implement, batched or
  not), and never batch a change to a contract other repositories consume
  (the order across repos is carried by release and deployment tasks, and
  a group sits wholly between deployments, §5.1). Two stay: never batch a
  documentation review, whose sweep may change nothing and so cannot give
  the batch its one commit, and respect the order rule above.

### 5.3 Building a group

- **When a group is runnable.** A group can be built once every blocker
  *outside* the group is closed (a deployment task among them). Blockers
  between members do not count. `vrg-epic-audit` reports each group as
  runnable or blocked, with what blocks it, as it reports tasks.
- **`epic-implement`** dispatches a runnable group as one batch, its
  members in build order, and may add other runnable tasks or groups to the
  batch on its own throughput judgment. The rule that a batch takes only
  tasks on the runnable frontier gives way, for group members, to the
  group's own runnability.
- **`issue-implement`** builds each member on top of the ones before it, as
  batch mode does today, with one commit per member and its merge review.
- **Security issues may be batched.** The existing safeguards stay: a
  security finding is a batch stop, `batch-extract` refuses while a
  security details file exists, and public text names security items only
  as counts.

### 5.4 Review after a rewrite

Any change to a commit of a batch (a fold into it, a rebase that changes
its patch) makes that commit **and every later commit** due for a fresh
merge review before handoff, or before pr-watch pushes the fix after
submit. Today only the changed commit's issue is reviewed again
(research §6). The merge-review records name the commits as they are after
autosquash, not the fixup commits folded away.

The gate enforces this, not skill prose. Today the gate matches each batch
commit's record by `commit_fingerprint`, the commit's own diff against its
parent (`vergil-tooling`, `lib/change_fingerprint.py`), so a fold into an
earlier commit that leaves a later commit's own diff unchanged leaves that
later record valid. Instead, a batch commit's record covers its own
fingerprint together with the fingerprints of every commit before it in the
batch, so a changed earlier commit voids the records of all later ones and
`vrg-submit-pr` and push refuse until they are reviewed again. A clean
rebase changes no commit's own diff, so it voids nothing, as today.

### 5.5 Changing a group

- A task cannot leave a group without the human's approval (below). A
  batch stop (extraction, handing off what is done, unbatching) is refused
  by the tooling only when it would split a group: leave part of a group
  in the batch, or send part of one out. Moving whole groups is allowed,
  needs no group change and is not recorded as one.
- **The check comes before anything is discarded.** Extraction discards
  the stopped issue's work (a hard reset of the batch branch), so the
  group check runs first, as a read-only check the skill calls before the
  reset; `batch-extract` repeats it and refuses as a backstop. A refusal
  therefore never follows lost work.
- **The choices at any stop in a batch that holds a group** are
  `continue`; any of `extract`, `handoff` or `unbatch` that moves only
  whole groups and standalone tasks; or a group change (below) followed by
  the action the human rules. This holds whether the stopped task is
  grouped or standalone. `unbatch` removes the batch's markers, then
  starts each whole group afresh as a batch of its own (`batch-start`)
  and leaves each standalone task to its own PR.
  `issue-implement`'s batch stops offer the choices in that form.
- The agent stops for the human with the reason it found (for example: the
  fourth member turns out to depend on a feature planned later in the
  epic). The human decides.
- On approval, the group tool posts a new group record on the epic
  (§5.1): the group's full membership after the change, its reason and its
  approval.
  Then the extraction, handoff or unbatch the human ruled proceeds.
- The comments are the record from which criteria for these decisions are
  later derived, so that agents can eventually make them alone. That later
  step is not part of this epic.
- **Only a removal needs the human's approval.** Taking a task out of a
  group, splitting a group, dissolving it, or removing a member closed
  without a merge can let a release carry partial work, so each needs the
  human's approval. Adding a task to a group, or merging two groups, only
  makes a group larger, which costs throughput and never correctness (§4):
  the agent may make it alone while membership is open (below), and the
  group tool still records it, with its reason.
- **The approval is verified by the tooling.** A change that removes a
  member is made only on a verified human approval naming the change (the
  group, and its members before and after), checked as merge-review
  exemptions are (`rulings.approvals.verified_approvals`); the group tool
  refuses without one. A group record that drops a member from the
  previous record is accepted only with its verified approval: the
  whole-group rule (§5.2) refuses to read membership from one that lacks
  it, and the audit flags it (§5.6). A record that only adds members needs
  none. A group's first record has no earlier record, so it drops nothing
  and needs no approval; every later record that drops a member needs one,
  one written by a re-entry into `file-tasks` included.
- **Every write is checked.** The group tool refuses a record that breaks
  a group rule (§5.1), checked against the tasks as they are. On an
  addition, or a merge of two groups, each incoming task must also be
  clean: open, and held by no open PR, no handed-off batch and no batch
  marker. So an addition can never itself make a broken group.
- **When a group can change.** Membership is open while the group is
  planned (`plan`, `file-tasks`) and until its batch starts
  (`batch-start`); within that window the agent may add members alone.
  Once the batch has started, any change stops for the human: a task found
  during implementation to belong in the group means the batch is unbatched
  and started again with the new membership, at the human's ruling. Such a
  find is usually a planning miss, and the stop gives the human the chance
  to examine how it happened, so that later planning avoids it. Membership
  **freezes at the `report-ready` of the batch that holds the group**,
  where the branch itself freezes, and stays frozen until that PR merges
  or is closed. While a handed-off batch or an open PR holds any member,
  the group tool refuses every change to the group, an added member
  included. A change needed after the freeze first takes the PR back
  (`vrg-pr-workflow unfreeze` before submit, closing the PR after it); the
  change is then recorded and the batch rebuilt and handed off again. This
  always stops for the human.
  So the whole-group check at handoff and submit (§5.2) holds for the life
  of the PR: a re-arm or a pushed fix cannot change the membership it
  checked.
- **A group ends only by a recorded change.** A group that shrinks to one
  member stays a group: nothing dissolves it by itself, so a group named
  outside its epic (§5.9) never drops out of view. Dissolving a group is a
  removal, made with the human's verified approval (above): its record
  says the group is dissolved and lists no members. Its tasks are then
  standalone.
- **Closing a member never changes the group.** Membership is the latest
  group record (§5.1), so a member closed without a merge (as not planned,
  say) is still a member, and the group is broken (§5.6). The repair is a
  recorded change that removes it. The whole-group rule (§5.2) counts the
  members of the latest record.

### 5.6 Detecting a broken invariant

A break is judged by the merged PRs that close the members (each
member's `Closes` line on a merged PR), not by issue state. `vrg-epic-audit`
flags a group whose members are closed by different merged PRs, or a
member closed with no merged PR closing it, naming the group, its members
and their PRs. An open member whose closing PR merged with the others is
drift, which the sweep's drift-close pass closes, not a break; the group
check runs after that pass in the same run. It also flags a group record
that has been edited since it was posted, and one that drops a member
without its verified approval (§5.5). The audit never repairs either.

The flag is a failure, not a report line. `vrg-epic-audit --close`, the
mode the daily `ops-epic-sweep` runs, checks every group in the org and
exits non-zero while any group is broken; a group it cannot read fails it
the same way. The sweep's `report` job already posts each run's result to
the org's operations queue, which hoshin's daily review reads, so a broken
group reaches the human within a day. The sweep stays failed until the
group is repaired: merged work cannot be unmerged, so the repair is a
group change the human approves and records (§5.5), bringing the group's
record in line with what shipped. A test proves that a broken group fails
the sweep.

The sweep is detection, not a release gate: `vrg-release` runs no group
check of its own and trusts the layers before it (§7).

### 5.7 Hardening the batch mechanism

Before groups are enforced, every edge case below gets an automated test
in `vergil-tooling` against real git repositories, and each gap a test
exposes is fixed:

- a fix folded into a middle commit of a chain of three or more;
- a fold that conflicts with a later commit;
- a long-lived batch rebased onto a `develop` that has moved;
- the fresh review of every later commit after a rewrite (§5.4);
- a rebase after merge review (vergil-claude-plugin#1140);
- a fix pushed after submit through pr-watch, with `batch-check`;
- merge-review records after autosquash and after a later clean rebase;
- a collapsed batch and an extracted member under the group rules, a stop
  that would split a group refused, and a `handoff` or `unbatch` of whole
  groups allowed;
- the group rules themselves (§5.2), each refusal and each read failure;
- a single-issue PR of a grouped task, refused on every path of §5.2;
- a group change refused while a handed-off batch or an open PR holds a
  member (§5.5);
- a removal refused without a verified approval, and a group record that
  drops a member without one refused by the whole-group rule and flagged
  by the audit (§5.5);
- an edited group record refused by the whole-group rule and flagged by
  the audit, and a retried write that posts nothing twice (§5.1);
- each refusal of the group tool's write check: a broken group rule, a
  dependency path through a non-member, and an incoming task that is not
  clean (§5.5);
- a change after `batch-start` that stops for the human (§5.5);
- a move, an intake route and a transfer of a grouped task, each refused
  (§5.1);
- a `vrg-gh pr edit` that changes a closing line refused, and the finalize
  watch's re-run of the whole-group rule (§5.2).

Then **one live run**: a validation task drives a deliberately awkward
batch through a fixture repo with the real skills: a dependency chain of
about six tasks as an ordinary batch, with no release group, a
forced fix to the second commit, a rebase onto a `develop` that moved, and
a fix after submit, ending merged as one PR with one commit per task. What
it proves is the batch mechanics, which groups depend on; the group rules
themselves are proved by the automated tests above. It runs against
released Vergil, so it is blocked by the release that ships the hardened
mechanism, and the enforcement of the group rules is blocked by it.

**When the group rules are enforced.** The group rules are live in the
tooling as soon as they ship, with no switch: with no group record on
any epic, they refuse nothing. Enforcement begins when
`epic-create`'s plan and `file-tasks` steps first write groups, so that
skill change is blocked by the live run.

The run's shape is not a standard fixture run: `fixture-run` files one
change task per fixture, and this run needs a chain of change tasks
driven as one batch. The plan gives it its own task shape, or extends
`fixture-run` for a multi-task change.

### 5.8 Documentation

The batching guidelines are rewritten around the principle of §4, as the
way work that must ship together is released, not an optional saving,
keeping only the guidelines §5.2 keeps:
`issue-implement`'s "What to batch" and batch stops, `epic-implement`'s
batching section, `epic-create`'s plan and `file-tasks` steps (release
groups) and its "Task sizing" section (sub-tasks of a grouped task ship in
one batch, not each on its own PR), the plugin's site docs, and the tooling's `git-workflow.md` and
`standards/github-issues.md`.

### 5.9 Interface for epic ownership (#1678)

The epic ownership follow-on (planning issue vergil-project/.github#1678)
coordinates releases across repos, and builds on release groups for each
repo's share. What it may rely on:

- **A repo's share is its release group.** Where that follow-on speaks of
  a repo's planned batch, it means a release group of this design: planned
  in `epic-create`'s plan, changed only as §5.5 says. A share of one task is
  a one-member group (§5.1).
- **A group is referenced as its epic and its group ID** (§5.1), never by
  its members.
- **The group tool's list command is the supported read path.** It reads
  through the trust layer and fails closed, and returns the group's latest
  record. No other tool parses the group records.
- **A handed-off batch may wait for another task to submit it.** Between
  the batch's `report-ready` and its submit, however long, membership stays
  frozen (§5.5), and the whole-group rule runs at that submit (§5.2), as it
  does at every submit.

What that follow-on adds (its own lines on the tasks, its submit guard, its
coordinated release task) is its design, not this one's.

## 6. Approaches

**For the coordination problem** (decided before this design, recorded in
the direction notes):

- **Release blocks and PR sets.** A merged PR places a block on the
  release until other work lands. A new coordination protocol, with races
  between epics to surface and resolve. Rejected: it lets unreleasable work
  into `develop` and coordinates around it.
- **Batching as the unit of release.** Chosen: it keeps `develop`
  releasable at every merge and reuses a mechanism that works, at the cost
  of bigger PRs, which no longer matter (§4).

**For where "ship together" is decided:**

- **A. The plan declares groups, and the tooling enforces them.** Chosen:
  the invariant becomes a hard rule.
- **B. The plan declares groups, and only the skills follow them.**
  Rejected: one slip breaks the invariant silently.
- **C. `epic-implement` decides at run time.** Rejected: groups would be
  decided after their tasks are filed and partly built.

**For changing a group:**

- **An explicit, recorded edit, human-approved when it removes a member.**
  Chosen (§5.5).
- **Extraction removes the task from its group automatically.** Rejected:
  a quick unbatch could ship partial work, with the ruling its only record.

**For the hardening bar:**

- **Automated tests only.** Rejected: the riskiest paths are skill-driven.
- **Automated tests plus one live run.** Chosen (§5.7).

## 7. Constraints

- A PR closes only issues in its own repo (the placement law), so a group
  is one repo's.
- Every read of a task body or a group record goes through the trust layer
  and fails closed.
- Every rule the tooling can check is checked in the tooling, not in skill
  prose.
- A deployment needs a merge and a release, so it can never sit inside a
  group.
- The fixture run's live check runs against released Vergil.
- **No group check at release time, by design.** `vrg-release` trusts the
  layers before it: the group rules on every write of a record (§5.5), the
  freeze, the move and transfer refusals (§5.1), the whole-group rule on
  every PR path and the closing-line guard (§5.2), and the daily sweep
  (§5.6). The accepted gap: a break made outside the tooling (a human's
  change in the web UI) or caused by a tooling bug can land between one
  sweep and the next, and ship in a release cut before the sweep reports
  it. While humans cut releases, the human reads that report in hoshin's
  daily review. Agent-run releases (the epic ownership follow-on,
  vergil-project/.github#1678) revisit this trade-off.

## 8. Left to the builder

- The group tool's name and command shape.
- The exact hard ceiling (about 20).
- How a batch commit's record chains the fingerprints of the commits before
  it, or otherwise voids later records when an earlier commit changes
  (§5.4).
- The group ID format (`G1`, or another short form) and how the plan's
  delivery table shows it.
- The exact grammar of a group record's first line, within §5.1.
- How `vrg-epic-audit` presents groups in its JSON and text output.
- Which fixture repo hosts the live run, and the exact shape of the awkward
  batch, within §5.7.
- How the tasks split between `vergil-tooling` and `vergil-claude-plugin`,
  and their order, provided the `epic-create` change that first writes
  groups is blocked by the live run (§5.7), and the group tool's list
  command and the group record's format (§5.1, §5.9) come ahead of the
  hardening work (§5.7), so the epic ownership follow-on is not blocked on
  the rest. The group tool's write commands ship in the same release as
  the whole-group rule (§5.2), never before it, so no group can be written
  while nothing enforces it.
