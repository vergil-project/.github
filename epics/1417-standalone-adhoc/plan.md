# standalone-adhoc — implementation plan

- **Epic:** vergil-project/.github#1417
- **Planning issue:** vergil-project/.github#1146 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ad-hoc epics hold only standalone work: another org's epic tasks
live under a same-org proxy epic driven by the primary, hoshin's logbook
tasks live under a standing hoshin epic, advisories can follow a fix across
orgs, and every misplaced task is re-homed.

**Architecture:**

- **vergil-tooling** does every deterministic part: the `HOSHIN` standing
  kind and its `vrg-queue hoshin` view; the proxy model (`epics.is_proxy`,
  the `Primary:` line, the trusted `Proxy:` comments) and its exemptions
  from rollup and the development-epic checks; `vrg-epic-create
  --proxy-for`; the audit's proxy children, link check, proxy close pass and
  misplacement report; merge review's proxy-aware open tasks; and the
  cross-org advisory move in `vrg-security-record`.
- **vergil-claude-plugin** changes the skills that place or drive work:
  `hoshin`, `epic-implement`, `issue-implement`, `merge-review`,
  `epic-retrospective`, `epic-create` and `epic-review`.
- **Migration** is operational: deployment tasks that move the existing
  misplaced tasks once the releases are installed, and a validation.
- The **hoshin group** (Tasks 2a to 6d) ships first and depends on no proxy
  task (spec §6).

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown skills
(vergil-claude-plugin), Markdown documents (.github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- Proxies exist only between public repos: the primary's repo and every
  repo whose tasks a proxy holds are public, and both epics home in a public
  `<owner>/.github` (spec §5).
- A guard rests on deterministic metadata, never on matching text. The
  misplacement check is a report, never a refusal, move or close (spec
  §4.8).
- Machine-read lines are alone on their line: `Primary: <owner>/<repo>#<N>`
  (proxy body), `Proxy: <owner>/.github#<N>` (comment on the primary),
  `Standalone-confirmed` (comment on an ad-hoc task), `Moved from:
  <GHSA-id>` and `Moved to: <org>/.github <GHSA-id>` (advisories).
- Comments count only when their author is listed by `vrg-trusted-authors`
  (`trusted_authors.load_trusted_authors()`), a leading `app/` stripped. A
  failed load is an error, never an empty list.
- **Fail loudly** (spec §5): a `Primary:` or `Proxy:` line that cannot be
  parsed or resolved, a proxy whose primary does not name it, or an
  unreadable proxy is an error in every reader, never a skipped item.
- Every App-token call runs under `github.target_org(<owner>)` of the org it
  touches; a tool that writes in two orgs enters each in turn, and refuses
  when a token is missing (spec §5).
- Each tooling task updates the site docs for the CLI or behaviour it
  changes (`docs/site/docs/reference/cli-tools-overview.md`, and
  `docs/site/docs/guides/security-records.md` for Task 11).
- No fixture runs are seeded: this epic changes no language support,
  container image, shared CI workflow or release tooling.

## Review Focus

1. **A forged `Proxy:` comment on a public primary** (an untrusted author
   names a proxy). It is ignored. Pinned in Task 7
   (`test_proxies_of_ignores_untrusted_comment`).
2. **A proxy whose `Primary:` line is missing, duplicated, or names a
   non-epic.** Every reader raises `ProxyLinkError`. Pinned in Task 7
   (`test_primary_of_missing_line`, `test_primary_of_two_lines`) and Task 9a
   (`test_epic_children_bad_proxy_fails`).
3. **A primary closed as not planned while a proxy task is open.** The sweep
   leaves the proxy open and reports drift. Pinned in Task 9b
   (`test_proxy_with_open_task_after_primary_closed_is_drift`).
4. **A cross-org move re-run after the new advisory exists but before the
   original is closed.** The re-run reuses it. Pinned in Task 11
   (`test_move_rerun_reuses_moved_from`).
5. **`--proxy-for` re-run when the proxy exists but its comment was never
   posted.** It posts the comment and creates nothing. Pinned in Task 8
   (`test_proxy_for_rerun_posts_missing_comment`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

### Hoshin group

- **1** (.github, task #1418): spec and plan, this PR.
- **2a** (vergil-tooling): the `HOSHIN` standing kind, its sentinel (and the
  `--epic` help that lists it), its drain, and `vrg-intake-route`'s refusal
  of it.
- **2b** (vergil-tooling): `vrg-queue hoshin`. Blocked by 2a.
- **3** (.github, deployment): the vergil-tooling release carrying 2a and 2b
  attested and installed. Blocked by 2a and 2b.
- **4** (vergil-claude-plugin): hoshin files under the hoshin epic and reads
  it in its stale check; `epic-implement` refuses the hoshin epic;
  `issue-implement` refuses its tasks. Blocked by 3.
- **5** (.github, deployment): the vergil-claude-plugin release carrying 4
  attested and installed, and every open `Logbook:` task moved from an
  ad-hoc epic to the hoshin epic. Blocked by 4.
- **6** (vergil-claude-plugin): the `Logbook:` title skip removed from
  `epic-implement`'s adhoc mode. Blocked by 5.
- **6d** (.github, deployment): the vergil-claude-plugin release carrying 6
  attested and installed. Blocked by 6. This closes the hoshin group on its
  own, whatever the state of the proxy group.

### Proxy group

- **7** (vergil-tooling): the proxy model and its exemptions from rollup,
  drift close, the Documents audit and the roadmap.
- **8** (vergil-tooling): `vrg-epic-create --proxy-for`. Blocked by 7.
- **9a** (vergil-tooling): `vrg-epic-audit --epic` lists proxy children and
  fails on link drift. Blocked by 7.
- **9b** (vergil-tooling): the sweep's proxy close pass. Blocked by 7.
- **9c** (vergil-tooling): the misplaced ad-hoc task report and
  `Standalone-confirmed`. Blocked by 7.
- **10** (vergil-tooling): merge review's open tasks resolve through a proxy.
  Blocked by 9a.
- **11** (vergil-tooling): the cross-org advisory move and the moved state.
- **12** (.github, deployment): the vergil-tooling release carrying 7 to 11
  attested and installed. Blocked by 8, 9a, 9b, 9c, 10 and 11.
- **13a** (vergil-claude-plugin): `epic-implement` and `epic-retrospective`
  for proxies. Blocked by 12.
- **13b** (vergil-claude-plugin): `merge-review` takes a proxy task's design
  sources from its primary. Blocked by 12.
- **13c** (vergil-claude-plugin): `epic-create` and `epic-review` follow the
  placement rule. Blocked by 12.
- **14** (.github, deployment): the vergil-claude-plugin release carrying
  13a to 13c attested and installed. Blocked by 13a, 13b and 13c.
- **15** (.github, deployment): the migration: the proxy for
  logical-minds-foundry/.github#319, #3188 and #3189 re-homed, every org's
  ad-hoc and deferred queues swept, and, for each primary, a task filed
  under that primary for its plan's evolution entry. Blocked by 12 and 14.
- **16** (.github, validation): no misplaced task remains, every confirmed
  candidate is re-homed, the audits are clean, and each primary's
  evolution-entry task is filed. Blocked by 15.

### Bookends

- **Documentation review** (vergil-tooling, bookend #4790). Blocked by 2a,
  2b, 4, 6, 7 to 11 and 13a to 13c. Its sweep covers the docs repo's
  `docs/site/docs/management/` pages (`epics-and-tasks.md`, `intake.md`,
  `workflow.md`, `tools-and-skills.md`), which define ad-hoc epics; it
  spawns a docs-repo task for them.
- **Retrospective** (.github, bookend #1419). Blocked by everything else.

Notes:

- **The two groups run in parallel and ship separately.** Nothing in 2a to
  6d depends on 7 to 16, in merging or in deploying: the hoshin group has
  its own tooling release (3) and plugin releases (5, 6d). Task 7, 11 and
  the hoshin tasks can start at once.
- **A primary's plan evolution stays in its own org.** The evolution entry
  that names a re-homed task in its primary's plan (Task 15) is the
  primary's own work, filed under the primary as a native sub-issue, not a
  task of this epic, so it needs no proxy. This epic checks only that it is
  filed; when it lands is the primary's business.
- **Merged is not deployed.** Each plugin task that runs the new tooling
  waits for its release (3, 12); the migration waits for both releases (12
  and 14), so it runs only the sanctioned, installed tools.
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1417, with its `Blocked-by:` refs. The security
  advisories tracking the planning issue are retargeted then to the tasks
  that implement their requirements: GHSA-xj7j-xg99-xhhx to Task 8;
  GHSA-8fmf-cp85-xh7r, GHSA-pm2g-vjv4-pmvh, GHSA-rhfx-48qr-xfqw,
  GHSA-rwxm-5qfw-7jmj, GHSA-rvjx-26r5-6255 and GHSA-r3mh-rr3j-6qmh to Task
  11. All are in
  vergil-project, as are both tasks.

---

### Task 2a: The `HOSHIN` standing kind (vergil-tooling)

Spec §4.5.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`HOSHIN`, `STANDING_KINDS`,
  `DRAINED_STANDING_KINDS`, `ensure_hoshin_epic`, the sentinel table in
  `resolve_epic_ref`, `drain_standing_org`)
- Modify: `src/vergil_tooling/lib/intake_route.py` (the one-off route's
  epic check)
- Modify: `src/vergil_tooling/bin/vrg_issue_create.py`,
  `src/vergil_tooling/bin/vrg_epic_move.py` (the `--epic` help and module
  docstrings list `hoshin`)
- Modify: `src/vergil_tooling/data/labels.json` (the `hoshin` label; the
  existing `test_standing_kind_labels_are_in_registry` checks it)
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_intake_route.py`,
  `tests/vergil_tooling/test_vrg_issue_create.py`,
  `tests/vergil_tooling/test_vrg_epic_move.py`

**Interfaces:**

- Produces: `HOSHIN = StandingKind(key="hoshin", sentinel="hoshin",
  title_prefix="Epic (hoshin): ", labels=("epic", "hoshin"),
  archive_prefix="Archive (hoshin): ", archive_labels=("archive",
  "hoshin"), body=…, archive_body=…)`; `STANDING_KINDS` gains it;
  `QUEUE_KINDS` does not. `DRAINED_STANDING_KINDS = tuple(k for k in
  STANDING_KINDS if k is not ADHOC)`. `ensure_hoshin_epic(target_repo: str)
  -> IssueRef`: the `Epic (hoshin): <org>` epic in the org's `.github`,
  created and reused idempotently like `ensure_intake_epic`.

- [ ] **Step 1: Write the failing tests** in `test_epics.py`:
  `test_hoshin_is_standing_not_queue`; `test_resolve_epic_ref_hoshin`
  (`--epic hoshin` for `vergil-project/docs` resolves to `Epic (hoshin):
  vergil-project` in `vergil-project/.github`);
  `test_drain_standing_child_hoshin` (a closed task under the hoshin epic
  moves to `Archive (hoshin): vergil-project` for its quarter);
  `test_drain_standing_org_covers_hoshin` (the sweep's plan includes the
  hoshin epic's closed children); `test_hoshin_epic_docs_skipped`
  (`epic_docs.is_skipped` is true for it). In `test_intake_route.py`:
  `test_one_off_refuses_hoshin_sentinel` (the refusal names
  `/vergil:hoshin`, and no epic is created: no create call is made);
  `test_one_off_refuses_hoshin_epic_ref` (an explicit ref to the hoshin
  epic is refused). In `test_vrg_issue_create.py` and
  `test_vrg_epic_move.py`: `test_epic_help_lists_hoshin`.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_intake_route.py
  -k hoshin`. Expect FAIL.
- [ ] **Step 3: Implement.** `drain_standing_org` loops over
  `DRAINED_STANDING_KINDS` instead of `QUEUE_KINDS`; the hoshin home comes
  from `queue_home` like the queue kinds. The one-off route refuses the
  `hoshin` sentinel by name before resolution, so a refused route creates
  nothing (as `vrg-epic-move` refuses `deferred`), and refuses an explicit
  ref whose `standing_kind_of` is `HOSHIN`. `ADHOC.body` states the
  rule (spec §4.6): an ad-hoc epic holds only standalone, single-issue
  work, and a task owned by any epic or skill never goes there.
- [ ] **Step 4:** Run the tests again, then `vrg-container-run --
  vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epics --message
  "add the hoshin standing epic"`, then report-ready.

### Task 2b: `vrg-queue hoshin` (vergil-tooling)

Spec §4.5.

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py` (`hoshin_items`, sharing
  `adhoc_items`'s body)
- Modify: `src/vergil_tooling/bin/vrg_queue.py` (the `hoshin` mode, same
  flags as `adhoc`: `--json`, `--repo`, `--root`)
- Test: `tests/vergil_tooling/test_queue_view.py`,
  `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**

- Consumes: `epics.HOSHIN`, `epics.find_standing_epic` (Task 2a).
- Produces: `hoshin_items(home: str, *, repo: str | None = None, roots:
  Sequence[Path] | None = None) -> list[AdhocItem]`, the same fields, states
  and dispositions as `adhoc_items`, read from the home's hoshin epic.
  `vrg-queue hoshin --json --repo vergil-project/docs --root <dir>` prints
  `adhoc_to_json` output.

- [ ] **Step 1: Write the failing tests:** `test_hoshin_items_lists_open_tasks`
  (a `Logbook:` task with a worktree mark is `in-flight`, `resume`);
  `test_hoshin_items_no_epic_is_empty`; `test_vrg_queue_hoshin_json`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** by extracting the shared body of `adhoc_items`
  into `_standing_items(epic_refs, roots)`, used by both.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope queue --message
  "add vrg-queue hoshin"`, then report-ready.

### Task 3: vergil-tooling release for the hoshin epic (deployment, .github)

Blocked by 2a and 2b.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 2a and 2b is published and installed on the host and the VMs, its
  version named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list Tasks 2a and 2b's PRs.
  2. Run `vrg-queue hoshin --json --repo vergil-project/docs --root <dir>`
     from the installed release: it exits 0.
  3. Run `vrg-issue-create --help` and confirm `hoshin` is a listed
     `--epic` sentinel.
- **SUCCESS:** every check passes, recorded with the version.

### Task 4: hoshin uses the hoshin epic (vergil-claude-plugin)

Spec §4.5. Blocked by 3.

**Files:**

- Modify: `skills/hoshin/SKILL.md` (preflight step 6, the stale check; the
  handoff's step 1, which holds the task-filing command)
- Modify: `skills/epic-implement/SKILL.md` (preflight: refuse an `epic` +
  `hoshin` issue)
- Modify: `skills/issue-implement/SKILL.md` (preflight: refuse a task whose
  native parent is `epic` + `hoshin`)

- [ ] **Step 1:** In `SKILL.md`'s handoff step 1, file the task with
  `vrg-issue-create --epic hoshin --repo vergil-project/docs --title
  "Logbook: <date> <mode>" --body-file <tmp>` (today `--epic adhoc`).
  Search the skill and its references for any other copy of the command.
- [ ] **Step 2:** In `SKILL.md` step 6, list with `vrg-queue hoshin --json
  --repo vergil-project/docs --root <projects-dir>`. Keep the `Logbook:`
  title filter: only `Logbook:` tasks are candidates for the stale close,
  and any other open task under the hoshin epic is named to the human and
  left open.
- [ ] **Step 3:** `epic-implement` refuses the hoshin epic with `<ref> is the
  hoshin epic; its tasks belong to /vergil:hoshin.`; `issue-implement`
  refuses a task under it with the same pointer.
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope hoshin --message
  "file and read logbook tasks under the hoshin epic"`, then report-ready.

### Task 5: plugin release and logbook tasks moved (deployment, .github)

Spec §4.7. Blocked by 4.

- **Precondition (human-attested):** the vergil-claude-plugin release
  carrying Task 4 is published and installed, its version named in a
  comment.
- **Procedure (agent-safe):**
  1. Confirm the installed hoshin skill files its task with `--epic hoshin`
     and lists with `vrg-queue hoshin`, and that no copy of `--epic adhoc`
     remains in it.
  2. `vrg-queue adhoc --json --repo vergil-project/docs --root <dir>`; take
     every open task titled `Logbook:`.
  3. For each, `vrg-epic-move --task vergil-project/docs#<N> --epic hoshin`.
  4. Re-run step 2: no `Logbook:` task remains. Run `vrg-queue hoshin` for
     the docs repo: each moved task is listed.
- **SUCCESS:** no `Logbook:` task remains under an ad-hoc epic, recorded
  with the moved refs (or "none open").

### Task 6: retire the `Logbook:` skip (vergil-claude-plugin)

Spec §4.5. Blocked by 5.

- **Files:** Modify `skills/epic-implement/SKILL.md` (the adhoc mode's
  step 2: remove the `Logbook:` skip and its "Skipped (logbook tasks…)"
  summary line), `skills/hoshin/SKILL.md` (the sentence saying
  `epic-implement`'s adhoc mode skips `Logbook:` tasks).
- [ ] **Step 1:** Remove the skip. Reword the hoshin sentence: logbook tasks
  live under the hoshin epic, which `epic-implement` and `issue-implement`
  refuse. Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 2:** Commit `vrg-commit --type refactor --scope epic-implement
  --message "retire the Logbook: title skip now that logbook tasks have
  their own epic"`, then report-ready.

### Task 6d: plugin release retiring the skip (deployment, .github)

Blocked by 6.

- **Precondition (human-attested):** the vergil-claude-plugin release
  carrying Task 6 is published and installed, its version named in a
  comment.
- **Procedure (agent-safe):** confirm the release notes list Task 6's PR,
  and that the installed `epic-implement` skill no longer carries the
  `Logbook:` skip.
- **SUCCESS:** both confirmed, recorded with the version.

### Task 7: the proxy model and its exemptions (vergil-tooling)

Spec §4.2.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`PROXY_LABEL`, `is_proxy`;
  `rollup` skips a proxy)
- Create: `src/vergil_tooling/lib/proxies.py` (the `Primary:` line, the
  `Proxy:` comments)
- Modify: `src/vergil_tooling/lib/epic_docs.py` (`is_skipped`),
  `src/vergil_tooling/lib/roadmap.py` (`_is_perpetual` excludes proxies, so
  `gather` and `epic_audit.epic_drift` skip them)
- Modify: `src/vergil_tooling/data/labels.json` (the `proxy` label)
- Test: `tests/vergil_tooling/test_proxies.py`,
  `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_epic_docs.py`,
  `tests/vergil_tooling/test_roadmap.py`,
  `tests/vergil_tooling/test_epic_audit.py`

**Interfaces:**

- Produces, in `epics`: `PROXY_LABEL = "proxy"`; `is_proxy(labels:
  Collection[str]) -> bool`.
- Produces, in `proxies`: `class ProxyLinkError(RuntimeError)`;
  `primary_line(primary: IssueRef) -> str` (`"Primary: <slug>"`);
  `proxy_line(proxy: IssueRef) -> str` (`"Proxy: <slug>"`);
  `primary_of(proxy: IssueRef) -> IssueRef` (reads the proxy body; exactly
  one `Primary:` line naming an `epic`-labelled issue, else
  `ProxyLinkError`); `proxies_of(primary: IssueRef) -> tuple[IssueRef,
  ...]` (the union of `Proxy:` lines in trusted comments, sorted, never
  shrinking); `is_proxy_ref(ref: IssueRef) -> bool`.

- [ ] **Step 1: Write the failing tests** in `test_proxies.py`:
  `test_primary_of_reads_line`; `test_primary_of_missing_line`;
  `test_primary_of_two_lines`; `test_primary_of_non_epic`;
  `test_proxies_of_union_of_trusted_comments` (two comments, two proxies);
  `test_proxies_of_ignores_untrusted_comment`;
  `test_proxies_of_strips_app_prefix`;
  `test_proxies_of_trusted_load_failure_raises`;
  `test_proxies_of_line_not_alone_ignored`. In the other files:
  `test_rollup_skips_proxy` (all tasks closed, the proxy stays open);
  `test_epic_drift_skips_proxy`; `test_is_skipped_proxy`;
  `test_gather_excludes_proxy`; `test_proxy_label_in_registry`.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_proxies.py tests/vergil_tooling/test_epics.py
  tests/vergil_tooling/test_epic_docs.py tests/vergil_tooling/test_roadmap.py
  tests/vergil_tooling/test_epic_audit.py -k proxy`. Expect FAIL.
- [ ] **Step 3: Implement.** Comments come from `github` with their authors,
  as `epic_size` reads them; the trusted set comes from
  `trusted_authors.load_trusted_authors()`.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epics --message
  "add the proxy epic model and exempt proxies from rollup and the
  development-epic checks"`, then report-ready.

### Task 8: `vrg-epic-create --proxy-for` (vergil-tooling)

Spec §4.2, §5. Blocked by 7.

**Files:**

- Modify: `src/vergil_tooling/lib/proxies.py` (`create_proxy`)
- Modify: `src/vergil_tooling/bin/vrg_epic_create.py` (`--proxy-for`;
  `--title` not required with it)
- Test: `tests/vergil_tooling/test_proxies.py`,
  `tests/vergil_tooling/test_vrg_epic_create.py`

**Interfaces:**

- Consumes: `primary_of`, `proxies_of`, `proxy_line`, `primary_line`,
  `PROXY_LABEL` (Task 7).
- Produces: `class ProxyRefusedError(RuntimeError)`;
  `@dataclass(frozen=True) class ProxyResult: proxy: IssueRef; created:
  bool; reopened: bool; commented: bool`; `create_proxy(primary: IssueRef,
  target_repo: str) -> ProxyResult`. Title `Epic (proxy): <primary title
  without "Epic: ">`, labels `epic` + `proxy`, body the `Primary:` line and
  one explanation line.

- [ ] **Step 1: Write the failing tests:** `test_proxy_for_creates_and_comments`;
  `test_proxy_for_same_org_refused`; `test_proxy_for_primary_not_open_dev_epic_refused`
  (standing, proxy, closed); `test_proxy_for_private_primary_refused`;
  `test_proxy_for_private_target_refused` (both refusals say the task must
  not be filed in an ad-hoc epic); `test_proxy_for_existing_open_proxy_prints_it`;
  `test_proxy_for_closed_proxy_reopened` (open primary; reported, no
  duplicate); `test_proxy_for_rerun_posts_missing_comment`;
  `test_proxy_for_rerun_is_noop`; `test_proxy_for_token_missing_refuses`;
  `test_proxy_for_ensures_labels_before_create` (`ensure_labels` for `epic`
  and `proxy` in the target home runs before the issue is created).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** Find an existing proxy by searching the target
  home for `epic` + `proxy` issues whose `primary_of` is the primary. Write
  the proxy under `target_org(target owner)` and the comment under
  `target_org(primary owner)`.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epic-create
  --message "add vrg-epic-create --proxy-for"`, then report-ready.

### Task 9a: proxy children and the link check (vergil-tooling)

Spec §4.3, §4.4. Blocked by 7.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`epic_children`,
  `proxy_link_problems`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (`--epic` prints
  `epic_children`, exits 1 on link problems)
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `proxies_of`, `primary_of`, `ProxyLinkError` (Task 7).
- Produces: `epic_children(epic: IssueRef) -> list[ChildState]` (the epic's
  own children, then each proxy's, each with its repo);
  `proxy_link_problems(epic: IssueRef) -> list[str]` (for a primary: each
  listed proxy that is missing or whose `Primary:` disagrees; for a proxy:
  a primary without the matching `Proxy:` comment). On a proxy, `--epic`'s
  JSON gains a `"primary": "<owner>/<repo>#<N>"` key, which merge review
  reads (Task 13b); on any other epic the key is absent.

- [ ] **Step 1: Write the failing tests:**
  `test_epic_children_includes_proxy_tasks`;
  `test_epic_children_bad_proxy_fails`;
  `test_vrg_epic_audit_epic_link_drift_exits_1` (problems on stderr, no
  JSON); `test_vrg_epic_audit_epic_on_proxy_reports_primary`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**, keeping `render_epic_children`'s JSON shape.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epic-audit
  --message "list proxy tasks among an epic's children and fail on proxy
  link drift"`, then report-ready.

### Task 9b: the sweep's proxy close pass (vergil-tooling)

Spec §4.2. Blocked by 7.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`proxy_closures`,
  `close_proxies`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (the `--close` path and
  its report)
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: `primary_of` (Task 7).
- Produces: `@dataclass(frozen=True) class ProxyClosure: proxy: IssueRef;
  primary: IssueRef; open_tasks: int`; `proxy_closures(home: str) ->
  list[ProxyClosure]` (each open proxy in *home* whose primary is closed),
  with `proxy_errors: list[str]` returned beside it, one line per proxy
  whose `Primary:` link fails, never raised; `close_proxies(closures:
  list[ProxyClosure]) -> list[str]` (closes those with `open_tasks == 0`;
  the rest are returned as drift lines).

- [ ] **Step 1: Write the failing tests:**
  `test_proxy_closed_when_primary_closed_and_no_open_task`;
  `test_proxy_with_open_task_after_primary_closed_is_drift`;
  `test_proxy_left_open_while_primary_open`;
  `test_proxy_unreadable_primary_reported_and_run_fails` (the error is on
  stderr and in the closed report, and the run exits non-zero at the end);
  `test_proxy_link_error_does_not_stop_other_passes` (in
  `test_vrg_epic_audit.py`: the reopen, drain and orphan passes still run).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**; `--close` runs `close_proxies` after
  `close_drift`, and the preview lists both kinds. A proxy link error is
  collected, never raised mid-run, as `drain_standing_org` treats a
  corrupted home; the exit status reflects it once every pass has run.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epic-audit
  --message "close proxies whose primary has closed in the daily sweep"`,
  then report-ready.

### Task 9c: the misplaced ad-hoc task report (vergil-tooling)

Spec §4.8. Blocked by 7.

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`misplaced_adhoc`, and
  `render`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (the report runs only
  without `--close`)
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Produces: `@dataclass(frozen=True) class MisplacedTask: task: IssueRef;
  reason: str`; `misplaced_adhoc(org: str) -> list[MisplacedTask]`: each
  open task under a live ad-hoc epic titled `Logbook:`, or whose body names
  an `epic`-labelled issue in another org, unless a trusted comment carries
  `Standalone-confirmed`. A cross-org ref whose labels cannot be read
  (private, deleted, an org the App cannot reach) is reported as its own
  candidate, with the reason `names <ref>, which could not be read`: never
  raised and never skipped. The report runs only in a human-run
  `vrg-epic-audit` without `--close`: the daily sweep (`--close` under
  `VRG_EPIC_SWEEP`) has no trusted-authors configuration and never runs it.

- [ ] **Step 1: Write the failing tests:** `test_misplaced_logbook_title`;
  `test_misplaced_cross_org_epic_ref`;
  `test_cross_org_ref_to_non_epic_not_reported`;
  `test_cross_org_ref_unreadable_reported`;
  `test_sweep_close_never_loads_trusted_authors` (in
  `test_vrg_epic_audit.py`: `--close` with `VRG_EPIC_SWEEP=1` neither
  renders the report nor calls `load_trusted_authors`);
  `test_standalone_confirmed_suppresses`;
  `test_standalone_confirmed_untrusted_ignored`;
  `test_misplaced_report_never_writes` (no move, close or label call).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**; the report is a section of `render`.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epic-audit
  --message "report ad-hoc tasks that look misplaced"`, then report-ready.

### Task 10: merge review's open tasks through a proxy (vergil-tooling)

Spec §4.3. Blocked by 9a.

**Files:**

- Modify: `src/vergil_tooling/lib/merge_review/request.py` (the task fetch
  behind `TaskContext.epic_open_tasks`)
- Test: `tests/vergil_tooling/test_merge_review_request.py`

**Interfaces:**

- Consumes: `epics.is_proxy`, `proxies.primary_of` (Task 7),
  `epic_audit.epic_children` (Task 9a).
- Produces: no new field. `epic_open_tasks` for a task whose native parent
  is a proxy is the primary's open children (its own and its proxies'),
  less the task itself. Design sources are not resolved here: the skill
  fetches them before it builds the request (Task 13b).

- [ ] **Step 1: Write the failing tests:**
  `test_proxy_task_open_tasks_from_primary`;
  `test_proxy_task_unresolvable_primary_raises`;
  `test_plain_epic_task_unchanged`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope merge-review
  --message "resolve a proxy task's epic context through its primary"`,
  then report-ready.

### Task 11: the cross-org advisory move (vergil-tooling)

Spec §4.6.

**Files:**

- Modify: `src/vergil_tooling/lib/security_records.py` (`retarget`, a
  `_move` step, `classify`, `Classification`, `records_pass`)
- Modify: `src/vergil_tooling/bin/vrg_security_record.py` (the retarget
  output for a move)
- Modify: `docs/site/docs/guides/security-records.md` (moves)
- Test: `tests/vergil_tooling/test_security_records.py`,
  `tests/vergil_tooling/test_vrg_security_record.py`

**Interfaces:**

- Produces: `moved_from_line(ghsa_id: str) -> str`; `moved_to_line(home:
  str, ghsa_id: str) -> str`; `Classification.moved: list[Advisory]`.
  `retarget(...)` keeps its signature; when every target is in another org
  it moves, returning the new advisory's ID among its output.

- [ ] **Step 1: Write the failing tests:**
  `test_move_creates_in_fix_org_with_moved_from`;
  `test_move_writes_moved_to_then_closes_original`;
  `test_move_rerun_reuses_moved_from`;
  `test_move_ignores_external_report_with_moved_from` (a triage-state or
  submitted advisory carrying the line is never reused; the move refuses
  and prints it);
  `test_move_two_moved_from_refused`;
  `test_move_moved_from_not_alone_ignored`;
  `test_move_keeps_drop_rule` (another open tracked issue not in `--drop`:
  refused before any write, the command printed);
  `test_targets_span_orgs_refused` (the refusal names both ways out);
  `test_classify_moved_not_broken`;
  `test_sync_reports_moved_target_missing`;
  `test_same_org_retarget_unchanged`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement** in the order of spec §4.6's three steps; every
  check runs before the first write. Step 1 (reuse) takes a candidate only
  when it is in `draft` state, is not external (`Advisory.external` false),
  and carries a `Moved from:` line naming exactly this ID, alone on its
  line. Any other match, or more than one candidate, refuses the move and
  prints the candidates; it is never reused.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope security-record
  --message "move an advisory to its fix's org on a cross-org retarget"`,
  then report-ready.

### Task 12: vergil-tooling release for proxies (deployment, .github)

Blocked by 8, 9a, 9b, 9c, 10 and 11.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 7 to 11 is published and installed on the host and the VMs, its
  version named in a comment.
- **Procedure (agent-safe):**
  1. Confirm the release notes list Tasks 7 to 11's PRs.
  2. `vrg-epic-create --help` lists `--proxy-for`.
  3. `vrg-epic-audit --epic vergil-project/.github#1417` exits 0 and prints
     the epic's children.
  4. The org's tokens cover both orgs the migration writes in
     (vergil-project and logical-minds-foundry): a read of
     logical-minds-foundry/.github#319 and of a vergil-project issue each
     succeed under their own `target_org`.
- **SUCCESS:** every check passes, recorded with the version.

### Task 13a: `epic-implement` and `epic-retrospective` for proxies (vergil-claude-plugin)

Spec §4.3, §4.4. Blocked by 12.

- **Files:** Modify `skills/epic-implement/SKILL.md`,
  `skills/epic-retrospective/SKILL.md`, `skills/issue-implement/SKILL.md`
  ("Documentation left for the sweep").
- [ ] **Step 1:** `epic-implement` builds its frontier from
  `vrg-epic-audit --epic <primary>`'s `children`, which include proxy
  tasks, and records plan evolution for them in the primary's plan.
  Preflight refuses an `epic` + `proxy` issue: `<ref> is a proxy for
  <primary ref>; drive the primary epic instead.`
- [ ] **Step 2:** `epic-retrospective`'s gate is unchanged (the children
  include proxy tasks); its refusal names the proxy of each open proxy
  task, and an exit 1 from `vrg-epic-audit --epic` (link drift) stops it.
- [ ] **Step 3:** The documentation-review sweep (`epic-implement` §6): a
  per-repo doc task for a repo in another org is filed with `--epic <that
  org's proxy ref>` (created with `--proxy-for` if none exists), never
  under the primary and never ad-hoc. `issue-implement`'s "Documentation
  left for the sweep" records a proxy task's items on the **primary's**
  documentation-review bookend.
- [ ] **Step 4:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope epic-implement
  --message "drive proxy tasks from the primary epic"`, then report-ready.

### Task 13b: design sources through a proxy (vergil-claude-plugin)

Spec §4.3. Blocked by 12.

- **Files:** Modify `skills/merge-review/SKILL.md` ("Before the review",
  steps 1 and 2).
- [ ] **Step 1:** In step 1, when the task's native parent carries `proxy`,
  resolve the primary with `vrg-epic-audit --epic <proxy ref>` and take its
  `primary` key (Task 9a). Step 2 then reads the primary's Documents
  section for the design sources, before the request is built, as today.
  An audit that exits non-zero, a missing `primary` key, or a primary whose
  Documents section lists no design source raises a new stop item, `D2`
  ("a proxy task's design sources cannot be resolved from its primary"),
  through `vrg-ruling` like the skill's other stops. Its only replies are
  `retry` (the link is fixed; resolve again) and `stop`; it has no
  `proceed-without`, so a proxy task is never reviewed against nothing.
  The open tasks need no skill change (Task 10).
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type feat --scope merge-review
  --message "take a proxy task's design sources from its primary"`, then
  report-ready.

### Task 13c: placement in `epic-create` and `epic-review` (vergil-claude-plugin)

Spec §4.6. Blocked by 12.

- **Files:** Modify `skills/epic-create/SKILL.md` (the cross-org note in
  Notes, step 8, the delivery table rule),
  `skills/epic-review/SKILL.md` (the one-off route, deferred revival).
- [ ] **Step 1:** `epic-create`: the cross-org note becomes the proxy
  convention; step 8 creates a proxy with `vrg-epic-create --proxy-for`
  when the plan files a task in another org, files it with `--epic <proxy
  ref>`, and lists proxy tasks with full refs; step 8's security retarget
  notes that a cross-org `--to` moves the advisory and that fixes spanning
  orgs are refused, naming the two ways out. The bookend convention (the
  documentation-review task's sweep) says a doc task for a repo in another
  org is filed under that org's proxy.
- [ ] **Step 2:** `epic-review`: a one-off for work that belongs to another
  org's epic goes to that epic's proxy, never ad-hoc; revival of an item
  deferred from ad-hoc checks for this first.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type feat --scope epic-create
  --message "place cross-org work under proxy epics"`, then report-ready.

### Task 14: vergil-claude-plugin release for proxies (deployment, .github)

Blocked by 13a, 13b and 13c.

- **Precondition (human-attested):** the vergil-claude-plugin release
  carrying Tasks 13a to 13c is published and installed, its version named
  in a comment.
- **Procedure (agent-safe):** confirm the release notes list those PRs, and
  that the installed `epic-implement` skill carries the proxy refusal and
  the installed `merge-review` skill resolves a proxy's primary.
- **SUCCESS:** both confirmed, recorded with the version.

### Task 15: the migration (deployment, .github)

Spec §4.7. Blocked by 12 and 14.

- **Precondition:** Tasks 12 and 14 closed with SUCCESS.
- **Procedure (agent-safe, the candidate list confirmed by the human
  before step 3):**
  1. For each org Vergil manages, list every open task under its ad-hoc and
     deferred epics (`vrg-queue adhoc --json`, `vrg-queue deferred
     --json`), and the misplacement report of `vrg-epic-audit` (Task 9c).
  2. Choose the candidates: the report's for ad-hoc tasks, plus a reading of
     each listed ad-hoc and deferred task's body and comments for a tie to
     an epic in another org, and each `Logbook:` task. Present the list,
     with what the reading found for each, and the human confirms it. The
     confirmed list is recorded in a comment on this task.
  3. `vrg-epic-create --proxy-for logical-minds-foundry/.github#319 --repo
     vergil-project/vergil-tooling`, and likewise for any other primary
     the list names.
  4. Each deferred candidate (#3188 and #3189 among them): carry its
     deferral reason into a comment, then `vrg-intake-route <ref> one-off
     --repo <repo> --epic <proxy ref>`. Its order relative to its
     prerequisites is the primary's plan's (#319's plan orders REG after
     OP1), recorded by step 6's evolution entry. Each ad-hoc candidate: `vrg-epic-move --task
     <ref> --epic <proxy ref>` (or `--epic hoshin` for a `Logbook:` task).
  5. Each standalone task the report flagged gets a `Standalone-confirmed`
     comment, on the human's ruling.
  6. For each primary that gained proxy tasks (#319 for #3188 and #3189),
     file a task under that primary, in its own org, as a native sub-issue:
     "Record the re-homed tasks in the plan's evolution log". Its PR appends
     an `## Evolution during execution` entry to the primary's `plan.md`
     naming each re-homed task by full ref and its place in the order (for
     #319: REG = vergil-project/vergil-tooling#3188, TRUST-SYNC =
     vergil-project/vergil-tooling#3189). The plan's seeding is the
     approval to file it; its own epic's driver works it.
- **SUCCESS:** every confirmed candidate re-homed and each evolution-entry
  task filed, recorded with the refs and the proxies created.

### Task 16: no misplaced task remains (validation, .github)

Spec §4.7. Blocked by 15.

- **Procedure:**
  1. Each evolution-entry task Task 15 recorded is filed as a native
     sub-issue of its primary. When it lands is the primary's business, not
     checked here. Every candidate on Task 15's confirmed list is under its
     proxy or the hoshin epic.
  2. `vrg-epic-audit` for each swept org: no misplaced ad-hoc task is
     reported.
  3. `vrg-epic-audit --epic logical-minds-foundry/.github#319` exits 0 and
     lists #3188 and #3189 among its children; the same for each other
     primary from Task 15.
  4. `vrg-queue adhoc` does not list #3188 or #3189.
  5. A sweep preview (`vrg-epic-audit` without `--close`) does not plan to
     close any proxy whose primary is open.
- **SUCCESS:** every check passes.
