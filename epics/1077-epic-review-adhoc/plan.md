# epic-review and epic-implement adhoc mode — implementation plan

- **Epic:** vergil-project/.github#1077
- **Planning issue:** vergil-project/.github#1076 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task below is one issue, one worktree and one
> PR in the repo it names, driven by `epic-implement` / `issue-implement`
> (operational tasks by `issue-deploy` / `issue-validate`). Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `intake-review`, `planning-review` and `deferred-review`
with one `epic-review` skill (planning review made prune-only), and give
`epic-implement` an `adhoc` mode that works the org's ad-hoc tasks, backed by
a new `vrg-queue adhoc` view with in-flight detection.

**Architecture:** Two delivery groups that touch different code. Group A is
the review consolidation: a `vrg-intake-route` route change and message,
template and docs revisions in vergil-tooling, then the new skill and the
retirement of the old three in vergil-claude-plugin, then the `docs` repo's
management pages. Group B is the ad-hoc driver: a `vrg-queue adhoc` view
and its in-flight detection in vergil-tooling, then the `adhoc` mode in
`epic-implement`. Each group ends with a deployment and a live validation.

**Tech Stack:** Python 3 (vergil-tooling: `lib/github.py`, `lib/git.py`,
`lib/epics.py`, pytest with the existing fakes), Markdown skills and site
docs (vergil-claude-plugin, docs, vergil-tooling).

**Spec:** [`spec.md`](spec.md) in this directory.

## Global Constraints

- Validation is `vrg-container-run -- vrg-validate` only, in every repo.
  "Run the tests" below always means this command.
- Every GitHub call goes through `github.read_json` / `github.graphql`, and
  every git call through `git.read_output`. No new retry logic.
- A read that fails or is truncated is an error that stops the command with
  its cause, never a short list or a task wrongly shown as ready (spec
  §5.4). Truncation uses `epics.refuse_truncated` with
  `epics.LISTING_LIMIT`.
- Reviews only put issues in the right state and place; they never process
  them. Planning review never starts `epic-create` (spec §1, §5.2).
- `epic-review` never offers `one-off` for an `in-progress` planning item
  (spec §5.2, §5.6).
- Private work stays private: a self-homed private repo's queues and ad-hoc
  epic are reviewed and worked only with its own `--repo` (spec §6).
- Historical records keep the old skill names: `releases/`, `CHANGELOG.md`,
  and every published `epics/*/spec.md`, `plan.md` and `retrospective.md`.
- The tooling and plugin releases of each group ship back to back. The
  window in which the released tooling names `epic-review` before it is
  installed is accepted (spec §6).

## Review Focus

1. **An archive labelled `epic` + `ad-hoc`** (title `Archive (ad hoc): …`
   or the legacy `Epic (ad hoc): <repo> — <quarter>` form) is never read as
   a live ad-hoc epic (Task 5,
   `test_adhoc_items_skips_archives_and_legacy_titles`).
2. **A task in a batch** shows as in flight from its local batch marker,
   even though no `feature/<N>-` branch exists for it (Task 6,
   `test_in_flight_batch_marker_counts`).
3. **A repo with ad-hoc tasks but no local clone** under the repo roots is
   shown with `local: no clone`, never silently treated as checked (Task 6,
   `test_in_flight_repo_without_clone_is_noted`).
4. **`--repo` on a public repo with no ad-hoc epic** gives an empty list
   and exit 0, not an error (Task 5,
   `test_adhoc_repo_without_epic_is_empty`).
5. **A planning item whose development epic is closed but which still
   carries `in-progress`** can be routed `one-off` and loses the label
   (Task 1, `test_route_one_off_from_planning_removes_in_progress`).

## Delivery

| Task | Repo | Group | Blocked by |
|---|---|---|---|
| 1. `one-off` from planning | vergil-tooling | A | — |
| 2. Skill names in tooling text and docs | vergil-tooling | A | — |
| 3. The `epic-review` skill | vergil-claude-plugin | A | — |
| 4. `docs` repo management pages | docs | A | — |
| D1. Deploy group A | vergil-claude-plugin | A | 1, 2, 3, 4 |
| V1. Validate `epic-review` live | vergil-claude-plugin | A | D1 |
| 5. `vrg-queue adhoc` | vergil-tooling | B | — |
| 6. In-flight detection | vergil-tooling | B | 5 |
| 7. The `adhoc` mode in `epic-implement` | vergil-claude-plugin | B | 3, 6 |
| D2. Deploy group B | vergil-claude-plugin | B | 5, 6, 7 |
| V2. Validate `epic-implement adhoc` live | vergil-claude-plugin | B | D2 |

Group A and group B run in parallel. Only Task 7 waits on group A: it edits
`epic-review`'s ad-hoc refusal (which Task 3 creates) to point at the new
`adhoc` mode, so the refusal pointing to the later part lands with the later
part (spec §6). The bookends are already filed: the docs task
vergil-project/.github#1078 (this spec and plan), the documentation review
vergil-project/vergil-claude-plugin#1041 (after every task above), and the
retrospective vergil-project/.github#1079 (last).

Releases are human-gated preconditions of D1 and D2, never performed by an
agent.

---

## Group A — the review consolidation

### Task 1: `one-off` from planning

**Files:**
- Modify: `src/vergil_tooling/lib/intake_route.py` (`ROUTES`, the one-off
  path)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (the
  `vrg-intake-route` section)
- Test: `tests/vergil_tooling/test_intake_route.py`

**Interfaces:**
- Produces: `ROUTES["planning"] == frozenset({Defer, Drop, OneOff})`.
  `vrg-intake-route <planning item> one-off --repo … [--epic …]` behaves as
  from intake, behind `_guard_open_planning_epics(ref, view, "routing
  one-off")`.

- [ ] **Step 1: Write the failing tests**

  - `test_route_one_off_from_planning_moves_under_adhoc`: a planning item
    with no development epic, routed `one-off --repo <org>/<repo>`, ends
    linked under that repo's ad-hoc epic and no longer under the planning
    epic.
  - `test_route_one_off_planning_refuses_open_marker_epic`: with an open
    epic recorded by an `Epic:` marker, the route raises the same refusal
    type as `test_route_drop_planning_refuses_open_marker_epic`, and the
    message contains `routing one-off`.
  - `test_route_one_off_planning_refuses_open_epic_found_by_planning_line`:
    as above, for an epic found by its `Planning:` body line.
  - `test_route_one_off_from_planning_removes_in_progress`: an item
    labelled `in-progress` whose epics are all closed is routed and loses
    the label (Review Focus 5).
  - `test_route_one_off_planning_refuses_a_truncated_search`: mirrors
    `test_route_defer_planning_refuses_a_truncated_search`.

- [ ] **Step 2: Run the tests; confirm the five fail**

  Run: `vrg-container-run -- vrg-validate`
  Expected: FAIL, the planning container refusing `one-off`.

- [ ] **Step 3: Allow the route**

  Add `OneOff` to `ROUTES["planning"]`. In the one-off path, when the
  parent's kind is planning, call `_guard_open_planning_epics(ref, view,
  "routing one-off")` before any move, and remove `in-progress` the way the
  defer path does (reuse its helper; do not copy it).

- [ ] **Step 4: Update the route table and the refusal text in
  `cli-tools-overview.md`**

  The planning row of the `vrg-intake-route` table lists `defer`, `drop`,
  `one-off`. The sentence describing the planning refusal says the route is
  refused while any development epic of the item is open (the rule since
  vergil-project/vergil-tooling#4170), replacing the `Epic:`-marker wording.

- [ ] **Step 5: Run the tests; confirm they pass**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS.

- [ ] **Step 6: Commit**

  `vrg-commit --type feat --scope intake-route --message "allow one-off from the planning queue"`

### Task 2: Skill names in tooling text and docs

**Files:**
- Modify: `src/vergil_tooling/lib/epics.py` (`_QUEUE_REFUSAL_TAILS`, the
  `INTAKE` and `DEFERRED` `StandingKind.body` templates)
- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_QUEUE_FULL_HINT`)
- Modify: `src/vergil_tooling/bin/vrg_intake_create.py` (docstring, epilog)
- Modify: `docs/site/docs/standards/github-issues.md`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (the `vrg-queue`
  section's "three review skills" sentence)
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_epic_audit.py` (and
  `tests/vergil_tooling/test_vrg_epic_audit.py`, which pins the old hints)

**Interfaces:**
- Produces: the user-facing strings below, exactly.

- [ ] **Step 1: Write the failing tests**

  - `test_queue_epic_bodies_name_no_skill`: for each kind in
    `epics.QUEUE_KINDS`, `kind.body` contains none of `intake-review`,
    `planning-review`, `deferred-review`, `epic-review`.
  - `test_queue_refusal_tails_name_epic_review`:
    `_QUEUE_REFUSAL_TAILS["intake"]` ends `Route it with epic-review intake
    first.` and `["deferred"]` ends `Revive it with epic-review deferred
    first.`; `["planning"]` is unchanged.
  - `test_queue_full_hints_name_epic_review`: `_QUEUE_FULL_HINT` is
    `{"intake": "Run epic-review intake.", "planning": "Run epic-review
    planning.", "deferred": "Run epic-review deferred."}`.

- [ ] **Step 2: Run the tests; confirm they fail**

  Run: `vrg-container-run -- vrg-validate`
  Expected: FAIL on the three tests.

- [ ] **Step 3: Change the strings**

  The intake body becomes `Perpetual intake queue for {name}: every
  captured, unreviewed item and every orphan the sweep finds, reviewed
  oldest first.\n`; the deferred body `Perpetual deferred queue for {name}:
  evaluated items parked until their revival conditions are met.\n`. Update
  any existing test that pinned the old strings. In `vrg_intake_create.py`
  the epilog reads `epic-review intake routes the capture into the
  epic/task model later.`

- [ ] **Step 4: Revise `github-issues.md` for meaning**

  Every `intake-review`, `planning-review` and `deferred-review` becomes
  `epic-review <mode>`. The planning row and the passage on which planning
  item to expand next describe planning review as pruning only (leave,
  defer, drop, one-off), with the choice of what to plan made by the human
  through `epic-create`. `one-off` is added to the planning exits.

- [ ] **Step 5: Revise `cli-tools-overview.md`'s `vrg-queue` sentence**

  "Used by the collect step of the three review skills" becomes "Used by
  the collect step of `epic-review`".

- [ ] **Step 6: Confirm no old name remains outside history**

  Run: `vrg-git grep -nE "intake-review|planning-review|deferred-review" -- src docs/site`
  Expected: no output.

- [ ] **Step 7: Run the tests; confirm they pass, then commit**

  Run: `vrg-container-run -- vrg-validate`, then
  `vrg-commit --type refactor --scope epics --message "name epic-review in queue messages and docs, and drop skill names from queue epic bodies"`

### Task 3: The `epic-review` skill

**Files:**
- Create: `skills/epic-review/SKILL.md`
- Delete: `skills/intake-review/`, `skills/planning-review/`,
  `skills/deferred-review/`
- Modify: `skills/epic-create/SKILL.md` (description triggers, the
  no-argument summary, the `intake-review` reference in Notes)
- Modify: `skills/intake-capture/SKILL.md`, `skills/issue-implement/SKILL.md`
  (references)
- Modify: `docs/paad-review-doctrine.md`, `docs/paad-policy.md`,
  `README.md`, `docs/site/docs/skills/index.md`

**Interfaces:**
- Consumes: Task 1's `one-off` from planning (the skill offers it; until
  Task 1's release, the tool refuses it with a refusal the skill reports,
  which is accepted under the back-to-back release constraint).
- Produces: `/vergil:epic-review <intake|planning|deferred|<epic-ref>>`, and
  the refusal lines Task 7 edits:
  - ad-hoc epic: `<ref> is an ad-hoc epic; it holds placed work, not a
    queue.` (Task 7 appends ` Its tasks are worked with
    /vergil:epic-implement adhoc.`);
  - development epic: `<ref> is a development epic; it is worked with
    /vergil:epic-implement <ref>.`

- [ ] **Step 1: Write `skills/epic-review/SKILL.md`**

  Frontmatter `name: epic-review`, and a `description` carrying the review
  phrases from spec §5.2 "Triggers", each stated as selecting its mode. The
  body has, in order:
  - **Kind detection** from labels (spec §5.1 table) and the keyword form;
    `--repo` as `vrg-queue --repo`. A non-epic ref is refused as not an
    epic.
  - **The shared procedure** (spec §5.2 steps 1–6), written once, carrying
    over from today's skills: the USER-agent preflight, the missing-queue
    rule, batches of 4, one confirmation per batch, applying with
    `vrg-intake-route`, the agent closing dropped and split items, "a
    refusal is information", and scope changes through `vrg-ruling
    draft-scope`.
  - **Per-mode sections** for intake, planning and deferred: order, mode
    check and outcomes from the spec §5.2 table, and each mode's tallies
    line. Intake keeps its five-route table and split mechanics; deferred
    keeps "checking is the point" and "never fabricate a check"; planning
    is the prune-only section of spec §5.2, including the `in-progress` /
    `ready-to-close` handling and the sorted list at the end.
  - **Drop in every mode**, stating the current rule (refused only while a
    development epic is open), which delivers
    vergil-project/vergil-claude-plugin#1007.
  - **Refusals** (the two lines above), and the Notes carried over from the
    old skills (no retired labels, private work stays private, queue items
    are not PR-workable, in and out of deferred only through the route).

- [ ] **Step 2: Delete the three old skill directories**

- [ ] **Step 3: Move the choosing phrases and guidance into `epic-create`**

  Its `description` gains "what should we plan next", "prioritize the
  planning queue" and "what's in planning worth planning". Its no-argument
  entry point gains the guidance: plan an epic only when it will be
  implemented soon, since a plan left sitting goes stale; usually one pick,
  rarely more than two or three.

- [ ] **Step 4: Update every reference in the plugin, for meaning**

  `intake-capture`, `epic-create` Notes, `issue-implement` (its queue
  refusal quotes and routing sentence), the doctrine's §1.7 exemption list,
  `paad-policy.md`, `README.md`'s skill table, and the site docs' skill
  table and the three skill sections (replaced by one `epic-review`
  section). Wherever a passage describes planning review as choosing what
  to plan, rewrite it to the prune-only role.

- [ ] **Step 5: Confirm no old name remains outside history**

  Run: `vrg-git grep -nE "intake-review|planning-review|deferred-review" -- . ':!releases' ':!CHANGELOG.md'`
  Expected: no output.

- [ ] **Step 6: Run the tests; confirm they pass, then commit**

  Run: `vrg-container-run -- vrg-validate` (the SKILL.md frontmatter check
  and markdown lint), then
  `vrg-commit --type feat --scope epic-review --message "add epic-review and retire the three queue review skills"`

### Task 4: `docs` repo management pages

**Files:**
- Modify: `docs/site/docs/management/tools-and-skills.md`
- Modify: `docs/site/docs/management/workflow.md`

- [ ] **Step 1: Revise both pages for meaning**

  The skill list names `epic-review` (intake, planning, deferred) in place
  of the three skills. `workflow.md` says planning review keeps the
  planning queue correct, and that the human chooses what to plan next and
  runs `epic-create` on it.

- [ ] **Step 2: Confirm and commit**

  Run: `vrg-git grep -nE "intake-review|planning-review|deferred-review" -- docs/site`
  Expected: no output. Then `vrg-container-run -- vrg-validate` and
  `vrg-commit --type docs --scope management --message "describe epic-review and prune-only planning review"`

### Task D1: Deploy group A (deployment)

Filed with `vrg-issue-create --kind deployment`, blocked by Tasks 1–4.

- **Preconditions (human-attested):** vergil-tooling released with Tasks 1
  and 2, and vergil-claude-plugin released with Task 3, back to back.
- **Steps:**
  1. `vrg-host plugins` and `vrg-vm update --all`, and install the tooling
     release where it runs.
  2. Update the live queue epics' bodies once to the Task 2 templates:
     every open `Epic (intake): …` and `Epic (deferred): …` in the org's
     `.github` and in each self-homed private repo, found with `vrg-gh
     issue list --label epic --label intake` (and `deferred`), each edited
     with `vrg-gh issue edit <N> --repo <home> --body-file <tmp>`, keeping
     any text below the template line.
- **Acceptance:** `/vergil:epic-review` is listed in a new session, and no
  live queue epic body names a skill.

### Task V1: Validate `epic-review` live (validation)

Filed with `vrg-issue-create --kind validation`, blocked by D1.

- **Acceptance**, each recorded with its output:
  1. `/vergil:epic-review vergil-project/.github#1077` prints the
     development-epic refusal and does nothing else.
  2. "review deferred" starts the deferred mode directly, without the
     which-queue question; stop after the first batch is presented.
  3. `/vergil:epic-review planning` presents the oldest 4 items with leave,
     defer, drop or one-off recommendations only, and never offers a pick;
     stop before confirming.
  4. `vrg-intake-route <a planning item with an open development epic>
     one-off --repo <repo>` is refused with the open-epic reason.

---

## Group B — the ad-hoc driver

### Task 5: `vrg-queue adhoc`

**Files:**
- Modify: `src/vergil_tooling/lib/queue_view.py` (new `AdhocItem`,
  `adhoc_items`, `render_adhoc_text`, `adhoc_to_json`)
- Modify: `src/vergil_tooling/bin/vrg_queue.py` (the `adhoc` choice)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (`vrg-queue`)
- Test: `tests/vergil_tooling/test_queue_view.py`,
  `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**
- Produces:

  ```python
  AdhocState = Literal["ready", "blocked", "in-flight"]

  @dataclass(frozen=True)
  class AdhocItem:
      ref: IssueRef
      title: str
      created_at: str
      kind: str            # "task", "validation" or "deployment"
      blocked_by: tuple[IssueRef, ...]   # open blockers only
      state: AdhocState
      marks: tuple[str, ...]             # Task 6 fills it; () here
      note: str            # Task 6: "local: no clone" when applicable

  def adhoc_items(home: str, *, repo: str | None = None) -> list[AdhocItem]
  def render_adhoc_text(items: Sequence[AdhocItem], *, now: datetime) -> str
  def adhoc_to_json(items: Sequence[AdhocItem]) -> list[dict[str, Any]]
  ```

  `adhoc_items` returns items grouped by repo (repos in name order), oldest
  first within each.

- [ ] **Step 1: Write the failing tests** (with the existing `_FakeGh`)

  - `test_adhoc_items_lists_open_tasks_across_epics`: two ad-hoc epics with
    open and closed children give only the open ones, grouped by repo,
    oldest first within each.
  - `test_adhoc_items_skips_archives_and_legacy_titles`: issues labelled
    `epic` + `ad-hoc` whose title does not start `ADHOC.title_prefix`, or
    that match the legacy archive form, are ignored (Review Focus 1).
  - `test_adhoc_items_refuses_a_truncated_search`: a label search returning
    `LISTING_LIMIT` rows raises.
  - `test_adhoc_items_blocked_state`: a task with an open `Blocked-by:` is
    `blocked` with that ref in `blocked_by`; with every blocker closed it is
    `ready`.
  - `test_adhoc_items_kind_from_labels`: `validation` and `deployment`
    labels give that kind; otherwise `task`.
  - `test_adhoc_repo_narrows_to_one_epic`: `repo="<org>/vergil-tooling"`
    gives only that repo's tasks.
  - `test_adhoc_repo_without_epic_is_empty` (Review Focus 4).
  - `test_vrg_queue_adhoc_text_and_json`, and
    `test_vrg_queue_bare_does_not_list_adhoc`.

- [ ] **Step 2: Run the tests; confirm they fail**

- [ ] **Step 3: Implement**

  Find the epics with one `issue list --label epic --label ad-hoc --state
  open` in *home*, refused when truncated, keeping titles that start
  `ADHOC.title_prefix` and do not match `_LEGACY_ADHOC_ARCHIVE_RE`. Open
  children come from `epics.child_states`; open blockers from
  `epics.blockers_of` and each blocker's state. For `repo`, resolve its
  home with `epics.resolve_epic_home` and keep only the epic titled
  `ADHOC.title_prefix + <bare repo>`. In `vrg_queue.py`, add `adhoc` to the
  choices but not to `_VIEWS`, so a bare run still shows three queues; for
  `adhoc`, `--repo` narrows instead of selecting a queue home, except for a
  self-homed private repo, whose own home is used. Text lines are
  `<slug>  <age>  <state>  <kind>  <title>[  — blocked by <refs>]`.

- [ ] **Step 4: Document `vrg-queue adhoc`** in `cli-tools-overview.md`
  beside the other queues: what it lists, its states, `--repo`.

- [ ] **Step 5: Run the tests; confirm they pass, then commit**

  `vrg-commit --type feat --scope queue --message "add vrg-queue adhoc"`

### Task 6: In-flight detection

**Files:**
- Create: `src/vergil_tooling/lib/in_flight.py`
- Modify: `src/vergil_tooling/lib/queue_view.py` (`adhoc_items` uses it)
- Modify: `src/vergil_tooling/bin/vrg_queue.py` (`--root DIR`, repeatable,
  for `adhoc`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_in_flight.py`

**Interfaces:**
- Consumes: Task 5's `AdhocItem`, `adhoc_items`;
  `live_branches.default_repo_roots`, `live_branches.discover_repos`.
- Produces:

  ```python
  MARKS = ("open-pr", "report-ready", "branch", "worktree", "merge-review", "batch")

  @dataclass(frozen=True)
  class RepoSignals:
      repo: str                    # "owner/name"
      clone: Path | None           # main clone found under the roots
      open_pr_issues: frozenset[int]
      remote_branches: frozenset[str]
      relay_branches: frozenset[str]
      local_branches: frozenset[str]

  class InFlightError(RuntimeError): ...

  def repo_signals(repo: str, clone: Path | None) -> RepoSignals
  def marks_for(issue: int, signals: RepoSignals) -> tuple[str, ...]
  def find_clones(repos: Iterable[str], roots: Sequence[Path]) -> dict[str, Path]
  ```

  `adhoc_items(home, *, repo=None, roots=None)` gains `roots`; a task with
  any mark is `in-flight` (it outranks `blocked`).

- [ ] **Step 1: Write the failing tests**

  - `test_in_flight_open_pr_closing_the_issue`: an open PR whose
    `closingIssuesReferences` names the issue gives `open-pr`.
  - `test_in_flight_open_pr_on_feature_branch`: an open PR whose head is
    `feature/<N>-x` gives `open-pr`.
  - `test_in_flight_report_ready_relay_ref`:
    `refs/vergil/pr-workflow/feature/<N>-x` on the remote gives
    `report-ready`.
  - `test_in_flight_remote_and_local_branch`: `feature/<N>-x` on the remote
    gives `branch`; a local branch or worktree of that name gives
    `worktree`.
  - `test_in_flight_merge_review_directory`: an existing
    `build/.vergil-state/issue-implement/<N>/merge-review/` in the clone
    gives `merge-review`.
  - `test_in_flight_batch_marker_counts` (Review Focus 2): a batch marker at
    `build/.vergil-state/issue-implement/<N>/batch` gives `batch`.
  - `test_in_flight_prefix_does_not_overmatch`: `feature/12-x` is not a
    mark for issue 1.
  - `test_in_flight_repo_without_clone_is_noted` (Review Focus 3): no clone
    gives no local marks and the item's note `local: no clone`.
  - `test_in_flight_read_failure_raises`: a failed `pr list` or `ls-remote`
    raises `InFlightError` naming the repo and the source.
  - `test_vrg_queue_adhoc_no_repo_roots_fails`: with no `--root` on a
    platform with no default, the command exits 1 with the
    `RepoSetError` message.

- [ ] **Step 2: Run the tests; confirm they fail**

- [ ] **Step 3: Implement**

  `repo_signals` reads, once per repo: `pr list --state open --json
  number,headRefName,closingIssuesReferences`, refused when truncated;
  `ls-remote https://github.com/<repo> refs/heads/feature/*
  refs/vergil/pr-workflow/*`; and, when a clone is given, its local
  branches and worktree branches. The merge-review and batch marks are
  path checks in the clone's `build/.vergil-state/issue-implement/<N>/`.
  A branch matches issue *N* only by the exact prefix `feature/<N>-`. Any
  failed read raises `InFlightError`; `vrg_queue` reports it and exits 1.
  `find_clones` maps each `owner/name` to a discovered repo whose `origin`
  is that GitHub repo.

- [ ] **Step 4: Document the in-flight state, its marks and `--root`** in
  `cli-tools-overview.md`.

- [ ] **Step 5: Run the tests; confirm they pass, then commit**

  `vrg-commit --type feat --scope queue --message "report in-flight ad-hoc tasks in vrg-queue adhoc"`

### Task 7: The `adhoc` mode in `epic-implement`

**Files:**
- Modify: `skills/epic-implement/SKILL.md`
- Modify: `skills/epic-review/SKILL.md` (the ad-hoc refusal)
- Modify: `docs/site/docs/skills/index.md` (the `epic-implement` section),
  `README.md` (its table row)

**Interfaces:**
- Consumes: Task 5 and 6's `vrg-queue adhoc [--repo R] [--root DIR] --json`
  and its `state`, `marks`, `blocked_by`, `kind` fields; Task 3's
  refusal line.

- [ ] **Step 1: Add kind detection and the queue refusal**

  In Preflight, detect the ref's kind (spec §5.1). An intake, planning or
  deferred epic or keyword prints `<ref> is the <kind> queue; review it
  with /vergil:epic-review <kind>.` and stops.

- [ ] **Step 2: Add the `adhoc` mode section** with spec §5.3's six steps:
  collect with `vrg-queue adhoc`; summarize by repo with in-flight tasks
  listed apart, with their marks; propose a strategy that judges overlap
  per repo and names what runs at the same time; one confirmation;
  dispatch by kind with the batch-marker check; gate and stop. Name the
  session `epic-adhoc` or `epic-adhoc-<repo>`. Records go on the ad-hoc
  epic of the repo concerned, cross-repo ones on the queue home's own
  ad-hoc epic. List what the mode does not do (plan reading, evolution log,
  terminal handoff, retrospective).

- [ ] **Step 3: Qualify the independence claims** in §3 "Batching ready
  tasks" and §4 "Batch PRs per repository": in the `adhoc` mode, the
  confirmed strategy supplies the independence that a plan supplies by
  construction.

- [ ] **Step 4: Point `epic-review`'s ad-hoc refusal at the mode** (append
  Task 3's second sentence).

- [ ] **Step 5: Update the site docs and README**, then run
  `vrg-container-run -- vrg-validate` and
  `vrg-commit --type feat --scope epic-implement --message "add the adhoc mode"`

### Task D2: Deploy group B (deployment)

Filed with `vrg-issue-create --kind deployment`, blocked by Tasks 5–7.

- **Preconditions (human-attested):** vergil-tooling released with Tasks 5
  and 6, and vergil-claude-plugin released with Task 7, back to back.
- **Steps:** `vrg-host plugins`, `vrg-vm update --all`, and install the
  tooling release where it runs.
- **Acceptance:** `vrg-queue adhoc --help` lists `--root`, and a new session
  lists the `adhoc` mode in `epic-implement`.

### Task V2: Validate `epic-implement adhoc` live (validation)

Filed with `vrg-issue-create --kind validation`, blocked by D2.

- **Acceptance**, each recorded with its output:
  1. `vrg-queue adhoc` lists every open ad-hoc task in the org, and a task
     with a known open PR shows `in-flight` with `open-pr`.
  2. `/vergil:epic-implement adhoc` presents the by-repo summary with
     in-flight tasks apart, and a strategy naming what runs at the same
     time. Stop before confirming.
  3. `/vergil:epic-implement vergil-project/.github#99` (one repo's ad-hoc
     epic) limits the summary to vergil-tooling.
  4. `/vergil:epic-implement intake` prints the queue refusal and stops;
     `/vergil:epic-review vergil-project/.github#99` prints the ad-hoc
     refusal naming `/vergil:epic-implement adhoc`.
