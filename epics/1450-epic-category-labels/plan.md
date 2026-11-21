# epic-category-labels — implementation plan

- **Epic:** vergil-project/.github#1450
- **Planning issue:** vergil-project/.github#1447 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment) are worked with `issue-deploy`. Steps use checkbox (`- [ ]`)
> syntax for tracking.

**Goal:** Every epic in the six orgs carries exactly one category label
(`epic-management`, `ad-hoc`, `proxy` or `epic-development`): the tools apply
it at creation, and a one-off deployment labels the epics that already
exist.

**Architecture:**

- **vergil-tooling** registers the two new labels, adds one function that
  derives an epic's category from the labels and title it already has, and
  applies the derived label in the two creation paths (`vrg-epic-create` and
  `ensure_standing_epic`). `vrg-ensure-label` gains a form that creates one
  registered label by name.
- **Deployment** is operational: one task attests the release is installed
  everywhere, and one task creates the labels in every epic home of the six
  orgs and labels every existing epic, open or closed.

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown documents
(.github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- The four category labels are exactly `epic-management`, `ad-hoc`,
  `proxy` and `epic-development`; only `epic-development` and
  `epic-management` are new (spec §5.1).
- Registry descriptions, verbatim: `epic-development` "Development epic:
  planned with epic-create, closes when its tasks close"; `epic-management`
  "Management epic: the intake, planning, deferred or hoshin epic; never
  auto-closes" (spec §5.1). Colours are the implementer's choice, distinct
  from each other and from `ad-hoc` (`5319e7`) and `proxy` (`bfd4f2`).
- **Display only** (spec §2.3): no code reads `epic-development` or
  `epic-management` to decide behaviour or to find an epic. The category
  function ignores both. `StandingKind.labels` and every lookup that
  filters on it stay unchanged.
- The category function raises on an ambiguous label set; it never guesses
  (spec §5.2).
- The six orgs are `vergil-project`, `logical-minds-foundry`,
  `mq-rest-admin-project`, `mnemosys-project`, `diogenes-project` and
  `vergils-nemesis` (spec §3). Archived repos are never touched.
- No ongoing drift repair: no audit pass or sweep is added (spec §2.2).
- Each tooling task updates the site docs for the CLI it changes
  (`docs/site/docs/reference/cli-tools-overview.md`).
- No fixture runs are seeded: this epic changes no language support,
  container image, shared CI workflow or release tooling.
- Every tooling task ends with a refactor step before its commit: tests
  green, remove any duplication the task introduced, re-run the tests.

## Review Focus

1. **An existing standing epic with no `epic-management` label** (every
   one, until the one-off runs). `ensure_standing_epic` finds and reuses
   it; it never creates a duplicate. Pinned in Task 2a
   (`test_ensure_standing_epic_reuses_epic_without_category_label`).
2. **An epic that carries a new category label it should not** (for
   example `epic` + `epic-management` with no management kind). The
   function derives from the markers alone and returns development. Pinned
   in Task 2a (`test_epic_category_ignores_new_category_labels`).
3. **A caller passing `--label epic-development`** to `vrg-epic-create`.
   It refuses and creates nothing. Pinned in Task 2a
   (`test_refuses_caller_category_label`).
4. **A label name not in the registry** given to `vrg-ensure-label --label`
   with no colour or description. It exits 1 naming the label, and runs no
   `gh label create`. Pinned in Task 2b
   (`test_main_registry_label_unknown_fails`).
5. **A legacy archive that kept `epic` but lost `archive`** (its stamped
   title is the only marker). It is not an epic and gets no category
   label. Pinned in Task 2a (`test_epic_category_stamped_archive_title_is_none`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by.

- **1** (.github, task #1451): spec and plan, this PR.
- **2a** (vergil-tooling, task #4861): the two registry labels, the
  category function, and the derived label applied by `vrg-epic-create`
  and `ensure_standing_epic`.
- **2b** (vergil-tooling, task #4862): `vrg-ensure-label --label <name>`
  creates one label from its registry definition.
- **3** (.github, deployment #1459): the vergil-tooling release carrying 2a
  and 2b attested and installed on every host and runner that creates
  epics. Blocked by 2a and 2b.
- **4** (.github, deployment #1460): the one-off: the legacy
  `Epic (standing):` issues stop being epics, the two labels are created
  in every epic home of the six orgs, and every existing epic is labelled.
  Blocked by 3.

### Bookends

- **Documentation review** (vergil-tooling, bookend #4843). Blocked by 2a
  and 2b. Its sweep covers the docs repo's
  `docs/site/docs/management/epics-and-tasks.md`, which defines the epic
  kinds; it spawns a docs-repo task for it.
- **Retrospective** (.github, bookend #1452). Blocked by everything else.

Notes:

- **2a and 2b are independent** and can run in parallel.
- **Merged is not deployed.** The one-off (4) waits for the release to be
  installed (3), not merely cut, so no host on old tooling creates an
  unlabelled epic after the backfill (spec §5.5).
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1450, with its `Blocked-by:` refs.

---

### Task 2a: category labels applied at creation (vergil-tooling)

Spec §5.1, §5.2, §5.3.

**Files:**

- Modify: `src/vergil_tooling/data/labels.json`
- Modify: `src/vergil_tooling/lib/epics.py` (new function beside
  `standing_kind_of` and `is_proxy`; `ensure_standing_epic`)
- Modify: `src/vergil_tooling/bin/vrg_epic_create.py`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (`vrg-epic-create`)
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_vrg_epic_create.py`

**Interfaces:**

- Produces, in `lib/epics.py`:
  - `EpicCategory = Literal["management", "ad-hoc", "proxy", "development"]`
  - `CATEGORY_LABELS: Mapping[EpicCategory, str]`, mapping each category
    to its label (`"management"` → `"epic-management"`, `"ad-hoc"` →
    `"ad-hoc"`, `"proxy"` → `"proxy"`, `"development"` →
    `"epic-development"`).
  - `DERIVED_CATEGORY_LABELS = frozenset({"epic-development", "epic-management"})`:
    the labels only the tools apply.
  - `class AmbiguousEpicCategoryError(ValueError)`, its message naming the
    conflicting markers.
  - `epic_category(labels: Collection[str], title: str) -> EpicCategory | None`
- Consumes: `standing_kind_of`, `is_proxy`, `is_archive`,
  `labels.ensure_labels`, `labels.ensure_labels_or_report`.

**`epic_category` rules** (spec §5.2), in order:

1. `"epic"` not in labels, or `is_archive(labels, title)`: `None`.
2. Collect the markers present: management (any of `intake`, `planning`,
   `deferred`, `hoshin`), `ad-hoc`, `proxy`. Two or more categories among
   them: raise `AmbiguousEpicCategoryError`. Several management kinds
   together are one category, not ambiguous.
3. One marker: that category. None: `"development"`.

`epic-development` and `epic-management` are never consulted.

- [ ] **Step 1: Write the failing tests** in `test_epics.py`:
  - `test_epic_category_plain_epic_is_development`: `{"epic"}` →
    `"development"`.
  - `test_epic_category_each_management_kind`: each of `intake`,
    `planning`, `deferred`, `hoshin` with `epic` → `"management"`.
  - `test_epic_category_adhoc_and_proxy`: `{"epic","ad-hoc"}` →
    `"ad-hoc"`; `{"epic","proxy"}` → `"proxy"`.
  - `test_epic_category_not_an_epic_is_none`: `{"intake"}` → `None`;
    `{"archive","intake"}` → `None`.
  - `test_epic_category_stamped_archive_title_is_none`: `{"epic","intake"}`
    with a stamped archive title (build it from an existing archive-title
    test fixture) → `None`.
  - `test_epic_category_ambiguous_raises`: `{"epic","ad-hoc","proxy"}` and
    `{"epic","intake","proxy"}` raise `AmbiguousEpicCategoryError`.
  - `test_epic_category_two_management_kinds_is_management`:
    `{"epic","intake","planning"}` → `"management"`.
  - `test_epic_category_ignores_new_category_labels`:
    `{"epic","epic-management"}` → `"development"`;
    `{"epic","intake","epic-development"}` → `"management"`.
  - `test_category_labels_are_in_registry`: every value of
    `CATEGORY_LABELS` is a registry label, with the two descriptions above
    verbatim.
  - `test_ensure_standing_epic_category_label_at_creation`, parametrized
    over every kind in `epics.STANDING_KINDS`: for `INTAKE`, `PLANNING`,
    `DEFERRED` and `HOSHIN`, `create_issue`'s `labels` are
    `[*kind.labels, "epic-management"]` and `ensure_labels` was called with
    the same names before it; for `ADHOC`, they equal `list(ADHOC.labels)`.
    A second assertion pins the parametrization: the set of kinds given
    `epic-management` is exactly `{INTAKE, PLANNING, DEFERRED, HOSHIN}`.
  - `test_ensure_standing_epic_reuses_epic_without_category_label`: the
    lookup returns a row carrying only `DEFERRED.labels`; the existing epic
    is returned, `create_issue` is not called, and every `--label` the
    lookup passed is from `DEFERRED.labels`.
- [ ] **Step 2: Write the failing tests** in `test_vrg_epic_create.py`
  (patch `ensure_labels_or_report` as `vrg_epic_create` imports it, and
  update the existing tests to patch it too):
  - `test_plain_epic_gets_development_label`: `--title T` →
    `labels == ["epic", "epic-development"]`.
  - `test_management_label_derived`: `--label intake` →
    `["epic", "intake", "epic-management"]`.
  - `test_adhoc_gets_no_new_label`: `--label ad-hoc` → `["epic", "ad-hoc"]`.
  - `test_refuses_caller_category_label`: `--label epic-development`, and
    separately `--label epic-management`: rc 1, a stderr line naming the
    label, `create_issue` not called.
  - `test_refuses_ambiguous_labels`: `--label ad-hoc --label proxy`: rc 1,
    `create_issue` not called.
  - `test_ensures_labels_before_create`: `ensure_labels_or_report` is
    called with the home and the final label list before `create_issue`;
    when it returns False, rc 1 and `create_issue` is not called.
- [ ] **Step 3: Run the tests to verify they fail**

  Run `vrg-container-run -- pytest -q` on `tests/vergil_tooling/test_epics.py`
  and `tests/vergil_tooling/test_vrg_epic_create.py`.
  Expected: the new tests FAIL (`epic_category` not defined, labels
  missing).

- [ ] **Step 4: Implement.**
  - Add the two registry entries to `labels.json`.
  - Add `EpicCategory`, `CATEGORY_LABELS`, `DERIVED_CATEGORY_LABELS`,
    `AmbiguousEpicCategoryError` and `epic_category` to `lib/epics.py`.
  - `ensure_standing_epic`: derive the label with `epic_category(kind.labels,
    title)`, as `vrg-epic-create` does; when `CATEGORY_LABELS[category]` is
    in `DERIVED_CATEGORY_LABELS`, append it to the names passed to
    `ensure_labels` and to `create_issue`'s `labels`. Do not test kind
    membership in `QUEUE_KINDS` or any other kind tuple. Leave
    `_find_epic_by_title`, `_await_listed` and `find_standing_epic`
    filtering on `kind.labels`.
  - `vrg_epic_create.main` (not `--proxy-for`): refuse any `--label` in
    `DERIVED_CATEGORY_LABELS`; derive the category from `["epic",
    *args.label]` and `args.title` (an `AmbiguousEpicCategoryError` is rc 1
    with its message); append `CATEGORY_LABELS[category]` when it is in
    `DERIVED_CATEGORY_LABELS`; call `ensure_labels_or_report("vrg-epic-create",
    home, labels)` inside the `target_org` block and return 1 if it fails;
    then create. Each refusal prints one `vrg-epic-create: …` line and
    creates nothing.
  - Document in `cli-tools-overview.md` that `vrg-epic-create` applies the
    derived category label and refuses a caller-given one.
- [ ] **Step 5: Run the tests to verify they pass**

  Run `vrg-container-run -- pytest -q` on `tests/vergil_tooling/test_epics.py`
  and `tests/vergil_tooling/test_vrg_epic_create.py`.
  Expected: PASS.

- [ ] **Step 6: Refactor, then run `vrg-container-run -- vrg-validate`.**
  Expected: PASS.
- [ ] **Step 7: Commit** with `vrg-commit --type feat --scope epics
  --message "apply a derived category label when an epic is created"`.

### Task 2b: create one registered label by name (vergil-tooling)

Spec §5.4.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_ensure_label.py`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (`vrg-ensure-label`)
- Test: `tests/vergil_tooling/test_vrg_ensure_label.py`

**Interfaces:**

- Consumes: `labels.ensure_labels(repo: str, names: Sequence[str]) -> None`,
  `labels.UnknownLabelError`.
- Produces: the CLI form `vrg-ensure-label --repo <owner>/<repo> --label
  <name>` with neither `--color` nor `--description`, used by Task 4.

- [ ] **Step 1: Write the failing tests:**
  - `test_main_registry_label_from_registry`: `--repo o/r --label
    epic-development` calls `ensure_labels("o/r", ["epic-development"])`
    and runs no other `gh label create`.
  - `test_main_registry_label_unknown_fails`: `--label no-such-label`
    with no colour or description: exit 1, stderr names the label, and no
    `gh label create` runs.
  - `test_main_single_label_with_color_description` keeps passing
    unchanged (the explicit form keeps today's behaviour); update
    `test_main_single_label` to the new expectation (registry-backed when
    neither flag is given), using a registered label.
  - `test_main_single_label_color_only_keeps_explicit_form`: `--color`
    alone still runs `gh label create` with that colour, without reading
    the registry.
- [ ] **Step 2: Run the tests to verify they fail**

  Run `vrg-container-run -- pytest -q` on
  `tests/vergil_tooling/test_vrg_ensure_label.py`.
  Expected: the new tests FAIL.

- [ ] **Step 3: Implement:** in single-label mode, when both `args.color`
  and `args.description` are `None`, call `ensure_labels(args.repo,
  [args.label])` and print the name; catch `UnknownLabelError` and return
  1 with `vrg-ensure-label: <message>` on stderr. Otherwise keep
  `_ensure_single`. Document the form in `cli-tools-overview.md`.
- [ ] **Step 4: Run the tests to verify they pass**

  Run `vrg-container-run -- pytest -q` on
  `tests/vergil_tooling/test_vrg_ensure_label.py`.
  Expected: PASS.

- [ ] **Step 5: Refactor, then run `vrg-container-run -- vrg-validate`.**
  Expected: PASS.
- [ ] **Step 6: Commit** with `vrg-commit --type feat --scope labels
  --message "create one registered label by name from the registry"`.

### Task 3: vergil-tooling release for category labels (deployment, .github)

Spec §5.5. Blocked by 2a and 2b.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 2a and 2b is published and installed on the host and every VM,
  its version and tag named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release contains both changes: for the merge commit of
     each of Task 2a's and Task 2b's PRs, `vrg-git merge-base
     --is-ancestor <merge-sha> <tag-sha>` in vergil-tooling exits 0.
  2. `uv tool list` on this host shows `vergil-tooling` at the attested
     version.
  3. For each org whose `.github` runs the daily epic sweep
     (`vergil-project`, `logical-minds-foundry`, `mq-rest-admin-project`),
     read the `vergil` pin in its `.github` repo's `vergil.toml` on
     `develop`: it is an exact version at or after the release, or the
     `v2.1` line pin. For a line pin, confirm the line resolves to this
     release or later under the resolver in force: if vergil-tooling has a
     rolling `v2.1` tag (epic vergil-project/.github#1440 moves line pins to
     it), `vrg-git merge-base --is-ancestor <release-tag-sha>
     <v2.1-tag-sha>` exits 0; otherwise the release is the newest `v2.1.Z`
     tag. This makes the sweep (which can create intake epics) install it.
  4. `vrg-ensure-label --help` lists the form, and `vrg-epic-create --help`
     mentions the derived category label.
- **SUCCESS:** every check passes, recorded with the version and tag.

### Task 4: the one-off label deployment (deployment, .github)

Spec §5.5. Blocked by 3.

- **Precondition:** Task 3 closed with SUCCESS.
- **Procedure (agent-safe; the human confirms the change list before step
  5).** Work files (listings, the script, its output) go in the session's
  scratch directory, never in a repo.
  1. **Find the epic homes.** For each of the six orgs: its `.github`,
     plus each private, unarchived repo of the org (`vrg-gh repo list
     <org> --json name,isArchived,visibility`) that holds at least one
     issue labelled `epic` (`vrg-gh issue list --repo <r> --label epic
     --state all --limit 1`). Record the list per org.
  2. **Retire the legacy standing epics.** In every unarchived repo of the
     six orgs, list the issues labelled `epic` whose title starts with
     `Epic (standing):` (`vrg-gh issue list --repo <r> --label epic
     --state all --search 'in:title "Epic (standing)"' --json
     number,title,state`), keep only titles that start with that prefix,
     and remove `epic` from each closed one (`vrg-gh issue edit <n> --repo
     <r> --remove-label epic`). An open one is left unchanged and reported
     for a human: removing `epic` from an open issue would make the sweep's
     orphan pass move it into intake. On 2026-11-20 there were 13, all
     closed: in vergil-project's `.github`, `vergil-tooling`,
     `vergil-claude-plugin`, `vergil-actions`, `vergil-containers` and
     `vergil-vm`; in logical-minds-foundry's `.github` and
     `mq-resiliency-lab-for-linux`; and in mq-rest-admin-project's
     `mq-rest-admin-rust`, `mq-rest-admin-python`, `mq-rest-admin-go`,
     `mq-rest-admin-dev-environment` and `mq-rest-admin-common`. They are
     dead umbrellas from before the ad-hoc migration and stop being epics.
     Record each ref.
  3. **Create the labels.** For each home, `vrg-ensure-label --repo <home>
     --label epic-development` and `vrg-ensure-label --repo <home> --label
     epic-management`. Each exits 0.
  4. **Work out the changes.** For each home, save every `epic` issue,
     open and closed, to `<home-slug>.json` (`vrg-gh issue list --repo
     <home> --label epic --state all --limit 1000 --json
     number,title,labels`; if a home returns 1000 rows, stop: the listing
     may be truncated). Classify it with this script, `classify.py`, run by
     the installed tooling's own interpreter:
     `"$(dirname "$(readlink -f "$(command -v vrg-epic-create)")")/python"
     -I classify.py < <home-slug>.json`.

     ```python
     import json
     import sys

     from vergil_tooling.lib import epics

     for row in json.load(sys.stdin):
         names = {label["name"] for label in row["labels"]}
         if epics.is_archive(names, row["title"]):
             print(f"{row['number']}\tarchive\t-")
             continue
         try:
             category = epics.epic_category(names, row["title"])
         except epics.AmbiguousEpicCategoryError as exc:
             print(f"{row['number']}\tambiguous\t{exc}")
             continue
         if category is None:
             print(f"{row['number']}\tnot-an-epic\t-")
             continue
         label = epics.CATEGORY_LABELS[category]
         add = label in epics.DERIVED_CATEGORY_LABELS and label not in names
         print(f"{row['number']}\t{category}\t{'add ' + label if add else '-'}")
     ```

     **Sanity check, before presenting:** the script printed one line per
     row of the listing; and every row whose labels include `intake`,
     `planning`, `deferred` or `hoshin`, and that the script did not print
     as `archive` (the §5.2 archive rule, label or stamped title), is
     classified `management` or `ambiguous`. A failed check stops the run.

     Present per home: counts by category, the `add` lines, and the
     `ambiguous` lines. The human confirms.
  5. **Apply.** For each confirmed `add` line, `vrg-gh issue edit <n>
     --repo <home> --add-label <label>`. No issue's state changes.
  6. **Verify.** Repeat step 4's listing, script and sanity check: no
     `add` line remains in any home, and step 2's listing finds no
     `Epic (standing):` issue labelled `epic`.
- **Result comment:** per org, the homes, the legacy epics retired, the
  number of epics labelled per label, and the ambiguous list (each left
  unlabelled, for a human).
- **SUCCESS:** steps 2, 3, 5 and 6 completed for all six orgs. A failed
  `gh` call is recorded with its error and the task stays open; a re-run
  is safe, since removing an absent label, creating a present label and
  adding a present label are all no-ops.

## Evolution during execution

The task list above is left as planned; each entry below records a change
made during execution. The spec stays as approved.

### 2026-11-21: a fix task for the `vrg-ensure-label` help text

Task 3 (#1459) step 4 checks that `vrg-ensure-label --help` describes the
registry-backed `--label` form, but Task 2b's steps did not change the
help text, so the check failed on `v2.1.332`. The run filed
vergil-project/vergil-tooling#4870 under this epic, fixed by PR #4872. The
human ruled that the installed code (the registry-backed branch and its
`UnknownLabelError` handling) satisfies that half of the check, so Task 3
closed with SUCCESS and Task 4 did not wait for #4870's release.

### 2026-11-21: the documentation review spawned two doc tasks

The sweep (vergil-tooling#4843) filed vergil-project/docs#152 for
`epics-and-tasks.md`, as the plan foresaw, and also
vergil-project/vergil-claude-plugin#1234: the `epic-create` skill said a
new epic gets only the `epic` label.

### 2026-11-21: the one-off retired 14 legacy umbrellas, not 13

Task 4 step 2's live listing found a 14th closed `Epic (standing):` issue,
mq-rest-admin-project/.github#22. The human ruled that the step's rule
governs and the list above is a dated snapshot. The human also ruled that
two closed `[archived]`-titled epics (vergil-project/.github#683 and #443),
which the archive rule does not recognise, get no special handling.
