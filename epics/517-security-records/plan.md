# Security records — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every security finding a private draft advisory in
`<org>/.github`, paired with a neutral `security`-labelled intake issue. The
pair stays consistent through routing, planning and the scheduled sweep.

**Architecture:** A new library, `lib/security_records.py`, owns the
advisories API, the two reference lines, pair classification and `retarget`.
A new CLI, `vrg-security-record`, exposes create, list, show and retarget.
Three existing tools call the library:

- `vrg-epic-audit --close` syncs advisory state and reports broken pairs;
- `vrg-intake-route` retargets the advisory on transfer, recreate and split;
- `vrg-queue planning` holds a planning issue back while an advisory still
  tracks it.

The plugin's `epic-create` skill gains the planning handoff step. A probe
(Task 1) runs before anything is built.

**Tech Stack:** Python 3.12+ (`argparse`, `dataclasses`, pytest), the `gh`
CLI via `lib/github.py`, the GitHub REST repository-advisories API, Markdown
skill and docs.

**Spec:** `epics/517-security-records/spec.md`
(vergil-project/.github#517)

**Shape:** this is an epic-level plan. Each task below is **one issue and
one PR in the repo named in its heading**, worked in a worktree of that repo.
Paths are relative to that repo's root. Deployment and validation tasks are
operational: they have no code PR and close on `Outcome: SUCCESS`.

## Global Constraints

- Advisory home: always `<org>/.github`. If it is not public, refuse loudly
  with: `vrg-security-record: <org>/.github is not public; security records
  need a public org .github (private orgs are unsupported)`.
- Advisory create body: `vulnerabilities: [{"package": {"ecosystem": "other",
  "name": "<affected repo bare name>"}}]`. Never publish, never request a
  CVE, never pass `start_private_fork`.
- Reference lines, exact and each alone on its line:
  - in the issue body: `Security advisory: <org>/.github <GHSA-id>`;
  - in the advisory description, one line per tracked issue:
    `Tracking issue: <owner>/<repo>#<N>`.
- Severity values: `critical`, `high`, `medium`, `low` (GitHub's terms). The
  per-paad-scale mapping table lives in the docs (spec 4.1).
- Labels: `security` ("Has a private security record in the org's .github
  advisories") and `security-disclosed` ("Past issue that disclosed a
  security problem publicly; fixed, no private record").
- **Output rule:** advisory `summary` and `description` reach stdout only
  through `show` and `list --details`. `--details` is refused when
  `VRG_EPIC_SWEEP` is set. Every other output (sweep, consistency report,
  errors, `list`) names only GHSA IDs, issue refs, labels and states.
- The advisory-create POST is sent **exactly once**: no retry wrapper.
- Issues are created in `epics.queue_home(<affected repo>)` and linked under
  its intake epic, never in the code repo itself.
- No silent failures: every partial pair is reported with what exists and
  the exact command that finishes it, and the exit code is non-zero.
- The agent never changes App permissions, cuts releases, dispatches
  workflows, or flips repo visibility; those are human actions.
- Validation: `vrg-container-run -- vrg-validate`. Commit with `vrg-commit`.
  Hand off with `vrg-pr-workflow report-ready`. Report-ready notes and PR
  bodies carry no real finding's details.

## Review Focus

1. **A transient error during the advisory-create POST**: retrying it would
   create a second, undeletable draft. The POST must not retry, and on
   failure the duplicate guard decides. Pinned by
   `test_create_advisory_posts_once_without_retry` (Task 2).
2. **The sweep printing an advisory's summary into public Actions logs.**
   Pinned structurally by `test_advisory_record_has_no_text_fields` (Task 2)
   and by `test_security_pass_output_names_only_ids_and_states` (Task 4).
3. **A split or recreate whose retarget fails**: the original intake issue
   must stay open, and the error must name the advisory. Pinned by
   `test_split_security_item_retarget_failure_leaves_original_open` and
   `test_recreate_security_item_retargets_before_closing` (Task 5).
4. **An org whose `.github` is private**: every command refuses loudly; none
   falls back to issue-only. Pinned by `test_advisory_home_refuses_private`
   (Task 2).
5. **A planning issue whose advisory has not been handed off**: it never
   shows `ready-to-close`. Pinned by
   `test_planning_state_held_while_advisory_tracks_it` (Task 6).

---

### Task 1: Probe the advisories API with both App tokens (.github, validation)

No dependencies. **Preconditions (human-attested):**

- "Repository security advisories: Read and write" has been added to the
  user App (`<user>-vergil-user`) and the org ops App (the
  `APP_CLIENT_ID` App of `.github`'s `epic-sweep.yml`), and every
  installation has accepted the change.

`vrg-gh api` is denied to agents, so **the human runs the probe** and the
agent records the outcome. For each App, mint a token with that App's
credentials:

```bash
TOKEN=$(VRG_APP_ID=<app id> VRG_PRIVATE_KEY_PATH=<pem> uv run python -c \
  "from vergil_tooling.lib import github; print(github.get_installation_token('vergil-project', require_installation=True))")
```

Then call `GH_TOKEN=$TOKEN gh api …`.

- [ ] User App: create a draft in `vergil-project/.github` with summary
  `PROBE: security-records API check (no real finding)`, description
  `Probe for epic #517. Not a vulnerability.`, severity `low`, and
  vulnerabilities `[{"package":{"ecosystem":"other","name":"vergil-tooling"}}]`.
  Expected: 201 and a `ghsa_id`.
- [ ] User App: `GET` it, then `PATCH` its description (append
  `Tracking issue: vergil-project/.github#517`). Expected: 200 both times.
- [ ] Ops App: `GET /repos/vergil-project/.github/security-advisories?state=draft`
  lists it. `PATCH state=closed`, then `PATCH state=draft`. Expected: 200,
  and the state reads back each time.
- [ ] User App: `PATCH state=closed` to leave it closed.
- [ ] Record `Outcome: SUCCESS` with the probe's GHSA ID, and say which
  calls each App made. If any call fails, record `Outcome: FAILURE` with
  the status and message. The epic then stops for re-planning with the
  human (spec 4.9): no implementation task starts.

---

### Task 2: `lib/security_records.py` core and the labels (vergil-tooling, implementation)

Blocked by Task 1.

**Files:**

- Create: `src/vergil_tooling/lib/security_records.py`
- Modify: `src/vergil_tooling/data/labels.json` (add `security` and
  `security-disclosed`, with the Global Constraints descriptions)
- Modify: `src/vergil_tooling/lib/github.py`: add `post_json_once`
- Test: `tests/vergil_tooling/test_security_records.py`,
  `tests/vergil_tooling/test_data_labels.py`

**Interfaces:**

- Produces, in `github.py`:
  `post_json_once(endpoint: str, body: dict[str, object]) -> dict[str, Any]`.
  It runs one `gh api <endpoint> -X POST --input -` with `_gh_env()` and no
  `_run_with_retry`. It raises `GitHubAPIError` on failure.
- Produces, in `security_records.py`:
  - constants `SECURITY_LABEL = "security"`,
    `DISCLOSED_LABEL = "security-disclosed"`,
    `SEVERITIES = ("critical", "high", "medium", "low")`;
  - `class PrivateHomeError(RuntimeError)`,
    `class RetargetError(RuntimeError)` (the message names the GHSA ID and
    the step that failed);
  - `@dataclass(frozen=True) class Advisory: ghsa_id: str; state: str;
    affected: str | None; tracking: tuple[IssueRef, ...]; external: bool`.
    It has **no summary or description field**, so the output rule is
    enforced by type;
  - `advisory_home(org: str) -> str`: returns `"<org>/.github"` or raises
    `PrivateHomeError`;
  - `advisory_line(home: str, ghsa_id: str) -> str`,
    `tracking_line(ref: IssueRef) -> str`;
  - `parse_advisory_ref(issue_body: str) -> str | None`,
    `parse_tracking(description: str) -> tuple[IssueRef, ...]`;
  - `create_advisory(home: str, *, summary: str, details: str, affected_repo:
    str, severity: str) -> str`: returns the GHSA ID, using `post_json_once`;
  - `find_duplicate(home: str, *, summary: str, affected_repo: str) -> str | None`;
  - `list_advisories(home: str) -> list[Advisory]`: every state, paginated.
    `external` is true for `state == "triage"` or a non-null `submission`.
    The fetch projects with `--jq` to the GHSA ID, state, package name,
    `submission` and the parsed `Tracking issue:` lines only; summary text is
    never fetched;
  - **error hygiene:** every advisory helper catches `GitHubAPIError` and
    re-raises it without its stdout (status, endpoint and stderr only), so a
    failed read can never echo advisory text;
  - `read_text(home: str, ghsa_id: str) -> tuple[str, str]`: returns
    `(summary, description)`, used only by `show` and `--details`;
  - `set_state(home: str, ghsa_id: str, state: Literal["draft", "closed"]) -> None`;
  - `mark_issue(ref: IssueRef, home: str, ghsa_id: str) -> None`: ensures the
    `security` label and appends the advisory line. Idempotent;
  - `unmark_issue(ref: IssueRef, moved_to: Sequence[IssueRef]) -> None`:
    removes the `security` label and the advisory line, and comments
    `Security record moved to <refs>`. Idempotent;
  - `retarget(home: str, ghsa_id: str, targets: Sequence[IssueRef]) -> None`:
    `mark_issue` each target, replace every `Tracking issue:` line with one
    per target, then `unmark_issue` every previously tracked issue that is
    not a target. Raises `RetargetError` on any failure;
  - `@dataclass(frozen=True) class SecurityIssue: ref: IssueRef; open: bool;
    ghsa_id: str | None`;
  - `security_issues(org: str) -> list[SecurityIssue]`: lists
    `security`-labelled issues per repo, in every state, over the same repo
    enumeration as `epic_audit.open_issues`. Never search;
  - `@dataclass(frozen=True) class Classification: paired: list[Advisory];
    external: list[Advisory]; broken: list[str]`. `broken` holds
    one-line, ID-only reasons;
  - `classify(advisories: Sequence[Advisory], issues: Sequence[SecurityIssue])
    -> Classification`. Implements spec 4.3's classification and
    consistency rules.

- [ ] **Step 1: Write the failing tests**:
  - `test_security_labels_registered`: both labels are present, with the
    exact descriptions;
  - `test_advisory_home_refuses_private`: `repo_visibility` returns
    `"PRIVATE"` → `PrivateHomeError`, and the message matches Global
    Constraints;
  - `test_create_advisory_posts_once_without_retry`: one transient failure
    → exactly one subprocess call, and `GitHubAPIError` raised;
  - `test_create_advisory_body`: ecosystem `other`, the package name is the
    bare repo, no `cve_id` and no `start_private_fork`;
  - `test_advisory_record_has_no_text_fields`:
    `{f.name for f in fields(Advisory)}` has no `summary` and no
    `description`;
  - `test_parse_tracking_multiple_lines` and
    `test_parse_advisory_ref_ignores_lookalike_text`;
  - `test_retarget_marks_targets_before_rewriting_tracking`: a
    `mark_issue` failure → `RetargetError`, and no PATCH sent;
  - `test_retarget_unmarks_dropped_issues`: the dropped issue loses the
    label and the line and gets the moved comment; a kept target is
    untouched;
  - `test_list_advisories_failure_does_not_echo_stdout`: a page-2 failure
    whose stdout holds a fixture summary → the raised error's text lacks it;
  - `test_classify_paired_external_broken`, one case per spec 4.3 rule:
    - paired;
    - triage, and submission present → external;
    - a draft no issue references → broken;
    - an issue whose GHSA doesn't exist → broken;
    - a `security` issue with no advisory line → broken;
    - an advisory with no `Tracking issue:` lines → broken;
    - a tracking line naming an issue that doesn't exist → broken;
    - a tracking ref that doesn't point back → broken;
    - an issue unmarked by `retarget` → not reported.
- [ ] **Step 2: Run them; expect FAIL**:
  `vrg-container-run -- uv run pytest tests/vergil_tooling/test_security_records.py tests/vergil_tooling/test_data_labels.py`.
- [ ] **Step 3: Implement** the interfaces above. Build REST calls on
  `github.read_json_paginated` for lists, on `read_json` for gets, and on
  `write_json("PATCH", …)` for updates.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS**,
  with 100% coverage.
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 3: `vrg-security-record` CLI (vergil-tooling, implementation)

Blocked by Task 2.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_security_record.py`
- Modify: `pyproject.toml` `[project.scripts]`: add
  `vrg-security-record = "vergil_tooling.bin.vrg_security_record:main"`
- Modify: `CLAUDE.md`: add a one-line bullet to the tool list
- Test: `tests/vergil_tooling/test_vrg_security_record.py`

**Interfaces:**

- Consumes everything Task 2 produces, plus `epics.queue_home`,
  `epics.ensure_intake_epic`, `epics.add_child` and `github.create_issue`.
- Produces `main(argv: list[str] | None = None) -> int`, with these
  subcommands:
  - `create --repo O/R --title T --body-file F --details-file D --severity S
    [--summary TEXT] [--issue REF] [--resume GHSA]`. The summary defaults to
    the details file's first line. `--issue` is attach mode;
  - `list [--repo O/R] [--state draft|closed|all] [--details]`;
  - `show (GHSA-ID | ISSUE-REF)`;
  - `retarget GHSA --to REF [--to REF …]`.
- Every run wraps its writes in `github.target_org(<org>)`, as
  `vrg-intake-create` does.

- [ ] **Step 1: Write the failing tests**:
  - `test_create_full_pair_order`: the calls run in order: duplicate check
    → advisory → `ensure_labels(<queue home>, ["security"])` → issue in the
    queue home → `add_child` under intake → tracking PATCH;
  - `test_create_add_child_failure_prints_resume_with_issue`;
  - `test_resume_with_issue_links_under_intake_idempotently`: `--resume
    GHSA --issue REF` re-runs `add_child` (a no-op if already linked), then
    the tracking PATCH. In attach mode (`--issue` without `--resume`), the
    existing issue keeps its current parent;
  - `test_create_issue_failure_prints_resume_command`: exit 1, and stderr
    contains `--resume GHSA-` but no summary text;
  - `test_create_tracking_failure_prints_resume_with_issue`: stderr has
    both `--resume` and `--issue`;
  - `test_create_resume_finds_issue_among_intake_children`: no search call;
  - `test_create_refuses_duplicate`: `find_duplicate` returns an ID → exit
    1, and the message names that ID;
  - `test_create_attach_mode_marks_existing_issue`: no `create_issue` call;
  - `test_list_omits_summary_without_details` and
    `test_list_details_refused_under_epic_sweep`;
  - `test_retarget_cli_failure_exits_nonzero`;
  - `test_private_home_refused_for_every_subcommand`.
- [ ] **Step 2: Run them; expect FAIL.**
- [ ] **Step 3: Implement `main`.** The neutral issue body is the
  `--body-file` contents with the advisory line appended. The repo's
  visibility comes from `advisory_home`, never from a flag.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 4: Security pass in the scheduled sweep (vergil-tooling, implementation)

Blocked by Task 2.

**Files:**

- Modify: `src/vergil_tooling/lib/security_records.py`: add
  `sync_states(home: str, classification: Classification, issues:
  Sequence[SecurityIssue]) -> int`, which returns the number of advisories
  changed
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py`: add
  `_apply_security_pass(org: str) -> int`. Call it from the `--close` path
  after `_apply_orphan_pass`. Add a read-only preview line to the
  report-only path.
- Test: `tests/vergil_tooling/test_security_records.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes `list_advisories`, `security_issues`, `classify` and
  `set_state` (Task 2).
- Rule: for each **paired** advisory, the target state is `closed` when all
  of its tracking issues are closed, and `draft` otherwise. Call
  `set_state` only on a mismatch. External reports are never touched.
- Output: `Security records: <n> advisory state(s) synced, <m> broken
  pair(s).`, then one line per broken pair taken from
  `Classification.broken`. Exit 1 when `m > 0`, after syncing.
- `advisory_home` is called only when the org has at least one `security`
  issue or `.github` exists. An org with neither prints `Security records:
  none` and is not an error.
- Exit codes: the `--close` path runs the orphan pass, then the security
  pass, **even if** the orphan pass returned 1, and returns
  `max(orphan, security)`.

- [ ] **Step 1: Write the failing tests**:
  - `test_sync_closes_when_all_tracked_issues_closed`;
  - `test_sync_keeps_draft_while_any_tracked_issue_open`;
  - `test_sync_reopens_when_issue_reopened`;
  - `test_sync_skips_external_reports`;
  - `test_security_pass_output_names_only_ids_and_states`: the fixture
    summary string is absent from captured stdout and stderr;
  - `test_security_pass_exits_1_on_broken_pairs`;
  - `test_report_only_mode_previews_without_writes`;
  - `test_close_runs_security_pass_after_failed_orphan_pass` (returns 1);
  - `test_org_without_security_items_skips_home_lookup`.
- [ ] **Step 2: Run them; expect FAIL.**
- [ ] **Step 3: Implement**, scoped in `github.target_org(org)` like the
  orphan pass.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 5: `vrg-intake-route` keeps the link (vergil-tooling, implementation)

Blocked by Task 2.

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py`:
  - in `_route` (Split branch), before `_close`;
  - in `_recreate`, before `_close`;
  - in `_one_off`, after a transfer resolves `final`.
- Test: `tests/vergil_tooling/test_intake_route.py`

**Interfaces:**

- Consumes `parse_advisory_ref`, `advisory_home` and `retarget` (Task 2).
- Rule: if the item carries the `security` label and an advisory line, then:
  - Split: `retarget(home, ghsa, action.into)`;
  - recreate: `retarget(home, ghsa, (new,))`;
  - transfer: `retarget(home, ghsa, (final,))`.

  - **Split and recreate:** the retarget runs before `_close`. A
    `RetargetError` stops the route and is re-raised as
    `RouteRefusedError`; the message names the advisory, and the original
    stays open.
  - **Transfer:** the move has already happened, so the hook runs after
    `_settle_parent`. A `RetargetError` is not a refusal. It raises
    `RuntimeError("<old> moved to <final>, but advisory <GHSA> was not
    retargeted; run: vrg-security-record retarget <GHSA> --to <final>")`.
- `security` is already carried on recreate (it is in the registry after
  Task 2). `mark_issue` is idempotent, so the copied advisory line isn't
  duplicated.

- [ ] **Step 1: Write the failing tests**:
  - `test_split_security_item_retargets_to_all_tasks_before_close`;
  - `test_split_security_item_retarget_failure_leaves_original_open`: no
    `_close` call;
  - `test_recreate_security_item_retargets_before_closing`;
  - `test_transfer_security_item_retargets_to_final`;
  - `test_transfer_retarget_failure_reports_moved_not_refused`;
  - `test_non_security_item_never_touches_advisories`;
  - `test_planning_route_leaves_advisory_on_item`.
- [ ] **Step 2: Run them; expect FAIL.**
- [ ] **Step 3: Implement the three hooks.**
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 6: Planning-queue guard (vergil-tooling, implementation)

Blocked by Task 2.

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py`:
  - `_planning_state` gains `advisory_held: bool`;
  - `planning_items` reads each item's body and computes it.
- Test: `tests/vergil_tooling/test_queue_view.py`

**Interfaces:**

- Consumes `parse_advisory_ref`, `list_advisories` and `advisory_home`
  (Task 2). Advisories are listed **once per call**, not once per item,
  and only if some planning item's body has an advisory line. Otherwise
  `advisory_home` is never called, so a queue view works the same for an
  org without security items.
- Rule: `advisory_held` is true when the item's body names an advisory
  whose `tracking` includes the item. A held item that would otherwise be
  `ready-to-close` is reported as `in-progress`, with the note
  `held: advisory <GHSA-id> still tracks this planning issue — retarget it
  to the fix task(s) with vrg-security-record retarget`.
  - No new queue state (the skill allows three).

- [ ] **Step 1: Write the failing tests**:
  - `test_planning_state_held_while_advisory_tracks_it`;
  - `test_planning_state_ready_after_retarget`;
  - `test_planning_items_lists_advisories_once`;
  - `test_non_security_planning_item_unchanged`;
  - `test_no_advisory_lines_never_resolves_home`.
- [ ] **Step 2: Run them; expect FAIL.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`; expect PASS.**
- [ ] **Step 5: Commit and run report-ready.**

---

### Task 7: `epic-create` planning handoff (vergil-claude-plugin, implementation)

Blocked by Tasks 3 and 6.

**Files:**

- Modify: `skills/epic-create/SKILL.md`, steps 9 and 10.

- [ ] Step 9 gains: *If the planning issue carries the `security` label,
  retarget its advisory to the task(s) that fix the problem once they are
  filed: `vrg-security-record retarget <GHSA> --to <task> [--to …]`. Only
  then post `Tasks filed: yes`.*
- [ ] Step 10 gains: *A `security` planning issue stays held (`in-progress`)
  while its advisory still tracks it; `vrg-queue planning` says so. Never
  close it by hand before the retarget.*
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit and run
  report-ready.

---

### Task 8: Security-records docs (vergil-tooling, implementation)

Blocked by Tasks 3–6.

**Files:**

- Create: `docs/site/docs/guides/security-records.md`, covering:
  - the policy and the about-a-day lifetime target;
  - paad's definition of a security finding, quoted;
  - neutral-wording rules;
  - the severity mapping table (spec 4.1);
  - managing a record through its issue (4.3);
  - the planning handoff (4.4b);
  - the pre-flip checklist (4.5);
  - the accepted trade-offs (4.1).
- Create: `docs/site/docs/reference/dev/security-record.md` (CLI reference).
- Modify: `docs/site/mkdocs.yml` (nav, both pages).
- Modify: `docs/site/docs/guides/account-setup.md`: the user App permission
  table gains "Repository security advisories: Read and write". Note that
  the org ops App needs it too, and the audit App does not (deferred,
  spec 4.8).

- [ ] Write the pages. Every example uses the placeholder text from Task 1
  (no real finding).
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS (markdownlint,
  mkdocs build). Commit and run report-ready.

---

### Task 9: Release and deploy (vergil-tooling, deployment)

Blocked by Tasks 2–8. **Preconditions (human-attested):**

- a vergil-tooling release containing Tasks 2–6 and 8 has been cut, and the
  `v2.1` line resolves to it;
- a vergil-claude-plugin release containing Task 7 is published;
- the sweep's tooling install (`ops-epic-sweep.yml@v2.1` through
  `actions/shared/setup/vergil`) resolves to that release.

- [ ] In the agent VM, run `vrg-vm update`. Then check that
  `vrg-security-record --help` lists `create`, `list`, `show` and
  `retarget`.
- [ ] Check that the plugin's `epic-create` step 9 text includes `vrg-security-record retarget`.
- [ ] Record `Outcome: SUCCESS` with both released versions.

---

### Task 10: End-to-end check (.github, validation)

Blocked by Task 9. **Precondition (human-attested):** the human will
dispatch `epic-sweep.yml` when asked (agents can't run workflows).

- [ ] Create a test pair: `vrg-security-record create --repo
  vergil-project/vergil-tooling --title "Test record: security-records
  end-to-end check" --body-file <neutral> --details-file <text "PROBE-E2E
  not a real finding"> --severity low`. Expected: exit 0, one advisory, one
  intake issue, linked both ways.
- [ ] `list` doesn't show `PROBE-E2E`; `show` does.
- [ ] Create a second plain intake issue, then `retarget <GHSA> --to <both
  issues>`. Expected: both labelled, two tracking lines.
- [ ] Close one issue, then ask the human to dispatch `epic-sweep.yml`.
  Expected: the advisory is still a draft.
- [ ] Close the other issue and dispatch again. Expected: the advisory is
  closed, and `vrg-gh run view <id> --log` does not contain `PROBE-E2E`.
- [ ] Record `Outcome: SUCCESS` with the GHSA ID and both run IDs.

---

### Task 11: Audit past public issues (.github, validation)

Blocked by Task 10. Kind: `validation`, because the acceptance is a recorded
audit, not a code change.

- [ ] Search every public vergil-project repo's open and closed issues for
  security disclosures: candidate search first, then the agent's judgment
  per issue.
- [ ] For each hit:
  - unfixed → `vrg-security-record create --issue <ref> …`;
  - fixed → add the `security-disclosed` label;
  - an exposed secret → list it for human rotation.
- [ ] Record the outcome comment, with only counts and issue refs (already
  public) and no new detail: hits, records created, `security-disclosed`
  labels, and secrets to rotate. Then ask the human whether to scrub (spec
  4.6). `Outcome: SUCCESS` once the human has decided.

---

### Bookends (already filed)

- Documentation: vergil-project/.github#518 (this spec and plan).
- Documentation review: vergil-project/vergil-tooling#3671. It also checks
  `.github/SECURITY.md` and spawns a same-repo task if that needs a mention.
- Retrospective: vergil-project/.github#519.

## Self-Review

- **Spec coverage:**
  - 4.1 → Tasks 2 and 3;
  - 4.2 → Task 3 (output rule in 2 and 3);
  - 4.3 → Tasks 2 and 4;
  - 4.4 → Task 5;
  - 4.4b → Tasks 6 and 7;
  - 4.5 → Task 8;
  - 4.6 → Task 11;
  - 4.7 → Task 2;
  - 4.8 → Task 1's precondition and Task 8;
  - 4.9 → Task 1;
  - 4.10 → Tasks 8 and 7, plus the doc-review bookend.
- **Types:** `Advisory`, `SecurityIssue`, `Classification` and
  `RetargetError` are defined in Task 2 and used under the same names in
  Tasks 3–6.
- **Proportion:** this plan is about 1.3 times the spec's length, and it
  carries no function bodies, only signatures, rules and test names. The
  extra length is the five implementation tasks' interface blocks.
