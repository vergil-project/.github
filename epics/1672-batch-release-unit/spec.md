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

### 5.1 Release groups in the plan and on the tasks

- **In the plan.** The delivery table gives each task that must ship with
  others a **release group ID** (for example `G1`), and the plan lists each
  group's members in build order: a dependency chain is listed from the
  bottom up, so where A builds on B and B on C, the order is C, B, A. A task
  with no group is **standalone** and ships alone, as every task does today.
  Planning marks only the exceptions.
- **The plan's group rules**, which `epic-create`'s plan stage and its
  pushback rounds hold the plan to:
  - every member is a code task in one repo (no deployment or validation
    task, no documentation review, no task in another repo or org);
  - a group sits wholly between deployments: no member is blocked by a
    deployment task that another member blocks;
  - a group has at least 2 members and at most the hard ceiling (§5.2);
  - no member's plan entry leaves a design question open;
  - each group exists only in a development epic.
- **On each task.** `file-tasks` writes, through the group tool, a
  `Release group: <id>` line into each member's body, alone on its line, next to its `Plan task:` key. A
  split task (13 into 13a and 13b) carries its group onto every sub-task.
- **The group tool.** A new tool (name left to the builder; §8) owns the
  `Release group:` lines: it lists an epic's groups and members (read
  through the trust layer), and is the only writer of the lines, at filing
  and after it (§5.5). It writes task bodies through the tooling's own
  GitHub layer, never through `vrg-gh` or `vrg-issue-create`.
- **The group record.** Each group's membership is also recorded on the
  epic, in a trusted comment the group tool posts, with machine-readable
  lines giving the group's ID and its full membership in build order. The
  tool posts one when the group is filed and a new one at every change
  (§5.5); the latest trusted record of a group is its membership. The
  lines on the tasks and the record must agree (§5.6).
- **Only the group tool writes the lines.** The tooling enforces it:
  `vrg-gh issue edit` and `vrg-issue-create` refuse every body that adds,
  changes or removes a `Release group:` line, with no exception. An edit that rewrites a whole body (to add a `Blocked-by:`
  line, say) and drops the line is refused, not applied. An edit outside
  the wrappers, such as a human's in the web UI, is caught by the audit.

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
  skips it because the PR holds one issue.
- **Other tasks may come along.** A batch may hold several whole groups and
  standalone tasks: over-batching is allowed (§4).
- **Order.** Members of a group appear in the batch in the group's build
  order. A `Blocked-by` between two members is satisfied by commit order
  within the batch.
- **Same epic, same repo** as today. A group line on a task whose parent is
  not a development epic is refused.
- **Reads fail closed,** like every batch rule today: a group line that
  cannot be read, or one read from an untrusted body, refuses.
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
  grouped or standalone. `unbatch` re-batches each whole group as a batch
  of its own and leaves each standalone task to its own PR.
  `issue-implement`'s batch stops offer the choices in that form.
- The agent stops for the human with the reason it found (for example: the
  fourth member turns out to depend on a feature planned later in the
  epic). The human decides.
- On approval, the group tool rewrites the affected `Release group:` lines
  and posts a new group record on the epic (§5.1), carrying the change,
  its reason and the group's full membership after it.
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
  none. The records written at `file-tasks` take their approval from the
  plan.
- **When a group can change.** Membership is open while the group is
  planned (`plan`, `file-tasks`) and while its batch is built, up to the
  batch's handoff: a task found during implementation to belong in the
  group is added then, before anything is pushed. Membership **freezes at
  the `report-ready` of the batch that holds the group**, where the branch
  itself freezes, and stays frozen until that PR merges or is closed. While
  a handed-off batch or an open PR holds any member, the group tool refuses
  every change to the group, an added member included. A change needed
  after the freeze first takes the PR back (`vrg-pr-workflow unfreeze`
  before submit, closing the PR after it); the change is then recorded and
  the batch rebuilt and handed off again. This always stops for the human.
  So the whole-group check at handoff and submit (§5.2) holds for the life
  of the PR: a re-arm or a pushed fix cannot change the membership it
  checked.
- **A group below two members dissolves.** A change that leaves a group
  with one member removes that member's `Release group:` line in the same
  recorded change, and the record says the group is dissolved. The task is
  then standalone.
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
check runs after that pass in the same run. It also flags a group whose
`Release group:` lines disagree with its latest group record (§5.1): a
line missing, added or changed outside the group tool, and a group record
that drops a member without its verified approval (§5.5). The audit never
repairs either.

The flag is a failure, not a report line. `vrg-epic-audit --close`, the
mode the daily `ops-epic-sweep` runs, checks every group in the org and
exits non-zero while any group is broken; a group it cannot read fails it
the same way. The sweep's `report` job already posts each run's result to
the org's operations queue, which hoshin's daily review reads, so a broken
group reaches the human within a day. The sweep stays failed until the
group is repaired: merged work cannot be unmerged, so the repair is a
group change the human approves and records (§5.5), bringing the group's
lines in line with what shipped. A test proves that a broken group fails
the sweep.

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
  by the audit (§5.5).

Then **one live run**: a validation task drives a deliberately awkward
batch through a fixture repo with the real skills: a dependency chain of
about six tasks as an ordinary batch, with no `Release group:` lines, a
forced fix to the second commit, a rebase onto a `develop` that moved, and
a fix after submit, ending merged as one PR with one commit per task. What
it proves is the batch mechanics, which groups depend on; the group rules
themselves are proved by the automated tests above. It runs against
released Vergil, so it is blocked by the release that ships the hardened
mechanism, and the enforcement of the group rules is blocked by it.

**When the group rules are enforced.** The group rules are live in the
tooling as soon as they ship, with no switch: with no `Release group:`
line on any task, they refuse nothing. Enforcement begins when
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
groups), the plugin's site docs, and the tooling's `git-workflow.md` and
`standards/github-issues.md`.

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
- Every read of a task body goes through the trust layer and fails closed.
- Every rule the tooling can check is checked in the tooling, not in skill
  prose.
- A deployment needs a merge and a release, so it can never sit inside a
  group.
- The fixture run's live check runs against released Vergil.

## 8. Left to the builder

- The group tool's name and command shape.
- The exact hard ceiling (about 20).
- How a batch commit's record chains the fingerprints of the commits before
  it, or otherwise voids later records when an earlier commit changes
  (§5.4).
- The group ID format (`G1`, or another short form) and how the plan's
  delivery table shows it.
- How `vrg-epic-audit` presents groups in its JSON and text output.
- Which fixture repo hosts the live run, and the exact shape of the awkward
  batch, within §5.7.
- How the tasks split between `vergil-tooling` and `vergil-claude-plugin`,
  and their order, provided the `epic-create` change that first writes
  groups is blocked by the live run (§5.7).
