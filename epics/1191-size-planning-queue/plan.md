# Size the planning queue — implementation plan

- **Epic:** vergil-project/.github#1191
- **Planning issue:** vergil-project/.github#1185 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`; operational tasks are worked
> with `issue-deploy`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** every planning and deferred item can carry an `epic-size:*` label and
a dated sizing comment written by one command, and `vrg-queue planning` sorts
and filters by them, so the small end of the planning queue can be worked on a
regular cadence.

**Architecture:** a new pure module, `lib/epic_size.py`, owns the scale, the
comment format and the size-record derivation. `vrg-intake-route` gains a
`size` route and sizing on the planning route, and its one-off clears a size.
`vrg-queue` reads the size record per item, sorts, filters and exits 2 when an
item's sizing is inconsistent. The `epic-review` and `epic-create` skills
learn the new commands and exit status. A backfill applies the 2026-11-16
estimates once the tooling is deployed.

**Tech Stack:** Python 3 (vergil-tooling), pytest, `gh` through
`vergil_tooling.lib.github`, Markdown skills (vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- The task count is the estimate; the size is derived from it: S 1–9, M 10–19, L 20–39, XL 40+ (spec §6.1). No command takes a size as input.
- Label names are exactly `epic-size:S`, `epic-size:M`, `epic-size:L`, `epic-size:XL` (spec §6.1, §6.2).
- The sizing comment's first line is `Sized YYYY-MM-DD: epic-size <X>, about <N> tasks.`, followed by the `Epic-size:`, `Estimated-tasks:`, `Confidence:` and `Reason:` lines in that order (spec §6.3).
- The cleared marker's first line is `Size cleared YYYY-MM-DD: moved out of planning as a one-off.` (spec §6.7).
- Only comments by trusted authors (`load_trusted_authors()`, `app/` stripped) count; trusted authors not configured or invalid fail loudly (spec §6.3, §7).
- An inconsistent size is never resolved to a size; it is listed as `!`, named on stderr, and `vrg-queue` exits 2. Exit 1 keeps its meaning and wins over 2 (spec §6.4, §6.8).
- Fail loudly: no swallowed exceptions, no fallback that hides an error (repo engineering principles).
- Validation runs only through `vrg-container-run -- vrg-validate` in the task repo's worktree.
- Commit with `vrg-commit`; hand off with `vrg-pr-workflow report-ready`. Releases are human-gated; no agent cuts one.

## Review Focus

1. **The identity that posts a sizing comment must be trusted.** A sizing comment posted under the item owner's App token whose author `load_trusted_authors()` does not list would leave every routed item "label without comment". Pinned in Task 3 (a test that the comment is posted through the same `github.run` path the route's other comments use) and checked live by Task 8's end check.
2. **A sizing comment edited or forged by an untrusted author after a trusted one.** The trusted one still wins; the untrusted comment neither changes the size nor raises an error. Pinned in Task 2.
3. **A deferred item sized, deferred again (a refresh) and revived to planning.** The size survives every step unchanged. Pinned in Task 5 (the deferred view reads the same record) and Task 3 (planning route without `--tasks` leaves an existing size alone).
4. **A `--size S` filter over a queue with an item in error.** The item is listed, flagged `!`, and the run exits 2. Pinned in Task 5.
5. **A bare `vrg-queue --json` with one planning item in error and the deferred queue unreadable.** It exits 1 and prints nothing on stdout, as today. Pinned in Task 5.

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1192) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | `epic_size` module and the four labels | — |
| 3 | vergil-tooling | task | `vrg-intake-route size`, and sizing on the planning route | 2 |
| 4 | vergil-tooling | task | A one-off clears the size | 3 |
| 5 | vergil-tooling | task | `vrg-queue` size column, sort, filter and exit 2 | 2 |
| 6 | vergil-claude-plugin | task | `epic-review` and `epic-create` skills size the queue | 3, 5 |
| 7 | .github | deployment | Tooling and plugin releases attested and installed | 2, 3, 4, 5, 6 |
| 8 | .github | deployment | Backfill the 2026-11-16 estimates | 7 |
| — | vergil-tooling | bookend (#4477) | Documentation review | 2, 3, 4, 5, 6 |
| — | .github | bookend (#1193) | Retrospective | all |

Task 4 follows Task 3 because both change `lib/intake_route.py`. No fixture
run is seeded: the epic changes no language support, container image, shared
CI workflow or release tooling.

---

### Task 2: `epic_size` module and the four labels (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/epic_size.py`
- Modify: `src/vergil_tooling/data/labels.json` (four entries)
- Test: `tests/vergil_tooling/test_epic_size.py`, `tests/vergil_tooling/test_data_labels.py`

**Interfaces:**

- Produces, all in `vergil_tooling.lib.epic_size`:
  - `Size = Literal["S", "M", "L", "XL"]`; `SIZES: tuple[Size, ...] = ("S", "M", "L", "XL")`.
  - `Confidence = Literal["high", "med", "low"]`; `CONFIDENCES: tuple[Confidence, ...]`.
  - `LABEL_PREFIX = "epic-size:"`; `label_for(size: Size) -> str`.
  - `size_for_tasks(tasks: int) -> Size`; raises `ValueError` for `tasks < 1`.
  - `@dataclass(frozen=True) class Sizing: tasks: int; confidence: Confidence; reason: str` with property `size -> Size`. `Sizing.__post_init__` raises `ValueError` when `tasks < 1`, `confidence` is not in `CONFIDENCES`, or `reason` is empty after `strip()` or contains a newline.
  - `render_sizing_comment(sizing: Sizing, day: date) -> str` and `render_cleared_marker(day: date) -> str`, exact text per Global Constraints.
  - `@dataclass(frozen=True) class SizeRecord: size: Size | None; tasks: int | None; confidence: Confidence | None; error: str | None` with property `display -> str` (`"M/14"`, `"?"` unsized, `"!"` error).
  - `trusted_logins() -> frozenset[str]`: `human + agents` from `trusted_authors.load_trusted_authors()`, letting its errors propagate.
  - `size_record(labels: Iterable[str], comments: Sequence[tuple[str, str]], trusted: frozenset[str]) -> SizeRecord`: `comments` are `(author_login, body)` oldest first; authors compared after stripping a leading `app/`. Implements spec §6.4's table, counting only sizing comments after the latest trusted cleared marker.
  - `has_size(labels: Iterable[str], comments: Sequence[tuple[str, str]], trusted: frozenset[str]) -> bool`: an `epic-size:*` label is present, or a trusted sizing comment follows the latest cleared marker.

- [ ] **Step 1: Write the failing tests** in `test_epic_size.py`:
  - `test_bands`: `size_for_tasks` gives S for 1 and 9, M for 10 and 19, L for 20 and 39, XL for 40 and 400; 0 and -1 raise `ValueError`.
  - `test_render_sizing_comment`: `Sizing(14, "med", "One sentence.")` on 2026-11-16 renders exactly `Sized 2026-11-16: epic-size M, about 14 tasks.\nEpic-size: M\nEstimated-tasks: 14\nConfidence: med\nReason: One sentence.`.
  - `test_sizing_rejects`: empty reason, a two-line reason, confidence `"medium"`, tasks 0 each raise `ValueError`.
  - `test_record_unsized`, `test_record_sized` (label `epic-size:M` + that comment by `app/vergil-user` with `vergil-user` trusted → `SizeRecord("M", 14, "med", None)`, display `M/14`).
  - One test per error row of spec §6.4: `test_record_comment_without_label`, `test_record_label_without_comment`, `test_record_disagreement`, `test_record_several_labels`, `test_record_malformed` (missing `Reason:`; `Estimated-tasks: fourteen`; `Confidence: medium`; `Epic-size: S` with `Estimated-tasks: 14`). Each has `size is None`, a non-empty `error`, display `!`.
  - `test_latest_malformed_is_not_skipped`: a well-formed comment followed by a malformed one is an error.
  - `test_untrusted_sizing_ignored`: a trusted M/14 comment followed by an untrusted XL/50 comment, label `epic-size:M` → M/14, no error.
  - `test_cleared_marker_resets`: label-less item with a trusted M/14 comment then a trusted cleared marker → unsized; a later trusted S/5 comment plus label `epic-size:S` → S/5. An untrusted cleared marker is ignored.
  - `test_has_size`: true for a label alone, true for a sizing comment alone, false after a cleared marker with no label.
- [ ] **Step 2: Write the failing test** `test_epic_size_labels_registered` in `test_data_labels.py`: the registry holds the four `epic-size:*` names, each with a description that names its band.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** the module and the four `labels.json` entries (one colour family; descriptions such as `Planning estimate: epic of 10–19 tasks`).
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope epic-size --message "add the epic-size scale, sizing comment and size record"`, then report-ready.

### Task 3: `vrg-intake-route size`, and sizing on the planning route (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (new `SizeItem` action, `ROUTES`, `_ROUTE_NAMES`, `ToPlanning`, new `_apply_sizing`, module docstring)
- Modify: `src/vergil_tooling/bin/vrg_intake_route.py` (the `size` subparser, `--tasks/--confidence/--reason` on `planning`, the success line, module docstring)
- Test: `tests/vergil_tooling/test_intake_route.py`, `tests/vergil_tooling/test_vrg_intake_route.py`

**Interfaces:**

- Consumes: Task 2's `Sizing`, `render_sizing_comment`, `label_for`, `LABEL_PREFIX`.
- Produces: `@dataclass(frozen=True) class SizeItem: sizing: Sizing` (named to avoid clashing with `epic_size.Size`); `ROUTES["planning"]` and `ROUTES["deferred"]` include it, no other container does; `_ROUTE_NAMES[SizeItem] = "size"`.
- Produces: `ToPlanning(sizing: Sizing | None = None)`.
- Produces: `_apply_sizing(ref: IssueRef, sizing: Sizing, day: date) -> None`: posts the comment, then reads the item's labels and runs one `issue edit` that adds `label_for(sizing.size)` and removes every other `epic-size:*` label present, after `ensure_labels(repo, [label])`. A failed label step raises `RouteHalfAppliedError` naming the item, "sizing comment posted, label not set", and the exact re-run command (`vrg-intake-route <slug> size --tasks N --confidence C --reason <shell-quoted>`).
- CLI: `vrg-intake-route <ref> size --tasks N [--confidence high|med|low] --reason TEXT` (`--confidence` default `med`); `planning [--tasks N [--confidence …] --reason TEXT]`. Success for `size` prints `Sized <slug>: epic-size X, about N tasks.`; `planning` with sizing prints the usual `Routed …` line followed by the `Sized …` line.

- [ ] **Step 1: Write the failing tests** in `test_intake_route.py`, with the module's existing fake `github.run`/`read_json` fixtures:
  - `test_size_planning_item`: posts exactly the rendered comment, then one edit adding `epic-size:M` and removing `epic-size:S` (present before); comment before edit in the recorded call order.
  - `test_size_deferred_item`: allowed.
  - `test_size_refused_outside_planning_and_deferred`: intake, ad-hoc, development-epic and unparented items raise `RouteRefusedError` naming `size`; no call made.
  - `test_size_label_failure_is_half_applied`: the edit fails → `RouteHalfAppliedError` whose message contains `size --tasks 14 --confidence med --reason`.
  - `test_size_comment_uses_route_comment_path`: the comment goes through `_comment` (the same path as the drop and defer comments), so it is posted under the item owner's token.
  - `test_planning_with_sizing`: an intake item is linked, then sized; with an invalid `Sizing` nothing is called (validation precedes the link, since `Sizing` raises on construction).
  - `test_planning_sizing_failure_after_link`: the comment fails after the link → `RouteHalfAppliedError` saying the item is now in planning and naming the `size` command.
  - `test_planning_without_sizing_leaves_size`: a deferred item with `epic-size:M` routed to planning without sizing makes no label edit and no comment.
- [ ] **Step 2: Write the failing CLI tests** in `test_vrg_intake_route.py`: `test_size_parses` (`size --tasks 14 --reason x` → `SizeItem(Sizing(14, "med", "x"))`); `test_size_requires_tasks_and_reason`; `test_size_rejects_non_integer_tasks`; `test_planning_tasks_without_reason_refused` and `test_planning_confidence_without_tasks_refused` (exit 1, nothing routed); `test_size_success_line`.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** the action, the routing branch in `_route` (sizing a planning or deferred item reparents nothing), `_apply_sizing`, the CLI, and both module docstrings (they are the commands' reference text).
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope intake-route --message "add the size route and sizing on the way into planning"`, then report-ready.

### Task 4: A one-off clears the size (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (`_one_off`, `_recreate`'s `dropped` set, new `_clear_size`, module docstring)
- Test: `tests/vergil_tooling/test_intake_route.py`

**Interfaces:**

- Consumes: Task 2's `has_size`, `trusted_logins`, `render_cleared_marker`, `LABEL_PREFIX`; Task 3's routing structure.
- Produces: `_clear_size(final: IssueRef, day: date) -> None`: reads `final`'s labels, removes every `epic-size:*` label in one edit when any is present, then posts the cleared marker. Failure raises `RuntimeError` with the by-hand fix, which `_one_off` wraps in `RouteHalfAppliedError` the way it wraps `_remove_in_progress`.
- `_one_off` computes `sized = has_size(...)` from the item's labels and authored comments before the move, for items whose container is planning or deferred, calling `trusted_logins()` first so a missing configuration refuses before any change. After the move settles, it calls `_clear_size(result.final, today)` when `sized` and `result.moved_by != "recreate"`.
- `_recreate` drops every label starting with `LABEL_PREFIX`, beside `RETIRED_LABELS` and `in-progress`.

- [ ] **Step 1: Write the failing tests:**
  - `test_one_off_transfer_clears_size`: a sized planning item transferred → final issue gets one edit removing `epic-size:M`, then the cleared marker comment, after the move's settle.
  - `test_one_off_same_repo_clears_size`: the reparent-only path does the same.
  - `test_one_off_recreate_drops_size_label`: the recreated issue is created without `epic-size:M`; no cleared marker posted.
  - `test_one_off_unsized_item_untouched`: no edit, no marker.
  - `test_one_off_comment_only_size_gets_marker`: a transfer whose label was lost (comment only) → no label edit, marker posted.
  - `test_one_off_clear_failure_is_half_applied`: the marker fails → `RouteHalfAppliedError` naming the moved item.
  - `test_one_off_trusted_authors_missing_refuses_first`: `load_trusted_authors` raises → nothing moved.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 3: Implement** the changes above and extend the module docstring's one-off paragraph.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type feat --scope intake-route --message "clear an item's size when it leaves as a one-off"`, then report-ready.

### Task 5: `vrg-queue` size column, sort, filter and exit 2 (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py` (`QueueItem`, `planning_items`, `deferred_items`, `render_text`, `to_json`, new `sort_planning`, `filter_planning`)
- Modify: `src/vergil_tooling/bin/vrg_queue.py` (`--sort`, `--size`, exit status, module docstring)
- Test: `tests/vergil_tooling/test_queue_view.py`, `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**

- Consumes: Task 2's `size_record`, `trusted_logins`, `SizeRecord`, `SIZES`.
- Produces: `QueueItem.size: SizeRecord | None = None` (last field, defaulted); planning and deferred items always set it, intake items never. Both views read `comments` with authors (`comments` already in the `--json` field list; each entry's `author.login`), calling `trusted_logins()` once per view.
- Produces: `render_text` inserts `<size.display>` after the age when `size` is set; an item in error shows its `size.error` as the note. `to_json` flattens a set `size` into `epic_size`, `estimated_tasks`, `confidence`, `size_error` keys and omits them when `size` is `None`, so intake JSON is unchanged.
- Produces: `sort_planning(items: list[QueueItem], key: Literal["age", "size", "tasks"]) -> list[QueueItem]` per spec §6.8 (groups: sized in order, then unsized, then error; ties oldest first, then slug).
- Produces: `filter_planning(items: list[QueueItem], sizes: frozenset[str]) -> list[QueueItem]`: keeps items whose size is in `sizes`, unsized items when `"unsized"` is in `sizes`, and every item in error.
- CLI: `--sort age|size|tasks` (default `age`) and `--size <comma list>` are accepted only with `planning`; with any other view, or bare, they exit 1 naming the flag. An unknown `--size` token exits 1. With a filter, the header reads `planning queue for <name> in <home>: <N> open item(s), <M> shown`. After printing, each item in error is named on stderr as `vrg-queue: <slug>: <error>`, and the run exits 2; a failed queue still exits 1 and wins.

- [ ] **Step 1: Write the failing view tests** in `test_queue_view.py`: `test_planning_item_carries_size` (record from labels and authored comments); `test_deferred_item_carries_size`; `test_intake_item_has_no_size`; `test_render_text_size_column` (`M/14`, `?`, `!` with the error as note); `test_to_json_size_fields` (the four keys, and none for intake); `test_sort_size` and `test_sort_tasks` over S/3 old, S/8 new, M/12, unsized, error; `test_filter_keeps_errors` (`{"S"}` keeps both S items and the error); `test_filter_unsized`.
- [ ] **Step 2: Write the failing CLI tests** in `test_vrg_queue.py`: `test_exit_2_on_size_error` (text and `--json`: full output on stdout, stderr names the item, exit 2); `test_exit_0_when_clean`; `test_bare_exit_2`; `test_bare_json_failure_beats_size_error` (exit 1, empty stdout); `test_sort_and_size_only_with_planning` (deferred, intake, adhoc and bare each exit 1); `test_unknown_size_token`; `test_filtered_header`; `test_trusted_authors_missing_fails` (exit 1 with the loader's message on stderr, never a traceback: add `TrustedAuthorsNotConfiguredError` to `_VIEW_ERRORS`).
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new tests to fail.
- [ ] **Step 4: Implement** the view and CLI changes and update the module docstring (exit statuses, the size column, `--sort`, `--size`).
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type feat --scope queue --message "show, sort and filter the planning queue by epic size"`, then report-ready.

### Task 6: `epic-review` and `epic-create` skills size the queue (vergil-claude-plugin)

**Files:**

- Modify: `skills/epic-review/SKILL.md` (step 2 "Collect the queue", intake mode's planning route, planning mode, deferred mode's revive, the notes)
- Modify: `skills/epic-create/SKILL.md` (the no-argument entry-point bullets after the entry-point table, "Choosing what to plan next")

**Interfaces:**

- Consumes: Task 3's `vrg-intake-route <ref> size --tasks N [--confidence …] --reason …` and `planning --tasks … --reason …`; Task 5's `vrg-queue planning --sort size --size S`, the `M/14`/`?`/`!` column, the JSON keys and exit 2.

- [ ] **Step 1: `epic-review` step 2.** Document the size column and JSON keys, and add: exit 2 means the listing is complete with items in error; show it, flag the `!` items, and continue. Exit 1 other than the missing-queue message stays a real failure.
- [ ] **Step 2: Intake mode.** The planning route always carries `--tasks` and `--reason` (and `--confidence` when not `med`), recommended in the item's paragraph and confirmed with the batch.
- [ ] **Step 3: Planning mode.** Each waiting item that is unsized or `!` gets a recommended task count (with the size it derives), confidence and one-line reason in its paragraph; a sized item gets a re-size only when the review shows the estimate is off. Confirmed sizes run `vrg-intake-route <ref> size …` after the batch's routes, skipped for items deferred, dropped or moved out. Sizing is not an outcome and not a tally. The end-of-pass list may be shown with `--sort size` on request; oldest first stays the default.
- [ ] **Step 4: Deferred mode.** A revive to planning of an unsized item carries `--tasks` and `--reason`.
- [ ] **Step 5: Notes.** "Do not invent fields, labels or priority scores" gains the exception: the `epic-size:*` family, written only through `vrg-intake-route`. Security items' sizing reasons stay neutral.
- [ ] **Step 6: `epic-create`.** In the no-argument bullets, exit 2 from `vrg-queue planning` is a listing to act on, not a failure. In "Choosing what to plan next", one line: `vrg-queue planning --sort size --size S` lists the small items oldest first, for the small-end cadence.
- [ ] **Step 7: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 8: Commit** `vrg-commit --type feat --scope skills --message "size planning items in epic-review and read vrg-queue exit 2"`, then report-ready.

### Task 7: Tooling and plugin releases attested and installed (deployment, .github)

Blocked by 2, 3, 4, 5, 6.

- **Precondition (human-attested):** a vergil-tooling release containing Tasks 2–5 and a vergil-claude-plugin release containing Task 6 are published, their versions named in a comment.
- **Human-run steps:** on the host, `vrg-host upgrade` and `vrg-host plugins`; for the VMs, `vrg-vm update --all`.
- **Procedure (agent-safe), in the agent's VM:**
  1. Confirm each release's notes list its tasks' PRs.
  2. `vrg-intake-route --help` lists `size`; `vrg-intake-route '#1' size --help` shows `--tasks`, `--confidence`, `--reason`.
  3. `vrg-queue planning --help` lists `--sort` and `--size`; `vrg-queue planning` prints the size column and exits 0 or 2 (2 is expected before the backfill only if an item is already in error).
  4. The installed plugin's `skills/epic-review/SKILL.md` contains `epic-size`.
- **SUCCESS:** every check passes, recorded with both release versions.

### Task 8: Backfill the 2026-11-16 estimates (deployment, .github)

Blocked by 7.

- **Precondition (machine probe):** Task 7 is closed with `Outcome: SUCCESS`, and `vrg-intake-route --help` lists `size`.
- **Procedure (agent-safe):**
  1. Read the `## Backfill data: sizing pass of 2026-11-16` comment on vergil-project/.github#1185, only if its author is a trusted author (`vrg-trusted-authors`); otherwise stop with "blocked: preconditions not met". Parse its `tsv` block: ref, size, tasks, confidence, reason.
  2. Skip the line for vergil-project/.github#1185 itself.
  3. For each other line: check that the size the task count derives (spec §6.1's bands) equals the size column, or record a mismatch and do not apply it; check that the item is open and its parent is the planning or deferred epic, or record it as skipped with its state; otherwise run `vrg-intake-route <ref> size --tasks <tasks> --confidence <confidence> --reason "<reason>"`. Run the lines one at a time; a failure is recorded and the run continues to the next line.
  4. Run `vrg-queue planning --sort size` and `vrg-queue deferred`.
- **SUCCESS:** every line has a recorded outcome (sized, skipped with reason, or mismatch); no line failed; `vrg-queue planning --sort size` and `vrg-queue deferred` both exit 0 and list no `!` item; the items still `?` are listed for the next `epic-review planning` pass.
- **FAILURE:** any line failed or any `!` item remains; the comment names each, and the task stays open.
