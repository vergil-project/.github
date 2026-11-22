# trusted-content-only — implementation plan

- **Epic:** vergil-project/.github#1467
- **Planning issue:** vergil-project/.github#579 (review history is in its comments)
- **Status:** approved

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** every read of GitHub content that drives Vergil's behaviour goes
through one trust layer in vergil-tooling and acts only on content from the
`human`, `agents` or per-org `automation` roles of the human's Vergil
configuration; untrusted content stops the item it is load-bearing for and is
otherwise reported, and a guard test and an agent hook keep new raw reads out.

**Architecture:**

- **vergil-tooling** does all of it but the skills: the configuration (roles
  and the Vergil org list), the trust layer, the publish and check commands,
  every migrated site, the trusted read command, the outsider report, the
  guard test and the hook blocklist.
- **vergil-actions** passes the delivered configuration into the jobs that
  read content, and stops treating a fork's `release/*` head as a release.
- **vergil-containers** turns its lifecycle job off; its re-architecture
  (one PR per run, through the trust layer) is vergil-project/.github#1128.
- **vergil-claude-plugin** switches every skill that reads issue or PR
  content to the trusted read command, and adds the outsider report and the
  configuration check to `hoshin start`.
- **Operational tasks:** an inventory of content reads outside
  vergil-tooling; a first vergil-tooling release (with the vergil-actions
  tag and the org variables published); a plugin release; a second
  vergil-tooling release carrying the hook; fixture runs; and a validation
  on real data.

**Tech Stack:** Python 3 and pytest (vergil-tooling), GitHub Actions YAML
(vergil-actions), Python (vergil-containers), Markdown skills
(vergil-claude-plugin).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Start.** No implementation task starts before epic
  vergil-project/.github#1440 is closed; this plan assumes all of it.
  Tasks 2 and 3a carry `Blocked-by: vergil-project/.github#1442`, #1440's
  retrospective, and every other task chains from them. The one
  exception is Task 5b, which uses nothing of #1440 or of the trust
  layer and has no blocker (spec §3).
- **One definition.** Every trust decision goes through `lib/trust.py`
  (Task 3b); the roles come only from `trusted_authors.load_trusted_authors()`
  (Task 3a). No module compares a login with a list itself.
- **Roles and types** (spec §5.1): `human` must be a GraphQL `User`;
  `agents` and `automation` must be a `Bot`. A login is normalised by
  stripping a leading `app/` and a trailing `[bot]`. A role match with the
  wrong type is never trusted: it is reported as skipped (cause
  `wrong-type`) on a comment, review, inline review comment or comment
  editor, and a stop only where the content is load-bearing (the body under
  `require_body`, a close or reopen acted on, the author of an item a tool
  targets). An `automation` identity is trusted only in repos of its own
  org.
- **Deleted accounts** (spec §5.1): a `null` author, editor or actor, or the
  `ghost` login, is untrusted, never a read failure.
- **Fail loudly** (spec §3): an API error, a missing field or a truncated
  connection (`pageInfo.hasNextPage` true where the reader does not page)
  raises `TrustReadError` naming the call. Never "trusted", "absent" or
  "nothing found".
- **Metadata only** (spec §5.3): stop errors and skip reports print repo,
  number, kind, login, time and field, never untrusted text. Under
  `GITHUB_ACTIONS=true`, batch runs print counts by cause, listing only
  read-failure stops by repo and number.
- **Batch runs** stop the item, not the run, and exit non-zero at the end
  when any item stopped. An outsider-opened issue a batch run merely
  encounters is reported with a clean exit.
- **Content reads live in vergil-tooling** (spec §3): Python automation calls
  `gh` and `git` directly with its own credentials; it never calls `vrg-gh`,
  `vrg-git` or the trusted read command.
- **Proportion** (spec §3): the simplest mechanism that protects; nothing
  for cases that cannot occur today.
- **Public wording:** issues, commits, PR text and docs describe mechanisms
  only and name findings by GHSA ID alone.
- **Fixtures** copy the field shapes of real recorded GitHub responses, and
  synthetic spam keeps the shape (structure, marker placement, link
  patterns) with neutralised URLs, never real text (spec §6.6).
- **Tests and validation** run through `vrg-container-run`; a task is done
  when `vrg-container-run -- vrg-validate` passes.
- **Site docs:** each tooling task updates the docs for what it changes
  (`docs/site/docs/reference/cli-tools-overview.md` and the reference page
  of each command it touches); the documentation review sweeps the rest.
- **Site tests assert the report or the stop** (spec §6.6): every site
  test with untrusted input asserts the metadata-only skip report on
  stderr (or the stop), and that the fixture's untrusted text appears
  nowhere in the output; a test that only checks the content was ignored
  is not enough. A site reader that returns a plain value (`parent_of`,
  `blockers_of`, `child_states` and the like) passes its skips to
  `trust.report_skipped`, and each site task says so for its readers.
- **Refactor step:** every code task ends with tests green, then removes the
  duplication it introduced (each replaced login comparison), then re-runs
  the tests.
- **Re-check first:** every site task first re-checks its sites against
  current `develop` (some are fixed, removed or moved since the audit, and
  #1440 changed the release ones), and records in its PR notes each site it
  found fixed or gone.
- **Fixture runs:** the epic changes release tooling and a shared CI
  workflow, which every language fixture uses, so it seeds a fixture run for
  all six fixtures (Task 19).

## Review Focus

1. **An edit history or comment list longer than one page.** The reader
   pages through all of it, and never judges the first page alone. Pinned
   in Task 3b (`test_read_issue_pages_all_comments`,
   `test_read_issue_pages_all_edits`).
2. **The same App author in each form GitHub returns** (`app/x`, `x`,
   `x[bot]`) in GraphQL, the layer's only source. Each maps to its role.
   Pinned in
   Task 3b (`test_role_of_app_login_forms`).
3. **An automation comment in an org that is not its own, or in an org with
   no stanza.** Untrusted and reported. Pinned in Task 3b
   (`test_automation_untrusted_in_other_org`,
   `test_unlisted_org_has_no_automation_role`).
4. **A same-repo PR and a fork PR on one branch name.** The same-repo PR is
   returned, the fork reported. Pinned in Task 3c
   (`test_prs_for_branch_skips_fork_and_reports`).
5. **A job under Actions with the delivered configuration missing.** The
   loader stops before any write. Pinned in Task 3a
   (`test_load_under_actions_without_env_raises`).

---

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

### Foundation

- **1** (.github, task #1468): spec and plan, this PR.
- **2** (.github, validation): inventory every script and workflow step
  outside vergil-tooling that reads GitHub content, filing a task for each
  one that reads behaviour-driving content; and inventory every `vrg-gh`
  read in the plugin's skills and agents for Tasks 16, 17a and 17b.
  Blocked by #1442.
- **3a** (vergil-tooling): the `[orgs.<org>]` configuration, roles in the
  loader, VM delivery, the Actions loader. Blocked by #1442.
- **3b** (vergil-tooling): `lib/trust.py`: identities, roles, issue bodies
  and comments, errors, skip reporting. Blocked by 3a.
- **3c** (vergil-tooling): close and reopen events, branch PR lookup,
  reviews and inline review comments. Blocked by 3b.
- **3d** (vergil-tooling): #1440's identity rule, trusted closing references
  and record rule onto the layer with #1440's parameters. Blocked by 3c.
- **4** (vergil-tooling): `vrg-trusted-authors publish` and `check`.
  Blocked by 3a.
- **5a** (vergil-actions): pass `VRG_TRUSTED_AUTHORS` into every job that
  runs a GitHub-touching `vrg-*` command. Blocked by 3a.
- **5c** (vergil-actions): the daily config audit runs `vrg-trusted-authors
  check` as the automation App. Blocked by 4.
- **5b** (vergil-actions): a fork's `release/*` head never gets the release
  matrix. No blocker.

### Sites

- **6a** (vergil-tooling): release PR lookups and the deps-gate branch
  delete. Blocked by 3c and 3d.
- **6b** (vergil-tooling): finalize, worktree and tracking-issue resolver
  lookups. Blocked by 3c.
- **6c** (vergil-tooling): CI evidence's PR choice and check-run matching.
  Blocked by 3c.
- **7a** (vergil-tooling): the queue views and intake routing's
  planning-epic guard. Blocked by 3b.
- **7b** (vergil-tooling): the operational-outcome audit and the
  security-record comment dedup. Blocked by 3b.
- **7c** (vergil-tooling): the remaining callers of the old login-only
  helpers (proxies, planning close, rulings, scope changes, merge-review
  request and open-PR reader, the release's trusted logins, the sizing
  record), and the old helpers removed. Blocked by 3c, 3d, 7a and 7b.
- **8a** (vergil-tooling): body parents, child search, blockers and the
  rollup's closer check. Blocked by 3c.
- **8b** (vergil-tooling): the sweep's body parents, nativising and drift
  closes. Blocked by 3d and 8a.
- **8c** (vergil-tooling): security-record body lines, `show`, resume and
  retarget. Blocked by 3b.
- **8d** (vergil-tooling): the open-task check, the release marker test,
  the submit linkage and the relay-ref payload. Blocked by 8a.
- **9** (vergil-tooling): the dependency-update fix task and the
  release-start gate. Blocked by 3c.
- **10** (vergil-tooling): PR provenance roles and the inline review-comment
  reader. Blocked by 3c.
- **11** (vergil-tooling): outsider-opened issues are never attached,
  routed, sized or published. Blocked by 3b.
- **12** (vergil-containers): the lifecycle job disabled until
  vergil-project/.github#1128. Blocked by #1442.

### Commands, enforcement, skills

- **13** (vergil-tooling): the trusted read command `vrg-issue-read`.
  Blocked by 3c.
- **14** (vergil-tooling): the outsider report. Blocked by 3c.
- **15** (vergil-tooling): the guard test. Blocked by 6a to 11 (7c
  included), 13, 14 and any task Task 2 files.
- **16** (vergil-tooling): the hook blocklist. Blocked by 20, so it cannot
  merge, and no routine release can carry it, until the migrated skills are
  installed.
- **17a** (vergil-claude-plugin): `issue-implement`, `epic-implement`,
  `issue-validate`, `issue-deploy`, `pr-watch`, `merge-review`, and every
  other body or comment read Task 2 assigns here. Blocked by 18.
- **17b** (vergil-claude-plugin): `epic-review`, `epic-create`,
  `epic-retrospective`, `deprecation-triage`. Blocked by 18.
- **17c** (vergil-claude-plugin): `hoshin start` runs the outsider report
  and shows the latest configuration-audit result. Blocked by 18.

### Operational

- **18** (.github, deployment): the human adds the `[orgs.<org>]` stanzas
  and grants each automation App read-only Variables permission; the
  vergil-actions tag with 5a to 5c; the vergil-tooling release with 3a to 15
  (all but 12), installed; `vrg-trusted-authors publish` for each org; Task
  19 filed. Blocked by 3a to 15 (all but 12), 5a to 5c and any task Task 2
  files.
- **19** (fixture runs, all six fixtures): filed with `fixture-run` by Task
  18, `--after` Task 18.
- **20** (.github, deployment): the vergil-claude-plugin release with 17a to
  17c, installed. Blocked by 17a, 17b and 17c.
- **21** (.github, deployment): the vergil-tooling release with 16,
  installed. Blocked by 16 and 20.
- **22** (.github, validation): the layer on real data. Blocked by 21, 12
  and the six fixture runs' validate tasks.

### Bookends

- **Documentation review** (vergil-tooling, #4887). Blocked by every task
  above but 19 and 22. Its sweep covers the docs repo and the plugin's
  skill docs, and spawns a per-repo task for each.
- **Retrospective** (.github, #1469). Blocked by everything else.
- **Follow-on:** vergil-project/.github#892, planned right after this plan's
  docs PR. Not a task of this epic.

Notes:

- **Merged is not deployed.** The skills (17a to 17c) call `vrg-issue-read`
  and the report, so they wait for Task 18's release. The hook (16) refuses
  raw reads, so it is blocked by Task 20: it cannot merge until the skills
  that stop needing them are installed, and Task 21's release is the first
  that can carry it.
- **Actions first.** Task 18 tags vergil-actions with 5a before the
  vergil-tooling cut, so every job that runs the new tooling already
  receives `VRG_TRUSTED_AUTHORS`; it then publishes the variable right after
  installing. A scheduled job in that gap stops loudly and is re-run.
- **Filing.** Every task above but 19 is filed at epic-create's filing step,
  in its repo, linked under #1467, with its `Blocked-by:` refs. The security
  advisories tracking the planning issue are retargeted then, one at a time,
  all in vergil-project:
  - GHSA-jj85-qr28-7jwh to Tasks 3b, 15 and 16;
  - GHSA-r4j2-cwcp-xg56 to Tasks 5b, 6a, 6b, 6c and 12;
  - GHSA-hrq4-7qg2-7xxf to Tasks 7a, 7b and 7c;
  - GHSA-w5hq-5p7v-qgc8 to Tasks 8a, 8b, 8c and 8d;
  - GHSA-777j-j57r-92ff to Tasks 8a and 9;
  - GHSA-q6hx-8c85-9f56 to Tasks 6c and 10;
  - GHSA-crcw-h2ff-qgr5 to Tasks 13, 17a and 17b;
  - GHSA-h24x-mm5p-j8vg to Tasks 3b and 7c;
  - GHSA-f7mp-c8cv-mxjv and GHSA-rcf4-699q-jgpr to Task 3b;
  - GHSA-3qxq-x48m-v8jj to Task 11;
  - GHSA-363j-9x97-29fm to Task 16;
  - GHSA-5mvv-5h3p-64m3 to Tasks 3a, 4 and 5c;
  - GHSA-578x-r7xr-gcx8 to Tasks 2 and 12;
  - GHSA-f4jm-rjc4-vv6c to Tasks 3b and 7c;
  - GHSA-q488-wcfh-87rg to Task 15;
  - GHSA-mf2v-j2rw-7r72 to Task 3b;
  - GHSA-m299-r8x7-m947 to Task 15;
  - GHSA-q98q-xp48-p9vj to Tasks 3b and 7c;
  - GHSA-q736-hg2x-m5hg to Tasks 4, 5c and 17c;
  - GHSA-g63x-36x6-4c9c to Task 3b;
  - GHSA-vmpw-wqxx-r7wj to Task 16;
  - GHSA-m3j7-p7h2-h5hf to Task 5b;
  - GHSA-qp8h-4r9w-vv37 to Tasks 3b and 3c;
  - GHSA-p5wm-892c-jw6j to Task 15;
  - GHSA-mfqq-jvrp-94q5 to Tasks 3b and 3d;
  - GHSA-g5qg-42hw-q5cw to Task 2;
  - GHSA-8x97-j643-qxv6 to Task 8b.

  A task Task 2 files later joins the advisory it fixes by a retarget from
  one of that advisory's tasks, with the complete set as `--to`.

---

## Task 1: spec and plan (.github, #1468)

This PR. Done when it merges.

## Task 2: inventory of content reads outside vergil-tooling (validation, .github)

Spec §3, §6.2 ("Outside vergil-tooling").

- **Preconditions:** #1440 closed.
- **Procedure:** at `origin/develop` of every repo of every org listed in
  the configuration (enumerated when the task runs; today including
  vergil-actions, vergil-containers, vergil-claude-plugin, vergil-vm,
  vergil-python, packages, docs, `.github` and the six fixture repos),
  except vergil-tooling itself, list every
  `gh` call, `gh api` path, `actions/github-script` step and script that
  reads issue or PR bodies, comments, reviews, titles, branch names or
  events. For each, record repo, file and line, what it reads, and whether
  the result drives behaviour. Then, in vergil-claude-plugin `skills/` and
  `agents/`, list every `vrg-gh` read and its `--json` fields or API path,
  classed as a body or comment read or a display read.
- **Acceptance:**
  - Every behaviour-driving read outside vergil-tooling is either the
    lifecycle publisher (Task 12) or has a task filed under #1467 moving it
    into vergil-tooling (`vrg-issue-create --epic
    vergil-project/.github#1467`, Blocked-by 3c), with its vergil-actions or
    vergil-containers counterpart where one is needed. Each filed task is
    added as a `Blocked-by:` of Task 15 (its tooling side) and Task 18 (it
    and its counterpart), and recorded in this plan under "Evolution during
    execution".
  - Every skill body or comment read is assigned to Task 17a or 17b (each
    issue's body is edited to list them).
  - The comment lists each read found, by file and line, with no content
    quoted. `Outcome: SUCCESS`.

## Task 3a: roles and the Vergil org list in the configuration (vergil-tooling)

Spec §5.1, §5.1a.

**Files:**

- Modify: `src/vergil_tooling/lib/trusted_authors.py`
- Modify: `src/vergil_tooling/lib/identity.py`
  (`_load_trusted_authors_stanza` passes the `[orgs.*]` tables to
  `parse_trusted_authors`, so `vrg-vm`'s identity config carries them)
- Modify: `src/vergil_tooling/lib/vm_guest.py` (`inject_trusted_authors`
  renders the orgs)
- Modify: `src/vergil_tooling/bin/vrg_trusted_authors.py` (prints
  `org: <org> automation: <slug>` lines; `--json` adds `orgs`)
- Modify: `src/vergil_tooling/lib/automation_identity.py`
  (`automation_app_slug(org)`: the job's `$VRG_AUTOMATION_APP` only when
  *org* is the job's own org (`$GITHUB_REPOSITORY_OWNER`), else the
  configuration; never the API)
- Test: `tests/vergil_tooling/test_trusted_authors.py`,
  `tests/vergil_tooling/test_automation_identity.py`,
  `tests/vergil_tooling/test_identity.py`

**Interfaces:**

- Produces: `OrgConfig(automation_app: str)`;
  `TrustedAuthors(human: tuple[str, ...], agents: tuple[str, ...], orgs:
  Mapping[str, OrgConfig])`; `parse_trusted_authors(raw: Mapping[str,
  object], orgs: Mapping[str, object]) -> TrustedAuthors` (rejects an
  `app/` or `[bot]` suffix on any value, an org with no `automation_app`,
  a login in two lists, and a configuration with no org at all);
  `load_trusted_authors() -> TrustedAuthors`:
  the VM file, else the host `identities.toml` (`[trusted_authors]` plus
  `[orgs.*]`), else, only when `GITHUB_ACTIONS == "true"` and neither file
  exists, `$VRG_TRUSTED_AUTHORS` (compact JSON: `{"human": [...], "agents":
  [...], "orgs": {"<org>": {"automation_app": "<slug>"}}}`); otherwise
  `TrustedAuthorsNotConfiguredError`. `vergil_orgs(authors) -> tuple[str,
  ...]`. `render_vm_file(authors)` writes `[orgs.<org>]` tables.
  `render_actions_value(authors) -> str`, the JSON carrying every listed
  org's `automation_app`, so a job reading another org's content (a proxy's
  primary) judges it as the host does.

- [ ] **Step 1: Write the failing tests:**
  `test_parse_orgs_stanza`; `test_parse_rejects_org_without_automation_app`;
  `test_parse_rejects_bot_suffix`; `test_parse_rejects_no_orgs` (a
  `ValueError` naming the file); `test_vm_file_round_trips_orgs`;
  `test_load_config_carries_orgs` (`identity.load_config` on an
  `identities.toml` with `[orgs.x]` yields `TrustedAuthors.orgs`);
  `test_automation_app_slug_env_first`;
  `test_automation_app_slug_other_org_ignores_env`;
  `test_load_under_actions_reads_env` (no files, `GITHUB_ACTIONS=true`,
  valid env); `test_load_under_actions_without_env_raises`;
  `test_load_env_ignored_when_file_exists`;
  `test_load_env_ignored_outside_actions`;
  `test_render_actions_value_has_every_org`;
  `test_automation_app_slug_reads_config`;
  `test_automation_app_slug_unlisted_org_raises`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** The `ValueError` messages name the file and
  the key, as today.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope trust --message
  "add the Vergil org list and per-org automation identities to the trusted
  authors"`, then report-ready. The PR notes give the human the stanza to add
  to `identities.toml` before Task 18.

## Task 3b: the trust layer — identities, issues and comments (vergil-tooling)

Spec §5.1 to §5.3, §6.1.

**Files:**

- Create: `src/vergil_tooling/lib/trust.py`
- Test: `tests/vergil_tooling/test_trust.py` (new)

The old login-only helpers (`trusted_authors.trusted_comment_bodies`,
`trusted_comment_lines`, `rulings/comments.trusted`,
`epic_size.trusted_logins`) are **not** wrapped: they cannot see account
types or editors. Each site task moves its callers onto this layer, and
Task 7c moves the rest and deletes the helpers.

**Interfaces:**

- Consumes: Task 3a's `TrustedAuthors`, `load_trusted_authors`.
- Produces:
  - `class Role(StrEnum)`: `HUMAN = "human"`, `AGENTS = "agents"`,
    `AUTOMATION = "automation"`; `ALL_ROLES: frozenset[Role]`.
  - `Identity(login: str | None, typename: str | None)`; `None` login is a
    deleted account.
  - `Judgement(role: Role | None, cause: str | None)`; `role_of(identity:
    Identity, *, org: str, authors: TrustedAuthors) -> Judgement`: `role`
    is `None` for every untrusted identity, so no caller can mistake one
    for a role; `cause` is `untrusted`, `deleted` or `wrong-type` (a role
    match with the wrong type, or a login with no `typename`). Readers
    report the cause on `Skipped` for a comment, review or editor, and
    raise `UntrustedContentError` where the content is load-bearing.
  - Every read the layer makes is GraphQL with `__typename` on each actor;
    the layer reads no REST user object and no `gh ... --json author`.
  - `EditorRule = Literal["every", "last"]`.
  - `Skipped(repo: str, number: int, kind: str, login: str | None, at: str,
    field: str)`, with `kind` one of `issue`, `pr`, `comment`, `review`,
    `review-comment`, `edit`, `event`.
  - `UntrustedContentError(item: Skipped, reason: str)` and
    `TrustReadError(call: str, detail: str)`; both messages metadata only.
  - `TrustedComment(id: str, body: str, role: Role, at: str)`;
    `IssueContent(body: str | None, body_role: Role | None, author_role:
    Role | None, comments: tuple[TrustedComment, ...], skipped:
    tuple[Skipped, ...])`.
  - `read_issue(repo: str, number: int, *, roles: frozenset[Role] =
    ALL_ROLES, editors: EditorRule = "every", require_body: bool = False,
    authors: TrustedAuthors | None = None) -> IssueContent`: a paged
    GraphQL read, through `repository.issueOrPullRequest(number:)` so it
    reads an issue or a PR alike, of the author, `userContentEdits`
    editors, `lastEditedAt`,
    comments with their authors and editors, each with `__typename`. It
    pages the comments connection and every nested `userContentEdits` until
    `hasNextPage` is false (as `rulings/comments.fetch_comments` pages
    today), and raises `TrustReadError` only on an API error or a malformed
    page. `require_body=True` raises `UntrustedContentError` when the body
    is untrusted.
  - `judge_comments(comments: Sequence[Mapping[str, object]], *, repo: str,
    number: int, roles=ALL_ROLES, editors: EditorRule = "every",
    authors) -> tuple[list[TrustedComment], list[Skipped]]` for callers
    that already fetched comments with the fields `COMMENT_FIELDS`
    names. It checks the input is complete: every field
    `COMMENT_FIELDS` names is present on each node (`__typename` on
    every actor) and every connection carries `pageInfo` with
    `hasNextPage` false; otherwise it raises `TrustReadError`. Callers
    that can use `read_issue` do.
  - `COMMENT_FIELDS: str`, the GraphQL selection a caller must request.
  - `report_skipped(skipped: Sequence[Skipped], *, stream: TextIO) ->
    None` and `BatchStops` (`add(item, cause)`, `exit_code() -> int`,
    `print_summary(stream)`), which under `GITHUB_ACTIONS=true` print counts
    by cause and list read-failure stops by repo and number only.

- [ ] **Step 1: Write the failing tests:** `test_role_of_human_user`;
  `test_role_of_app_login_forms` (`app/x`, `x`, `x[bot]` with `Bot` all
  map to `agents`); `test_role_of_user_matching_app_slug_is_wrong_type`;
  `test_role_of_login_without_typename_is_wrong_type`;
  `test_judgement_untrusted_role_is_none` (no untrusted cause yields a
  truthy role); `test_read_issue_reads_pr` (a PR number, through
  `issueOrPullRequest`);
  `test_read_issue_comment_from_user_named_like_app_is_skipped`;
  `test_read_issue_body_from_user_named_like_app_require_body_raises`;
  `test_role_of_deleted_is_untrusted`; `test_role_of_ghost_is_untrusted`;
  `test_automation_untrusted_in_other_org`;
  `test_unlisted_org_has_no_automation_role`;
  `test_read_issue_body_untrusted_after_outsider_edit`;
  `test_read_issue_last_editor_rule`; `test_read_issue_require_body_raises`;
  `test_read_issue_skips_spam_comment_and_reports`;
  `test_read_issue_roles_parameter_narrows`;
  `test_read_issue_pages_all_comments`;
  `test_read_issue_pages_all_edits`;
  `test_read_issue_malformed_page_raises`;
  `test_read_issue_api_error_raises`;
  `test_judge_comments_unpaged_edits_raises`;
  `test_judge_comments_missing_typename_raises`;
  `test_read_issue_body_edited_by_deleted_account_is_untrusted`;
  `test_read_issue_comment_edited_by_deleted_account_is_skipped`;
  `test_errors_carry_no_untrusted_text`;
  `test_batch_summary_counts_only_under_actions`;
  `test_batch_summary_lists_read_failures_under_actions`. Fixtures: recorded
  response shapes for a User, an App in each login form, a deleted account
  and a synthetic spam comment.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope trust --message
  "add the trust layer for issue bodies and comments"`, then report-ready.

## Task 3c: events, branch PR lookup and reviews (vergil-tooling)

Spec §5.2, §6.1.

**Files:**

- Modify: `src/vergil_tooling/lib/trust.py`
- Modify: `src/vergil_tooling/lib/mechanized_pr.py` (`_same_repo` becomes
  `trust.is_same_repo`)
- Test: `tests/vergil_tooling/test_trust.py`,
  `tests/vergil_tooling/test_mechanized_pr.py`

**Interfaces:**

- Consumes: Task 3b.
- Produces: `IssueEvent(kind: str, actor: Identity, at: str)`, `kind`
  one of `closed`, `reopened`, `labeled`, `unlabeled`, `mentioned`,
  `cross-referenced`, `subscribed` and the other timeline kinds the
  release guard warns on today; `read_events(repo: str, number: int) ->
  tuple[IssueEvent, ...]` (paged); `close_judgement(repo: str, number:
  int, *, roles=ALL_ROLES) -> Judgement` (the latest close's actor, so a
  caller or #892 can require a role; callers that act on the close
  raise `UntrustedContentError` when `role` is `None`);
  `is_same_repo(repo: str, item: Mapping[str, object]) -> bool`
  (`isCrossRepository` false and `headRepository` owner/name equal to
  *repo*); `PRRef(number: int, url: str, title: str, head: str,
  head_oid: str, author_role: Role, state: str)`, with `head` the branch
  name and `head_oid` the head commit;
  `prs_for_branch(repo: str, branch: str, *, state: Literal["open",
  "merged", "all"], roles=ALL_ROLES) -> tuple[list[PRRef],
  list[Skipped]]`, a GraphQL read; `PRReviewContent(reviews:
  tuple[TrustedComment, ...], review_comments: tuple[TrustedComment, ...],
  skipped: tuple[Skipped, ...])`; `read_reviews(repo: str, pr: int, *,
  roles=ALL_ROLES, editors: EditorRule = "every") -> PRReviewContent`.

- [ ] **Step 1: Write the failing tests:**
  `test_close_judgement_human`; `test_close_judgement_outsider_has_no_role`;
  `test_close_judgement_deleted_account_has_no_role`;
  `test_prs_for_branch_skips_fork_and_reports`;
  `test_prs_for_branch_only_fork_returns_empty_and_reports`;
  `test_prs_for_branch_untrusted_author_skipped`;
  `test_is_same_repo_checks_head_repository`;
  `test_read_reviews_skips_outsider_review`;
  `test_read_reviews_skips_edited_by_outsider`;
  `test_read_events_truncated_raises`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope trust --message
  "add trusted events, branch lookups and reviews to the trust layer"`, then
  report-ready.

## Task 3d: #1440's rules onto the layer (vergil-tooling)

Spec §6.1 ("Readers are parameterised").

**Files:**

- Modify: `src/vergil_tooling/lib/release/guard.py` (`read_identities`
  is replaced by Task 3b's `read_issue` and Task 3c's `read_events`;
  `check_tracking_issue` uses `trust.role_of` with `roles={Role.HUMAN}`,
  and keeps its warnings for the other event kinds)
- Modify: `src/vergil_tooling/lib/pending_release.py`
  (`trusted_closing_refs` takes `roles` and uses the layer; default
  `{Role.HUMAN, Role.AGENTS}`)
- Modify: `src/vergil_tooling/lib/release/records.py` (`read_comments` is
  replaced by `read_issue`; record authors: `roles={Role.HUMAN}`,
  `editors="last"`)
- Test: the existing tests of the three modules

**Interfaces:**

- Consumes: Tasks 3b and 3c.
- Produces: `trusted_closing_refs(pr: Mapping[str, object], *, repo: str,
  roles: frozenset[Role] = frozenset({Role.HUMAN, Role.AGENTS}), authors:
  TrustedAuthors) -> tuple[tuple[int, ...], list[Skipped]]`, which Task 8b
  reuses.

- [ ] **Step 1: Write the failing tests:**
  `test_tracking_guard_rejects_agent_with_layer`;
  `test_tracking_guard_rejects_user_named_like_app`;
  `test_closing_refs_exclude_automation_by_default`;
  `test_record_last_editor_rule_kept`;
  `test_tracking_guard_still_warns_on_foreign_label_and_mention`;
  `test_trusted_closing_refs_unpaged_raises`. Existing tests of the three modules
  change only to add `__typename` to their actors and to follow the new
  signatures and return types; each keeps the outcome it asserts.
- [ ] **Step 2:** Run them. Expect the new ones to FAIL.
- [ ] **Step 3: Implement.** No behaviour of #1440 changes beyond the
  account-type check.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type refactor --scope release
  --message "move the release identity rules onto the trust layer"`, then
  report-ready.

## Task 4: publish and check the Actions copies (vergil-tooling)

Spec §5.1a.

**Files:**

- Create: `src/vergil_tooling/lib/trust_publish.py`
- Modify: `src/vergil_tooling/bin/vrg_trusted_authors.py` (subcommands
  `publish [--org ORG]` and `check [--org ORG]`)
- Test: `tests/vergil_tooling/test_trust_publish.py` (new)

**Interfaces:**

- Consumes: Task 3a's `render_actions_value`, `vergil_orgs`.
- Produces: `ORG_VARIABLES = ("VRG_TRUSTED_AUTHORS", "VRG_AUTOMATION_APP")`;
  `publish(org: str, authors: TrustedAuthors) -> None` (sets both org
  variables, visibility `all`, refusing first if `check` finds a repo-level
  shadow); `check(org: str) -> CheckResult(findings: list[str], values:
  dict[str, str])`: one finding per missing or invalid org variable and per
  repository-level variable of either name in any repo of the org, and the
  variables' current values. `check` exits 1 with findings, 0 without, and
  with `--values-out PATH` writes the values as JSON (logins and slugs
  only) for the workflow to upload as the artifact
  `trusted-authors-values`.
  `publish` is a setup step the human runs. `check` is automation: it runs
  as the org's automation App (which Task 18 grants read-only Variables
  permission at org and repository level) and refuses the agent identity
  modes, which never hold that permission. `render [--org ORG]` prints the
  values the host file implies (`render_actions_value` and the org's
  `automation_app`), with no GitHub call, for `hoshin start`'s drift
  comparison (Task 17c).

- [ ] **Step 1: Write the failing tests:** `test_publish_sets_both_variables`;
  `test_publish_refuses_when_repo_shadow_exists`;
  `test_check_clean`; `test_check_flags_missing_or_invalid`;
  `test_check_flags_repo_level_shadow`;
  `test_check_writes_values_to_summary`;
  `test_check_api_error_raises`; `test_check_refuses_agent_mode`;
  `test_check_runs_as_automation`; `test_render_matches_publish`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope trust --message
  "publish and check the Actions copies of the trusted authors"`, then
  report-ready.

## Task 5a: deliver the configuration to content-reading jobs (vergil-actions)

Spec §5.1a.

**Files:**

- Modify: every workflow and composite action whose steps run a `vrg-*`
  command that touches GitHub, at least
  `.github/workflows/ops-epic-sweep.yml`,
  `.github/workflows/ops-epic-rollup.yml`,
  `.github/workflows/ops-upstream-refresh.yml`,
  `.github/workflows/ops-github-config.yml`,
  `.github/workflows/cd-docs.yml`, `.github/workflows/cd-docs-refresh.yml`
  and `actions/cd/release/ci-evidence-gate/action.yml` (the task lists every
  one in its PR notes)

- [ ] **Step 0: Write the failing test** (the yq-based workflow shell-test
  pattern of `actions/ops/github-config/tests/automation-env.test.sh`):
  `trusted-authors-env.test.sh` asserts every step that runs a
  GitHub-touching `vrg-*` command receives `VRG_TRUSTED_AUTHORS` from
  `vars`. Run it. Expect FAIL.
- [ ] **Step 1:** Deliver `VRG_TRUSTED_AUTHORS: ${{ vars.VRG_TRUSTED_AUTHORS
  }}` to each such step: once in `actions/shared/setup/vergil` if a
  composite action there can receive it (as an input its callers pass from
  `vars`), otherwise in each step's `env`. Do not set `VRG_AUTOMATION_APP`
  where the job does not run as the automation identity (that variable
  selects the identity mode).
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate` (actionlint).
  Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope ops --message "pass
  the trusted authors into the jobs that read content"`, then report-ready.

## Task 5c: the configuration check in the daily audit (vergil-actions)

Spec §5.1a.

**Files:**

- Modify: `.github/workflows/ops-github-config.yml` (a step that runs
  `vrg-trusted-authors check --org ${{ github.repository_owner }}` as the
  automation App, its token minted with read-only organization and
  repository Variables permission)

- [ ] **Step 0: Write the failing test** (the same shell-test pattern):
  the check step runs as the automation App and the artifact upload runs
  on failure too (`if: always()`). Run it. Expect FAIL.
- [ ] **Step 1:** Add the step; a finding fails the job the way the audit's
  other drift findings do, and the job uploads `check --values-out`'s file
  as the artifact `trusted-authors-values`, on success and on failure.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope ops --message "check
  the trusted-authors variables in the daily config audit"`, then
  report-ready.

## Task 5b: no release matrix for a fork's `release/*` head (vergil-actions)

**Files:**

- Modify: `.github/workflows/ci-package.yml` (the full-matrix choice)

- [ ] **Step 0: Write the failing test** (the same shell-test pattern): a
  fork's `release/*` head does not get the full matrix; a same-repo one
  does. Run it. Expect FAIL.
- [ ] **Step 1:** Choose the full matrix for a `release/*` head only when
  `github.event.pull_request.head.repo.full_name == github.repository`.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope ci-package --message
  "give only same-repo release heads the full matrix"`, then report-ready.

## Task 6a: release PR lookups and the deps-gate delete (vergil-tooling)

Spec §5.2 (PR found by branch name), §6.2.

**Files:**

- Modify: `src/vergil_tooling/lib/github.py` (`_first_pr_for_branch`,
  `pr_for_branch` call `trust.prs_for_branch`)
- Modify: `src/vergil_tooling/lib/release/prepare.py`,
  `src/vergil_tooling/lib/release/bump.py`,
  `src/vergil_tooling/lib/release/preflight.py` (deps gate)
- Test: `tests/vergil_tooling/test_github.py`,
  `tests/vergil_tooling/test_release_prepare.py`,
  `tests/vergil_tooling/test_release_bump.py`,
  `tests/vergil_tooling/test_release_preflight.py`

**Interfaces:**

- Consumes: Task 3c's `prs_for_branch`, `is_same_repo`.

- [ ] **Step 1: Write the failing tests:**
  `test_prepare_ignores_fork_release_pr`;
  `test_bump_ignores_fork_post_release_pr`;
  `test_deps_gate_never_deletes_for_fork_pr`;
  `test_pr_for_branch_reports_fork`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope release --message
  "adopt only same-repo trusted PRs in the release"`, then report-ready.

## Task 6b: finalize, worktree and resolver lookups (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (`_resolve_open_prs`,
  `_infer_pr`), `src/vergil_tooling/lib/worktrees.py`
  (`_resolve_pr_state`), `src/vergil_tooling/bin/vrg_resolve_tracking_issue.py`
  (no first-PR fallback)
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_worktrees.py`,
  `tests/vergil_tooling/test_vrg_resolve_tracking_issue.py`

- [ ] **Step 1: Write the failing tests:**
  `test_finalize_never_adopts_fork_pr`;
  `test_worktree_state_ignores_fork_pr`;
  `test_resolve_tracking_issue_no_first_pr_fallback` (several PRs and no
  merged one: an error naming them, never the first).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope pr --message "find
  PRs by branch only among same-repo trusted PRs"`, then report-ready.

## Task 6c: CI evidence (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/ci_evidence.py`
- Test: `tests/vergil_tooling/test_ci_evidence.py`

- [ ] **Step 1: Write the failing tests:**
  `test_evidence_has_no_first_pr_fallback`;
  `test_evidence_ignores_fork_pr_for_commit`;
  `test_check_runs_matched_by_app_and_workflow` (a check run with the right
  name from another app does not count).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** Match check runs by app slug `github-actions`
  and the workflow name the gate configuration names.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope ci-evidence
  --message "attest only the commit's same-repo PR and its own check runs"`,
  then report-ready.

## Task 7a: the queue views and the planning-epic guard (vergil-tooling)

Spec §5.2 (markers in comments), §6.2.

**Files:**

- Modify: `src/vergil_tooling/lib/queue_view.py` (planning markers, deferral
  notes and latest notes from trusted comments; skips reported),
  `src/vergil_tooling/lib/intake_route.py` (`_planning_epic_states`)
- Test: `tests/vergil_tooling/test_queue_view.py`,
  `tests/vergil_tooling/test_intake_route.py`

- [ ] **Step 1: Write the failing tests:**
  `test_forged_planning_markers_ignored_and_reported`;
  `test_forged_deferral_ignored`; `test_latest_note_from_trusted_only`;
  `test_forged_epic_marker_does_not_block_drop`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope queue --message
  "read queue markers and notes only from trusted comments"`, then
  report-ready.

## Task 7b: the outcome audit and the record dedup (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py`
  (`closed_operational_without_success`),
  `src/vergil_tooling/lib/security_records.py` (unmark comment dedup)
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_security_records.py`

- [ ] **Step 1: Write the failing tests:**
  `test_forged_outcome_does_not_hide_hand_closed_task`;
  `test_unmark_dedup_ignores_outsider_comment`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope audit --message
  "count outcomes and dedup comments only from trusted authors"`, then
  report-ready.

## Task 7c: the remaining login-only readers (vergil-tooling)

Spec §5.1, §5.2 (markers in comments).

**Files:**

- Modify: `src/vergil_tooling/lib/proxies.py` (`Proxy:` lines),
  `src/vergil_tooling/lib/planning_close.py` (`Docs task:` marker),
  `src/vergil_tooling/bin/vrg_ruling.py`,
  `src/vergil_tooling/bin/vrg_ruling_reply.py`,
  `src/vergil_tooling/bin/vrg_scope_change.py`,
  `src/vergil_tooling/lib/merge_review/request.py` (the task body, through
  `read_issue(require_body=True)`),
  `src/vergil_tooling/lib/merge_review/open_pr.py` (the PR body's `Closes`
  refs, through Task 3d's `trusted_closing_refs`),
  `src/vergil_tooling/lib/pending_release.py` (`_trusted_logins`),
  `src/vergil_tooling/lib/epic_audit.py` and `src/vergil_tooling/lib/queue_view.py`
  (the sizing record), and every other caller of the old helpers
- Modify: `src/vergil_tooling/lib/trusted_authors.py`,
  `src/vergil_tooling/lib/rulings/comments.py`,
  `src/vergil_tooling/lib/epic_size.py` (the old helpers deleted)
- Test: the tests of those modules

- [ ] **Step 1: Write the failing tests:** for each module, a forged-input
  test with a User account named like an agent App and one with a trusted
  comment edited by an outsider (`test_proxy_line_from_user_named_like_app`,
  `test_planning_close_marker_edited_by_outsider`,
  `test_ruling_reply_from_user_named_like_app`,
  `test_scope_change_edited_by_outsider`,
  `test_merge_review_request_untrusted_body_raises`,
  `test_open_pr_closes_from_untrusted_body_ignored`,
  `test_sizing_record_from_user_named_like_app`), and
  `test_old_helpers_removed`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** No caller of the old helpers remains.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope trust --message
  "move the remaining comment and body readers onto the trust layer"`, then
  report-ready.

## Task 8a: parents, blockers and the rollup (vergil-tooling)

Spec §5.2 (structural lines, close), §6.2.

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (`parent_of` body fallback and
  `child_states` search fallback from trusted bodies only; `blockers_of`
  from a trusted body; `rollup` acts on a close only when
  `trust.close_judgement` gives a role), `src/vergil_tooling/lib/intake_route.py`
  (`_route`)
- Test: `tests/vergil_tooling/test_epics.py`,
  `tests/vergil_tooling/test_intake_route.py`

- [ ] **Step 1: Write the failing tests:**
  `test_parent_of_ignores_untrusted_body_line`;
  `test_child_states_search_ignores_untrusted_bodies`;
  `test_blockers_of_untrusted_body_raises` (load-bearing: a stop);
  `test_rollup_stops_on_outsider_close`;
  `test_route_parent_fallback_trusted_only`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope epics --message
  "read parents and blockers only from trusted bodies, and roll up only
  trusted closes"`, then report-ready.

## Task 8b: the sweep's body parents and drift closes (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`body_only_parents`,
  `nativize_parents`, `task_drift`, `close_drift`, `epic_drift`,
  `closed_epic_open_child`, `reopen_epics_with_open_children`,
  `proxy_closures`, `proxy_link_drift`),
  `src/vergil_tooling/bin/vrg_epic_audit.py` (its trusted checks run under
  `--close` too, with `BatchStops`)
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: Task 3d's `trusted_closing_refs`; Task 8a.

- [ ] **Step 1: Write the failing tests:**
  `test_nativize_skips_untrusted_body_parent`;
  `test_task_drift_ignores_untrusted_closing_line`;
  `test_task_drift_ignores_body_edited_after_merge_by_outsider`;
  `test_close_runs_trusted_checks`;
  `test_close_stop_is_per_item_and_exits_nonzero`;
  `test_proxy_primary_closed_by_its_own_automation_is_trusted_cross_org`
  (a primary closed by its org's automation App, read from a job in
  another org with the delivered copy);
  `test_close_drift_ignores_outsider_closed_child` (a child's close counts
  only when `close_judgement` gives a role);
  `test_epic_drift_outsider_close_is_per_item_stop`;
  `test_audit_skips_and_reports_unlisted_org` (an org with no stanza is
  never swept, through `vergil_orgs`).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope audit --message
  "nativize parents and close drift only from trusted content"`, then
  report-ready.

## Task 8c: security-record body lines (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (`Security advisory:`
  lines and `_recreate` from a trusted body),
  `src/vergil_tooling/lib/security_records.py` (`retarget`'s re-run reads
  the moved comment from trusted authors only),
  `src/vergil_tooling/bin/vrg_security_record.py` (`show <issue>` prints
  only advisories whose tracking names the issue; `_find_resumed_issue`
  from trusted bodies)
- Test: `tests/vergil_tooling/test_intake_route.py`,
  `tests/vergil_tooling/test_security_records.py`,
  `tests/vergil_tooling/test_vrg_security_record.py`

- [ ] **Step 1: Write the failing tests:**
  `test_show_issue_omits_untracked_advisory`;
  `test_resume_ignores_untrusted_body_match`;
  `test_retarget_rerun_ignores_outsider_moved_comment`;
  `test_route_security_lines_untrusted_body_raises`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope security-record
  --message "trust security-record lines only from trusted bodies"`, then
  report-ready.

## Task 8d: task checks, release marker, linkage and relay (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_git.py` (`_check_issue_is_open_task`
  requires a trusted author), `src/vergil_tooling/lib/release/__init__.py`
  (`is_release_tracking_issue` requires a trusted body),
  `src/vergil_tooling/bin/vrg_submit_pr.py` (`_task_linkage` through Task
  8a's `parent_of`), `src/vergil_tooling/lib/pr_workflow/github_transport.py`
  (the relay payload's commit author is a trusted identity)
- Test: the tests of those four modules

- [ ] **Step 1: Write the failing tests:**
  `test_open_task_check_refuses_outsider_issue`;
  `test_release_marker_untrusted_body_not_tracking`;
  `test_submit_linkage_ignores_untrusted_parent_line`;
  `test_relay_payload_untrusted_commit_author_raises`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope trust --message
  "require trusted authors for task checks, release markers, linkage and
  relay payloads"`, then report-ready.

## Task 9: the deps fix task and the release-start gate (vergil-tooling)

Spec §6.2 (adoption by title and acting on closes).

**Files:**

- Modify: `src/vergil_tooling/lib/update_deps/pr.py` (fix task adopted only
  with a trusted author and body), `src/vergil_tooling/lib/release/preflight.py`
  (acts on the fix task's close only when `close_judgement` gives a role; the fix-task
  link from the deps PR body only from a trusted body; the start gate's open
  `release:` issues: an untrusted one is reported, never a blocker)
- Test: `tests/vergil_tooling/test_update_deps_pr.py`,
  `tests/vergil_tooling/test_release_preflight.py`

- [ ] **Step 1: Write the failing tests:**
  `test_fix_task_by_outsider_not_adopted`;
  `test_outsider_close_of_fix_task_does_not_clear_gate`;
  `test_fix_task_link_from_untrusted_body_ignored`;
  `test_outsider_release_issue_does_not_block_start`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope release --message
  "adopt fix tasks and release issues only when trusted"`, then
  report-ready.

## Task 10: PR provenance and review comments (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/pr_provenance.py` (roles from `trust`; a
  login with no role is `UNTRUSTED` and flagged, never `HUMAN`),
  `src/vergil_tooling/bin/vrg_pr_review_comments.py` (returns trusted
  comments only through `trust.read_reviews`; skips reported)
- Test: `tests/vergil_tooling/test_pr_provenance.py`,
  `tests/vergil_tooling/test_vrg_pr_review_comments.py`

- [ ] **Step 1: Write the failing tests:**
  `test_provenance_stranger_is_untrusted`;
  `test_provenance_roles_from_config_not_suffix`;
  `test_review_comments_omit_outsider_and_report`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope pr --message
  "classify PR identities and review comments through the trust layer"`,
  then report-ready.

## Task 11: outsider-opened issues (vergil-tooling)

Spec §5.3 (outsider-opened issues and PRs).

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`attach_orphans` never
  attaches one; reports it; clean exit),
  `src/vergil_tooling/lib/intake_route.py` (routing one is a stop),
  `src/vergil_tooling/lib/epic_size.py` (sizing one is a stop),
  `src/vergil_tooling/lib/activity_log.py` (leaves out outsider-opened or
  outsider-closed issues)
- Test: the tests of those four modules

- [ ] **Step 1: Write the failing tests:**
  `test_attach_orphans_skips_outsider_issue_clean_exit`;
  `test_route_outsider_issue_stops`; `test_size_outsider_issue_stops`;
  `test_activity_log_omits_outsider_issue`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope intake --message
  "never attach, route, size or publish outsider-opened issues"`, then
  report-ready.

## Task 12: disable the lifecycle job (vergil-containers)

Spec §6.2 (lifecycle publisher).

The lifecycle publisher's GitHub handling is re-architected under
vergil-project/.github#1128 (one PR per run, through the trust layer), not
ported here. Until then its job is off, so it reads nothing.

**Files:**

- Modify: `.github/workflows/ops.yml` (the lifecycle job's `if:` is
  `false`, with a comment naming vergil-project/.github#1128 as the work
  that re-enables it)

- [ ] **Step 1:** Disable the job; leave `docker/versions/lifecycle/` as it
  is.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type ci --scope lifecycle --message
  "disable the lifecycle job until its re-architecture"`, then
  report-ready. (A `.github/workflows/` change: the human's submit pushes
  the branch.)

## Task 13: the trusted read command (vergil-tooling)

Spec §6.4.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_issue_read.py` (entry point
  `vrg-issue-read`)
- Test: `tests/vergil_tooling/test_vrg_issue_read.py` (new)

**Interfaces:**

- Consumes: Task 3b's `read_issue`, Task 3c's `read_reviews`.
- Produces: `vrg-issue-read OWNER/REPO#N [--pr] [--json]
  [--allow-untrusted-body] [--roles human,agents,automation]`. Text
  output: the trusted body, then
  each trusted comment (and, with `--pr`, reviews and inline review
  comments) with its role and time. `--json` gives `IssueContent` and
  `PRReviewContent` as JSON. Skips go to stderr as metadata. Exit 0 on
  success; 3 when the body is untrusted (`UntrustedContentError`),
  unless `--allow-untrusted-body` is given, which only listing or
  display reads use (the body is then left out and reported); 1 on a
  read failure.

- [ ] **Step 1: Write the failing tests:** `test_prints_trusted_only`;
  `test_untrusted_body_exits_3_by_default`;
  `test_allow_untrusted_body_omits_body_and_reports`; `test_json_shape`;
  `test_skips_to_stderr_metadata_only`; `test_pr_reviews`;
  `test_read_failure_exits_1`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope trust --message
  "add vrg-issue-read for skills"`, then report-ready.

## Task 14: the outsider report (vergil-tooling)

Spec §6.5.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (`--untrusted --since
  ISO`), Create: `src/vergil_tooling/lib/outsider_report.py`
- Test: `tests/vergil_tooling/test_outsider_report.py` (new)

**Interfaces:**

- Consumes: Task 3a's `vergil_orgs`; Tasks 3b and 3c.
- Produces: `OutsiderItem(org, repo, number, kind, login, at, url,
  tamper: bool)`; `outsider_activity(org: str, *, since: str) ->
  list[OutsiderItem]` (issues and PRs opened by untrusted accounts,
  untrusted comments and reviews, untrusted edits to trusted bodies,
  untrusted closes and reopens, newest first, from items updated since
  *since*); `vrg-epic-audit --untrusted --since ISO [--org ORG]` prints
  the items of ORG, or of every listed org without it, tamper items
  (edits to trusted bodies, closes) first,
  metadata and links only; it writes nothing on GitHub.

- [ ] **Step 1: Write the failing tests:**
  `test_lists_outsider_issue_and_comment`;
  `test_tamper_items_first`; `test_no_untrusted_text_in_output`;
  `test_only_listed_orgs`; `test_org_flag_limits_to_one_org`; `test_since_filters`;
  `test_read_failure_raises`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope audit --message
  "add the outsider report"`, then report-ready.

## Task 15: the guard test (vergil-tooling)

Spec §6.3.

**Files:**

- Create: `tests/vergil_tooling/test_trust_guard.py`,
  `tests/vergil_tooling/trust_guard_exemptions.toml`

- [ ] **Step 1: Write the test** `test_no_content_read_outside_trust_layer`:
  it scans `src/vergil_tooling/` (AST), outside `lib/trust.py`,
  `bin/vrg_gh.py` (the agents' passthrough wrapper, excluded by name: its
  arguments are the agent's own and the hook covers them) and the six
  low-level read wrappers' own bodies in `lib/github.py` (`graphql`,
  `read_json`, `read_json_paginated`, `read_output`, `run`,
  `open_prs_for_head`; the rest of `lib/github.py` is scanned like any
  module), for two channels:
  - calls to those wrappers, resolving a module-level string constant
    passed as the query or path, so each higher-level content reader in
    `lib/github.py` (`pr_closing_references`, `head_ref`, `pr_for_branch`,
    `closed_pr_for_branch` and the like) is caught and either moves into
    `lib/trust.py` or gets an exemption with a reason;
  - calls that run `gh` or `vrg-gh` directly with `issue view`, `pr view`,
    `issue list`, `pr list`, `search` or `api`.

  It fails when one requests a content field (`body`, `comments`,
  `reviews`, `userContentEdits`, `timelineItems`, `headRefName`,
  `--comments`) or a content API path (`.../comments`, `.../reviews`,
  `issues/<n>`, `pulls/<n>`), or passes a query or path it cannot resolve,
  unless the exemption file lists the call with a reason. Also
  `test_exemptions_are_display_only` (each entry names fields from the
  display-only set) and `test_guard_catches_planted_reads` (fixture source
  files: a direct `gh` read, a GraphQL constant selecting `body`, a
  paginated REST comments path, and calls to `pr_closing_references` and
  `head_ref`). If constant resolution proves brittle,
  replace it with a call-site registry (every GitHub read registered with
  its fields; an unregistered one fails), as spec §6.3 allows.
- [ ] **Step 2:** Run it. Fix any remaining raw read by moving it onto the
  layer or exempting a display-only read with a reason.
- [ ] **Step 3:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 4:** Commit `vrg-commit --type test --scope trust --message
  "fail on content reads outside the trust layer"`, then report-ready.

## Task 16: the hook blocklist (vergil-tooling)

Spec §6.4. A guard against accidents, not a security boundary.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_hook_guard.py`
- Test: `tests/vergil_tooling/test_vrg_hook_guard.py`

**Interfaces:**

- Produces: for agent identity modes, the hook refuses a `vrg-gh` command
  that has `--comments`, a `--json` list naming `body`, `comments` or
  `reviews`, an `api` path ending in `/comments` or `/reviews`, or an
  `issue view` or `pr view` with no `--json` at all (whose default output
  prints the body), with a message naming `vrg-issue-read`. It matches
  with the guard's existing quote-strip and command-position rule; forms
  it cannot see are outside it. Human mode is unchanged.

- [ ] **Step 1: Write the failing tests:**
  `test_agent_issue_view_comments_refused`;
  `test_agent_json_body_refused`; `test_agent_api_comments_path_refused`;
  `test_agent_bare_view_refused`;
  `test_agent_display_read_allowed`; `test_human_mode_unchanged`;
  `test_refusal_names_vrg_issue_read`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope hook --message
  "refuse raw content reads by agents in the hook guard"`, then
  report-ready.

## Task 17a: work skills read through the layer (vergil-claude-plugin)

Spec §6.2 (skills), §6.4.

**Files:**

- Modify: `skills/issue-implement/SKILL.md` (the body and resume comments
  through `vrg-issue-read`; exit 3 is a stop for the human),
  `skills/epic-implement/SKILL.md` (dispatch only a task whose body
  `vrg-issue-read` accepts), `skills/issue-validate/SKILL.md`,
  `skills/issue-deploy/SKILL.md`,
  `skills/issue-validate/references/operational-task-lifecycle.md` (the
  commands and steps only from a trusted body), `skills/pr-watch/SKILL.md`
  (review feedback through `vrg-issue-read --pr`),
  `skills/merge-review/SKILL.md` (the PR and issue body reads), and every
  other body or comment read Task 2's inventory assigns here

- [ ] **Step 1:** Replace each raw `vrg-gh` body or `--comments` read with
  `vrg-issue-read`, and state the stop.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type fix --scope skills --message
  "read task bodies and comments through vrg-issue-read"`, then
  report-ready.

## Task 17b: review and planning skills (vergil-claude-plugin)

**Files:**

- Modify: `skills/epic-review/SKILL.md`, `skills/epic-create/SKILL.md`
  (resume reads through `vrg-issue-read`), `skills/epic-retrospective/SKILL.md`,
  `skills/deprecation-triage/SKILL.md` (dedup only against trusted issues),
  `skills/hoshin/references/daily.md` (the rulings read), and every other
  body or comment read Task 2's inventory assigns here

- [ ] **Step 1:** Replace each raw read with `vrg-issue-read`; drop the
  hand-written trusted-author instructions it makes redundant. A read
  that only lists or displays items (not one the skill acts on) passes
  `--allow-untrusted-body`, and the skill text says why at that read.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type fix --scope skills --message
  "read queue and planning content through vrg-issue-read"`, then
  report-ready.

## Task 17c: the morning review (vergil-claude-plugin)

Spec §5.1a, §6.5.

**Files:**

- Modify: `skills/hoshin/SKILL.md`, `skills/hoshin/references/daily.md`

- [ ] **Step 1:** `hoshin start` runs `vrg-epic-audit --untrusted --since
  <last daily review> --org <org>` for every listed org and shows the
  human what is new
  (tamper items first). For each org it reads the latest daily config
  audit run (Task 5c) through display reads, flags a failed or missing run,
  downloads its `trusted-authors-values` artifact (`vrg-gh run download`)
  and compares it with `vrg-trusted-authors render --org <org>` from the
  host file, flagging any difference as drift
  (the human runs `vrg-trusted-authors publish` to fix it). It records in
  the logbook only that both were reviewed, with no count or detail.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expect PASS.
- [ ] **Step 3:** Commit `vrg-commit --type feat --scope hoshin --message
  "review the outsider report and the configuration audit each morning"`,
  then report-ready.

## Task 18: first vergil-tooling release (deployment, .github)

- **Preconditions:** Tasks 3a to 15 (all but 12), 5a to 5c and every task
  Task 2 filed merged. The human attests that the `[orgs.<org>]` stanzas
  are in the host `identities.toml` for every Vergil org
  (`vrg-trusted-authors` lists them), and that each org's automation App
  holds read-only Variables permission at org and repository level.
- **Human-run steps** (one-off setup): grant the automation Apps that
  permission; tag vergil-actions with 5a to 5c; cut and install the
  vergil-tooling release; run `vrg-trusted-authors publish` for each listed
  org; `vrg-vm update --all`.
- **Procedure:** confirm each merge is in the release with `vrg-git
  merge-base --is-ancestor <merge-sha> <tag-sha>` for every task above;
  dispatch each org's config audit (`vrg-gh workflow run`) and confirm its
  `vrg-trusted-authors check` step passes; file Task 19 with
  `fixture-run <epic-ref> python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app
  --validates "vergil-tooling <version> and vergil-actions <tag>" --after
  <this task>`.
- **Acceptance:** the release and the tag carry every task; the check is
  clean; the fixture runs are filed. `Outcome: SUCCESS`.

## Task 19: fixture runs (all six fixtures)

Filed by Task 18 with `fixture-run`, `--after` Task 18.

## Task 20: plugin release (deployment, .github)

- **Preconditions:** Tasks 17a to 17c merged; Task 18 closed.
- **Human-run steps:** cut and install the vergil-claude-plugin release.
- **Procedure:** ancestry check of 17a to 17c's merges against the tag.
- **Acceptance:** `Outcome: SUCCESS`.

## Task 21: second vergil-tooling release (deployment, .github)

- **Preconditions:** Task 16 merged; Task 20 closed.
- **Human-run steps:** cut and install the vergil-tooling release;
  `vrg-vm update --all`.
- **Procedure:** ancestry check of 16's merge; in an agent session, a raw
  `vrg-gh issue view <ref> --comments`, and a bare `vrg-gh issue view
  <ref>`, are refused and name `vrg-issue-read`.
- **Acceptance:** `Outcome: SUCCESS`.

## Task 22: the layer on real data (validation, .github)

Spec §6.7.

- **Preconditions:** Task 21 closed; Task 12 merged; the six fixture
  runs' validate tasks closed.
- **Procedure:**
  1. `vrg-issue-read` on three issues that carry existing untrusted
     comments: each is reported on stderr, never printed.
  2. `vrg-issue-read` on issues with comments by the human, an agent and
     the automation App: all three pass with their roles.
  3. `vrg-epic-audit --untrusted --since <a date before the release>` lists
     the existing untrusted items, metadata only.
  4. The next scheduled sweep and an upstream-refresh run under Actions
     succeed, and their logs show the trusted checks ran (counts only).
  5. `vrg-issue-read --pr` on real PRs that together carry content from
     the human, an agent and the automation App (for example an
     upstream-refresh PR and a task PR), with conversation comments and
     reviews, prints their trusted body, comments and reviews with each
     of the three roles.
  6. `hoshin start` shows the outsider listing and a passing configuration
     audit whose values artifact matches the host file, and the logbook
     entry records only that both were reviewed.
  7. A scheduled config audit's `vrg-trusted-authors check` step ran as the
     automation App, passed, and uploaded its values artifact.
  8. The vergil-containers lifecycle job is disabled, naming
     vergil-project/.github#1128.
- **Acceptance:** every step as stated. `Outcome: SUCCESS`.

## Evolution during execution

Tasks filed during execution (by Task 2 or otherwise) are recorded here,
each with its repo, what it delivers and what it is blocked by, and the
tasks it blocks.
