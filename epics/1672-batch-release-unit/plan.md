# batch-release-unit: batches as the unit of release — implementation plan

- **Epic:** vergil-project/.github#1672
- **Planning issue:** vergil-project/.github#1663 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each implementation task is one PR, run with
> `issue-implement`; operational tasks are run with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** tasks that must ship together are declared in the plan as
**release groups**, built and merged as one batch PR, and enforced by the
tooling on every PR path, so that a green PR merged into `develop` is
releasable from that moment.

**Architecture:** three phases, in order. Phase 1 ships the group record
(a snapshot log of comments on the epic) and its read path,
`vrg-release-group list`, which the epic ownership follow-on
(vergil-project/.github#1678) builds on. Phase 2 hardens the batch
mechanism: the merge-review gate chains each batch commit's record to the
commits before it, every edge case of spec §5.7 gets a real-git test, a
middle commit can be re-reviewed before handoff (in the tooling and the
skills, with a read-only gate probe), and one live run drives an awkward
chain batch through a fixture. Phase 3
enforces groups: the group rules, the whole-group rule on every PR path,
the group tool's write commands, the guards around them and the audit,
then the skill changes, the last of which (epic-create's plan and
`file-tasks` steps) is what first writes groups.

**Tech Stack:** Python 3 (vergil-tooling, pytest, real git repositories in
`tmp_path`), GitHub GraphQL through the trust layer, Markdown skills
(vergil-claude-plugin), Zensical site docs.

**Spec:** [`spec.md`](spec.md) in this directory. Section references (§5.4)
are to it.

## Global Constraints

- The group tool is **`vrg-release-group`**, in vergil-tooling. It is the
  only writer of group records; its `list` command is the only supported
  read path outside `lib/release_groups/` (§5.1, §5.9).
- A group ID matches `G[1-9][0-9]*` (`G1`, `G2`, …) and is never reused
  within an epic, a dissolved group's included (§5.1). The plan's delivery
  table shows it in a **Group** column; a standalone task's cell is `—`.
- **A group's epic** is a development epic, or a proxy epic of one for
  the proxy's own-org tasks (§5.1). Its `epics.task_epic` kind is
  `development`, or `proxy` with `epic_audit.proxy_link_problems(epic)`
  empty and its `Primary:` link naming an epic whose kind is
  `development`. Every other kind (ad-hoc, a standing management kind, the
  intake, planning and deferred queues) is not a group's epic. A proxy's
  members are its own tasks, all in one repo, and its group IDs are its
  own. Where this plan says "development epic" for a group, it means a
  group's epic in this sense.
- A group record is a trusted comment on the epic (the chain's head, as
  `epics.task_epic` resolves it, a proxy for a proxy task). Its first line
  is exactly one of:

  ```text
  release-group <id> rev <n>: <owner>/<repo>#<N> [<owner>/<repo>#<N> …]
  release-group <id> rev <n>: dissolved
  ```

  `<n>` is a positive integer without leading zeros; members are fully
  qualified, in build order, each once. Any later line starting `Reason:`
  is the change's reason; no other line is parsed. A comment whose first
  line starts `release-group` but does not match is malformed (§5.1).
- **The latest record wins:** a group's membership is the record with the
  highest revision. The log is valid only when each record of the group
  has a higher revision than every record posted before it, so the top
  revision is held by exactly one record and it is the latest one posted.
  Anything else (two records at one revision, a lower revision posted
  after a higher one) is a **conflict**, and every read of that group
  refuses (§5.1). A missing revision is not a conflict: a deleted record
  leaves the latest remaining one in force.
- Records are trusted only from the `human` and `agents` roles
  (`trust.read_comment_stream`, `roles=frozenset({Role.HUMAN,
  Role.AGENTS})`). A trusted record that has been edited (`edited=True`)
  refuses every read of its group. The stream gives a skipped comment as
  metadata only (`trust.Skipped`), never its text, and no code here reads
  a skipped comment's text in any other way. So a comment on the epic that
  the stream skipped because an identity outside those roles edited it
  (`kind == "edit"`, a `comment <id>` field) cannot be scoped to a group:
  it is a failure that names no group, and refuses every read on the epic,
  naming the comment ID. Any other untrusted comment is ignored and
  reported on stderr, metadata only (§5.1, §5.2).
- Every group read fails closed, scoped to the group: an unreadable epic,
  a malformed or edited record, a conflict, or a record on an epic that is
  not a group's epic (above) refuses, naming the epic and group (§5.2,
  §7). A failure confined to one group refuses only reads of that group,
  so a task outside it, standalone tasks included, is unaffected. A
  failure that names no group (an unreadable epic, a malformed record with
  no readable group ID, a record on an epic that is not a group's epic, a
  comment skipped for an outsider's edit) refuses every read on the epic.
- **The human recovers by deleting the bad comment.** With one driver per
  epic (below), a conflict or an edited record is rare, so the tooling has
  no recovery command. Every refusal names the comments involved, and the
  human reads them in the web UI and deletes the bad one (one side of a
  conflict, the edited record, or a comment skipped for an outsider's
  edit). The latest remaining record is then in force again, and the
  driver runs `vrg-release-group set` if the group still needs a change.
- Nothing about a group is written into a task body, and no code parses a
  task body for group membership (§5.1).
- `MAX_BATCH = 20` (the hard ceiling of §5.2; the spec leaves the value to
  the builder). `MIN_BATCH = 2` stays (§5.3).
- **Every merge keeps `develop` releasable, this epic's own included.** The
  group tool's write commands (Task 12) are blocked by the whole-group rule
  (Task 11) and the group rules (Task 10), so no release can carry a write
  command without the rule that enforces what it writes (§8).
- The live run (Task 8) uses an ordinary chain batch with no group record;
  epic-create's group writing (Task 20) is blocked by it (§5.7).
- **One driver per epic** (§7). An epic is driven by one `epic-implement`
  session at a time, in one place (local or cloud), and the human keeps it
  so; no tooling locks it. Group changes come only from that driver or the
  epic's planning run, so the batch markers that matter are in that
  driver's main clone, and Task 12's "Started" check reads them locally by
  design. If two drivers ever ran one epic, the whole-group rule on every
  PR path (Task 11) would still refuse a batch that is not its whole group.
- Every repo a group's members live in has its main clone at
  `~/dev/projects/<owner>/<repo>` on the host that plans or drives the
  epic; the group tool's write commands run from it (Task 12, Task 20).
- Every vergil-tooling and vergil-claude-plugin task is test-first where it
  changes behaviour, ends with `vrg-container-run -- vrg-validate` green,
  and commits with `vrg-commit`.
- Releases are human actions; a deployment task never cuts one. Deployment
  tasks follow the plugin's `issue-deploy/references/release-coverage.md`:
  the precondition is `vrg-release-record check` on each task shipped; the
  release has installed itself on the host and in every running VM; the
  only thing left over for the plugin is `/reload-plugins` in each running
  session, a human-run step.

## Review Focus

1. **Two writers change one group at once.** Two agents post `rev 4` of
   `G1` within seconds. Expected: every read of `G1` (the whole-group rule,
   `list`, the audit) refuses as a conflict naming both comments; `set`
   refuses too and says the human must delete one of them (Task 2,
   `test_conflict_at_top_revision_refuses`; Task 12,
   `test_set_refuses_on_conflict`).
2. **A PR that closes a member by a keyword other than `Closes`.** A human
   PR body says `Fixes vergil-project/vergil-tooling#900`, and #900 is in
   a group. Expected: refused at arm, since the rule reads GitHub's closing
   references and not only the body's `Closes` lines (Task 11,
   `test_arm_reads_github_closing_refs_not_body_only`).
3. **A member under an epic continuation.** The epic has rolled over past
   100 sub-issues, and a member's native parent is a continuation.
   Expected: its group is found on the chain's head, and the whole-group
   rule applies (Task 2, `test_groups_of_resolves_continuation_to_head`).
4. **A member transferred or moved by hand in the web UI.** Expected: the
   next audit reports the group as breaking the one-repo or same-epic rule;
   nothing silently drops the member (Task 17,
   `test_audit_flags_member_moved_out_of_epic`).
5. **A batch PR that holds members of two groups, interleaved.** A batch
   lists `G1`'s A, then `G2`'s X, then `G1`'s B. Expected: allowed, since
   only each group's own build order must hold (A before B), and refused
   when B comes before A (Task 11, `test_interleaved_groups_keep_own_order`).

---

## Delivery

Every task is filed under the epic, vergil-project/.github#1672, in the
repo its PR lands in. The plan task ID is the `Plan task:` key `file-tasks`
writes. No task of this epic is in a release group: groups do not exist
until Task 20 ships.

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | `.github` | task | Spec and plan (vergil-project/.github#1673) | — |
| **Phase 1** | | | | |
| 2 | `vergil-tooling` | task | The group record and its reader; `vrg-release-group list` | — |
| 3 | `vergil-tooling` | deployment | A vergil-tooling release carries Task 2 (the release gate) | 2 |
| **Phase 2** | | | | |
| 4 | `vergil-tooling` | task | The merge-review gate chains each batch commit's record to the commits before it | 2 |
| 5 | `vergil-tooling` | task | Real-git batch fixture and the fold, conflict and rebase tests | 4 |
| 6 | `vergil-tooling` | task | Batch review-record tests: autosquash, clean rebase, a fix after submit through push mode | 5 |
| 6a | `vergil-tooling` | task | A merge-review record for a middle batch commit; the read-only gate probe `batch-gates` | 4, 5 |
| 6b | `vergil-claude-plugin` | task | `merge-review` and `issue-implement`: re-review after a rewrite and a clean rebase before handoff | 6a |
| 7 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 4–6a (the release gate) | 4, 5, 6, 6a |
| 7a | `vergil-claude-plugin` | deployment | A plugin release carries Task 6b; every running session reloads its plugins | 6b, 7 |
| 8 | `vergil-fixture-python-lib` | validation | The live run: an awkward six-task chain batch through the real skills | 7, 7a |
| **Phase 3** | | | | |
| 9 | `vergil-tooling` | task | The ceiling: `MAX_BATCH = 20` | — |
| 10 | `vergil-tooling` | task | The group rules and the holder check (`lib/release_groups/rules.py`) | 2 |
| 11 | `vergil-tooling` | task | The whole-group rule on every PR path and stage | 2, 5, 9, vergil-project/vergil-tooling#5275 |
| 12 | `vergil-tooling` | task | `vrg-release-group set` and `dissolve` | 10, 11 |
| 13 | `vergil-tooling` | task | `vrg-gh pr edit` keeps a develop PR's closing lines | — |
| 14 | `vergil-tooling` | task | Move, route and transfer of a grouped task refused | 2 |
| 15 | `vergil-tooling` | task | `batch-extract` takes a set and a read-only `--check`; a split refused | 2 |
| 16 | `vergil-tooling` | task | `vrg-epic-audit`: group runnability in `--epic` | 10 |
| 17 | `vergil-tooling` | task | `vrg-epic-audit`: broken groups, rule breaks and edited records fail the sweep | 16 |
| 18 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 9–17 (the release gate) | 9, 10, 11, 12, 13, 14, 15, 16, 17 |
| 19a | `vergil-claude-plugin` | task | `issue-implement` and `pr-watch`: batching rewritten, stops, groups | 18 |
| 19b | `vergil-claude-plugin` | task | `epic-implement`: dispatching groups | 18 |
| 20 | `vergil-claude-plugin` | task | `epic-create`: groups in the plan, `file-tasks` writes them, "Task sizing" | 8, 18 |
| 21 | `vergil-claude-plugin` | deployment | A plugin release carries Tasks 19a, 19b and 20; every running session reloads its plugins | 19a, 19b, 20 |
| 22 | `vergil-tooling` | task | Tooling docs: `git-workflow.md`, `standards/github-issues.md`, the group tool's reference | 18 |
| doc-review | `vergil-tooling` | task | Documentation review: `Documentation review: batch-release-unit` | 21, 22 |
| retrospective | `.github` | retrospective | `Retrospective: batch-release-unit` | all others |

### Operational tasks

- **Task 3** makes the group record format and `list` usable before
  anything else, so that the epic ownership follow-on (#1678) is not
  blocked on the rest (§8, §5.9). Nothing in this epic waits for it.
- **Task 7** ships the hardened mechanism the live run needs: the run uses
  released Vergil (§7).
- **Task 7a** ships the skills that make re-review after a rewrite
  possible before handoff, which the live run drives.
- **Task 8** is the live run, with its own shape (§5.7): it files its own
  chain of change tasks in the fixture when it runs and drives them as one
  batch, so nothing makes the chain runnable before the run starts it.
  It is not a `fixture-run` trio.
- **Task 18** ships all of phase 3's tooling at once. Its tasks are ordered
  so that any release cut part-way carries no write command without its
  rules (Global Constraints).
- **Task 21** ships the skills. Task 20 is what first writes groups, so
  enforcement begins at Task 21's release.
- No `fixture-run` is seeded: the epic changes no language support,
  container image, shared CI workflow or release tooling.

### Notes for `file-tasks`

- **Advisories.** The advisories raised in the spec pushback rounds were
  recorded without being attached to the planning issue, on the human's
  ruling, so no advisory tracks vergil-project/.github#1663 and no
  retarget is needed. Confirm with `vrg-security-record list` (no
  `tracks:` line naming #1663) before posting `Tasks filed: yes`.
- **Task 8's chain tasks** are filed by Task 8's own run (Appendix A), not
  here.
- **The follow-on planning issues** raised while this epic was planned
  (vergil-project/.github#1678, vergil-project/.github#1825,
  vergil-project/.github#1830) are not tasks of this epic.

---

## Phase 1: the group record

### Task 2: The group record and `vrg-release-group list` (vergil-tooling)

Spec §5.1, §5.9.

**Files:**

- Create: `src/vergil_tooling/lib/release_groups/__init__.py`,
  `src/vergil_tooling/lib/release_groups/record.py`
- Create: `src/vergil_tooling/bin/vrg_release_group.py` (the `list`
  command only)
- Modify: `pyproject.toml` (`[project.scripts]` entry `vrg-release-group`)
- Test: `tests/vergil_tooling/test_release_group_record.py`,
  `tests/vergil_tooling/test_vrg_release_group.py`

**Interfaces:**

- Produces, in `lib/release_groups/record.py`:

  ```python
  GROUP_ID_RE: re.Pattern[str]          # r"G[1-9][0-9]*"
  RECORD_PREFIX = "release-group "

  class GroupRecordError(Exception): ...  # names the epic, the group and why

  @dataclass(frozen=True)
  class ParsedRecord:
      group: str
      rev: int
      members: tuple[epics.IssueRef, ...]   # () when dissolved
      dissolved: bool
      reason: str | None

  @dataclass(frozen=True)
  class Group:
      epic: epics.IssueRef
      group: str
      rev: int
      members: tuple[epics.IssueRef, ...]
      dissolved: bool
      comment_id: str

  @dataclass(frozen=True)
  class GroupLog:
      epic: epics.IssueRef
      groups: Mapping[str, Group]   # every readable group ever recorded, dissolved included
      broken: Mapping[str, GroupRecordError]   # groups whose reads refuse, by ID
      epic_error: GroupRecordError | None      # a failure that names no group
      ids_used: frozenset[str]                 # broken groups' IDs included

  def parse_record(body: str) -> ParsedRecord | None
      # None when the first line does not start RECORD_PREFIX; GroupRecordError when it does but is malformed
  def render_record(group: str, rev: int, members: Sequence[epics.IssueRef], *,
                    dissolved: bool = False, reason: str | None = None) -> str
  def read_log(epic: epics.IssueRef, *, authors: TrustedAuthors | None = None) -> GroupLog
      # raises only on a read failure; a bad record goes into broken or epic_error
  def groups_of(task: epics.IssueRef, *, authors: TrustedAuthors | None = None) -> list[Group]
      # the live (not dissolved) groups of task's epic (epics.task_epic) holding task;
      # raises GroupRecordError when epic_error is set, or when a broken group
      # may hold task: task is named in any readable record of the group (any
      # revision, either side of a conflict, an edited record's current text),
      # or the group has no readable record
  ```

  `read_log` takes the skipped comments from the stream's `skipped` list
  as metadata only, never reading their text, and scopes each failure as
  the Global Constraints say: a comment skipped for an outsider's edit
  sets `epic_error`. A broken group's records are used only to refuse,
  never to allow.

- Produces: `vrg-release-group list --epic <owner>/<repo>#<N> [--group <id>]
  [--json]`. Exit 0 on a read, 1 on any refusal or read failure (stderr
  names it), 2 on a usage error. JSON:
  `{"epic": "<ref>", "groups": [{"group": "G1", "rev": 3, "members":
  ["<ref>", …], "dissolved": false}]}`, groups in ID order. Plain output
  is one line per group: `G1 rev 3: <ref> <ref>` or `G2 rev 2: dissolved`.

- [ ] **Step 1: Write the failing tests** (comment streams stubbed at
  `trust.read_comment_stream`):
  - `test_parse_record_members_and_dissolved`: both grammars parse;
    `Reason: split 13` is read as the reason.
  - `test_parse_record_malformed_raises`: `release-group G01 rev 1: …`,
    `rev 0`, a bare `#5` member, a member listed twice, trailing text →
    each `GroupRecordError`.
  - `test_latest_revision_wins`: records rev 1, 2, 3 → rev 3's members.
  - `test_conflict_at_top_revision_refuses`: two rev 4 records → reading
    the group raises, naming both comment IDs; once one is deleted, the
    other reads cleanly.
  - `test_out_of_order_revision_refuses`: rev 3 posted after rev 4 → raises.
  - `test_edited_record_refuses`: the top record has `edited=True` → raises,
    naming the comment; once it is deleted, the record below it is in force.
  - `test_untrusted_edit_of_trusted_comment_refuses_epic`: a trusted
    comment the stream skipped for an outsider's edit, whether or not it was
    a group record → every read on the epic raises, a standalone task's
    `groups_of` included, naming the comment ID. The stub's skipped entries
    carry no text, and the test fails if any other read of the comment is
    made.
  - `test_revision_gap_reads`: records rev 1 and rev 3 (rev 2 deleted) →
    reads cleanly as rev 3's members.
  - `test_broken_group_may_hold_conflict_side`: two rev 4 records, only
    one of which adds T (rev 3 does not hold T) → `groups_of(T)` raises.
  - `test_broken_group_may_hold_edited_text`: rev 3 holds T and the edited
    rev 4's current text drops it → `groups_of(T)` raises.
  - `test_untrusted_record_ignored_and_reported`: an outsider's
    `release-group G1 rev 9: …` changes nothing and is reported on stderr.
  - `test_conflict_in_other_group_does_not_block`: `G2` in conflict →
    `groups_of` for a `G1` member reads cleanly.
  - `test_standalone_task_unaffected_by_broken_group`: `G1` broken →
    `groups_of` for a task in no group returns `[]`.
  - `test_unscoped_failure_refuses_every_read`: a malformed record with no
    readable group ID → `groups_of` raises for every task of the epic.
  - `test_member_in_two_live_groups_refuses`.
  - `test_record_on_ad_hoc_epic_refuses`,
    `test_record_on_management_epic_refuses` and
    `test_record_on_queue_epic_refuses` (a planning epic).
  - `test_group_on_proxy_epic_reads`: a record on a proxy whose `Primary:`
    link names a development epic; `groups_of` for a proxy task returns
    its group from the proxy epic.
  - `test_record_on_proxy_with_broken_link_refuses` (a proxy whose
    `Primary:` link has problems, or names an ad-hoc epic).
  - `test_groups_of_resolves_continuation_to_head` (Review Focus 3).
  - `test_dissolved_id_stays_used`: `ids_used` holds a dissolved group's ID.
  - `test_cli_list_json_and_plain`, `test_cli_list_read_failure_exits_1`.
- [ ] **Step 2: Run them and see them fail.**
  `vrg-container-run -- pytest tests/vergil_tooling/test_release_group_record.py
  tests/vergil_tooling/test_vrg_release_group.py` → FAIL.
- [ ] **Step 3: Implement** as the Global Constraints state the grammar,
  the latest-record rule and the trust rules. `read_log` reads the epic
  once; `groups_of` resolves the epic with `epics.task_epic` (a proxy
  task's epic is its proxy) and refuses when the epic is not a group's
  epic (Global Constraints) and the log holds any record.
- [ ] **Step 4: Pass, validate, commit.**

### Task 3: Release the group record (deployment)

- Precondition: `vrg-release-record check vergil-project/vergil-tooling#<Task 2>`
  exits 0.
- Procedure: that check, alone; the release installed itself.
- Acceptance: the check passes; the result names the release version.

---

## Phase 2: hardening the batch mechanism

### Task 4: Chained review records in the gate (vergil-tooling)

Spec §5.4.

**Files:**

- Modify: `src/vergil_tooling/lib/agent_gates.py` (`_Lookups`,
  `_merge_review_reasons`)
- Modify: `src/vergil_tooling/lib/merge_review/record.py` (`Record` gains
  `at: str`, the comment's `createdAt`; the record line does not change)
- Modify: `src/vergil_tooling/lib/rulings/approvals.py` (`Approval` gains
  `at: str`, the time of the human's approving reply)
- Test: `tests/vergil_tooling/test_agent_gates.py`,
  `tests/vergil_tooling/test_merge_review_record.py`,
  `tests/vergil_tooling/test_rulings_approvals.py`

**Interfaces:**

- Produces: `record.Record.at: str` (ISO 8601, from the comment stream's
  `at`). The stored format is unchanged, so no record migrates.
- Produces: `rulings.approvals.Approval.at: str`, the approving reply's
  time, never the request's.
- Consumes: `rulings.approvals.verified_approvals` (an exemption's
  coverage time is its approving reply's time).

The chain, computed at check time from what the records already hold, with
no git object other than the branch's own commits needed:

```text
c(0) = -infinity
for k = 1..n (the batch's issue commits, in order):
    c(k) = the earliest time of a record or approved exemption on issue k
           whose fingerprint equals commit_fingerprint(commit k)
           and whose time is at or after c(k-1)
    if none: commit k and every later commit are uncovered
```

A record posted before the review of the current content of an earlier
commit therefore no longer covers its commit. A clean rebase changes no
fingerprint, so it voids nothing. Records built in order (every batch to
date) pass as they are. A record covering the whole change still covers
every commit, as today.

- [ ] **Step 1: Write the failing tests** (a three-commit batch built in a
  real git repository, record streams stubbed):
  - `test_chain_fold_into_middle_voids_later_records`: records for 1, 2, 3
    in order; a fold changes commit 2; a fresh record for 2 → commit 3 is
    refused, and the reason names commit 3 and says its record predates the
    review of commit 2.
  - `test_chain_rereview_of_later_commits_restores`: then a fresh record for
    3 → passes.
  - `test_chain_clean_rebase_voids_nothing`.
  - `test_chain_fix_to_last_commit_needs_only_last`.
  - `test_chain_records_in_build_order_pass` (the no-migration case:
    records written before this task, in build order, pass unchanged).
  - `test_chain_exemption_counts_at_its_approval_time`: an exemption
    requested before an earlier commit's re-review and approved after it
    covers its commit; one approved before that re-review does not.
  - `test_whole_change_record_still_covers_every_commit`.
- [ ] **Step 2: Run them and see them fail.**
- [ ] **Step 3: Implement** the chain in `_merge_review_reasons`'
  per-commit branch; `_Lookups` keeps each issue's `(fingerprint, at)`
  pairs instead of fingerprints alone.
- [ ] **Step 4: Pass, validate, commit.**

### Task 5: Real-git batch fixture and the fold, conflict and rebase tests (vergil-tooling)

Spec §5.7 (the first four cases, and #1140).

**Files:**

- Create: `tests/vergil_tooling/batch_repo.py` (the shared fixture)
- Create: `tests/vergil_tooling/test_batch_mechanics.py`
- Modify: whatever module a test exposes a gap in (named in the PR)

**Interfaces:**

- Produces, in `tests/vergil_tooling/batch_repo.py`: a pytest fixture
  `batch_repo(tmp_path) -> BatchRepo` making a bare `origin` with a
  `develop` branch and a clone; `BatchRepo.commit_issue(number: int, files:
  Mapping[str, str]) -> str` (a commit whose subject ends `(#<number>)`),
  `BatchRepo.fixup(sha: str, files) -> str`, `BatchRepo.autosquash() ->
  None` (raises `RebaseConflict` on a conflict, leaving the rebase
  aborted), `BatchRepo.advance_develop(files) -> str`. GitHub reads
  (issues, comment streams, closing refs) are stubbed; git is real.
  Task 6 and Task 11 reuse it.

- [ ] **Step 1: Write the tests:**
  - `test_fold_into_middle_of_chain_of_four`: fixup to commit 2 of 4,
    autosquash → `check_commit_per_issue` passes and the gate (Task 4)
    refuses commits 3 and 4 (their re-review is Task 6a's).
  - `test_fold_conflicting_with_later_commit`: the fixup touches a line
    commit 3 changes → `autosquash` raises, the branch is unchanged, and
    `check_commit_per_issue` refuses the unfolded `fixup!` commit by name.
  - `test_long_lived_batch_rebased_onto_moved_develop`: develop advances
    twice; a clean rebase → `check_commit_per_issue` and the gate pass with
    the existing records.
  - `test_rebase_after_merge_review_keeps_records` (vergil-claude-plugin#1140):
    each issue reviewed, then a clean rebase → records still cover.
- [ ] **Step 2: Run them.** Any test that fails exposes a gap.
- [ ] **Step 3: Fix each gap** in the module it names, test-first, in this
  PR. A gap too large for this PR is a split, which `issue-implement`
  proposes to the human.
- [ ] **Step 4: Pass, validate, commit.**

### Task 6: Batch review-record tests: autosquash, clean rebase, a fix after submit (vergil-tooling)

Spec §5.7 (records after autosquash and rebase; a fix after submit with
`batch-check`).

**Files:**

- Modify: `tests/vergil_tooling/test_batch_mechanics.py`
- Modify: whatever module a test exposes a gap in

**Interfaces:**

- Consumes: `batch_repo` (Task 5); `pr_path.push_to_open_pr` and
  `vrg-pr-workflow batch-check` as they stand.

- [ ] **Step 1: Write the tests:**
  - `test_records_name_commits_after_autosquash`: a record written by
    `record.batch_member_change` before the fold, then a fold → only the
    folded commit's fingerprint covers it; the `fixup!` commit is never a
    covered commit.
  - `test_records_after_autosquash_then_clean_rebase`.
  - `test_fix_after_submit_through_push_mode`: an open batch PR (stubbed),
    a fold into commit 2, a whole-branch merge-review record →
    `push_to_open_pr` pushes; without the record it refuses and does not
    push.
  - `test_fix_after_submit_batch_check_passes_and_refuses`: `batch-check`
    passes on the folded branch, and refuses one with an unfolded `fixup!`.
- [ ] **Step 2–4:** as Task 5.

### Task 6a: A record for a middle batch commit, and the gate probe (vergil-tooling)

Spec §5.4 (the gate enforces re-review before handoff, so re-review must
be possible before handoff).

Today `record.batch_member_change` refuses unless the issue under review is
the last one on the branch ("… must be the last issue on the branch"), so
after a fold into commit k nothing can record a fresh review of issue k, or
of any later issue but the last, while later commits sit on the branch.
And no command evaluates the gate before report-ready:
`vrg-submit-pr --dry-run` returns before the gates run.

**Files:**

- Modify: `src/vergil_tooling/lib/merge_review/record.py`
  (`batch_member_change`, `covered_change`)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (a new
  `batch-gates` subcommand)
- Modify: `src/vergil_tooling/lib/agent_gates.py` (expose the merge-review
  evaluation the probe reuses, unchanged in behaviour)
- Test: `tests/vergil_tooling/test_batch_mechanics.py`,
  `tests/vergil_tooling/test_merge_review_record.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`

**Interfaces:**

- Produces: `batch_member_change` (and `covered_change`) give the change
  for issue k as issue k's own commit, wherever it sits in the batch, its
  fixups to that commit folded; later issues' commits are not part of it.
  Its fold base is that commit's parent, the first issue's included: the
  marker's `base_sha` is no longer used to find any issue's change, so a
  clean rebase onto a moved `develop` changes no issue's change and leaves
  the markers as they are.
- Produces: `vrg-pr-workflow batch-gates [--json]`, run in a batch's
  worktree before report-ready. It reads the batch's issues from the
  clone's markers, takes the base from a freshly fetched `origin/develop`
  as `vrg-submit-pr` does, and evaluates the gate's merge-review check
  exactly as `vrg-submit-pr` would, commit by commit. After report-ready
  (no markers left) it refuses with exit 1, saying the probe runs only
  before handoff. It writes nothing. JSON: `{"branch": "<name>", "head":
  "<sha>", "commits": [{"issue": "<ref>", "sha": "<sha>", "covered": bool,
  "reason": str | null}]}`; exit 0 when every commit is covered, 1 when
  any is not or on a read failure (stderr names it), 2 on usage.

- [ ] **Step 1: Write the failing tests** (real git through `batch_repo`;
  the record path itself is not stubbed, only GitHub reads):
  - `test_member_record_for_middle_commit_with_later_commits`: a batch of
    four, a fold into commit 2 → a record for issue 2 is written, and its
    fingerprint is commit 2's.
  - `test_rereview_restores_coverage_after_middle_fold`: then records for
    3 and 4 → the gate passes every commit (completes Task 5's
    `test_fold_into_middle_of_chain_of_four`).
  - `test_member_change_excludes_later_commits`.
  - `test_batch_gates_reports_uncovered_after_fold` and
    `test_batch_gates_all_covered_exits_0`.
  - `test_batch_gates_before_report_ready`: no handoff state needed.
  - `test_batch_gates_after_report_ready_refuses`: no markers → exit 1,
    naming the reason.
  - `test_member_record_for_first_commit_after_rebase`: a batch rebased
    onto a moved `develop`, then a fold into commit 1 → a record for issue
    1 is written, and its change holds only commit 1.
  - `test_batch_gates_after_rebase_onto_moved_develop`: every commit still
    reported covered; `develop`'s new commit is not part of the batch.
  - `test_batch_gates_matches_submit_gate`: the probe and
    `vrg-submit-pr`'s gate give the same verdict per commit.
  - `test_batch_gates_writes_nothing` and
    `test_batch_gates_read_failure_exits_1`.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 6b: Re-review after a rewrite in the skills (vergil-claude-plugin)

Spec §5.4, §5.7.

**Files:** Modify `skills/merge-review/SKILL.md` ("In a batch"),
`skills/issue-implement/SKILL.md` (its batch sections), and their pages
under `docs/site/docs/skills/`.

The changes:

- **merge-review, "In a batch":** a run may review issue k while later
  issues' commits sit on the branch, after a fold into k's commit; its
  fixups are folded into k's own commit only.
- **issue-implement:** a fix to a finished issue's commit in a running
  batch is folded into that commit (`fixup!` and autosquash), and that
  issue and every later one are merge-reviewed again before handoff;
  `vrg-pr-workflow batch-gates` confirms coverage before report-ready. A
  clean rebase of the batch branch onto `origin/develop` before handoff is
  allowed, voids no record and leaves the markers as they are; a rebase
  that conflicts is a batch stop.
  The rules "never reopening a finished issue" and "a batch never rebases
  onto `develop` before handoff" are rewritten to match.
- The batch size wording ("up to 8") is left for Task 19a, which changes
  it once the ceiling ships (Task 9).

- [ ] **Step 1–2:** as Task 19a.

### Task 7: Release the hardened mechanism (deployment)

- Precondition: `vrg-release-record check` exits 0 for each of
  `vergil-project/vergil-tooling#<Task 4>`, `#<Task 5>`, `#<Task 6>` and
  `#<Task 6a>`.
- Procedure: those checks, alone.
- Acceptance: every check passes; the result names the release version.

### Task 7a: Release the re-review skills (deployment)

- Precondition: `vrg-release-record check
  vergil-project/vergil-claude-plugin#<Task 6b>` exits 0, and Task 7 is
  closed.
- Procedure: that check.
- Human-run steps: `/reload-plugins` in each running Claude Code session.
- Acceptance: the check passes, and the human has confirmed the reloads.

### Task 8: The live run (validation)

Spec §5.7. Filed in `vergil-project/vergil-fixture-python-lib`, where the
batch PR lands.

- **Intro.** Drives a deliberately awkward chain batch through the fixture
  with the real skills against released Vergil, proving the batch
  mechanics groups depend on. No group record is written. The run files
  its own chain of change tasks (Appendix A), so the chain is never
  runnable before this run starts it as a batch.
- **Preconditions** (machine probes): `vrg-release-record check
  vergil-project/vergil-tooling#<Task 4>`, `#<Task 5>`, `#<Task 6>` and
  `#<Task 6a>`, and `vrg-release-record check
  vergil-project/vergil-claude-plugin#<Task 6b>`, each exit 0; Task 7a is
  closed (the sessions have reloaded); the fixture's managed files match the released
  vergil-tooling, as the `fixture-run` skill's preconditions check it; no
  open PR into the fixture's `develop`.
- **Procedure:**
  1. File the six chain tasks of Appendix A, 8.1 to 8.6, under
     vergil-project/.github#1672 with `vrg-issue-create --epic
     vergil-project/.github#1672 --repo
     vergil-project/vergil-fixture-python-lib --title … --body-file …`,
     in order, each 8.k (k ≥ 2) with `--blocked-by` 8.(k−1). Record their
     refs in a progress comment on this task. 8.0 is not filed yet: it is
     filed at step 5, so no driver can merge it before the batch starts.
  2. Start the batch at once, from the fixture's main clone:
     `vrg-pr-workflow batch-start --issue <8.1> … --issue <8.6>`, so the
     markers route any dispatch of a member to this batch.
  3. Run `issue-implement` in batch mode on 8.1–8.6, in order, to the end
     of 8.4 (each committed and merge-reviewed).
  4. **The forced fix to the second commit.** Fold the fix Appendix A
     gives for 8.2 into its commit (`fixup!` and autosquash). Confirm the
     gate refuses commits 3 and 4: `vrg-pr-workflow batch-gates --json`
     reports both as uncovered. Re-run merge review on 8.2, 8.3 and 8.4,
     and confirm `batch-gates` then reports every commit covered.
  5. **`develop` moves.** File 8.0 as step 1 files the others (no
     `--blocked-by`), record its ref in a progress comment on this task,
     then implement it as its own single-issue PR and watch it to the
     merge. Then rebase the batch branch onto the new
     `origin/develop` (a clean rebase) and confirm `batch-gates` still
     reports 8.1–8.4 covered.
  6. Finish 8.5 and 8.6, hand off the batch (`report-ready`) and submit it
     with `vrg-submit-pr`. 8.4's commit carries the planned CI-only
     failure (Appendix A), so the PR's CI fails and submit reports the PR
     not merged.
  7. **The fix after submit.** Through `pr-watch`'s ordinary fix path, fold
     the after-submit fix of Appendix A into 8.4's commit: merge review of
     the whole branch, `vrg-pr-workflow batch-check`, and the push with
     `vrg-submit-pr --push`, then `vrg-finalize-pr`. Watch the PR to the
     merge with `vrg-pr-await`.
- **Acceptance:** one merged PR closes 8.1–8.6, rebase-merged, with exactly
  six commits on `develop`, one per task in order, each subject ending in
  its issue's ref; step 4's refusal and recovery were seen in
  `batch-gates`' output; step 5's rebase kept the records; the PR's CI
  failed on the planned failure and passed after step 7's push, which the
  PR's timeline shows before the merge. If the PR merged without step 7's
  push, the outcome is `FAILURE` ("the fix after submit was not
  exercised"), and a re-run files a fresh chain whose keys end `-r2`.

---

## Phase 3: enforcing groups

### Task 9: The ceiling (vergil-tooling)

Spec §5.2 (size).

**Files:** Modify `src/vergil_tooling/lib/pr_workflow/batch_rules.py`
(`MAX_BATCH = 20`, the module and `check_hard_rules` docstrings say "2 to
20"); every other statement of the old limit in vergil-tooling: the
`vrg-pr-workflow` help ("a batch PR of 2 to 8 same-epic issues",
`src/vergil_tooling/bin/vrg_pr_workflow.py`), the
`src/vergil_tooling/bin/vrg_submit_pr.py` module docstring,
`docs/specs/worktree-convention.md`,
`docs/site/docs/reference/dev/submit-pr.md` ("2 to 8 issues of one epic"),
`docs/site/docs/reference/cli-tools-overview.md` ("The batch hard
rules are: 2 to 8 issues"), `docs/site/docs/standards/github-issues.md`
("A batch PR (2 to 8 small issues of…") and
`docs/site/docs/guides/git-workflow.md` ("2 to 8 small issues"), the
last two also dropping "small", since the size guideline is removed; Test
`tests/vergil_tooling/pr_workflow/test_batch_rules.py`.

- [ ] **Step 1:** `test_twenty_issues_allowed`, `test_twenty_one_refused`
  (the refusal names the ceiling), and the existing count tests updated.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 10: The group rules and the holder check (vergil-tooling)

Spec §5.1 (the group rules), §5.5 (every write checked; incoming tasks
clean; when a group can change).

**Files:**

- Create: `src/vergil_tooling/lib/release_groups/rules.py`
- Test: `tests/vergil_tooling/test_release_group_rules.py`

**Interfaces:**

- Consumes: `record.read_log`, `record.Group` (Task 2);
  `batch_rules.MAX_BATCH`; `epics.task_epic`, `epics.blockers_of`;
  `epic_audit.proxy_link_problems` (a proxy's links);
  `batch_marker.read_marker`; `github_transport.list_relay_branches` and
  the relay and local workflow-state readers; GitHub's
  `closedByPullRequestsReferences`.
- Produces:

  ```python
  @dataclass(frozen=True)
  class RuleBreak:
      group: str
      rule: Literal["open code task", "one repo", "not operational",
                    "not a documentation review", "dependency path",
                    "size", "group's epic", "same epic"]
      detail: str

  def check_group_rules(epic: epics.IssueRef, group: str,
                        members: Sequence[epics.IssueRef]) -> list[RuleBreak]
  def dependency_paths(members: Sequence[epics.IssueRef]
                       ) -> list[tuple[epics.IssueRef, epics.IssueRef, epics.IssueRef]]
      # (from member, through outside task, to member), open outside tasks only

  @dataclass(frozen=True)
  class Holder:
      task: epics.IssueRef
      kind: Literal["open PR", "handed-off batch", "batch marker"]
      where: str    # the PR URL, the branch, or the marker's batch

  def holders(task: epics.IssueRef, *, main_root: Path) -> list[Holder]
  ```

  Every read failure raises `GroupRuleReadError`, never an empty list.

The rules, as §5.1 gives them: each member open, a task (not an epic), in
one repo, under `epic` (the same epic), not labelled `validation` or
`deployment`, not a documentation review (its title starts
`Documentation review:` or its body carries `Plan task: doc-review`); 1 to
`MAX_BATCH` members; the epic a group's epic (Global Constraints: a
development epic, or a proxy of one with sound links); no dependency path from
one member to another through an open task outside the group (`Blocked-by`
lines and native links, followed transitively, across epics and repos; a
closed task cannot wait, so a path through one does not count). The
design-question rule stays a planning judgment and is not checked.

- [ ] **Step 1: Write the failing tests:** one per rule, each refused with
  its rule's name (`test_rule_<rule>_refused`, eight tests; the group's
  epic rule is tested with an ad-hoc and a management epic);
  `test_proxy_epic_group_valid` (a proxy's own-org tasks in one repo);
  `test_dependency_path_through_open_deployment`;
  `test_dependency_path_through_closed_task_ignored`;
  `test_dependency_path_across_epics`; `test_one_member_group_valid`;
  `test_holder_open_pr`, `test_holder_handed_off_batch_relay`,
  `test_holder_handed_off_batch_local`, `test_holder_batch_marker`;
  `test_holder_read_failure_raises`; `test_rules_read_failure_raises`
  (`check_group_rules` and `dependency_paths` raise `GroupRuleReadError`
  on a failed issue read, blocker read and proxy-link read).
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 11: The whole-group rule on every PR path (vergil-tooling)

Spec §5.2.

**Files:**

- Create: `src/vergil_tooling/lib/release_groups/pr_check.py`
- Modify: `src/vergil_tooling/lib/pr_workflow/batch_rules.py`
  (`check_hard_rules` calls it, with the batch's order)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (`cmd_report_ready`
  calls it for a single-issue PR too; `cmd_batch_check` calls it)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (`_run_gates` calls it,
  with the batch's order for a batch, in every mode, single-issue and the
  CLI emergency mode included, before any push or `gh pr create`; a refusal
  says nothing was pushed and no PR was opened, as submit's other gate
  refusals do)
- Modify: `src/vergil_tooling/lib/pr_path.py` (`arm` and `push_to_open_pr`
  call it among their judgments, after the provenance check and before
  `_through_the_gate` and `_enable_bounded`, so both the release lock's
  `holder_pass` branch and its arming-handshake branch reach it, and so do
  `arm`'s callers: the release back-merge, the re-arm sweep
  (`release_lock.sweeps`) and the merge watch's re-arm (`LockWatch`); for
  every PR into `develop`, agent involved or not; a refusal is the
  `refused` outcome, or `pushed=False`; for a PR that submit opened, this
  is the backstop for a group that broke between submit's gate and `arm`)
- Test: `tests/vergil_tooling/test_release_group_pr_check.py`,
  `tests/vergil_tooling/test_pr_path.py`,
  `tests/vergil_tooling/test_vrg_pr_workflow.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Consumes: `record.groups_of` (Task 2); `closing_refs.read_closing_refs`;
  `batch_rules.BatchRuleError`.
- Produces:

  ```python
  def check_whole_groups(closing: Sequence[epics.IssueRef], *,
                         order: Sequence[epics.IssueRef] | None = None) -> None
      # raises batch_rules.BatchRuleError(rule="whole groups" | "group order", ...)
  ```

  The message names each group, its epic, and its missing members (or the
  members out of order). At `arm` and `push_to_open_pr`, *closing* is a
  superset of every issue GitHub will close at the merge: every entry of
  the PR's `closingIssuesReferences`, `counted` and `uncounted` alike and
  in any repo, plus every issue the body names with a closing keyword (the
  whole autoclose family, `AUTOCLOSE_RE`, not only `Closes`). An
  untrusted reference is used here only to refuse, never to allow.
  *order* is the batch's commit order when the PR is a batch.

- [ ] **Step 1: Write the failing tests:**
  - `test_single_issue_of_group_refused` at `report-ready`, `batch-check`,
    `arm` and `push_to_open_pr` (four tests), naming the missing members;
  - `test_submit_modes_all_reach_the_rule`: template, `--all`/`--select`,
    named-branch local and relay, and the CLI emergency mode are each
    refused at submit's gate with no push and no PR opened (`gh pr create`
    never called), and `--push` is refused at `push_to_open_pr` (one
    parametrized test);
  - `test_grouped_single_issue_submit_opens_no_pr`: a single-issue submit
    of a grouped task is refused before any push, and no PR exists after
    it;
  - `test_arm_applies_rule_without_agent_involvement` and
    `test_push_applies_rule_without_agent_involvement` (a human PR);
  - `test_rearm_sweep_partial_group_refused` and
    `test_merge_watch_rearm_partial_group_refused`: a re-arm by
    `release_lock.sweeps` and by `LockWatch` of a PR holding part of a
    group is refused, and the refusal is reported as each caller reports
    a refused arm;
  - `test_back_merge_holder_pass_reads_no_group`: a release back-merge PR,
    which closes no issue, passes the `holder_pass` branch with no group
    read;
  - `test_batch_start_partial_group_refused` (a batch holding part of a
    group);
  - `test_arm_reads_github_closing_refs_not_body_only` (Review Focus 2);
  - `test_uncounted_fixes_reference_counts_for_rule`,
    `test_cross_repo_closing_reference_counts_for_rule` and
    `test_untrusted_body_reference_counts_for_rule`: a group member named
    only in each such way is in *closing*, so a PR holding part of a group
    that way is refused;
  - `test_whole_group_and_standalone_allowed`;
  - `test_interleaved_groups_keep_own_order` (Review Focus 5);
  - `test_group_order_violation_refused`;
  - `test_one_member_group_single_pr_allowed`, and
    `test_one_member_group_alone_batch_start_refused` (a batch takes at
    least two issues, so a lone one-member group goes to the single-issue
    path, §5.3);
  - `test_edited_record_refuses_pr` and `test_unreadable_epic_refuses_pr`;
  - `test_no_records_refuses_nothing` (the rule is live with no group).
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 12: `vrg-release-group set` and `dissolve` (vergil-tooling)

Spec §5.1 (the tool; safe to retry), §5.5.

**Files:**

- Create: `src/vergil_tooling/lib/release_groups/write.py`
- Modify: `src/vergil_tooling/bin/vrg_release_group.py`
- Test: `tests/vergil_tooling/test_release_group_write.py`,
  `tests/vergil_tooling/test_vrg_release_group.py`

**Interfaces:**

- Consumes: `record.read_log`, `record.render_record` (Task 2);
  `rules.check_group_rules`, `rules.holders` (Task 10).
- Produces:
  - `vrg-release-group set --epic <ref> --group <id> --member <ref>
    [--member <ref> …] --reason <text> [--at-batch-stop]`
  - `vrg-release-group dissolve --epic <ref> --group <id> --reason <text>
    [--at-batch-stop]`
  - Exit 0 when the record is posted or already the latest (prints
    `G1 rev 3: already recorded`), 1 on any refusal or failure, 2 on usage.
  - `write.set_group(epic, group, members, *, reason, at_batch_stop,
    main_root) -> WriteResult(posted: bool, rev: int)`.

Behaviour:

- Re-read the log first; when the latest record already holds the requested
  membership, post nothing (safe to retry).
- A new ID must be unused (`ids_used`); a dissolved ID is refused.
- Check the requested membership with `check_group_rules`; refuse on any
  break. The rules that need an open member (open, and the dependency
  path) are checked only against the members that are open, as the audit
  does (Task 17); the structural rules (a task, one repo, the same
  development epic, not operational, not a documentation review, the size)
  apply to every member.
- Every record posted needs a non-empty `--reason`, written as the
  record's `Reason:` line: a new group, an addition, a merge of two
  groups, a removal and `dissolve` alike. A retry that posts nothing
  needs none.
- A new record is posted at one past the group's latest revision.
- Every **incoming** task (not in the latest record, by an addition or a
  merge of two groups) must be open and have no holder, so an addition
  never itself makes a broken group. The open-only relaxation above covers
  only members already in the latest record. One exception: with
  `--at-batch-stop`, an incoming task whose only holder is a `batch marker`
  of the same batch as the group's members' markers is accepted (spec
  §5.5); it is already in that batch. Any other holder still refuses.
- **Frozen:** while any open member has an `open PR` or `handed-off batch`
  holder, every change is refused, naming the holder and saying the PR must
  be taken back first (`vrg-pr-workflow unfreeze`, or closing the PR).
- **Started:** while any member has a `batch marker` holder, the change is
  refused, naming the batch and saying it stops for the human, unless
  `--at-batch-stop` is given (the skills pass it only after the human has
  ruled a group change at a batch stop). The frozen rule still applies.
  The markers read are this clone's: with one driver per epic (Global
  Constraints), that is where the epic's batches run.
- A **conflict**, an **edited** record, or a comment skipped for an
  outsider's edit (Global Constraints) refuses, naming the comments and
  the human's recovery: read them in the web UI and delete the bad one.
  Any other read failure names what failed and says the human must look at
  it.
- Run from the main clone of the members' repo (its `origin`), since
  batch markers are local; refuse elsewhere.
- A group in a closed epic is never changed: `set` and `dissolve` refuse
  it, since the sweep no longer checks it and there is nothing to repair
  (spec §5.6).

- [ ] **Step 1: Write the failing tests:** `test_set_new_group_rev_1`;
  `test_set_on_proxy_epic_posts_on_proxy`;
  `test_set_on_ad_hoc_or_management_epic_refused`;
  `test_set_retry_posts_nothing`; `test_set_dissolved_id_refused`;
  `test_set_rule_break_refused`; `test_set_dependency_path_refused` (a
  path from one member to another through an open task outside the
  group); `test_set_without_reason_refused` (a new
  group, an addition, a removal and `dissolve`; a retry that posts nothing
  needs none);
  `test_dissolve_records_no_members`; `test_incoming_task_with_holder_refused`
  (one per holder kind); `test_incoming_closed_task_refused` (an addition,
  and a merge of two groups); `test_change_refused_while_frozen` (open PR, and a
  handed-off batch); `test_change_after_batch_start_stops_for_human` and
  `test_at_batch_stop_allows_marker_only`;
  `test_at_batch_stop_incoming_same_batch_marker_allowed` (a standalone
  task in the running batch joins the group) and
  `test_at_batch_stop_incoming_other_holder_refused` (another batch's
  marker, an open PR or a handed-off batch);
  `test_set_refuses_on_conflict` (Review Focus 1: naming both comments and
  the recovery; once one is deleted, `set` posts the next revision);
  `test_set_refuses_on_edited_record` (naming the comment; once it is
  deleted, `set` posts above the latest remaining revision);
  `test_skipped_edit_refuses_with_recovery` (while a comment skipped for
  an outsider's edit is on the epic, `set` refuses, naming the comment and
  the recovery);
  `test_split_is_one_record` (13 replaced by 13a and 13b, with a reason);
  `test_change_in_closed_epic_refused`; `test_open_rules_skip_closed_members`
  (a repair on an open epic that keeps closed, shipped members is posted);
  `test_run_outside_members_clone_refused`.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 13: `vrg-gh pr edit` keeps a develop PR's closing lines (vergil-tooling)

Spec §5.2 (the PR's closing lines stay as checked).

**Files:** Modify `src/vergil_tooling/bin/vrg_gh.py` (a `pr edit` guard
beside the `pr update-branch` one); Test `tests/vergil_tooling/test_vrg_gh.py`.

**Interfaces:**

- Consumes: the `linkage` module's line parser (`Closes` and `Ref` lines,
  plain or in `- <label>:` form) and the autoclose pattern
  (`AUTOCLOSE_RE`: `Fixes`, `Resolves` and the rest of the family).
- Produces: `_closing_line_refusal(args: list[str]) -> str | None`.

`pr edit` with `--body`, `-b`, `--body-file` or `-F` on a PR whose base is
`develop` reads the current body (`gh pr view --json body,baseRefName`) and
refuses when the multiset of `Ref` lines and closing-keyword references
(the whole autoclose family) differs. Any other base,
and any edit that leaves the body alone, passes. A read failure refuses.
The guard applies to every identity.

- [ ] **Step 1:** `test_pr_edit_adding_closes_line_refused`,
  `test_pr_edit_removing_ref_line_refused`,
  `test_pr_edit_changing_ref_target_refused`,
  `test_pr_edit_adding_fixes_line_refused`,
  `test_pr_edit_adding_resolves_line_refused`,
  `test_pr_edit_prose_only_allowed`, `test_pr_edit_non_develop_allowed`,
  `test_pr_edit_title_only_allowed`, `test_pr_edit_read_failure_refused`.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 14: Move, route and transfer of a grouped task refused (vergil-tooling)

Spec §5.1 (a grouped task stays where its group is).

**Files:** Modify `src/vergil_tooling/bin/vrg_epic_move.py`,
`src/vergil_tooling/lib/intake_route.py` (before any write),
`src/vergil_tooling/bin/vrg_gh.py` (`issue transfer`); Test
`tests/vergil_tooling/test_vrg_epic_move.py`,
`tests/vergil_tooling/test_intake_route.py`,
`tests/vergil_tooling/test_vrg_gh.py`.

**Interfaces:** Consumes `record.groups_of` (Task 2). Each refusal names
the group and its epic and says a recorded change
(`vrg-release-group set` or `dissolve`) must remove the task first.

- [ ] **Step 1:** `test_move_grouped_task_refused`,
  `test_route_grouped_task_refused`, `test_transfer_grouped_task_refused`,
  `test_move_after_dissolve_allowed`, and a read-failure refusal for each
  of the three.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 15: `batch-extract` takes a set, and a split is refused (vergil-tooling)

Spec §5.5 (the check comes before anything is discarded).

**Files:** Modify `src/vergil_tooling/bin/vrg_pr_workflow.py`
(`cmd_batch_extract`, its parser), `src/vergil_tooling/lib/pr_workflow/batch_marker.py`
(`extract` takes a list); Test `tests/vergil_tooling/test_vrg_pr_workflow.py`,
`tests/vergil_tooling/pr_workflow/test_batch_marker.py`.

**Interfaces:**

- Produces: `vrg-pr-workflow batch-extract --issue <N> [--issue <M> …]
  [--check]`. The set leaves the batch in one marker rewrite. A set that
  would split a group (a group with members both leaving and staying) is
  refused, naming the group, before any marker changes. `--check` runs the
  same judgment, writes nothing, and prints `{"ok": bool, "split":
  [{"group", "leaving", "staying"}], "remaining": [...]}`; exit 0 when
  `ok`, 1 otherwise.
- Produces: `batch_marker.extract(root: Path, issues: Sequence[str]) ->
  list[str]`; the existing security-file refusal applies to each issue.

- [ ] **Step 1:** `test_extract_split_refused_before_marker_change`,
  `test_extract_whole_group_allowed`, `test_extract_standalone_allowed`,
  `test_handoff_set_whole_groups_allowed`, `test_unbatch_all_allowed`
  (a batch holding a one-member group among others: its member leaves
  with the rest, and a single-issue PR of it then passes the rule),
  `test_check_writes_nothing`, `test_check_read_failure_exits_1`,
  `test_extract_set_one_marker_rewrite`, and
  `test_collapsed_batch_under_group_rules` (an extraction of whole groups
  that leaves one issue removes its marker, and that issue's group, whole,
  passes the whole-group rule on the single-issue path).
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 16: Group runnability in `vrg-epic-audit --epic` (vergil-tooling)

Spec §5.3 (when a group is runnable).

**Files:** Create `src/vergil_tooling/lib/release_groups/audit.py`; Modify
`src/vergil_tooling/bin/vrg_epic_audit.py` (`_epic_listing`); Test
`tests/vergil_tooling/test_release_group_audit.py`,
`tests/vergil_tooling/test_vrg_epic_audit.py`.

**Interfaces:**

- Produces: `audit.group_status(epic: IssueRef) -> list[GroupStatus]`,
  `GroupStatus(group: str, rev: int, members: tuple[IssueRef, ...],
  runnable: bool, blocked_by: tuple[IssueRef, ...])`. A group is runnable
  when every open blocker outside the group is closed; blockers between
  members do not count.
- Produces: `vrg-epic-audit --epic <ref>` JSON gains `"groups": [{"group",
  "rev", "members", "runnable", "blocked_by"}]` (live groups only); the
  text output lists each group after the tasks as `G1 rev 3: runnable` or
  `G1 rev 3: blocked by <ref>, <ref>`.

- [ ] **Step 1:** `test_group_runnable_when_outside_blockers_closed`,
  `test_group_blocked_by_outside_task`, `test_inner_blockers_ignored`,
  `test_epic_listing_json_groups`, `test_epic_listing_unreadable_group_exits_1`.
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 17: Broken groups fail the sweep (vergil-tooling)

Spec §5.6.

**Files:** Modify `src/vergil_tooling/lib/release_groups/audit.py`,
`src/vergil_tooling/bin/vrg_epic_audit.py` (`_audit`, `_close`); Test
`tests/vergil_tooling/test_release_group_audit.py`,
`tests/vergil_tooling/test_vrg_epic_audit.py`.

**Interfaces:**

- Produces:

  ```python
  @dataclass(frozen=True)
  class GroupFinding:
      epic: IssueRef
      group: str
      kind: Literal["split", "closed unmerged", "rule", "edited", "unreadable"]
      detail: str      # names the members and their PRs, or the rule broken

  def group_findings(org: str) -> list[GroupFinding]
  ```

  It checks the live groups of every open group's epic (Global
  Constraints: every development epic and every proxy epic) in the org's
  home. A closed epic is never checked (spec §5.6): every member is closed
  there, so nothing of the group can still be missing from `develop`.
- `split`: members closed by different merged PRs (each member's merged
  `closedByPullRequestsReferences`). `closed unmerged`: a closed member
  that no merged PR closes. `rule`: a group with an open member that
  breaks `check_group_rules` (the dependency-path rule above all, since
  `Blocked-by` lines change without any group write). `edited`, for an
  edited record, or for a comment on the epic skipped for an outsider's
  edit (Global Constraints; reported for the epic, naming the comment),
  and `unreadable` for a log that cannot be read or holds a conflict. An open
  member whose closing PR merged together with the others is drift, closed
  by the drift-close pass; `_close` runs the group check after that pass.
- Plain `vrg-epic-audit` prints the findings in a `Release groups` section
  and exits 1 when there are any. `--close` (the daily `ops-epic-sweep`)
  does too, after its other passes; the audit repairs nothing.

- [ ] **Step 1:** `test_split_group_fails_sweep` (the test the spec asks
  for: a group whose members merged through two PRs makes `--close` exit
  1, and its report names both PRs); `test_member_closed_unmerged_flagged`;
  `test_open_member_with_merged_pr_is_drift_not_break`;
  `test_dependency_path_from_blocked_by_edit_fails_audit`;
  `test_edited_record_flagged`; `test_skipped_record_flagged`;
  `test_conflict_flagged_unreadable`;
  `test_audit_flags_member_moved_out_of_epic` (Review Focus 4);
  `test_dissolved_group_not_checked`; `test_proxy_epic_groups_checked`;
  `test_closed_epic_not_checked` (a split group on a closed epic leaves
  `--close` at exit 0).
- [ ] **Step 2–4:** fail, implement, pass, validate, commit.

### Task 18: Release phase 3's tooling (deployment)

- Precondition: `vrg-release-record check` exits 0 for each of Tasks 9–17
  (`vergil-project/vergil-tooling#<N>`).
- Procedure: those checks, alone.
- Acceptance: every check passes; the result names the release version.

### Task 19a: `issue-implement` and `pr-watch` (vergil-claude-plugin)

Spec §5.2 (the guidelines), §5.3, §5.4, §5.5, §5.8.

**Files:** Modify `skills/issue-implement/SKILL.md` ("What to batch",
"Starting a batch", "Batch stops", "A batch left with one issue",
"Handing off a batch", "A change too big for one PR"),
`skills/pr-watch/SKILL.md` (a fix to a batch commit),
`skills/merge-review/SKILL.md` ("In a batch": "up to 8 same-epic issues"
becomes the tooling's ceiling), and their pages under
`docs/site/docs/skills/`. Task 6b's re-review rules are kept as they
shipped.

The changes:

- **What to batch** is rewritten around §4: a batch is how work that must
  ship together is released. The size guidelines (about 150 lines and 6
  files; the caps of 4 and 8) and the "never batch" rules for design work,
  cross-repo contracts and security issues are removed. Two rules stay:
  never batch a documentation review, and members keep their group's build
  order. The ceiling is the tooling's (20).
- **Security issues may be batched**, with the existing safeguards (§5.3).
- **Review after a rewrite:** any change to a batch commit makes it and
  every later commit due for a fresh merge review before handoff, or
  before pr-watch pushes; the gate refuses otherwise (§5.4). pr-watch
  re-reviews the whole branch before `vrg-submit-pr --push`.
- **Batch stops in a batch that holds a group:** before any reset, run
  `vrg-pr-workflow batch-extract --check --issue …` with the set the
  ruling would send out; offer `continue`, any of `extract`, `handoff` or
  `unbatch` that moves only whole groups and standalone tasks, or a group
  change (`vrg-release-group set … --at-batch-stop --reason …`) followed by
  the action the human rules. No command adds an issue to a running batch,
  so a group change at a batch stop that adds a member (a split included)
  is always followed by `unbatch`, never by joining the running batch. The
  stop's reason names what was found (the fourth member depends on a
  feature planned later, say). A stop that would split a group is never
  offered.
- **Unbatch** removes the markers, then starts each whole group afresh as a
  batch (`batch-start`), sends a one-member group to the single-issue path,
  and leaves each standalone task to its own PR. The members' finished
  commits stay on the branch for the fresh batch, each merge-reviewed
  again as the chained gate requires.
- **A change too big for one PR:** splitting a grouped task is one recorded
  group change, the sub-tasks added and the original removed with its
  reason. Before `batch-start` the sub-tasks join the group's batch when
  it starts; after it, the split is a batch stop, followed by `unbatch` as
  above, and the fresh batch holds the sub-tasks.
- **The batch-stop reason** "depends on another task's work being merged
  first" no longer applies to a dependency on another member of the same
  group, which commit order satisfies.

- [ ] **Step 1:** make the edits; `vrg-container-run -- vrg-validate
  --markdown --fix` on each file.
- [ ] **Step 2:** full `vrg-container-run -- vrg-validate`; commit.

### Task 19b: `epic-implement` dispatches groups (vergil-claude-plugin)

Spec §5.3, §5.8.

**Files:** Modify `skills/epic-implement/SKILL.md` (its batching section,
"Batching ready tasks") and its page under `docs/site/docs/skills/`.

The changes: read the epic's groups from `vrg-epic-audit --epic <ref>
--json` (`groups`, with `runnable` and `blocked_by`); dispatch each
runnable group as one batch, members in build order; for group members,
the group's own runnability replaces the runnable-frontier rule; other
runnable tasks or groups may join on throughput judgment; a one-member
group goes to the single-issue path; never dispatch a member alone; a
group the audit cannot read stops for the human. The skill states the
one-driver assumption (Global Constraints): one `epic-implement` session
drives an epic at a time, in one place, and running a second on the same
epic is unsupported.

- [ ] **Step 1–2:** as Task 19a.

### Task 20: `epic-create` writes groups (vergil-claude-plugin)

Spec §5.1 (in the plan; the group rules), §5.7 (enforcement begins here),
§5.8.

**Files:** Modify `skills/epic-create/SKILL.md` ("Task sizing", step 6
`plan`, step 7's plan pushback walk-through, step 8 `file-tasks` and
"Keyed filing"), and its page under `docs/site/docs/skills/`.

The changes:

- **Plan (step 6):** the delivery table gains the **Group** column; the
  plan lists each group's members in build order (a chain bottom-up) and
  states the group rules of §5.1, the design-question rule included, which
  the plan pushback rounds hold it to. Only the exceptions get a group.
- **Task sizing:** a grouped task's sub-tasks ship in one batch, not each
  on its own PR.
- **`file-tasks` (step 8):** once every member of a group is filed and its
  `Blocked-by:` refs are complete, record the group with
  `vrg-release-group set --epic <home>#<N> --group <id> --member … --reason
  "planned"`, members from the keyed listing's key-to-ref map, in build
  order, run from the members' repo main clone,
  `~/dev/projects/<owner>/<repo>` (Global Constraints); a missing clone is
  the tool's ordinary refusal. A re-entry re-runs it; an identical record posts nothing. A
  refusal stops the stage for the human. `Tasks filed: yes` is posted only
  after every group is recorded.

- [ ] **Step 1–2:** as Task 19a.

### Task 21: Release the plugin (deployment)

- Precondition: `vrg-release-record check` exits 0 for each of Tasks 19a,
  19b and 20 (`vergil-project/vergil-claude-plugin#<N>`).
- Procedure: those checks.
- Human-run steps: `/reload-plugins` in each running Claude Code session.
- Acceptance: every check passes, and the human has confirmed the reloads.

### Task 22: Tooling docs (vergil-tooling)

Spec §5.8.

**Files:** Modify `docs/site/docs/guides/git-workflow.md` (batches as the
unit of release; release groups; review after a rewrite and
`batch-gates`; the ceiling; one `epic-implement` driver per epic, which
the "Started" check relies on) and
`docs/site/docs/standards/github-issues.md` (a group is recorded on the
epic, never in a task body; moving a grouped task); add the
`vrg-release-group` command to the site's command reference, wherever the
other `vrg-*` commands are listed.

- [ ] **Step 1–2:** edit, lint each file, full validate, commit.

### Documentation review and retrospective

- **doc-review** (`vergil-tooling`, `Documentation review:
  batch-release-unit`): sweep the tooling's and the plugin's site docs and
  the `docs` repo for batching, size limits and "never batch" wording left
  behind, spawning a same-repo doc task in each repo that needs one.
- **retrospective** (`.github`, `Retrospective: batch-release-unit`):
  authored with `epic-retrospective`.

---

## Appendix A: Task 8's chain

All seven tasks are code tasks in `vergil-project/vergil-fixture-python-lib`
under vergil-project/.github#1672. `<pkg>` is the fixture's package under
`src/`. Each body carries its key on its own line, `Plan task: 8 chain-<k>`
(`8 mover` for 8.0).

| Task | Title | Change |
|---|---|---|
| 8.0 | `Live run: add NOTES.md` | Adds `NOTES.md` with one line; touches no file the chain touches. Standalone. |
| 8.1 | `Live run chain 1: add step_1` | Adds `src/<pkg>/chain.py` with `def step_1() -> int: return 1` and `tests/test_chain.py::test_step_1`. |
| 8.2 | `Live run chain 2: add step_2` | Adds `def step_2() -> int: return step_1() + 1` with a one-line docstring, and its test. |
| 8.3 | `Live run chain 3: add step_3` | Adds `step_3` (`step_2() + 1`) and its test. |
| 8.4 | `Live run chain 4: add step_4` | Adds `step_4` (`step_3() + 1`) and its test, and the planned CI-only failure (below). |
| 8.5 | `Live run chain 5: add step_5` | Adds `step_5` (`step_4() + 1`) and its test. |
| 8.6 | `Live run chain 6: add step_6` | Adds `step_6` (`step_5() + 1`) and its test. |

- **The fix to 8.2** (Task 8, step 4): `step_2`'s docstring becomes "Return
  the second step, built on `step_1`." No later commit touches that line,
  so the fold is clean.
- **The planned CI-only failure in 8.4** (step 6): a deliberate failure
  that the fixture's CI reports and that the local `vrg-container-run --
  vrg-validate` does not, so 8.4 validates and hands off locally but the
  PR's CI fails. 8.4's issue body states it as the run's planned failure
  and its removal as the after-submit fix, so merge review judges it in
  scope. The builder chooses the mechanism, touching no line a later
  commit touches.
- **The fix after submit to 8.4** (step 7): the planned failure is removed,
  and `step_4` gains the docstring "Return the fourth step." No later
  commit touches either line.
