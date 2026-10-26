# Deferred queue — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a third queue, `Epic (deferred): <name>`, in every queue home.
Defer moves an item into it with a recorded reason and revival conditions, and
three review skills process the queues. A one-time sweep empties intake and
planning of everything deferred today.

**Architecture:** vergil-tooling gains a `DEFERRED` standing kind next to
`INTAKE` and `PLANNING`, so it inherits the queue-home rule, the archives, the
drain, the audit treatment and the PR refusal. `vrg-intake-route` becomes the
router between all container epics (intake, planning, ad hoc, deferred), driven
by a per-source route table. `vrg-queue deferred` lists the queue. The
vergil-claude-plugin gains `deferred-review` and `planning-review` and updates
`intake-review` and `epic-create`. After both releases, an operational sweep
migrates the live queues.

**Tech Stack:** Python 3.12+ (`argparse`, `dataclasses`, pytest with
`unittest.mock`), `gh` via `lib/github.py`, Markdown skills.

**Spec:** `epics/508-deferred-queue/spec.md` (vergil-project/.github#508)

**Shape:** this is an epic-level plan. Each task below is **one issue and one
PR in the repo named in its heading**, worked in a worktree of that repo. Paths
are relative to that repo's root. Deployment tasks are operational: they have
no code PR and close on `Outcome: SUCCESS`.

## Global Constraints

- **Deferred state = native parent is a live deferred epic.** There is no per-item label. The epic carries `epic` + `deferred`.
- **Epic title** `Epic (deferred): <name>`. **Archive** `Archive (deferred): <name> — <YYYY>-Qn`, labels `archive` + `deferred`. **Sentinel** `deferred`.
- **Deferral comment**, exactly these line prefixes:
  `Deferred <YYYY-MM-DD>: <reason>` / `Revisit when: <conditions>` /
  `Deferred from: <owner>/<repo>#<N>`. The third line is written only when the
  item moves, never on a refresh. The content is never parsed for meaning.
- **Deferrable:** intake, planning and ad-hoc children only. A development-epic task, an epic, or an unparented issue is refused, and the refusal names the parent.
- **Entry and exit only through `vrg-intake-route`.** `vrg-epic-move` and `vrg-issue-create` refuse the deferred epic as a target, and `vrg-epic-move` refuses a task whose current parent is a deferred epic.
- **Existing guards hold for every route:** same org only, and a private item is never routed into a public repo or epic. Each route runs under the item owner's App token (`github.target_org`).
- **No silent failures.** A half-applied route raises with the state it left and the command that fixes it.
- **Releases are human-gated.** The agent never cuts one. During the accepted window between the tooling and plugin releases, intake review is not run.
- Validation gate: `vrg-container-run -- vrg-validate`. Commit with `vrg-commit`. Hand off with `vrg-pr-workflow report-ready`.

## Review Focus

1. **Deferring a task in a development epic** is refused, naming that epic. It is not moved, and nothing is posted. Pinned by `test_route_defer_refuses_development_epic_task` (Task 2).
2. **A planning item whose `epic-create` session already made an epic** (it has an `Epic:` marker) refuses both defer and drop. Pinned by `test_route_planning_with_epic_marker_refuses_defer_and_drop` (Task 2).
3. **An ad-hoc item with an open PR that would close it** is refused. A merged or closed PR does not block. Pinned by `test_route_defer_adhoc_refuses_open_closing_pr` and `test_route_defer_adhoc_ignores_closed_pr` (Task 2).
4. **A deferred item with no deferral comment** (hand-moved, or legacy) lists with an empty note, sorts first, and does not crash the view. Pinned by `test_deferred_items_without_deferral_sort_first` (Task 3).
5. **A PR against a deferred item** gets the deferred refusal text, not the `ValueError("not a queue kind")` that `queue_refusal_message` raises today for an unknown kind. Pinned by `test_queue_refusal_message_deferred` (Task 1).

---

### Task 1: Deferred standing kind (vergil-tooling, implementation)

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py`:
  - the `StandingKind` table, `STANDING_KINDS` and `EPIC_SENTINELS`
  - `queue_kind`, `queue_item_kind` and `_QUEUE_REFUSAL_TAILS`
  - `resolve_epic_ref`, `_QUEUE_KINDS` (renamed to public `QUEUE_KINDS`) and `drain_standing_org`
  - the docstrings that say "intake or planning"
- Modify: `src/vergil_tooling/lib/epic_audit.py`: `_QUEUE_FULL_HINT`, `public_self_homed_queues` and `move_public_queues` (line ~703 currently maps every non-intake key to `PLANNING`).
- Modify: `src/vergil_tooling/bin/vrg_epic_move.py`, `src/vergil_tooling/bin/vrg_issue_create.py`: the deferred refusals.
- Modify: `src/vergil_tooling/data/labels.json`: add `{"name": "deferred", "color": "c5def5", "description": "Perpetual deferred queue of evaluated items parked until their revival conditions are met; never auto-closes"}`.
- Test: `tests/vergil_tooling/test_epics.py`, `test_epic_audit.py`, `test_vrg_epic_move.py`, `test_vrg_issue_create.py`

**Interfaces:**

- Produces:
  - `epics.DEFERRED: StandingKind`, with key `"deferred"`, the sentinel, title, archive and labels from Global Constraints, and body
    `"Perpetual deferred queue for {name}: evaluated items parked until their revival conditions are met. Reviewed by deferred-review.\n"`
  - `epics.QUEUE_KINDS: tuple[StandingKind, ...] = (INTAKE, PLANNING, DEFERRED)`
  - `epics.QUEUE_KIND_BY_KEY: dict[str, StandingKind]`
  - `epics.ensure_deferred_epic(target_repo: str) -> IssueRef`
  - `epics.queue_kind(epic) -> str | None`, now also `"deferred"`
  - `epics.container_kind(epic: IssueRef) -> str | None`, which returns `"ad-hoc"`, `"intake"`, `"planning"` or `"deferred"` for a live standing epic and None otherwise
  - `epics.DEFERRED_ROUTE_ONLY: str`, the shared refusal text pointing at `vrg-intake-route`

- [ ] **Step 1: Write the failing tests**

```python
def test_deferred_is_a_standing_queue_kind() -> None:
    assert epics.DEFERRED in epics.STANDING_KINDS
    assert epics.QUEUE_KINDS == (epics.INTAKE, epics.PLANNING, epics.DEFERRED)
    assert "deferred" in epics.EPIC_SENTINELS
    assert epics.standing_kind_of({"epic", "deferred"}) is epics.DEFERRED
    assert epics.standing_kind_of({"archive", "deferred"}) is None

def test_archive_re_matches_deferred_archive() -> None:
    m = epics.archive_re(epics.DEFERRED).match("Archive (deferred): org — 2026-Q4")
    assert m and m["name"] == "org" and m["quarter"] == "2026-Q4"

def test_queue_kind_and_container_kind_deferred_and_adhoc() -> None:
    # patch _labels: {"epic","deferred"} -> "deferred" for both; {"epic","ad-hoc"} -> queue_kind None, container_kind "ad-hoc"
    ...

def test_ensure_deferred_epic_uses_queue_home() -> None:
    # public member repo "org/tool" -> ensure_standing_epic(DEFERRED, "org/.github", "org")
    # private self-homed "org/secret" -> ensure_standing_epic(DEFERRED, "org/secret", "secret")
    ...

def test_queue_refusal_message_deferred() -> None:
    msg = epics.queue_refusal_message("org/tool#5", "deferred")
    assert msg == ("--issue (org/tool#5) is in the deferred queue; it is deferred until its "
                   "revival conditions are met. Revive it with deferred-review first.")

def test_drain_standing_org_covers_deferred() -> None:
    # plan_standing_drain called once per (kind, home) for kind in (INTAKE, PLANNING, DEFERRED)
    ...
```

In `test_epic_audit.py`:

- `test_move_public_queues_moves_deferred_to_dotgithub_deferred`: a self-homed deferred epic's children are reparented under `ensure_standing_epic(DEFERRED, "org/.github", "org")`, not under `PLANNING`.
- `test_public_self_homed_queues_includes_deferred`.
- `test_queue_full_hint_deferred`, which asserts `_QUEUE_FULL_HINT["deferred"] == "Run deferred-review."`.

In the CLI tests:

- `test_epic_move_refuses_deferred_sentinel`, `test_epic_move_refuses_deferred_epic_number` (resolved epic's `queue_kind` is `"deferred"`), `test_epic_move_refuses_task_under_deferred_epic`: each exits 1, stderr contains `vrg-intake-route`, and no `add_child` / `remove_child` call is made.
- `test_issue_create_refuses_deferred_epic`: same shape for `--epic deferred`.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_epic_audit.py tests/vergil_tooling/test_vrg_epic_move.py tests/vergil_tooling/test_vrg_issue_create.py -q`
Expected: FAIL (`AttributeError: ... has no attribute 'DEFERRED'`).

- [ ] **Step 3: Implement** the Interfaces above.

  - `resolve_epic_ref` maps the `deferred` sentinel to `ensure_deferred_epic`, so internal callers can resolve it. The two CLIs check for it themselves: either `args.epic == epics.DEFERRED.sentinel`, or the resolved epic's `queue_kind` is `"deferred"`. Both checks run before any write.
  - `move_public_queues` uses `QUEUE_KIND_BY_KEY[key]`.
  - `ensure_standing_epic` already ensures the kind's labels exist at the point of use (#3378). Confirm that a test covers `deferred` the same way.

- [ ] **Step 4: Run the tests to verify they pass**, then run the full gate: `vrg-container-run -- vrg-validate`. Expected: PASS.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope epics --message "add the deferred standing queue kind"`

---

### Task 2: Route into and out of deferred (vergil-tooling, implementation)

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py`: the module docstring, the action dataclasses, the comment constants, `_route` and `_check_eligible`, plus new guards.
- Modify: `src/vergil_tooling/bin/vrg_intake_route.py`: `defer` flags and the new `intake` subcommand.
- Modify: `docs/site/docs/reference/cli-tools-overview.md`: the `vrg-intake-route` entry, covering the route table and the new flags.
- Test: `tests/vergil_tooling/test_intake_route.py`, `tests/vergil_tooling/test_vrg_intake_route.py`

**Interfaces:**

- Consumes (Task 1): `epics.container_kind`, `epics.ensure_deferred_epic`, `epics.ensure_intake_epic`, `epics.PLANNING_IN_PROGRESS_LABEL`; also `queue_view.parse_planning_markers` (existing).
- Produces:
  - `Defer(reason: str, revisit_when: str, day: date | None = None)`, where `day` overrides today.
  - `ToIntake()`.
  - `DEFER_COMMENT = "Deferred {day}: {reason}\nRevisit when: {revisit_when}"`
  - `DEFERRED_FROM_LINE = "Deferred from: {epic}"`
  - `DROP_COMMENT = "Dropped in {queue} review: {reason}"`
  - `ROUTES: dict[str | None, frozenset[type]]`, the table below, where `None` is unparented.
  - CLI: `defer --reason TEXT --revisit-when TEXT [--date YYYY-MM-DD]` and `intake`. Tasks 5, 6, 7 and 9 call these exactly.

| `container_kind(parent)` | allowed |
| --- | --- |
| `"intake"` | Drop, Defer, OneOff, ToPlanning, Split |
| `"planning"` | Defer, Drop |
| `"ad-hoc"` | Defer |
| `"deferred"` | Defer (refresh), ToIntake, ToPlanning, OneOff, Drop, Split |
| `None` with no parent | ToPlanning |
| anything else (development epic) | none — refused, naming the parent |

**Behavior the tests pin:**

- **Defer, moving** (from intake, planning or ad hoc):
  1. Run the guards below.
  2. `_link(ensure_deferred_epic(repo), ref)`, with the visibility guard.
  3. Remove `in-progress` if present.
  4. Post `DEFER_COMMENT + "\n" + DEFERRED_FROM_LINE` (the old parent's slug).

  The link comes before the comment. If the comment fails after the move, raise
  `RuntimeError`, saying the item is under the deferred epic without a record
  and giving the `vrg-intake-route … defer` command to re-run. A re-run is a
  refresh, so it posts the record.
- **Defer, refresh** (already deferred): post `DEFER_COMMENT` only. No link, and no `Deferred from:` line.
- **Planning guard:** for Defer or Drop on a planning item, read its comments. If `parse_planning_markers(...).epic is not None`, refuse with "…has a development epic (<slug>); deferring or dropping it would orphan that epic".
- **Ad-hoc guard:** for Defer on an ad-hoc item, read `gh issue view --json closedByPullRequestsReferences` and each PR's state. Refuse if any PR is `OPEN`, naming it.
- **ToIntake:** `_link(ensure_intake_epic(repo), ref)`, with the visibility guard.
- **Drop:** the comment names the source queue (`intake`, `planning`, `deferred`).

- [ ] **Step 1: Write the failing tests** (extend `Env`: `kinds` becomes the `container_kind` map, plus `DEFERRED = IssueRef("org", ".github", 10)`, `DEV_EPIC = IssueRef("org", ".github", 400)`, issue comments and labels, and closing-PR fixtures)

```python
def test_route_defer_moves_intake_item_and_records(env: Env) -> None:
    route(ITEM, Defer(reason="waits on X", revisit_when="X ships"), today=TODAY)
    assert env.native_parent[ITEM] == DEFERRED
    env.run.assert_any_call("issue", "comment", "352", "--repo", "org/.github", "--body",
        "Deferred 2026-10-23: waits on X\nRevisit when: X ships\nDeferred from: org/.github#7")

def test_route_defer_date_override(env: Env) -> None:  # day=date(2026,10,24) -> "Deferred 2026-10-24:"
def test_route_defer_refresh_does_not_move_or_add_from_line(env: Env) -> None:
def test_route_defer_from_planning_removes_in_progress(env: Env) -> None:
def test_route_defer_from_adhoc_moves(env: Env) -> None:
def test_route_defer_refuses_development_epic_task(env: Env) -> None:
    # parent DEV_EPIC (container_kind None) -> RouteRefusedError mentioning "org/.github#400"; run, add_child, reparent_child not called
def test_route_defer_refuses_unparented(env: Env) -> None:
def test_route_planning_with_epic_marker_refuses_defer_and_drop(env: Env) -> None:
def test_route_defer_adhoc_refuses_open_closing_pr(env: Env) -> None:
def test_route_defer_adhoc_ignores_closed_pr(env: Env) -> None:
def test_route_defer_comment_failure_after_move_fails_loudly(env: Env) -> None:
def test_route_to_intake_from_deferred(env: Env) -> None:
def test_route_to_intake_refused_from_intake(env: Env) -> None:
def test_route_one_off_from_deferred_same_repo_reparents(env: Env) -> None:
def test_route_drop_names_source_queue(env: Env) -> None:  # planning -> "Dropped in planning review: ..."
def test_route_defer_refuses_private_item_to_public_deferred_epic(env: Env) -> None:
def test_routes_table_matches_spec() -> None:  # ROUTES equals the table above exactly
```

Update the existing tests whose contract changes: `test_route_defer_comments_with_date_only` becomes the move test, and `test_route_refuses_non_intake_item` now targets a development epic. In `test_vrg_intake_route.py`, add these:

- `--revisit-when` is required (exit 2).
- `--date` must be a valid `YYYY-MM-DD` (exit 2 otherwise).
- `intake` maps to `ToIntake()`.

- [ ] **Step 2: Run the tests to verify they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_intake_route.py tests/vergil_tooling/test_vrg_intake_route.py -q`. Expected: FAIL.

- [ ] **Step 3: Implement** per Interfaces and the behavior list. Keep `route()`'s signature; `today` remains the default when `Defer.day` is None.

- [ ] **Step 4: Run the tests, then `vrg-container-run -- vrg-validate`.** Expected: PASS.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope intake-route --message "route items into and out of the deferred queue"`

---

### Task 3: `vrg-queue deferred` (vergil-tooling, implementation)

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py`, `src/vergil_tooling/bin/vrg_queue.py`, `docs/site/docs/reference/cli-tools-overview.md` (`vrg-queue` entry)
- Test: `tests/vergil_tooling/test_queue_view.py`, `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**

- Consumes (Task 1): `epics.DEFERRED`.
- Produces:
  - `Deferral(day: str, reason: str, revisit_when: str)` (frozen dataclass).
  - `parse_latest_deferral(comments: Sequence[str]) -> Deferral | None`: the latest comment whose first line matches `^Deferred (\d{4}-\d{2}-\d{2}): (.*)$`, with `revisit_when` taken from its `Revisit when:` line (`""` if absent).
  - `deferred_items(home: str, name: str) -> list[QueueItem]`. Every state is `"waiting"`. The note is `"Deferred <day>: <reason> | Revisit when: <revisit_when>"`, or `""` with no deferral. Ordering is by `(day or "", created_at, slug)`, so an item with no deferral sorts first.
  - `vrg-queue deferred [--repo] [--json]`. A bare `vrg-queue` (positional now optional) prints all three queues in order (intake, planning, deferred), each with its usual header. In that mode, a missing queue epic prints `<kind> queue for <name> in <home>: no queue yet` and the run continues with exit 0. A named queue that is missing keeps today's exit 1.

- [ ] **Step 1: Write the failing tests**

```python
def test_parse_latest_deferral_latest_wins_and_reads_revisit_line() -> None:
def test_parse_latest_deferral_ignores_non_deferral_comments() -> None:
def test_deferred_items_least_recently_deferred_first() -> None:
    # A deferred 2026-10-20, B deferred 2026-10-01 then refreshed 2026-10-25 -> order [A, B]
def test_deferred_items_without_deferral_sort_first() -> None:  # note == "", first in list, no exception
def test_deferred_items_missing_queue_epic_raises() -> None:
def test_vrg_queue_deferred_text_and_json() -> None:
def test_vrg_queue_bare_lists_all_three_and_tolerates_missing() -> None:
```

- [ ] **Step 2: Run the tests to verify they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_queue_view.py tests/vergil_tooling/test_vrg_queue.py -q`. Expected: FAIL.

- [ ] **Step 3: Implement** per Interfaces. Update the module docstrings to describe three queues.

- [ ] **Step 4: Run the tests, then `vrg-container-run -- vrg-validate`.** Expected: PASS.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope queue --message "add the vrg-queue deferred view"`

---

### Task 4: Release and install vergil-tooling (vergil-tooling, deployment)

`--kind deployment`, blocked by Tasks 1–3.

- **Precondition (human-attested):** a vergil-tooling release containing Tasks 1–3 is published under `v2.1`.
- **Procedure:** reinstall the host tool per the host-level-tool spec. Then confirm that `vrg-intake-route --help` lists `intake` and that `vrg-intake-route <ref> defer --help` shows `--revisit-when` and `--date`. Confirm that `vrg-queue deferred --repo vergil-project/.github` either exits 0 or prints the "no open 'Epic (deferred)…'" message.
- **Accepted window (spec §4):** from now until Task 8, intake review is not run.

---

### Task 5: Update `intake-review` and `epic-create` (vergil-claude-plugin, implementation)

**Files:**

- Modify: `skills/intake-review/SKILL.md`:
  - The route table row for **defer**: it moves the item to the deferred queue, and both `--reason` and `--revisit-when` are required.
  - The command block at line ~91.
  - The "keeps its age" lines (~18, ~138). Remove them; "leave it in intake for next time" is now the no-route outcome.
  - The tallies.
- Modify: `skills/epic-create/SKILL.md`:
  - The entry-point table: add a row for an issue in the **deferred** queue. Pull it into planning first with `vrg-intake-route <ref> planning`.
  - Line ~207: the "do not invent states" note says to defer with `vrg-intake-route … defer`, not with a progress-comment state.
  - The parent-label check that identifies queues: `epic` + `deferred` is the deferred epic.
- Modify: `skills/intake-capture/SKILL.md:18` (the route list) and `docs/site/docs/skills/index.md` (the intake-review row and section).
- Modify: `CLAUDE.md:228-229`: the three-queue sentence.

**Verification:** `vrg-container-run -- vrg-validate` passes. `grep -rn "keeps its age\|leaves the item in intake" skills docs` returns nothing.

---

### Task 6: New `deferred-review` skill (vergil-claude-plugin, implementation)

**Files:** Create `skills/deferred-review/SKILL.md`. Modify `docs/site/docs/skills/index.md` to add a table row and a section.

**The skill must specify:**

- **Frontmatter `description`**, with these triggers: "review deferred", "what's deferred", "revisit the backlog", "process the deferred queue".
- **Preflight:** USER agent (`vrg-whoami --mode`).
- **Collect:** `vrg-queue deferred [--repo <owner>/<private-repo>]`. A missing queue epic means an empty queue, not a failure; stop.
- **Per item,** in batches of 4, with one confirmation per batch:
  1. Read the issue and its latest deferral.
  2. **Actually check the conditions**: release pages, issue states, dates, upstream status.
  3. Report what was checked and what was found.
  4. Recommend exactly one outcome:
     - **keep:** no command.
     - **refresh:** `vrg-intake-route <ref> defer --reason … --revisit-when …`.
     - **revive:** `intake`, `planning`, or `one-off --repo <repo>`. Use `Deferred from:` to choose: an ad-hoc origin usually returns via one-off.
     - **drop:** `drop --reason …`.
- **Never fabricate a check:** if a condition can't be verified, say so and recommend keep.
- **Finish with tallies:** kept, refreshed, revived to each destination, and dropped.

**Verification:** `vrg-container-run -- vrg-validate` passes, and the skill appears in the index.

---

### Task 7: New `planning-review` skill (vergil-claude-plugin, implementation)

**Files:** Create `skills/planning-review/SKILL.md`. Modify `docs/site/docs/skills/index.md`.

**The skill must specify:**

- **Triggers:** "review planning", "what should we plan next", "prioritize the planning queue".
- **Collect:** `vrg-queue planning [--repo …]`.
- **Purpose:** decide **what to plan next** with the human. Rank the queue by priority and importance, and pick one or more items to hand to `epic-create <ref>`.
- **Doctrine, stated as the skill's core rule:** plan an epic only when you intend to implement it soon. A plan left sitting goes stale and needs a conceptual rebase, so the skill discourages picking more than the near-term capacity allows.
- **Per-item outcomes:**
  - **pick:** hand the item to `epic-create`.
  - **defer:** `vrg-intake-route <ref> defer --reason … --revisit-when …`. The tool refuses this when an `Epic:` marker exists.
  - **drop:** `vrg-intake-route <ref> drop --reason …`.
  - **leave:** the human's discretion, for items expected to be ready within days or a week.
- **Surface stalled `in-progress` items:** give each its latest progress note, and offer to resume it with `epic-create` or to defer it.
- **Starts simple:** no scoring formula. The ranking is a conversation.

**Verification:** `vrg-container-run -- vrg-validate` passes, and the skill appears in the index.

---

### Task 8: Release and install vergil-claude-plugin (vergil-claude-plugin, deployment)

`--kind deployment`, blocked by Tasks 4–7.

- **Precondition (human-attested):** a plugin release containing Tasks 5–7 is published.
- **Procedure:** update the installed plugin. Confirm that the skill list includes `vergil:deferred-review` and `vergil:planning-review`, and that `intake-review`'s defer row requires `--revisit-when`. This ends the accepted window.

---

### Task 9: Migration sweep (vergil-project/.github, deployment)

`--kind deployment`, blocked by Task 4. It does not need Task 8, because it calls
`vrg-intake-route` directly.

- **Precondition (machine):** `vrg-intake-route <ref> defer --help` shows `--revisit-when` and `--date`.
- **Procedure:**
  1. **List the queue homes:** `~/dev/projects/<org>/.github` for each org, plus `vrg-queue intake|planning --repo <org>/<private-repo>` for each private repo. "No queue" means empty.
  2. **Read every intake and planning item** (spec §3.9). A `Deferred <date>:` comment is a signal, not the rule.
  3. **Propose an outcome for each candidate,** in batches of 4, with one human confirmation per batch.
     - **defer:** `vrg-intake-route <ref> defer --reason <from the existing comment> --revisit-when <drafted where missing> --date <original deferral date>`.
     - **flag:** items whose "deferral" is really something else, such as becoming a task of #384 (vergil-tooling#1593, #1594), go to the human for a decision.
  4. **Record one results comment:** per home, the moved items, the flagged items with their decisions, and the before and after counts of `vrg-queue intake` and `vrg-queue planning`.
- **Acceptance:** every intake and planning item in every queue home was reviewed, and each one left is live work. `Outcome: SUCCESS`.

---

### Bookends (already filed)

- Documentation: vergil-project/.github#513, which this plan's PR closes.
- Docs review: vergil-project/vergil-tooling#3669, blocked by Tasks 1–9. It sweeps the site docs, and the `docs` repo and plugin docs as spawned tasks.
- Retrospective: vergil-project/.github#515 (terminal).

## Self-Review

- **Spec coverage:**
  - §3.1–3.2 → Task 2 (route table, guards).
  - §3.3 → Task 1.
  - §3.4–3.5 → Task 2.
  - §3.6 → Task 3.
  - §3.7 → Task 1. The PR refusal flows from `queue_kind`, and the audit treatment is in `epic_audit.py:113`.
  - §3.8 → Tasks 5–7.
  - §3.9 → Task 9.
  - §4 → task order and the deployment tasks.
- **Types:** `Defer`, `ToIntake`, `ROUTES`, `Deferral`, `deferred_items`, `container_kind`, `QUEUE_KINDS` and `QUEUE_KIND_BY_KEY` are named the same in every task that uses them.
- **Proportion:** the tooling tasks give interfaces and test names; the bodies are left to the implementer.
