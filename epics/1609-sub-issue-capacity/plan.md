# sub-issue-capacity — implementation plan

- **Epic:** vergil-project/.github#1609
- **Planning issue:** vergil-project/vergil-tooling#3840 (review history is in its comments)
- **Status:** draft

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` and
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** An epic of any kind grows past GitHub's 100-sub-issue limit as a
chain of containers behind one chain layer, so no caller ever meets the limit
and no item lands outside its epic.

**Architecture:**

- **vergil-tooling** adds `lib/epic_chain.py`, which owns walking, growing,
  healing and closing a chain. `lib/epics.py`'s public readers become
  chain-aware, its raw writers become private, and every caller that links,
  moves, unlinks or reads children goes through the chain layer.
- Two behaviour changes the chain depends on ship first, on their own:
  **closed is closed** (no tooling path reopens an epic) and an **atomic
  `vrg-epic-move`**.
- The audit and the epic sweep report and heal chain drift, and the sweep's
  queue cap becomes a nudge instead of a refusal.
- Rollover ships switched off. Every task up to the release in Task 12
  ships the chain layer with `epic_chain.ROLLOVER_ENABLED` off, so no
  continuation can be created. Task 11b turns it on only once the plugin
  skills read through the chain (Task 14) and that plugin release is
  installed everywhere (Task 14b), and Task 12b installs the tooling
  release that carries 11b.

**Tech Stack:** Python 3 and pytest, GitHub GraphQL sub-issue API
(vergil-tooling); Markdown (site docs, and the vergil-claude-plugin skills).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Container capacity:** a container holds at most **99 items**; the 100th
  slot of a non-tail container is its continuation (spec §4.1).
- **The count is all children, open and closed**, read from the sub-issue
  total, less one when the container holds a continuation (spec §4.3).
- **Continuation identity:** an issue in the head's repo, labels `epic` and
  `continuation` and no kind label, title `<head title> (continuation <k>)`
  with `k` from 2, body lines `Continues: <owner>/<repo>#<head>` and
  `Container: <k>`, each alone on its line (spec §4.1).
- **`Container:` and the title number are display only;** position is the
  native chain's, and no reader checks the number (spec §4.1).
- **Trust:** a continuation is followed only when it has the label, a trusted
  body, a `Continues:` line naming the head being walked, and a native link
  to the container before it; anything else is chain drift, raised by name
  (spec §4.1).
- **First fit:** link into the first container in chain order with fewer than
  99 items; roll over only when every container holds 99 (spec §4.3).
- **Rollover order:** find-or-create the continuation, link it into the
  tail's last slot, then link the item (spec §4.3).
- **Closed is closed:** no tooling path reopens an epic; `link_into` and
  `move_into` refuse a closed head by name (spec §4.2).
- **`close_chain` closes the head first,** then every continuation
  (spec §4.6).
- **No raw writer or raw `parent`/`subIssues` selection outside
  `lib/epics.py` and `lib/epic_chain.py`** once Task 11 lands (spec §4.2).
- **The sweep keeps its cap of 95 open children** as the review nudge, but
  no longer refuses at it (spec §4.7).
- **Archives are unchanged:** `ensure_writable_standing_archive` and its
  segment cap of 95 are not modified (spec §2, non-goals).
- **No silent failures:** every refusal, heal and rollover prints one line
  saying what it did (spec §5).
- **Rollover is off until Task 11b.** While `epic_chain.ROLLOVER_ENABLED`
  is `False`, nothing creates, adopts or links a continuation. A head takes
  up to 100 items, as today, and a link into a head at 100 is refused by
  name, before any create.

## Review Focus

1. **A continuation's ref pasted as `--epic`** (copied from GitHub's UI). It
   resolves to the head with a line saying so, and every kind check runs
   against the head. Pinned in Task 6
   (`test_resolve_epic_ref_maps_continuation_to_head`).
2. **A head already at 100 children with no continuation** (the state #438
   was in). The first link heals it and rolls over, and no item is left
   unlinked. Pinned in Task 5c
   (`test_link_into_heals_head_at_100_without_continuation`).
3. **The last open task of an epic sits in a continuation.** Closing a task
   in the head does not roll the epic up; closing that last task does, and
   closes the whole chain. Pinned in Task 6
   (`test_rollup_waits_for_open_item_in_continuation`).
4. **A continuation closed by hand in an open chain.** Readers still list its
   items, writers skip it, the audit notes it, and nothing reopens it.
   Pinned in Task 5b (`test_link_into_skips_closed_continuation`) and
   Task 9a (`test_audit_notes_closed_continuation_in_open_chain`).
5. **A create-then-link command at the depth limit.** `vrg-issue-create`
   refuses before creating, so no issue is orphaned. Pinned in Task 7
   (`test_issue_create_depth_refusal_creates_nothing`).
6. **A full head while rollover is off** (any release before Task 12b's).
   The link is refused by name, `check_capacity` refuses before a create,
   and no continuation is created. Pinned in Task 5b
   (`test_link_into_rollover_disabled_full_head_refused`) and Task 5c
   (`test_heal_rollover_disabled_creates_nothing`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by. Every implementation task but 2b and 14 is in
vergil-tooling.

- **1** (.github, task #1610): spec and plan, this PR.
- **2** (vergil-tooling): closed is closed: no reopen in `add_child`,
  `intake_route` or `proxies.create_proxy`; `ensure_standing_epic` refuses a
  closed one; `vrg-issue-create` refuses a closed epic before creating; the
  audit's reopen fix removed, and the sweep's link passes report a closed
  epic per item instead of aborting; `EpicRefusalError`, the base of every
  refusal this epic adds, which `cli_main` turns into a one-line exit.
- **2b** (vergil-claude-plugin): the `epic-create` and `epic-implement`
  skills say a closed proxy is refused, not reopened, and the agent stops
  for the human. Blocked by 2.
- **3** (vergil-tooling): `vrg-epic-move` moves atomically with
  `reparent_child`, refusing a closed target. Blocked by 2.
- **4** (.github, validation): a live probe of GitHub's nesting limit and of
  linking under a closed parent, fixing the depth constant.
- **5a** (vergil-tooling): `epic_chain` read side, the `continuation` label,
  the chain-drift error, the raw per-container reads and the in-memory
  sub-issue fake. Blocked by 2.
- **5b** (vergil-tooling): `epic_chain` write side: first-fit `link_into`,
  `move_into`, `unlink_from`, rollover, find-or-create with orphan adoption,
  the depth refusal, the capacity pre-check, the rollover messages and the
  `ROLLOVER_ENABLED` switch, off. Blocked by 2, 5a, 4.
- **5c** (vergil-tooling): race heals (limit error, full tail, duplicate
  continuations) and `close_chain`. Blocked by 5b.
- **6** (vergil-tooling): `epics.py` chain-aware readers, ref resolution,
  rollup, drain and the lookups that must not find a continuation. Blocked
  by 2, 5c.
- **7** (vergil-tooling): the CLI writers (`vrg-epic-link`,
  `vrg-issue-create`, `vrg-intake-create`, `vrg-security-record`,
  `vrg-epic-move`) and `ops_report` through the chain layer, with the
  capacity pre-check, their tests ported to the fake. Blocked by 3, 6, 9a,
  10.
- **8** (vergil-tooling): `intake_route` through the chain layer, with
  transfer settling and the forced re-link moved into it, its tests ported
  to the fake. Blocked by 6, 9a, 10.
- **9a** (vergil-tooling): audit read side: continuation skips, chain-aware
  children and `misplaced_adhoc`, the batched open-issue listing moved into
  `epics`, drift and orphan reporting, container count, and `epic_docs`'s
  continuation skip. Blocked by 6.
- **9b** (vergil-tooling): sweep writes: the orphan pass through the chain
  with the nudge, the heals, orphan adoption with a rollup re-run, and
  `close_drift`/`close_proxies` through `close_chain`, its tests ported to
  the fake. Blocked by 9a, 10.
- **10** (vergil-tooling): the remaining readers (`queue_view`, `roadmap`,
  `planning_close`, `security_records`, `merge_review/request`,
  `vrg-submit-pr` and `pr_workflow/batch_rules`). Blocked by 6.
- **10b** (vergil-tooling): `vrg-epic-parent`, a chain-aware command that
  prints a task's epic and its kind, for the plugin skills. Blocked by 6.
- **11** (vergil-tooling): the raw layer made private, with a source-scan
  test that keeps it so, and `vrg-gh` refusing `issue edit`'s raw sub-issue
  flags. Blocked by 7, 8, 9b, 10, 10b.
- **12** (.github, deployment): the vergil-tooling release carrying 2–11,
  with rollover off, installed on the host and every VM. Blocked by 11.
- **14** (vergil-claude-plugin): the skills that read a task's raw native
  parent (`issue-implement`, `merge-review`, `epic-create`,
  `epic-retrospective`) use `vrg-epic-parent` instead. Blocked by 12.
- **14b** (.github, deployment): the vergil-claude-plugin release carrying
  14, installed on every host and VM. Blocked by 14.
- **11b** (vergil-tooling): rollover turned on: the `ROLLOVER_ENABLED`
  switch and its disabled path removed. Blocked by 11, 14b.
- **12b** (.github, deployment): the vergil-tooling release carrying 11b,
  installed on the host and every VM. Blocked by 11b.
- **13** (.github, validation): #1384's 14 tasks re-homed into its chain, the
  first live rollover of a development epic. Blocked by 12b.

### Bookends

- **doc-review** (vergil-tooling, `Plan task: doc-review`): blocked by 13.
  Its sweep covers vergil-tooling's
  `docs/site/docs/standards/github-issues.md` (the 100-sub-issue cap, the
  orphan pass, archives' 95 cap beside the chain) and
  `docs/site/docs/reference/` CLI pages for the changed commands; the docs
  repo's summary of the epic model; and any vergil-claude-plugin skill that
  tells an agent what to do at the cap (`epic-implement`,
  `issue-implement`), each a spawned per-repo doc task. The skills' parent
  reads are Task 14's, not the sweep's.
- **retrospective** (.github, `Plan task: retrospective`): blocked by
  everything else.

Notes:

- **No fixture runs** and no `dev-` image validation: the change touches no
  language support, container image, shared workflow or release tooling.
- **No follow-on brainstorm task:** no enabling chain is known. Collecting
  empty continuations (spec §2, non-goals) is accrued as an intake item only
  if a real chain ever needs it.
- **No chain before every reader knows chains.** vergil-tooling releases
  from develop often, and each release is installed on the host and every
  VM, so any task may reach the fleet before Task 12. Readers include the
  plugin skills (Task 14), which run the installed tooling and so can only
  switch to `vrg-epic-parent` after a release. The writers' blocker edges
  alone cannot order them before the writers. So rollover ships switched
  off: until Task 11b is released, no writer creates a continuation, every
  head stays a single container, and the raw writers that remain public
  until Task 11 meet exactly today's 100-child limit, refused by name.
  Task 11b is blocked by 11 and 14b, so it is released only once every
  tooling reader reads through the chain and the skills that do are
  released and installed everywhere: plugin installs follow the plugin's
  releases, not its develop branch, so Task 14's merge alone orders
  nothing. Tasks 7, 8 and 9b stay
  blocked by the reader tasks (6, 9a and 10), so each writer is written and
  tested against chain-aware readers. Each task must still pass the full
  suite on its own.
- **Two releases.** Task 12 installs a release carrying every vergil-tooling
  implementation task but 11b, with rollover off; Task 14 then runs against
  it. Task 12b installs the release carrying 11b. Tasks 2 and 3 may reach an
  earlier release by the normal cadence; that is fine, as each stands alone.
- **A closed continuation with no successor.** A closed continuation fills
  the 100th slot of the container before it, so when it has no successor
  and every open container holds 99 items, no open container has a free
  slot. The next continuation is then linked into the closed continuation
  itself: the one link a closed container takes, which leaves its state
  unchanged (spec §4.2). Task 4 probes that GitHub accepts a link under a
  closed parent; if it refuses, Task 5b instead refuses by name, naming the
  closed continuation and saying a person may reopen or unlink it, with
  `check_capacity` detecting the case so a create-then-link command refuses
  before creating, and the spec is corrected to match.
- **Tests move to the fake with their code.** Every task that switches a
  caller to the chain layer ports that caller's existing tests from
  `patch(...epics.add_child)`-style mocks to `FakeSubIssueGraph`, asserting
  on the graph's end state instead of on raw calls, in the same PR. The
  port is named in each task's Files and steps.
- **Docs travel with behaviour.** Each task that changes documented
  behaviour updates `docs/site/docs/standards/github-issues.md` in the same
  PR (named in its Files); the doc-review bookend checks the whole.

---

### Task 2: closed is closed (vergil-tooling)

Spec §4.2 (closed is closed), goal 8.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`add_child`, `ensure_standing_epic`, `_find_epic_by_title`)
- Modify: `src/vergil_tooling/lib/intake_route.py:1545-1560` (the reopen before a re-parent)
- Modify: `src/vergil_tooling/lib/proxies.py:330-375` (`create_proxy`), `src/vergil_tooling/bin/vrg_epic_create.py:189` (the reopened report)
- Modify: `src/vergil_tooling/lib/epic_audit.py` (remove `reopen_epics_with_open_children`; `nativize_parents` and `attach_orphans` catch `ClosedEpicError` per item), `src/vergil_tooling/bin/vrg_epic_audit.py:300-372` (`_apply_orphan_pass` included)
- Modify: `src/vergil_tooling/bin/vrg_epic_link.py` (docstring: the reopen-on-late-child rule)
- Modify: `src/vergil_tooling/bin/vrg_issue_create.py:177` (a state check before `github.create_issue`)
- Modify: `src/vergil_tooling/lib/cli.py` (`cli_main` maps `EpicRefusalError` to a clean exit 1)
- Modify: `docs/site/docs/standards/github-issues.md` and `docs/site/docs/reference/cli-tools-overview.md` (every mention of a tooling reopen: the proxy reopen, `vrg-epic-link` reopening a closed epic, and the audit reopening epics closed with an open child; today lines 911, 1137 and 2085-2086 of the reference page), and any other `docs/site/docs/` page a grep for `reopen` finds describing them
- Test: `tests/vergil_tooling/test_epics.py`, `test_intake_route.py`, `test_proxies.py`, `test_epic_audit.py`, `test_vrg_epic_audit.py`, `test_vrg_issue_create.py`, `test_cli.py`, `test_vrg_epic_create.py`

**Interfaces:**

- Produces: `epics.EpicRefusalError(Exception)`, the base of every refusal
  this epic adds, which `lib/cli.py`'s `cli_main` maps to one
  `<prog>: <message>` line on stderr and exit 1, with no traceback, as it
  maps `EpicHomeError` today. Every later error this plan introduces
  (`ChainDriftError`, `RolloverDisabledError`, `ChainDepthError`,
  `ChainLimitError`) subclasses it.
- Produces: `epics.ClosedEpicError(EpicRefusalError, ValueError)` with
  attribute `epic: IssueRef` and message `epic <slug> is closed; closed
  epics are not reopened by the tooling`. Later tasks raise it from
  `link_into` and `move_into`.
- Produces: `epics.ClosedStandingEpicError(ClosedEpicError)`, raised by
  `ensure_standing_epic` when no open epic has the canonical title and a
  closed one does; its message adds `a person may reopen it by hand`.

- [ ] **Step 1: Write the failing tests**

  - `test_add_child_refuses_closed_epic`: with the epic's state `CLOSED`,
    `add_child` raises `ClosedEpicError` naming `org/.github#40`, and no
    `issue reopen` and no `addSubIssue` call is made. Replaces
    `test_add_child_reopens_closed_epic_before_linking`.
  - `test_ensure_standing_epic_reuses_open_beside_closed`: an open and a
    closed epic share the canonical title; the open one is returned, nothing
    is created.
  - `test_ensure_standing_epic_refuses_when_only_closed`: only a closed one
    exists; raises `ClosedStandingEpicError` naming it; nothing is created.
  - `test_ensure_standing_epic_creates_when_none`: neither exists; created as
    today.
  - `test_route_refuses_closed_target_epic` (intake_route): routing into a
    closed epic exits non-zero naming it; no `issue reopen` call.
  - `test_create_proxy_refuses_closed_proxy` (proxies): a closed proxy whose
    `Primary:` line names the primary raises `ClosedEpicError`; no reopen.
  - `test_audit_reports_closed_epic_open_child_without_reopen`
    (vrg-epic-audit `--close`): the violation is printed for the human and
    no `issue reopen` call is made.
  - `test_nativize_skips_closed_epic_and_continues` (epic_audit): two open
    tasks whose `Parent:` lines name a closed epic and an open one; the
    first is reported in a line naming the task and the closed epic, the
    second is linked, `attach_orphans` still runs, and the pass exits
    non-zero at the end.
  - `test_attach_orphans_reports_closed_intake_epic`: with only a closed
    intake epic, each orphan is reported naming it, nothing is linked or
    reopened, and the pass exits non-zero at the end.
  - `test_issue_create_closed_epic_refused_before_create`
    (vrg-issue-create): `--epic` names a closed epic → exit 1 with
    `ClosedEpicError`'s message, and `github.create_issue` is never called.
  - `test_cli_main_maps_epic_refusal_to_one_line` (cli): a command raising
    any `EpicRefusalError` subclass exits 1 with exactly one stderr line,
    `<prog>: <message>`, and no `Traceback`.
  - `test_proxy_for_closed_proxy_is_clean_refusal` (vrg-epic-create):
    `--proxy-for` whose org's proxy is closed → exit 1, one stderr line
    naming the proxy, no traceback, no reopen.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_intake_route.py tests/vergil_tooling/test_proxies.py tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py tests/vergil_tooling/test_vrg_issue_create.py tests/vergil_tooling/test_cli.py tests/vergil_tooling/test_vrg_epic_create.py -k "closed or reopen or refusal" -v`
  Expected: the new tests FAIL (reopen still happens).

- [ ] **Step 3: Implement**

  `add_child` checks the state and raises; `_find_epic_by_title` gains a
  `states` parameter (`"open"` default, `"all"` for `ensure_standing_epic`);
  remove `reopen_epics_with_open_children` and its CLI wiring, keeping
  `closed_epic_open_child` as a report. `ProxyResult.reopened` is removed.
  One task naming a closed epic never stops the orphan pass: it is
  reported, the pass goes on, and the exit is non-zero. `vrg-issue-create`
  reads the epic's state before it creates the issue and refuses a closed
  epic with `ClosedEpicError`'s message. Otherwise, once this task ships
  alone, the issue would be created, the link refused, and the suggested
  `vrg-epic-move` refused too (Task 3). Task 7 folds this check into
  `check_capacity`.

- [ ] **Step 4: Run the tests and the full gate**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "make closed epics final: refuse to link, move or reuse a closed epic instead of reopening it"`.

### Task 2b: the plugin skills say a closed proxy is refused (vergil-claude-plugin)

Spec §4.2 (closed is closed: `create_proxy` refuses a closed proxy). Blocked
by 2. Until Task 2 is released, the installed tooling still reopens a closed
proxy, so the new text describes a refusal the agent may not yet meet; that
is harmless, since the step it describes then simply succeeds.

**Files:**

- Modify: `skills/epic-create/SKILL.md` (the cross-org bullet of step 8 and
  the Notes: "a re-run finds the proxy (reopening one closed early)")
- Modify: `skills/epic-implement/SKILL.md` (the proxy passage: "reopens one
  closed early")

- [ ] **Step 1: Find every reopen claim.** Grep `skills/` and `agents/` for
  `reopen` and for prose saying a tooling path reopens a closed epic or
  proxy. Every hit is either changed in Step 2 or named in the PR with why it
  can stay (a reopen a person does by hand).

- [ ] **Step 2: Reword each one:** `vrg-epic-create --proxy-for` finds an
  open proxy and posts nothing twice; when the org's proxy is closed it
  refuses by name, and the agent stops and asks the human, who may reopen
  the proxy by hand. Never tell the agent to reopen it.

- [ ] **Step 3: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 4: Commit**: `vrg-commit --type docs --scope skills --message "say that a closed proxy is refused, not reopened, and that the agent stops for the human"`.

### Task 3: atomic `vrg-epic-move` (vergil-tooling)

Spec §4.5 (`vrg-epic-move` stops orphaning), §4.2 (closed is closed).
Blocked by 2.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_move.py:90-93`
- Test: `tests/vergil_tooling/test_vrg_epic_move.py`

**Interfaces:**

- Consumes: `epics.reparent_child(new_parent, task)`; `epics.add_child` for a
  task with no current parent; `epics.ClosedEpicError` (Task 2).

- [ ] **Step 1: Write the failing tests**

  - `test_move_uses_one_reparent`: a task under epic A moved to B makes one
    `addSubIssue` call with `replaceParent: true` and no `removeSubIssue`.
  - `test_failed_move_leaves_task_in_place`: the reparent raises; the command
    exits 1 with the error, and no `removeSubIssue` was called.
  - `test_move_unparented_task_links_it`: no current parent → `add_child`.
  - `test_move_into_closed_epic_refused`: target B is closed → exit 1 with
    `ClosedEpicError`'s message naming B; no `addSubIssue` or
    `removeSubIssue` call, and the task stays under A.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_epic_move.py -v`
  Expected: the first two and the closed-target test FAIL.

- [ ] **Step 3: Implement**: check the target's state first and raise
  `ClosedEpicError` when it is closed; then replace the remove-then-add
  with `reparent_child` when `current is not None`. `reparent_child`
  itself stays unchecked, for the archive drain.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type fix --scope epic-move --message "move a task between epics in one atomic re-parent"`.

### Task 4: probe GitHub's nesting limit (validation, .github)

Spec §4.5 (the depth headroom), §4.4 (GitHub's limit error), §4.2 (a closed
continuation with no successor). No blockers.

Every write in the probe is one the agent identity cannot make: `vrg-gh`
refuses `issue create` and `api graphql`, `vrg-issue-create` always links
under an epic (so a probe would never start unparented), and the agent's
only paths that remove a sub-issue link go through an epic (`vrg-epic-move`,
and `vrg-gh issue edit --remove-sub-issue` until Task 11 refuses it), never
a bare unlink of an unparented probe chain. So the probe is **human-run**: the human
runs the script below with a human identity and attests the result, and the
agent only checks and records it.

- **Precondition (human-attested):** a trusted human comment on this task
  says the probe script below was run to completion, and carries its output
  lines (`Max depth:`, `Depth error:`, `Limit error:`, `Link under closed
  parent:`), each alone on its line. If it is missing, comment "blocked:
  preconditions not met" and stop.
- **The human-run probe.** Run it in one sitting, away from the daily epic
  sweep's scheduled time (`epic-sweep.yml`), since the sweep links
  unparented, unlabelled issues under the intake epic. Paste it as one
  `! bash -c '…'` block, or save it to a scratch file and run that:

  ```bash
  set -uo pipefail
  repo=vergil-project/.github
  body="Throwaway for epic vergil-project/.github#1609 Task 4."
  mk()  { gh issue create --repo "$repo" --title "$1" --body "$body" | sed 's#.*/##'; }
  nid() { gh issue view "$1" --repo "$repo" --json id -q .id; }
  sub() { gh api graphql -f query="mutation(\$p:ID!,\$c:ID!){$1(input:{issueId:\$p,subIssueId:\$c}){issue{number}}}" \
            -f p="$(nid "$2")" -f c="$(nid "$3")" 2>&1; }
  parent() { gh api graphql -f query="query{repository(owner:\"vergil-project\",name:\".github\"){issue(number:$1){parent{number}}}}" -q .data.repository.issue.parent; }
  # 1. Depth: link each new probe under the last until a link fails (cap 50).
  probes=("$(mk 'Probe: sub-issue depth 1')"); depth=1
  while [ "$depth" -lt 50 ]; do
    next=$(mk "Probe: sub-issue depth $((depth + 1))")
    if out=$(sub addSubIssue "${probes[-1]}" "$next"); then probes+=("$next"); depth=$((depth + 1))
    else echo "Depth error: $out"; gh issue close "$next" --repo "$repo" --reason "not planned"; break; fi
  done
  [ -z "$(parent "${probes[0]}")" ] || echo "WARNING: probe 1 gained a parent; the depth is not valid"
  echo "Max depth: $depth"
  # 2. The 100-child error: set full=<N> to an open issue holding 100 sub-issues, or leave it empty.
  full=""
  if [ -n "$full" ]; then echo "Limit error: $(sub addSubIssue "$full" "${probes[-1]}")"; else echo "Limit error: unobserved"; fi
  # 3. A link under a closed parent.
  cp=$(mk 'Probe: closed parent'); cc=$(mk 'Probe: closed child')
  gh issue close "$cp" --repo "$repo" --reason "not planned"
  if out=$(sub addSubIssue "$cp" "$cc"); then echo "Link under closed parent: accepted"; else echo "Link under closed parent: refused: $out"; fi
  gh issue view "$cp" --repo "$repo" --json state -q .state   # must print CLOSED
  # 4. Clean up: unlink bottom-up and close each probe as not planned.
  sub removeSubIssue "$cp" "$cc" >/dev/null
  for ((i = ${#probes[@]} - 1; i > 0; i--)); do sub removeSubIssue "${probes[i-1]}" "${probes[i]}" >/dev/null; done
  for n in "${probes[@]}" "$cc"; do gh issue close "$n" --repo "$repo" --reason "not planned"; done
  ```

  If the loop reaches the cap of 50 without a failure, or the probe 1
  warning prints, stop: the result is not valid, and the human decides how
  to proceed.
- **Procedure (agent):** read the task with `vrg-issue-read` (exit 3 or 1
  stops), take the four lines from the trusted human comment, and check
  each is present and well formed (`Max depth:` an integer of at least 2).
  Post them in the result comment.
- **SUCCESS:** a result comment carrying the lines `Max depth: <d>` (levels
  including the top issue), `Depth error: <text>`, `Limit error: <text>`
  (or `Limit error: unobserved`) and `Link under closed parent: accepted`
  (or `refused: <text>`). Task 5b reads them. The limit text is only a hint:
  5b recognises a full container by its sub-issue total, so `unobserved`
  needs no live confirmation.

### Task 5a: chain read side and the sub-issue fake (vergil-tooling)

Spec §4.1, §4.2 (`chain_of`, `head_of`, `chain_child_states`,
`chain_contains`). Blocked by 2 (`EpicRefusalError`, the base of `ChainDriftError`).

**Files:**

- Create: `src/vergil_tooling/lib/epic_chain.py`
- Create: `tests/vergil_tooling/subissue_fake.py`
- Modify: `src/vergil_tooling/data/labels.json` (the `continuation` label)
- Modify: `src/vergil_tooling/lib/epics.py` (`CONTINUATION_LABEL`, `is_continuation`, `_container_parent`, `_container_child_states`)
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Produces in `epics`: `CONTINUATION_LABEL = "continuation"`;
  `is_continuation(labels: Collection[str]) -> bool`; and the raw
  per-container reads `_container_parent(task: IssueRef) -> IssueRef | None`
  and `_container_child_states(container: IssueRef) -> list[ChildState]`,
  the same reads `native_parent_of` and `child_states` make today, used only
  by `epics` and `epic_chain`. Every walk in `epic_chain` (`head_of`, the
  depth walk, the container listing) uses only these, never the public
  readers, so Task 6 making the public readers chain-aware changes nothing
  in the chain layer.
- Produces in `epic_chain`:
  - `ITEM_CAP = 99`
  - `class ChainDriftError(EpicRefusalError, RuntimeError)` with `container: IssueRef` and
    `rule: str` (one of `label`, `untrusted-body`, `continues-mismatch`,
    `not-native-child`, `two-continuations`), message naming both.
  - `@dataclass(frozen=True) class Container: ref: IssueRef; state: str;
    item_count: int; continuation: IssueRef | None`
  - `containers(head: IssueRef) -> list[Container]` (head first)
  - `chain_of(head: IssueRef) -> list[IssueRef]`
  - `head_of(container: IssueRef) -> IssueRef`
  - `chain_child_states(head: IssueRef) -> list[epics.ChildState]`
    (container order, continuations excluded)
  - `chain_contains(head: IssueRef, task: IssueRef) -> bool`
  - `parse_continues(body: str) -> IssueRef | None`
- Produces in the fake: `FakeSubIssueGraph` with `add_issue(ref, *, title,
  labels=(), body="", state="OPEN") -> IssueRef`, `link(parent, child)`,
  `close(ref)`, and `install(monkeypatch)`, which answers `github.graphql`
  for `epics`' sub-issue, parent and mutation queries and `github.read_output`
  for node ids, state, title and labels. `addSubIssue` fails at 100 children
  with `LIMIT_ERROR` and past `MAX_DEPTH` with `DEPTH_ERROR` (module
  constants, from Task 4's result); a trusted body is served through
  `trust_stubs.FakeItems`.

- [ ] **Step 1: Write the fake and the failing tests**

  - `test_chain_of_single_head`: a head with 3 items → `[head]`.
  - `test_chain_of_follows_continuations`: head (99 items + C2), C2 (99 + C3),
    C3 (5) → `[head, C2, C3]`; `chain_child_states(head)` has 203 items, no
    continuation among them, head's items first.
  - `test_head_of_continuation_walks_parents`: `head_of(C3) == head`;
    `head_of(head) == head`.
  - `test_continuation_with_wrong_continues_is_drift`: C2's `Continues:`
    names another head → `ChainDriftError(rule="continues-mismatch")`.
  - `test_continuation_untrusted_body_is_drift`: rule `untrusted-body`.
  - `test_continuation_missing_label_is_drift`: rule `label`.
  - `test_continuation_not_native_child_is_drift`: rule `not-native-child`,
    including a `head_of` case whose `Continues:` line names a head the
    native walk does not reach. With these, each of the four acceptance
    rules of spec §4.1 has its own test.
  - `test_two_continuations_under_one_container_is_drift`: rule
    `two-continuations` (5c heals it; readers raise).
  - `test_closed_continuation_is_walked`: C2 closed under an open head; its
    items are listed and no error is raised.
  - `test_item_count_excludes_continuation`: head with 99 items + C2 →
    `item_count == 99`; read from the sub-issue total, not a listing.
  - `test_chain_contains_item_in_continuation`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -v`
  Expected: FAIL (module does not exist).

- [ ] **Step 3: Implement** the read side. `head_of` on an issue with no
  native parent and a trusted `Continues:` line returns that head and prints
  `chain drift: <slug> is an orphaned continuation of <head>` to stderr
  (adoption is 5b's).

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "add the chain layer's read side and the continuation label"`.

### Task 5b: chain write side and rollover (vergil-tooling)

Spec §4.2 (`link_into`, `move_into`, `unlink_from`), §4.3, §4.5, §4.7
(rollover messages). Blocked by Task 2 (`ClosedEpicError`), 5a and Task 4.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Modify: `src/vergil_tooling/lib/epics.py` (`_add_child_unchecked`)
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Consumes: Task 5a's read side; `epics.ClosedEpicError` (Task 2);
  `epics.add_child`/`reparent_child`/`remove_child` (still public until 11).
- Produces in `epics`: `_add_child_unchecked(parent: IssueRef, child:
  IssueRef) -> None`, the raw `addSubIssue` link with no state check. It is
  closed is closed's one named exception: only
  `epic_chain.find_or_create_continuation` calls it, and only to link the
  next continuation under a closed continuation that has no successor
  (spec §4.2). Every other link goes through `add_child`, which refuses a
  closed parent. If Task 4 recorded `Link under closed parent: refused`,
  it is not added.
- Produces:
  - `ROLLOVER_ENABLED: bool = False`. While it is `False`, no writer
    creates, adopts or links a continuation, first fit sees only the head,
    and the head takes up to 100 items (today's limit). Task 11b removes
    it. Every chain test in this and later tasks turns it on with
    `monkeypatch`, except the tests named `*_rollover_disabled_*`.
  - `class RolloverDisabledError(EpicRefusalError, RuntimeError)`: message `epic <slug> is at
    GitHub's limit of 100 sub-issues, and rollover into a continuation is
    not enabled yet`.
  - `MAX_DEPTH: int` (Task 4's `Max depth`)
  - `class ChainDepthError(EpicRefusalError, RuntimeError)`: message `epic <slug> cannot grow:
    <items> items in <containers> containers, at GitHub's nesting limit of
    <MAX_DEPTH>; split the epic`. When it rewords GitHub's depth error
    during a `move_into`, the message adds `, or <task>'s own sub-issues
    are too deep to fit under it`, since the depth check does not count
    the moved item's own subtree.
  - `check_capacity(head: IssueRef) -> None`: raises `ClosedEpicError`,
    `RolloverDisabledError` (rollover off and the head at 100) or
    `ChainDepthError` when a link would fail for any of them; reads only.
  - `link_into(head: IssueRef, task: IssueRef) -> IssueRef` (the container
    linked into)
  - `move_into(head: IssueRef, task: IssueRef) -> IssueRef`
  - `unlink_from(head: IssueRef, task: IssueRef) -> None` (raises
    `ValueError` naming both when the task is not in the chain)
  - `find_or_create_continuation(head: IssueRef, tail: IssueRef) -> IssueRef`
    (raises `ClosedEpicError` naming the head, before any read of orphans
    and any write, when the head is closed: an orphan of a closed head is
    never adopted, and every adoption path goes through this function; a
    closed *tail* under an open head is the closed-continuation case below,
    not a refusal)
  - `find_orphans(head: IssueRef) -> list[IssueRef]` (open
    `continuation`-labelled issues in the head's repo whose trusted
    `Continues:` names *head* and that `chain_of(head)` does not reach)

- [ ] **Step 1: Write the failing tests**

  - `test_link_into_first_fit`: head 99 items + C2 with 40 → the item goes
    into C2; head 98 items + C2 with 40 → into the head.
  - `test_link_into_reuses_freed_slot_before_growing`: head 99 + C2 99, then
    one head item moved out → the next link goes into the head, no C3.
  - `test_link_into_rolls_over_at_99`: head 99 → C2 created with the exact
    title, labels and body lines, linked as the head's 100th child, then the
    item linked into C2 (assert the call order).
  - `test_link_into_skips_closed_continuation`: head 99 items + C2 closed
    (5 items) + C3 open (10 items) → the item goes into C3; C2 is not
    written to and stays closed.
  - `test_link_into_closed_continuation_without_successor`: head 99 items +
    C2 closed (99 items) with no successor → C3 is created and linked as
    C2's 100th child, C2 stays closed, and the item goes into C3. If Task 4
    recorded `Link under closed parent: refused`, the test instead expects
    a refusal naming C2 and saying a person may reopen or unlink it, with
    nothing created, and `check_capacity` raises the same refusal.
  - `test_closed_continuation_link_uses_unchecked_add`: in that case C3 is
    linked with `epics._add_child_unchecked`, and `add_child` under the
    closed C2 still raises `ClosedEpicError`.
  - `test_add_child_unchecked_has_one_caller`: an `ast` scan of `src/` finds
    `_add_child_unchecked` called only in
    `epic_chain.find_or_create_continuation`.
  - `test_link_into_refuses_closed_head`: `ClosedEpicError`, nothing written.
  - `test_find_or_create_refuses_closed_head`: a closed head with an orphan
    naming it → `ClosedEpicError`, the orphan is not linked and its body is
    not rewritten, nothing created.
  - `test_find_or_create_reuses_existing_child_continuation` and
    `test_find_or_create_adopts_orphan`: an orphan naming the head is linked
    into the tail's last slot and its `Container:` line and title rewritten;
    nothing created.
  - `test_orphan_failing_trust_is_not_adopted`: a `continuation`-labelled
    issue with an untrusted body raises `ChainDriftError`; a new
    continuation is not created over it either.
  - `test_depth_refusal_before_any_write`: a chain at `MAX_DEPTH` with every
    container full → `ChainDepthError` with the exact message; no issue
    created, nothing linked.
  - `test_github_depth_error_is_reworded`: the fake's `DEPTH_ERROR` comes
    first → `ChainDepthError`.
  - `test_move_into_depth_error_names_item`: `move_into` meets the fake's
    `DEPTH_ERROR` for an item with its own sub-issues → `ChainDepthError`
    whose message also names the item's own sub-issues as a possible cause.
  - `test_link_into_rollover_disabled_full_head_refused`: rollover off, the
    head at 99 items → the item takes the head's 100th slot, with no
    continuation; the head at 100 → `RolloverDisabledError` naming the
    head, nothing created or linked, and `check_capacity` raises the same.
  - `test_depth_counts_head_nested_under_chained_parent`: the head is itself
    an item in another epic's C2 → the head's depth counts that chain's
    containers (through `_container_parent`), and a chain one container
    short of `MAX_DEPTH` by that count is refused before any write.
  - `test_development_rollover_is_loud`, parametrised over a development
    head and a proxy head: stdout has `epic <head> rolled over into
    continuation <C2> at 99 children; consider splitting it`, and the same
    text is commented on the head.
  - `test_queue_rollover_prints_review_nudge`: an intake head → stdout has
    `queue <head> rolled over into continuation <C2>; run epic-review
    intake`; no comment on the head.
  - `test_other_standing_rollover_is_plain`: an ad-hoc head (and any
    standing kind not in `QUEUE_KINDS`) → stdout has one plain line,
    `epic <head> rolled over into continuation <C2>`, with no comment on
    the head and no advice to split or review.
  - `test_move_into_is_one_reparent` and `test_unlink_from_finds_container`.
  - `test_move_into_refuses_closed_head`: `ClosedEpicError`, the task stays
    where it was, nothing written.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement** the writers. Depth is the head's own depth
  (a `_container_parent` walk) plus the container count; the check runs
  before find-or-create. With `ROLLOVER_ENABLED` off, find-or-create and
  orphan adoption are never reached.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "link, move and unlink through the chain with first-fit rollover, orphan adoption and the depth refusal"`.

### Task 5c: race heals and `close_chain` (vergil-tooling)

Spec §4.4, §4.6 (`close_chain`). Blocked by 5b.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Test: `tests/vergil_tooling/test_epic_chain.py`

**Interfaces:**

- Produces:
  - `LINK_RETRIES = 3`
  - `heal_full_tail(head: IssueRef, *, own_item: IssueRef | None = None) -> bool`
  - `heal_duplicate_continuations(head: IssueRef) -> bool`
  - `close_chain(head: IssueRef) -> list[IssueRef]` (closed, head first)
  - `class ChainLimitError(EpicRefusalError, RuntimeError)`: message naming GitHub's limit of
    100, the container and the item, raised when retries run out.

- [ ] **Step 1: Write the failing tests**

  - `test_link_into_heals_head_at_100_without_continuation`: the head holds
    100 items and no continuation; `link_into` creates C2, moves the head's
    newest item into it with `replaceParent`, links C2 into the freed slot,
    then links the new item (into C2), and prints one heal line. At no step
    is any item without a parent (assert on the fake's graph after every
    mutation).
  - `test_full_tail_heal_moves_own_item`: after the writer's own link fills
    slot 100, the moved item is its own.
  - `test_limit_error_retries_then_succeeds`: the first link raises the
    fake's `LIMIT_ERROR` (a racing writer); the writer re-reads, rolls over
    and succeeds.
  - `test_limit_error_retries_exhausted_names_limit`: `ChainLimitError` after
    `LIMIT_ERROR` three times, naming 100, the container and the item.
  - `test_unrecognised_link_error_at_100_is_full`: the link fails with an
    error text other than `LIMIT_ERROR`, and the re-read total of the target
    is 100 → handled as full (re-read, roll over, succeed).
  - `test_link_error_below_100_surfaces`: the link fails and the re-read
    total is below 100 → the error is raised with GitHub's text, and no
    retry or heal runs.
  - `test_duplicate_continuations_keep_lower`: C2a (#201) and C2b (#205)
    under the head; items of #205 moved to #201, #205 unlinked, then closed
    as not planned with a comment naming #201 — assert that order.
  - `test_close_chain_head_first`: closes head, C2, C3 in that order.
  - `test_close_chain_skips_already_closed`.
  - `test_heal_rollover_disabled_creates_nothing`: rollover off, the head at
    100 with no continuation → `heal_full_tail` returns `False` and creates
    and moves nothing; a link meeting `LIMIT_ERROR` raises
    `RolloverDisabledError`, not `ChainLimitError`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -k "heal or limit or duplicate or close_chain" -v`
  Expected: FAIL.

- [ ] **Step 3: Implement**: `link_into` reads the tail back after each link
  and calls `heal_full_tail` and `heal_duplicate_continuations`. On any
  `addSubIssue` failure it re-reads the target's sub-issue total: a total of
  100 means full, whatever the error text says (`LIMIT_ERROR` is a hint
  only), and triggers the re-read and retry, never a raw surface. Any other
  failure is raised with GitHub's error. With `ROLLOVER_ENABLED` off, no
  heal creates a continuation, and a full head raises
  `RolloverDisabledError`.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "heal racing rollovers and close a chain head first"`.

### Task 6: chain-aware `epics.py` (vergil-tooling)

Spec §4.1 (lookups, ref resolution), §4.2 (public readers), §4.6. Blocked by
2 and 5c.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`child_states`,
  `child_states_with_blockers`, `native_children`, `all_children_closed`,
  `parent_of`, `native_parent_of`, `resolve_epic_ref`,
  `rollup`, `drain_standing_child`, `plan_standing_drain`,
  `apply_standing_drain`, `_find_epic_by_title`, `standing_kind_of`,
  `queue_kind`, `epic_category`, `unbuildable_parent_kind`)
- Modify: `src/vergil_tooling/bin/vrg_epic_create.py:166` (`--proxy-for`)
  and `src/vergil_tooling/bin/vrg_epic_docs.py:65` (`set --epic`): parse
  the ref through `resolve_epic_ref`, not raw `parse_issue_ref`
- Modify: `src/vergil_tooling/bin/vrg_epic_link.py:52`,
  `src/vergil_tooling/bin/vrg_epic_audit.py:66` and
  `src/vergil_tooling/lib/intake_route.py:1135`: take the epic ref through
  `resolve_epic_ref` instead of `qualify_epic_ref`, which stays a pure
  string function with no network call
- Modify: `src/vergil_tooling/lib/proxies.py:236-252`
  (`_require_open_development_epic` resolves a continuation to its head
  before its kind checks)
- Modify: `docs/site/docs/standards/github-issues.md` (a new "Chained epics"
  section: the chain, the label and lines, first fit, the messages, the
  depth refusal)
- Test: `tests/vergil_tooling/test_epics.py`, `test_proxies.py`,
  `test_vrg_epic_create.py`, `test_vrg_epic_docs.py`, `test_vrg_epic_link.py`,
  `test_vrg_epic_audit.py`, `test_intake_route.py`

**Interfaces:**

- Consumes: `epic_chain.chain_child_states`, `head_of`, `find_orphans`,
  `find_or_create_continuation`, `close_chain`.
- Produces: the readers keep their signatures and answer for the chain,
  built on `epic_chain` and on the raw per-container reads 5a introduced
  (`_container_child_states`, `_container_parent`), which are unchanged.
  `resolve_epic_ref` prints `<continuation> is a continuation of <head>;
  using <head>` to stderr when it maps one.

- [ ] **Step 1: Write the failing tests** (on `FakeSubIssueGraph`)

  - `test_resolve_epic_ref_maps_continuation_to_head`: `--epic` naming C2
    resolves to the head with the stderr line; a kind check on the result
    sees the head's labels.
  - `test_proxy_for_continuation_uses_head`: `vrg-epic-create --proxy-for`
    naming C2 → the stderr line, and the proxy's `Primary:` line names the
    head; `_require_open_development_epic` called directly with C2 checks
    the head's kind.
  - `test_epic_link_and_audit_continuation_use_head`: `vrg-epic-link --epic`
    and `vrg-epic-audit --epic` naming C2 → the stderr line, and each acts
    on the head; `intake_route` given C2 as its target routes to the head.
  - `test_epic_docs_continuation_uses_head`: `vrg-epic-docs set --epic`
    naming C2 → the stderr line, and the Documents section is written to the
    head's body, never C2's.
  - `test_parent_of_item_in_continuation_is_head`.
  - `test_parent_of_item_in_orphan_resolves_without_writing`: an item in an
    orphaned C2 → `parent_of` returns the head named by C2's trusted
    `Continues:` line and reports the drift; the fake records no write.
  - `test_child_states_spans_chain`,
    `test_child_states_with_blockers_spans_chain`,
    `test_native_children_spans_chain` and
    `test_all_children_closed_spans_chain`: each answers for head and C2
    together, continuations excluded.
  - `test_rollup_waits_for_open_item_in_continuation`: all head items closed,
    one C2 item open → closing a head item does not close the epic; closing
    the C2 item closes head then C2.
  - `test_rollup_continuation_close_is_noop`: closing C2 itself rolls nothing.
  - `test_rollup_adopts_orphan_before_judging`: an orphan of the head holding
    an open item → adopted, epic stays open.
  - `test_rollup_refuses_with_unadoptable_orphan`: an orphan with an
    untrusted body → no close, the orphan reported.
  - `test_drain_from_continuation_uses_head_kind`: a closed item in an
    intake chain's C2 drains to the intake archive.
  - `test_standing_lookup_never_finds_continuation`: an intake head and its
    continuation (same title prefix) → `ensure_standing_epic` returns the
    head; `standing_kind_of(continuation labels)` is `None`.
  - `test_batch_drain_spans_chain`: closed items in an intake head and in
    its C2 → `plan_standing_drain` lists both, and `apply_standing_drain`
    moves both to the intake archive.
  - `test_unbuildable_parent_kind_in_continuation`: an intake item linked in
    the intake chain's C2 → `unbuildable_parent_kind` returns `intake`, so
    report-ready, `vrg-submit-pr` and the batch rules still refuse it.
  - `test_epic_category_of_continuation_is_head_category`: `epic_category`
    of an intake chain's C2 returns the intake head's category, never
    `development`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_proxies.py tests/vergil_tooling/test_vrg_epic_create.py tests/vergil_tooling/test_vrg_epic_docs.py tests/vergil_tooling/test_vrg_epic_link.py tests/vergil_tooling/test_vrg_epic_audit.py tests/vergil_tooling/test_intake_route.py -v`
  Expected: the new tests FAIL; the existing ones still pass.

- [ ] **Step 3: Implement.** The reflink fallback in `child_states` and
  `parent_of` is kept, applied to the head.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "read epics, parents and rollups across the whole chain"`.

### Task 7: the CLI writers through the chain (vergil-tooling)

Spec §4.2 (writer callers), §4.5 (create-then-link checks first,
`vrg-epic-move`). Blocked by 3, 6, 9a and 10 (readers before writers).

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_link.py:61`,
  `vrg_issue_create.py:189`, `vrg_intake_create.py:75`,
  `vrg_security_record.py:237,343,389` (`_find_resumed_issue` included),
  `vrg_epic_move.py`
- Modify: `src/vergil_tooling/lib/ops_report.py:300-302` and `:448` (the
  create, which now runs `check_capacity` first)
- Test (new tests, and the existing raw-writer mocks ported to
  `FakeSubIssueGraph`): `tests/vergil_tooling/test_vrg_issue_create.py`,
  `test_vrg_epic_move.py`, `test_vrg_security_record.py`,
  `test_vrg_epic_link.py`, `test_vrg_intake_create.py`, `test_ops_report.py`

**Interfaces:**

- Consumes: `epic_chain.check_capacity`, `link_into`, `move_into`,
  `chain_contains`, `chain_child_states`.

- [ ] **Step 1: Write the failing tests**

  - `test_issue_create_depth_refusal_creates_nothing`: `check_capacity`
    raises `ChainDepthError` → exit 1 with its message, and
    `github.create_issue` is never called.
  - Task 2's `test_issue_create_closed_epic_refused_before_create` keeps
    passing once its state check is replaced by `check_capacity`.
  - `test_issue_create_rollover_disabled_full_epic_creates_nothing`:
    rollover off and the epic at 100 → exit 1 with `RolloverDisabledError`'s
    message, and `github.create_issue` is never called.
  - `test_intake_create_rolls_over_full_queue`: an intake head at 99 → the
    item lands in C2 and the queue nudge is printed.
  - `test_security_record_resume_finds_item_in_continuation`: the resumed
    intake issue sits in C2 and is found.
  - `test_epic_link_into_full_epic_rolls_over`.
  - `test_epic_move_into_chain`: moved with `move_into`; `already under epic`
    is decided with `chain_contains`.
  - `test_ops_report_links_through_chain`.
  - `test_intake_create_refusal_creates_nothing`,
    `test_security_record_refusal_creates_nothing` and
    `test_ops_report_refusal_creates_nothing`: for each, `check_capacity`
    raising `ChainDepthError`, and then rollover off with the epic at 100
    (`RolloverDisabledError`), → a non-zero exit with the error's message,
    and `github.create_issue` is never called.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_issue_create.py tests/vergil_tooling/test_vrg_intake_create.py tests/vergil_tooling/test_vrg_security_record.py tests/vergil_tooling/test_vrg_epic_link.py tests/vergil_tooling/test_vrg_epic_move.py tests/vergil_tooling/test_ops_report.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `check_capacity` runs before every create
  (`vrg-issue-create`, `vrg-intake-create`, `vrg-security-record` and
  `ops_report`), replacing Task 2's state check in `vrg-issue-create`; the link-failure
  hint that suggested `vrg-epic-move` names the chain error instead.

- [ ] **Step 4: Port the existing tests.** In each test file above, replace
  every `patch(...epics.add_child)`, `remove_child` and `reparent_child`
  mock and its `assert_called_*` with a `FakeSubIssueGraph` fixture and an
  assertion on the graph's end state (which container holds the task, and
  that it has one parent). Each ported test keeps the behaviour it pinned;
  none is deleted or weakened. After this step these files name no raw
  writer.

- [ ] **Step 5: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 6: Commit**: `vrg-commit --type feat --scope epics --message "link, move and create through the chain layer, checking capacity before creating"`.

### Task 8: `intake_route` through the chain (vergil-tooling)

Spec §4.2 (settling after a transfer, writer callers). Blocked by 6, 9a and
10 (readers before writers).

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (lines 924, 947, 1218,
  1530-1561 (`_link` and its `force` mode), 1613-1615; `_observe`,
  `_settle_parent`, `_clear_duplicate`, `_relink_after_failed_transfer`)
- Modify: `src/vergil_tooling/lib/epic_chain.py` (settling operations,
  `relink`)
- Test (new tests, and the existing raw-writer and raw-reader mocks ported
  to `FakeSubIssueGraph`): `tests/vergil_tooling/test_intake_route.py`,
  `test_epic_chain.py`

**Interfaces:**

- Produces in `epic_chain`:
  - `observe_parent(head: IssueRef, task: IssueRef) -> tuple[IssueRef | None, list[IssueRef]]`
    (the task's parent field, and the containers of *head*'s chain whose
    listing holds it)
  - `clear_stale_listing(head: IssueRef, task: IssueRef, keep: IssueRef) -> None`
  - `relink(head: IssueRef, task: IssueRef) -> IssueRef`: re-links *task*
    with `replaceParent` into the container of *head*'s chain that its
    parent field names (the #4291 case: the parent field names it, its
    listing lacks it), and returns that container. Raises `ValueError`
    naming both when the parent field names no container of the chain.
- `move_into(head, task)` for a task already linked in the chain (its parent
  field names a container whose listing holds it) changes nothing and
  returns that container; forcing a re-link is `relink`'s job.

- [ ] **Step 1: Write the failing tests**

  - `test_route_item_from_continuation`: an intake item in C2 routed to
    planning is unlinked from C2 and linked into the planning chain.
  - `test_route_into_full_planning_rolls_over`.
  - `test_settle_after_transfer_finds_listing_in_continuation`: the stale
    listing (the #4004 case) is in C2 and is cleared there.
  - `test_relink_after_failed_transfer_uses_chain`.
  - `test_relink_forces_listing_in_continuation` (epic_chain): the task's
    parent field names C2, C2's listing lacks it → one `addSubIssue` with
    `replaceParent: true` into C2, and the listing then holds it.
  - `test_move_into_task_already_in_chain_is_noop` (epic_chain).
  - The existing #3839/#4004/#4291 regression tests are ported (Step 4),
    with the same end-state assertions; the #4291 test's forced re-link
    goes through `relink`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_intake_route.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `intake_route` makes no raw read or write;
  `_link`'s `force` mode calls `epic_chain.relink`.

- [ ] **Step 4: Port the existing tests.** In `test_intake_route.py`,
  replace every mock of `epics.add_child`, `reparent_child`, `remove_child`,
  `native_parent_of` and `native_children` with a `FakeSubIssueGraph`
  fixture that models the same parent-field and listing states, and assert
  on the graph's end state instead of on the raw calls. Each ported test
  keeps the behaviour it pinned; none is deleted or weakened. After this
  step the file names no raw writer.

- [ ] **Step 5: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 6: Commit**: `vrg-commit --type feat --scope intake-route --message "route and settle intake items across chained queues"`.

### Task 9a: the audit's read side (vergil-tooling)

Spec §4.1 (consumers skip continuations), §4.2 (no raw queries), §4.7
(reporting). Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`list_home_epics` consumers:
  `epic_documents`, `closed_epic_open_child`, `proxy_link_drift`,
  `misplaced_adhoc`; `_OPEN_ISSUES_QUERY` and its listing, moved out;
  `OpenIssue`; `epic_children`; new `chain_drift`)
- Modify: `src/vergil_tooling/lib/epics.py` (new `list_open_issues`, holding
  the batched listing and its `parent` selection, inside the Task 11
  boundary)
- Modify: `src/vergil_tooling/lib/epic_docs.py` (`is_skipped`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (`containers` key and
  chain-drift exit with `--epic`; drift sections)
- Modify: `src/vergil_tooling/lib/epic_audit.py` `render_epic_children` (the
  `containers` key)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `test_vrg_epic_audit.py`, `test_epic_docs.py`

**Interfaces:**

- Produces: `epic_audit.ChainDrift` dataclass (`head: IssueRef`, `kind: str`
  one of `drift`, `full-tail`, `duplicate`, `closed-head-open-continuation`,
  `orphan`, `closed-continuation`; `containers: tuple[IssueRef, ...]`;
  `rule: str`), and `chain_drift(home: str) -> list[ChainDrift]`. 9b heals
  from this list.
- Produces in `epics`: `list_open_issues(repo: str) -> list[OpenIssueRecord]`,
  the same one batched GraphQL listing `_OPEN_ISSUES_QUERY` makes today,
  with each issue's native parent and that parent's labels. A parent that
  carries the `continuation` label is mapped to its head with
  `epic_chain.head_of`, once per distinct continuation, cached for the
  listing. Each record has `epic` (the head, or the parent itself when it
  is not a continuation; `None` when unparented) and `container` (the raw
  native parent). `OpenIssue.native_parent` is filled from `epic`, so the
  orphan pass, nativize, `misplaced_archives` and `misplaced_adhoc` keep
  reading the field they read today, with no per-issue read.
- Changes `vrg-epic-audit --epic`: its stdout stays exactly one JSON object
  (the contract its consumers pipe into `jq`), which gains a `containers`
  key, the integer count of the head's containers (1 for an unchained
  epic). When the chain walk raises `ChainDriftError`, `--epic` prints the
  drift (head, container and rule) on stderr, prints nothing on stdout and
  exits 1, as it already does for proxy link drift. A closed continuation
  in an open chain is not drift: it is counted, with a note on stderr, and
  the exit stays 0.

- [ ] **Step 1: Write the failing tests**

  - `test_continuation_not_reported_missing_documents`.
  - `test_continuation_not_a_closed_epic_with_open_child`.
  - `test_continuation_not_a_primary_in_proxy_drift`.
  - `test_misplaced_adhoc_sees_task_in_adhoc_continuation`: the task's raw
    parent is the ad-hoc C2; it maps to the ad-hoc head.
  - `test_epic_children_spans_chain_and_proxies`.
  - `test_audit_epic_shows_container_count`: `--epic` on a 3-container chain
    prints one JSON object whose `containers` is 3, and stdout parses as
    JSON with nothing else on it; an unchained epic gives `containers` 1.
  - `test_audit_epic_chain_drift_exits_1`: `--epic` on a chain whose C2
    fails the `Continues:` rule exits 1, names the head, C2 and the rule on
    stderr, and prints nothing on stdout.
  - `test_audit_reports_each_drift_kind`: one fixture per `ChainDrift.kind`.
  - `test_audit_notes_closed_continuation_in_open_chain`: reported as a note,
    exit 0.
  - `test_audit_lists_orphan_continuation`.
  - `test_orphan_detection_still_sees_native_parents`: an open issue with a
    native parent is not an orphan, and one with none is.
  - `test_misplaced_archives_still_sees_native_parent`.
  - `test_open_issue_in_continuation_maps_to_head` (epics): an issue in C2
    has `epic` the head and `container` C2.
  - `test_list_open_issues_is_one_batched_read` (epics): one listing query
    per page, and one `head_of` per distinct continuation however many
    issues it holds; no per-issue parent read.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py tests/vergil_tooling/test_epic_docs.py tests/vergil_tooling/test_epics.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `_OPEN_ISSUES_QUERY` and its listing move
  into `epics.list_open_issues`, keeping the `parent` selection there;
  `epic_audit` fills `OpenIssue.native_parent` from each record's `epic`,
  so every check that reads it sees the chain's head.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-audit --message "audit chained epics: skip continuations, read across the chain and report chain drift"`.

### Task 9b: the sweep's writes (vergil-tooling)

Spec §4.6 (every close closes the chain), §4.7 (nudge, heals, orphans).
Blocked by 9a and 10 (readers before writers).

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_QueueCapacity`,
  `StandingEpicFull`, `attach_orphans`, the re-links at line 1197,
  `close_drift`, `close_proxies`; new `heal_chains`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py`
- Modify: `docs/site/docs/standards/github-issues.md` (the orphan pass: the
  nudge replaces the stop; the chain heals)
- Test (new tests, and the existing raw-writer mocks ported to
  `FakeSubIssueGraph`): `tests/vergil_tooling/test_epic_audit.py`,
  `test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `ChainDrift`, `chain_drift` (9a); `epic_chain.link_into`,
  `heal_full_tail`, `heal_duplicate_continuations`, `close_chain`,
  `find_orphans`, `find_or_create_continuation`; `epics.rollup`.
- Produces: `heal_chains(drift: list[ChainDrift]) -> list[str]` (one line per
  heal).

- [ ] **Step 1: Write the failing tests**

  - `test_orphan_pass_past_95_links_and_nudges`: the intake head has 95 open
    children → the orphans are linked through the chain, exit 0, and the
    report has `intake queue <ref> holds 95 open items; run epic-review
    intake`. Replaces the `StandingEpicFull` stop test.
  - `test_sweep_heals_full_tail`, `test_sweep_heals_duplicates`,
    `test_sweep_closes_open_continuations_of_closed_head`.
  - `test_sweep_adopts_orphan_and_reruns_rollup`: the orphan held the last
    open item, now closed → after adoption the rollup closes the chain.
  - `test_sweep_reports_orphan_of_closed_head_without_adopting`: an orphan
    whose `Continues:` head is closed → reported as orphan drift naming the
    orphan and the closed head, nothing linked, the pass goes on and exits
    non-zero at the end.
  - `test_sweep_reports_chained_development_epics`: each open development or
    proxy epic with a continuation is a report line, not a failure.
  - `test_close_drift_closes_whole_chain` and
    `test_close_proxies_closes_whole_chain`.
  - `test_orphan_pass_rollover_disabled_reports_full_head`: rollover off and
    the intake head at 100 → each orphan is reported with
    `RolloverDisabledError`'s message, as a closed epic is (Task 2), nothing
    is linked or created, the pass goes on, and it exits non-zero at the
    end.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_audit.py -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: `StandingEpicFull` is removed; `_QueueCapacity`
  only counts, for the nudge.

- [ ] **Step 4: Port the existing tests.** In both test files, replace every
  `epics.add_child` and `reparent_child` mock and its `assert_called_*` with
  a `FakeSubIssueGraph` fixture and an assertion on the graph's end state.
  Each ported test keeps the behaviour it pinned; none is deleted or
  weakened. After this step these files name no raw writer.

- [ ] **Step 5: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 6: Commit**: `vrg-commit --type feat --scope epic-audit --message "sweep through chained epics: nudge past the queue cap, heal chains and close them whole"`.

### Task 10: the remaining readers (vergil-tooling)

Spec §4.2 (readers), §4.1 (lookups). Blocked by 6.

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py`, `roadmap.py`,
  `planning_close.py`, `security_records.py`, `merge_review/request.py`
  (`_fetch_task`, `_open_sub_issues`, `_primary_open_tasks` and their raw
  queries), `src/vergil_tooling/lib/pr_workflow/batch_rules.py` (the
  batch "same epic" rule `vrg-submit-pr` runs)
- Test: the matching test files

**Interfaces:**

- Consumes: the chain-aware `epics` readers (Task 6); `epics.is_continuation`.

- [ ] **Step 1: Write the failing tests**, one per module, each with an item
  in a continuation:
  - `test_queue_view_lists_items_in_continuation`, and the planning view
    does not list the continuation as an item;
  - `test_roadmap_skips_continuations`;
  - `test_planning_close_reads_chain`;
  - `test_security_records_reads_chain`;
  - `test_merge_review_open_tasks_span_chain`: a task in C2 gets its epic's
    open tasks from the whole chain, and `_fetch_task` reports the head as
    its parent;
  - `test_submit_pr_batch_same_epic_across_containers`.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling -k "continuation or chain" -v`
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement**: each module's own `parent`/`subIssues` GraphQL
  selection is replaced with an `epics` reader.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "read queues, roadmaps, planning closes, advisories and merge-review scopes across the chain"`.

### Task 10b: `vrg-epic-parent` (vergil-tooling)

Spec §4.2 (every raw parent read moves behind the chain layer). Blocked
by 6.

The plugin skills find a task's epic with `vrg-gh issue view <N> --json
parent` and the parent's labels, which names a continuation, not its epic,
for a task in a chain. This command gives them the chain-aware answer.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_epic_parent.py`
- Modify: `pyproject.toml` (the console script)
- Create: `docs/site/docs/reference/` page for the command, beside the other
  CLI pages
- Test: `tests/vergil_tooling/test_vrg_epic_parent.py`

**Interfaces:**

- Consumes: `epics.parent_of` (chain-aware since Task 6), the head's labels,
  `epics.standing_kind_of`, `epics.is_proxy`.
- Produces: `vrg-epic-parent <owner>/<repo>#<N> [--json]`. Plain output is
  one line, `<head ref> <kind>`, `kind` one of `development`, `proxy`,
  `ad-hoc`, `intake`, `planning`, `deferred` or another standing kind's
  name; `--json` prints `{"task", "epic", "kind", "container"}`, with
  `container` the native parent (the head itself, or the continuation that
  holds the task). A task with no parent prints `none` (`--json`: `epic`
  null) and exits 0; a read failure or chain drift exits 1 with its error.
  Two more kinds cover a parent that is not an open-work epic, so the skills
  keep today's rule that a task is under an epic only when its parent
  carries `epic`:
  - **`not-epic`:** the native parent lacks the `epic` label (a plain issue,
    such as a sub-task's parent). Plain output is `<parent ref> not-epic`;
    `--json` has `epic` null, `kind` `not-epic` and `container` the parent.
    Exit 0.
  - **`archive`:** the native parent is an archive (the `archive` label).
    Plain output is `<archive ref> archive`; `--json` has `epic` the archive,
    `kind` `archive` and `container` the archive. Exit 0. No kind falls
    through to `development` by default: `development` is printed only for
    an `epic`-labelled head that is none of the other kinds.

- [ ] **Step 1: Write the failing tests** (on `FakeSubIssueGraph`)

  - `test_task_in_head`: prints the head and its kind; `container` is the
    head.
  - `test_task_in_continuation_reports_head`: a task in an intake chain's C2
    prints the intake head and `intake`; `container` is C2.
  - `test_proxy_task_kind`: a task under a proxy epic prints `proxy`.
  - `test_unparented_task`: `none`, exit 0.
  - `test_non_epic_parent_is_not_epic`: a task whose parent is a plain issue
    prints `<parent> not-epic`; `--json` has `epic` null; exit 0.
  - `test_archive_parent_is_archive`: a task drained into an archive prints
    `<archive> archive`.
  - `test_chain_drift_exits_1`: an untrusted continuation body → exit 1
    naming the drift rule.

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_vrg_epic_parent.py -v`
  Expected: FAIL (command does not exist).

- [ ] **Step 3: Implement** the command; it makes no raw read.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epics --message "add vrg-epic-parent, which names a task's epic and kind across a chain"`.

### Task 11: lock the raw layer (vergil-tooling)

Spec §4.2 (raw writers private, agents' raw sub-issue flags included; no
raw queries outside the chain layer).
Blocked by 7, 8, 9b, 10, 10b.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`add_child` → `_add_child`,
  `remove_child` → `_remove_child`, `reparent_child` → `_reparent_child`;
  their uses in the archive drain stay, as `epics`-internal)
- Modify: `src/vergil_tooling/lib/epic_chain.py`
- Modify: every test file that still names a raw writer after Tasks 7, 8
  and 9b ported theirs: at least `tests/vergil_tooling/test_epics.py` (the
  raw writers' own tests, moved to the private names) and
  `test_vrg_epic_create.py`; the implementer greps `tests/` for the public
  names first and lists every hit in the PR.
- Create: `tests/vergil_tooling/test_chain_layer_boundary.py`
- Modify: `src/vergil_tooling/bin/vrg_gh.py` (refuse `issue edit`'s
  `--parent`, `--add-sub-issue` and `--remove-sub-issue`), and its test
  `tests/vergil_tooling/test_vrg_gh.py`

- [ ] **Step 1: Write the failing tests**

  `test_issue_edit_refuses_raw_subissue_flags` (vrg-gh): each of
  `issue edit <N> --parent <M>`, `--add-sub-issue <M>` and
  `--remove-sub-issue <M>` exits non-zero with a message naming the flag and
  pointing to `vrg-epic-link` and `vrg-epic-move`, and runs no `gh`; an
  `issue edit` with only other accepted flags still runs.

  `test_no_raw_subissue_access_outside_chain_layer`: parses every `.py`
  under `src/vergil_tooling` except `lib/epics.py` and `lib/epic_chain.py`
  with `ast` (so comments are skipped, and every string form is seen alike:
  plain, f-string and t-string) and asserts that no `Name` or `Attribute`
  is `add_child`, `remove_child` or `reparent_child` (or their private
  forms), and that no string constant, including each literal part of a
  `JoinedStr` (f-string) or template string, matches the GraphQL
  selections `\bsubIssues\s*[({]`, `\bparent\s*\{\s*number`,
  `\baddSubIssue\s*\(`, `\bremoveSubIssue\s*\(` or the REST path
  `/sub_issues`. Docstrings and messages that merely name these (today
  `intake_route.py:1565-1580`, `epic_audit.py:1599`,
  `planning_close.py:158`) do not match the selection patterns; the test
  asserts that too, with those three lines as negative fixtures. A
  positive fixture copied from today's `vrg_security_record.py:237`
  (`f"repos/{intake.owner}/{intake.repo}/issues/{intake.number}/sub_issues"`)
  must be flagged, as must an f-string GraphQL query with a `subIssues(`
  selection. A failure lists each file and line.

- [ ] **Step 2: Run it to verify it fails** (on the public names), then
  rename.

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_chain_layer_boundary.py tests/vergil_tooling/test_vrg_gh.py -v`
  Expected: FAIL, then PASS after the renames and the `vrg-gh` refusal.

- [ ] **Step 3: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 4: Commit**: `vrg-commit --type refactor --scope epics --message "make the raw sub-issue layer private to epics and epic_chain"`.

### Task 11b: turn rollover on (vergil-tooling)

Spec §4.3 (rollover). Blocked by 11 and 14b: rollover is enabled only once
every tooling reader (Tasks 6, 9a, 10 and 10b) reads through the chain and
the plugin skills that read through it (Task 14) are released and installed
on every host and VM (Task 14b). Blocking on Task 14's merge alone is not
enough: plugin installs follow vergil-claude-plugin's releases, not its
develop branch, while vergil-tooling releases from develop often, so
merging 11b is in practice what turns rollover on.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_chain.py` (remove `ROLLOVER_ENABLED`,
  `RolloverDisabledError` and the disabled path)
- Modify: `docs/site/docs/standards/github-issues.md` (drop any wording that
  rollover is not enabled yet)
- Modify: every test that sets `ROLLOVER_ENABLED` with `monkeypatch` (the
  monkeypatch removed); remove the `*_rollover_disabled_*` tests with the
  path they pin. The implementer greps `tests/` and `src/` for
  `ROLLOVER_ENABLED` and `RolloverDisabledError` and lists every hit in the
  PR.

- [ ] **Step 1: Write the failing test.**
  `test_link_into_full_head_rolls_over_by_default` (`test_epic_chain.py`, on
  `FakeSubIssueGraph`, with no `ROLLOVER_ENABLED` monkeypatch): a link into
  a head holding 99 children creates C2 and links the item into it. Run
  `vrg-container-run -- pytest tests/vergil_tooling/test_epic_chain.py -k
  rolls_over_by_default -v`; it FAILS while the switch defaults to off.

- [ ] **Step 2: Remove the switch.** Delete the constant, the error and
  every branch on them, and the tests named above. The new test PASSES.

- [ ] **Step 3: Confirm nothing names it.** `ROLLOVER_ENABLED` and
  `RolloverDisabledError` appear nowhere under `src/` or `tests/`.

- [ ] **Step 4: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 5: Commit**: `vrg-commit --type feat --scope epic-chain --message "turn on rollover into continuations"`.

### Task 12: the vergil-tooling release, installed (deployment, .github)

Spec §4.8 (once released and installed). Blocked by 11.

- **Precondition (human-attested):** a vergil-tooling release carrying
  Tasks 2, 3, 5a, 5b, 5c, 6, 7, 8, 9a, 9b, 10, 10b and 11 is published, its
  version and tag named. It ships with rollover off; Task 11b is not in it,
  or the order of Tasks 14 and 11b is lost.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for each of those tasks. For a
     task with no record (exit 4), `vrg-git merge-base --is-ancestor
     <merge-sha> <tag-sha>` in vergil-tooling exits 0 instead.
  2. Confirm the release's `install` stage ran: it installs the release on
     the Mac that ran it and runs `vrg-vm update --all` on every VM.
     **Human-run steps (need a human identity):** on any other host,
     `vrg-tooling-install <version>`; `vrg-vm update --all` only if a VM
     was not reached by the install stage.
  3. Check the installed tool itself, not the host's Python (the install is
     an isolated `uv tool` environment): `vrg-epic-parent --help` runs (the
     command is new in this release and imports `epic_chain`), and the
     installed version reported by the tooling matches the release tag.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 12b: the rollover release, installed (deployment, .github)

Spec §4.8 (once released and installed). Blocked by 11b.

- **Precondition (human-attested):** a vergil-tooling release carrying
  Task 11b is published, its version and tag named.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for Task 11b. With no record
     (exit 4), `vrg-git merge-base --is-ancestor <merge-sha> <tag-sha>` in
     vergil-tooling exits 0 instead.
  2. Confirm the release's `install` stage ran, as Task 12's step 2 says,
     with the same **Human-run steps (need a human identity)**.
  3. The installed version reported by the tooling matches the release tag.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 13: re-home #1384's tasks, the live rollover (validation, .github)

Spec §4.8, goals 1, 3, 5 and 7. Blocked by 12b, which follows 11b and so
Task 14b: the installed skills read a task's epic through the chain before
any task can move into a continuation.

- **Precondition (machine probe):** vergil-project/.github#1384 is open
  (`vrg-gh issue view 1384 --repo vergil-project/.github --json state`). Its
  sub-issue total was 96 on 2026-11-24; record the current total, and
  whether #1384 already has a continuation (`vrg-epic-audit --epic
  vergil-project/.github#1384 | jq .containers` prints `<n>`, 1 with none),
  since tasks filed after Task 12b may already have rolled it over.
- **Procedure (if #1384 is open):**
  1. Move each of the 14 tasks listed on the planning issue
     (vergil-project/vergil-tooling#3840, comment of 2026-11-24) with
     `vrg-epic-move --epic vergil-project/.github#1384 --task <ref>`.
  2. Observe the rollover. If #1384 had no continuation at the
     precondition: once the head holds 99 items, the next move prints the
     development-epic rollover line, and the head gets the same comment. If
     it already had one: record the existing rollover comment on the head
     as the evidence, and confirm the moves went first-fit into the chain.
  3. `vrg-epic-audit --epic vergil-project/.github#1384` exits 0 (an exit
     of 1 is chain drift, named on stderr), its JSON `children` lists all
     14, and its `containers` is at least 2.
  4. The epic sweep's next scheduled run (`epic-sweep.yml`) reports #1384
     as a chained development epic, not a failure. The agent waits for it
     and reads its log with `vrg-gh run view`. **Human-run step (needs a
     human identity), optional:** to avoid the wait, the human runs `gh
     workflow run epic-sweep.yml --repo vergil-project/.github` and names
     the run; `vrg-gh` refuses `workflow run` for the agent.
  5. Tell the #1384 driver the hand tracking can be dropped; note it on
     #1384.
- **Fallback (if #1384 has closed):** the 14 tasks stay where they are
  (closed is closed). Make the live check on a throwaway development epic
  instead: `vrg-epic-create --repo vergil-project/.github --title "Epic:
  chain rollover probe"`, file 100 throwaway tasks under it with
  `vrg-issue-create --epic …` (titles `Probe: rollover <n>`), and observe
  step 2 against it, the case with no continuation. Then check that
  `vrg-epic-audit --epic <probe epic>` exits 0, its JSON `children` lists
  all 100 probe tasks, and its `containers` is 2. Close every task as not
  planned and let the rollup close the chain, confirming head-first.
  Last, comment on #1384's retrospective task (its `retrospective`-labelled
  child) listing the 14 tasks by full ref, each under its ad-hoc epic, so
  the retrospective records them as #1384's work (spec §4.8); if that
  retrospective has already merged, comment on #1384 itself.
- **SUCCESS:** the rollover line, the head comment and the audit output are
  recorded in the result comment, with no item left outside the chain.

### Task 14: the plugin skills read a task's epic through the chain (vergil-claude-plugin)

Spec §4.2 (every raw parent read moves behind the chain layer). Blocked by
12: the skills run the installed tooling, so `vrg-epic-parent` must be
released and installed first.

**Files:**

- Modify: `skills/issue-implement/SKILL.md` (the planning, intake and
  deferred gate, today `vrg-gh issue view <N> --json parent` and the
  parent's labels)
- Modify: `skills/merge-review/SKILL.md` (finding the task's epic and its
  design sources)
- Modify: `skills/epic-create/SKILL.md` (finding where an issue stands, in
  Entry points)
- Modify: `skills/epic-retrospective/SKILL.md` (labelling a task as a proxy
  task)

- [ ] **Step 1: Find every raw parent read.** Grep `skills/` and `agents/`
  for `--json parent` and for prose that tells the agent to read a task's
  native parent and then its labels. Every hit is either changed in Step 2
  or named in the PR with why it can stay (a read of an issue that is never
  in a chain).

- [ ] **Step 2: Replace each read** with `vrg-epic-parent <ref>` (or
  `--json` where the skill needs the container), and each check of the
  parent's labels with the `kind` it prints. merge-review takes the epic's
  Documents section from the head that `epic` names, never from the
  container. epic-retrospective labels a task as a proxy task only when
  `kind` is `proxy`. Each changed skill also states its action for the two
  kinds that are not open-work epics, keeping what it does today for such a
  parent: `not-epic` and `none` mean the task is under no epic (merge-review
  loads no Documents section; issue-implement's gate treats it as an
  unparented task), and `archive` means a closed item that no skill builds
  (issue-implement stops for the human, naming the archive).

- [ ] **Step 3: Validate**: `vrg-container-run -- vrg-validate` passes.

- [ ] **Step 4: Commit**: `vrg-commit --type feat --scope skills --message "read a task's epic and kind with vrg-epic-parent, so a task in a chained epic is judged by its epic"`.

### Task 14b: the plugin release, installed (deployment, .github)

Spec §4.8 (once released and installed). Blocked by 14. Task 11b is
blocked by this task, so rollover cannot be turned on while any host or VM
runs skills that read a task's raw native parent.

- **Precondition (human-attested):** a vergil-claude-plugin release carrying
  Task 14 is published and promoted (its `vX.Y` tag moved, so the
  marketplace serves it), its version and tag named. Cutting and promoting
  the release is the human's; the agent never performs it.
- **Procedure:**
  1. `vrg-release-record check <ref>` exits 0 for Task 14. With no record
     (exit 4), `vrg-git merge-base --is-ancestor <merge-sha> <tag-sha>` in
     vergil-claude-plugin exits 0 instead.
  2. **Human-run steps (need a human identity):** `vrg-host plugins` on
     every host, and `vrg-vm update --all` for the VMs. The human attests
     each host and VM refreshed.
  3. On this host, the installed plugin is the release's version, and its
     `skills/issue-implement/SKILL.md` and `skills/merge-review/SKILL.md`
     name `vrg-epic-parent` (a read of the installed plugin cache, under
     `~/.claude/plugins/cache/`).
- **SUCCESS:** every check passes, recorded with the plugin version and tag
  and the hosts and VMs the human attested.

## Evolution during execution
