# resumable-release-records — implementation plan

- **Epic:** vergil-project/.github#1440
- **Planning issue:** vergil-project/.github#1437 (review history is in its comments)
- **Status:** draft

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `vrg-release` becomes one workflow that `vrg-release --resume`
always finishes, with its state on a locked, human-owned tracking issue, and
posts a `Released:` record on every issue the release contains, which agents
then read as release evidence through `vrg-release-record check`.

**Architecture:**

- **vergil-tooling** does every deterministic part: the `vX.Y` resolver fix;
  the state block, the lock and the identity rule; resume from the first
  undone stage; one release at a time and `--supersede`; the issue list;
  resumable publishing and dependency update; promote and install decoupled
  from packaging, and the install rename; the opt-outs removed;
  `release-records` and `close-tracking`; `vrg-release-record check` and the
  deployment scaffold.
- **vergil-claude-plugin** changes the skills that read release evidence:
  `issue-deploy`, `epic-implement` and `epic-create`, and its docs.
- **The config switch** (`[publish].install`, `v<MINOR>`) lands in each
  repo's `vergil.toml` only once the tooling that reads it is installed,
  because a release is run by the installed tooling, not the one it ships.
- **Operational tasks:** one deployment of the new vergil-tooling (which
  first clears the fleet's open tracking issues), one of the plugin, a
  live-release validation, and fixture runs.

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown skills
(vergil-claude-plugin), Markdown documents (.github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Human logins only.** The tracking issue's state and every `Released:`
  record are trusted only when the identities the spec names are `human`
  logins of `trusted_authors.load_trusted_authors()` (a leading `app/`
  stripped). A failed load is an error, never an empty list (spec §4.2,
  §4.9).
- **Machine-read markers**, each alone on its line:
  `<!-- vrg-release:state:begin -->` and `<!-- vrg-release:state:end -->`
  around a fenced `json` block (the state block), and
  `<!-- vrg-release:released:<owner>/<repo>:v<X.Y.Z> -->` (a record).
- **Fail loudly** (spec §6): every GitHub read the state, the identities,
  the closing references or the records depend on raises with the call that
  failed. None is read as "no issues", "no record" or "no other identity".
- **No silent fallback:** a malformed or missing state block on resume is a
  refusal naming the block, never in-memory defaults (spec §4.2).
- **Imports at start:** `vrg-release` imports everything it uses at module
  load, and after `install` starts no vergil-tooling subprocess, only `gh`
  and `git` (spec §4.4).
- **Tests and validation** run through `vrg-container-run`; a task is done
  when `vrg-container-run -- vrg-validate` passes.
- **Site docs:** each tooling task updates the docs for what it changes:
  `docs/site/docs/reference/dev/release.md`, `docs/site/docs/guides/releasing.md`,
  `docs/site/docs/reference/cli-tools-overview.md`, and the references
  named in its Files.
- **Refactor step:** every tooling task ends with tests green, then removes
  the duplication it introduced, then re-runs the tests.
- **Fixture runs:** this epic changes release tooling, which every language
  fixture releases with, so it seeds a fixture run for all six fixtures
  (Task 18).

## Review Focus

1. **A resume after a crash partway through `release-records`.** Only the
   issues without a counting record get one; none gets two. Pinned in
   Task 11 (`test_records_resume_posts_only_missing`).
2. **An issue closed by two `feature/` PRs in one release.** It appears once
   in the issue list, with both PRs, and gets one record. Pinned in Task 6
   (`test_issue_list_dedupes_issue_closed_twice`).
3. **An agent edits the tracking-issue body between two runs.** The next
   run stops before any write, naming the editor. Pinned in Task 3b
   (`test_guard_stops_on_agent_body_edit`).
4. **`vX.Y` not on this release when `install` runs** (a failed `promote`,
   or a hand demotion in between). `install` refuses. Pinned in Task 8
   (`test_install_refuses_when_line_not_on_release`).
5. **A GraphQL failure while reading identities or closing references.** It
   raises, never passes as "nothing found". Pinned in Task 3b
   (`test_guard_read_failure_raises`) and Task 6
   (`test_closing_refs_failure_raises`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it is
blocked by.

### Tooling

- **1** (.github, task #1441): spec and plan, this PR.
- **2** (vergil-tooling): `vX.Y` follows its tag in `resolve()`, and
  `vrg-tooling-install vX.Y`.
- **3a** (vergil-tooling): the state block module and the tracking issue's
  Contents section.
- **3b** (vergil-tooling): the lock and the identity rule. Blocked by 3a.
- **4** (vergil-tooling): resume from the first undone stage, the worktree
  invariant and fact checks. Blocked by 3a and 3b.
- **5** (vergil-tooling): one release at a time, and the old-tooling
  refusal. Blocked by 3a.
- **6** (vergil-tooling): the issue list from closing references, pinned to
  the cut. Blocked by 3a.
- **7a** (vergil-tooling): `cd-publish`, failed-job recording, and the CD
  release-job recovery. Blocked by 4.
- **7b** (vergil-tooling): `package-index` dispatch on resume; `update-deps`
  required; `OPTIONAL_STAGES` removed; `--without-deps` on resume. Blocked
  by 4.
- **8** (vergil-tooling): promote decoupled; `finalize` split from the
  close; `install` renamed, gated on `promote` and checking `vX.Y`; the
  `[publish].install` key and `<MINOR>`. Blocked by 2 and 4.
- **9** (vergil-tooling): the opt-outs removed; `vrg-submit-pr` always
  finishes; their docs. Blocked by 8.
- **10** (vergil-tooling): `release-records` and `close-tracking`. Blocked
  by 6, 7a, 7b and 8.
- **11** (vergil-tooling): `--supersede`. Blocked by 5 and 10.
- **12** (vergil-tooling): `vrg-release-record check` and the deployment
  scaffold. Blocked by 10.
- **13** (.github, deployment): the vergil-tooling release carrying 2 to 12,
  installed; the fleet's open tracking issues cleared first. Blocked by 2,
  3a, 3b, 4, 5, 6, 7a, 7b, 8, 9, 10, 11 and 12.

### Config, skills and validation

- **14a** (vergil-tooling): vergil-tooling's `vergil.toml` moves to
  `[publish].install` with `vrg-tooling-install v<MINOR>`. Blocked by 13.
- **14b** (vergil-claude-plugin): vergil-claude-plugin's `vergil.toml` moves
  to `[publish].install`. Blocked by 13.
- **15** (vergil-claude-plugin): `issue-deploy`, `epic-implement` and
  `epic-create` use `vrg-release-record check`; `repository-standards.md`
  drops `--no-finalize`. Blocked by 13.
- **16** (.github, deployment): the vergil-claude-plugin release carrying
  14b and 15, installed. Blocked by 14b and 15.
- **17** (.github, validation): a live vergil-tooling release with the new
  tooling: locked issue, records, install through `v<MINOR>`, the check
  command. Blocked by 14a and 16.
- **18** (fixture runs, all six fixtures): filed with `fixture-run`. After
  13.

### Bookends

- **Follow-on brainstorm** (.github, #1444). Blocked by 17, so it starts
  once the records have run on a real release.
- **Documentation review** (vergil-tooling, #4837). Blocked by 2 to 12, 14a,
  14b and 15. Its sweep covers the docs repo's release and workflow pages,
  and spawns a docs-repo task for them.
- **Retrospective** (.github, #1442). Blocked by everything else.

Notes:

- **Merged is not deployed.** The config switch (14a, 14b) waits for the
  deployment (13): a release is run by the installed tooling, which must
  already read `[publish].install` and expand `<MINOR>`. Until then the
  deprecated `consumer-refresh` key keeps every repo installing.
- **The first release with the new tooling** is the one after 13. The
  release that ships 2 to 12 is cut by the older tooling and posts no
  records (spec §5).
- **Filing.** Every task above is filed at epic-create's filing step, in its
  repo, linked under #1440, with its `Blocked-by:` refs; the fixture runs
  are filed with `fixture-run` once 13 exists. The security advisories
  tracking the planning issue are retargeted then, one at a time, all in
  vergil-project: GHSA-9rj7-jmfj-cjqq and GHSA-826g-5hcc-9p76 to Task 3b;
  GHSA-r67r-gh5h-7j92 to Tasks 10 and 12; GHSA-phxw-fpvx-rm39 to Tasks 12
  and 15; GHSA-m5jx-7295-476w to Tasks 3b, 10 and 12.

---

### Task 2: `vX.Y` follows its tag (vergil-tooling)

Spec §4.6.

**Files:**

- Modify: `src/vergil_tooling/lib/tooling_install.py` (`resolve`,
  `install_release`, module docstring)
- Modify: `src/vergil_tooling/lib/tooling_release.py` (docstring: a line
  installs the release its tag names)
- Modify: `src/vergil_tooling/bin/vrg_tooling_install.py` (metavar and help
  `vX.Y or vX.Y.Z`)
- Test: `tests/vergil_tooling/test_tooling_install.py`,
  `tests/vergil_tooling/test_tooling_release.py`,
  `tests/vergil_tooling/test_vrg_tooling_install.py`

**Interfaces:**

- Produces: `resolve(ref, *, ls_remote) -> ResolvedRef` where a line
  `vX.Y` lists `refs/tags/vX.Y` and `refs/tags/vX.Y^{}` and the `vX.Y.*`
  tags in one call, takes the commit the `vX.Y` tag peels to, and returns
  `tag` = the one `vX.Y.Z` tag on that commit. `install_release(tag: str)`
  accepts `classify_ref(tag) in {"line", "exact"}`.

- [ ] **Step 1: Write the failing tests:**
  `test_resolve_line_follows_tag_not_newest` (listing: `v2.1` peels to the
  commit of `v2.1.300`, while `v2.1.301` exists; result `tag == "v2.1.300"`);
  `test_resolve_line_without_tag_raises` (`v3.2` absent: raises naming the
  ref, never falling back to a patch); `test_resolve_line_commit_without_release_tag_raises`;
  `test_install_release_accepts_line`; `test_vrg_tooling_install_help_names_line`.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_tooling_install.py -k line`. Expect FAIL.
- [ ] **Step 3: Implement.** One `ls_remote(GIT_URL, "refs/tags/vX.Y*")`
  already returns the `vX.Y` tag and its peeled line; `_tag_shas` gives the
  commit. Exact pins are unchanged.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope tooling-install
  --message "resolve vX.Y through its rolling tag"`, then report-ready.

### Task 3a: the state block and the Contents section (vergil-tooling)

Spec §4.2.

**Files:**

- Create: `src/vergil_tooling/lib/release/state.py`
- Modify: `src/vergil_tooling/lib/release/tracking.py`
  (`create_tracking_issue` writes Contents and the state block;
  `read_state`, `write_state`)
- Modify: `src/vergil_tooling/lib/release/context.py` (`ReleaseContext.state:
  StateBlock | None`)
- Test: `tests/vergil_tooling/test_release_state.py` (new),
  `tests/vergil_tooling/test_release_tracking.py`

**Interfaces:**

- Produces, in `state.py`: `BEGIN`, `END` (the markers);
  `IssueEntry(ref: str, prs: tuple[int, ...])`;
  `FailedJob(branch: Literal["main", "develop"], run_id: str, job: str)`;
  frozen `StateBlock(version: str, cut: str, issues: tuple[IssueEntry, ...],
  release_pr: str | None, bump_pr: str | None, tag: str | None,
  main_cd_run: str | None, develop_cd_run: str | None,
  failed_jobs: tuple[FailedJob, ...], index_listed: bool, without_deps: bool,
  done: tuple[str, ...], install_done: bool, records_posted: tuple[str, ...],
  supersedes: str | None, superseded_main_commit: str | None)`;
  `parse_state(body: str) -> StateBlock` raising `StateBlockError` (missing,
  duplicated, bad JSON, unknown or missing key, wrong type);
  `render_state(state: StateBlock) -> str`; `upsert_state(body: str, state:
  StateBlock) -> str`.
- Produces, in `tracking.py`: `read_state(ctx) -> StateBlock` (body read,
  then `parse_state`, then the version check against the title);
  `write_state(ctx, state) -> None` (read-modify-write of the body).
  Task 3b inserts the guard into `read_state`.

- [ ] **Step 1: Write the failing tests:** `test_state_round_trip`;
  `test_parse_state_missing_block_raises`; `test_parse_state_two_blocks_raises`;
  `test_parse_state_unknown_key_raises`; `test_upsert_state_keeps_checklist`;
  `test_read_state_version_mismatch_raises`;
  `test_create_tracking_issue_body_has_contents_and_state`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** JSON keys are the field names; tuples render
  as lists. Contents is a `## Contents` section built from the
  `PendingRelease` table and the issue list (Task 6 fills the list; until
  then it is empty).
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "keep the release state in the tracking-issue body"`, then report-ready.

### Task 3b: the lock and the identity rule (vergil-tooling)

Spec §4.2.

**Files:**

- Create: `src/vergil_tooling/lib/release/guard.py`
- Modify: `src/vergil_tooling/lib/release/tracking.py` (`create_tracking_issue`
  locks; `read_state` runs `ensure_locked` then `check_tracking_issue`)
- Test: `tests/vergil_tooling/test_release_guard.py` (new),
  `tests/vergil_tooling/test_release_tracking.py`

**Interfaces:**

- Consumes: `trusted_authors.load_trusted_authors().human`; Task 3a's
  `read_state`.
- Produces: `IssueIdentities(author: str, editors: tuple[str, ...],
  state_actors: tuple[tuple[str, str], ...], traces: tuple[tuple[str, str],
  ...])`, the `(login, action)` pairs; `read_identities(repo: str, number:
  int) -> IssueIdentities` (one paginated GraphQL read: `author`,
  `userContentEdits.editor`, `editor`, `ClosedEvent`/`ReopenedEvent` actors,
  and comments, cross-references, mentions, subscriptions and label events
  as traces); `check_tracking_issue(identities: IssueIdentities, humans:
  Sequence[str]) -> list[str]`, returning one warning per foreign trace and
  raising `TrackingIssueBreach(login: str, action: str)` for a non-human
  author, editor or close or reopen actor; `ensure_locked(repo: str,
  number: int) -> None` (reads `locked`, runs `gh issue lock` only when
  false).

- [ ] **Step 1: Write the failing tests:** `test_guard_passes_human_only`;
  `test_guard_stops_on_agent_body_edit` (an agent login in
  `userContentEdits`: raises naming the login and `edited the body`);
  `test_guard_stops_on_foreign_reopen`;
  `test_guard_warns_on_foreign_comment_and_crossref` (two warnings, no
  raise); `test_guard_read_failure_raises`;
  `test_guard_truncated_connection_raises`;
  `test_ensure_locked_locks_unlocked_issue`; `test_ensure_locked_is_noop_when_locked`;
  `test_read_state_stops_before_any_write` (no `issue edit` or `comment`
  call is made after a breach).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** The breach error's message starts
  `suspected security breach on tracking issue #<N>:` and tells the human
  to inspect, clean up and resume. Warnings print once per run.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "lock the tracking issue and trust only human state changes"`, then
  report-ready.

### Task 4: resume from the first undone stage (vergil-tooling)

Spec §4.3.

**Files:**

- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (`_tracked`
  records `done` and the stage's facts in the state; a resumed run skips
  done stages)
- Modify: `src/vergil_tooling/lib/release/resume.py` (`find_resume_target`
  reads the state, not the checklist cursor)
- Modify: `src/vergil_tooling/lib/release/preflight.py` (on resume: host,
  auth and config checks only; the worktree adopted only when the first
  undone stage is at or before `teardown-worktree`)
- Modify: `src/vergil_tooling/lib/release/tracking.py` (the deferral
  comment's "do not resume" text goes)
- Test: `tests/vergil_tooling/test_release_orchestrator.py`,
  `tests/vergil_tooling/test_release_resume.py`,
  `tests/vergil_tooling/test_release_preflight.py`

**Interfaces:**

- Consumes: `read_state`, `write_state` (3a, 3b).
- Produces: `BRANCH_STAGES = ("prepare", "merge-release",
  "back-merge-bump", "teardown-worktree")`; `first_undone(state:
  StateBlock, stages: Sequence[str]) -> str | None`;
  `verify_facts(ctx, state) -> None` (the tag exists on the recorded
  commit, the PRs are merged, each recorded run's head SHA is the release's;
  a mismatch raises `ReleaseError` naming the fact). Each stage function
  writes its facts through `write_state`; later tasks add stages that follow
  the same rule.

- [ ] **Step 1: Write the failing tests:**
  `test_resume_after_finalize_skips_branch_stages` (state `done` through
  `finalize`: no worktree is created, `prepare` is not called);
  `test_resume_at_merge_release_adopts_worktree`;
  `test_resume_runs_only_host_auth_config_checks`;
  `test_verify_facts_tag_moved_raises`; `test_first_undone_all_done_is_none`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** The skip is in `_tracked`: a stage in
  `state.done` prints `already done` and returns. The checklist is still
  ticked for the human.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "resume from the first undone stage"`, then report-ready.

### Task 5: one release at a time (vergil-tooling)

Spec §4.7.

**Files:**

- Modify: `src/vergil_tooling/lib/release/preflight.py`
  (`_check_no_existing_tracking_issue` becomes `_check_no_open_release`)
- Modify: `src/vergil_tooling/lib/release/resume.py` (`_open_release_issues`
  shared)
- Test: `tests/vergil_tooling/test_release_preflight.py`

**Interfaces:**

- Consumes: `state.parse_state` (3a).
- Produces: `_check_no_open_release(repo: str) -> None`, raising
  `ReleaseError` naming the open issue and `vrg-release --resume`, or, for
  an issue whose body has no state block, the old-tooling recovery (finish
  the release by hand, then close the issue).

- [ ] **Step 1: Write the failing tests:**
  `test_fresh_start_refuses_other_version_open`;
  `test_fresh_start_refusal_names_old_tooling_recovery`;
  `test_fresh_start_passes_with_none_open`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "refuse a fresh release while any release is open"`, then report-ready.

### Task 6: the issue list (vergil-tooling)

Spec §4.1.

**Files:**

- Modify: `src/vergil_tooling/lib/pending_release.py` (`closingIssuesReferences`
  in the PR fields; `gather_release_contents`)
- Modify: `src/vergil_tooling/lib/release/preflight.py` (prints the table and
  the list; stores them on the context)
- Modify: `src/vergil_tooling/lib/release/prepare.py` (passes them to
  `create_tracking_issue`)
- Test: `tests/vergil_tooling/test_pending_release.py`,
  `tests/vergil_tooling/test_release_preflight.py`

**Interfaces:**

- Consumes: `IssueEntry` (3a).
- Produces: `gather_release_contents(*, cut: str, remote: str = "origin")
  -> tuple[PendingRelease, tuple[IssueEntry, ...]]`: the range `main..cut`
  (first parent); issues from PRs whose head branch starts `feature/`, in
  the released repo only, sorted by number, each with every closing PR.

- [ ] **Step 1: Write the failing tests:** `test_issue_list_from_feature_prs`;
  `test_issue_list_skips_deps_release_automation`;
  `test_issue_list_ignores_other_repo_refs`;
  `test_issue_list_dedupes_issue_closed_twice`;
  `test_closing_refs_truncated_raises`; `test_closing_refs_failure_raises`;
  `test_preflight_prints_issue_list`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** `closingIssuesReferences(first: 20) {
  totalCount nodes { number repository { nameWithOwner } } }`, checked
  with `_complete_nodes`.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "list the issues a release closes"`, then report-ready.

### Task 7a: `cd-publish` (vergil-tooling)

Spec §4.5.

**Files:**

- Create: `src/vergil_tooling/lib/release/cd_publish.py`
- Modify: `src/vergil_tooling/lib/release/confirm.py` (`_record_publish_state`
  writes `FailedJob`s; on resume `confirm_main` re-runs a failed release job:
  the release PR's CI run, then the CD run, each watched with
  `--exit-status`; `_failed_release_recovery`'s hand-run steps go)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (the stage)
- Test: `tests/vergil_tooling/test_release_cd_publish.py` (new),
  `tests/vergil_tooling/test_release_confirm.py`

**Interfaces:**

- Produces: `deliver_failed_jobs(ctx: ReleaseContext) -> None`. A fresh
  run with failed jobs records them and fails deferred. On resume, `main`
  jobs: `gh run rerun <id> --failed`, watch, re-judge with
  `confirm_main`'s rules; `develop` jobs: delivered when a later `develop`
  CD run's same job succeeded, else re-run. Delivered jobs leave
  `failed_jobs`; remaining ones fail the stage.

- [ ] **Step 1: Write the failing tests:**
  `test_cd_publish_fresh_run_defers`; `test_cd_publish_resume_reruns_main`;
  `test_cd_publish_resume_develop_superseded_by_later_success`;
  `test_cd_publish_second_resume_reruns_only_remaining`;
  `test_confirm_main_resume_reruns_ci_then_cd`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "deliver failed CD publish jobs on resume"`, then report-ready.

### Task 7b: `package-index` and `update-deps` resumable (vergil-tooling)

Spec §4.5.

**Files:**

- Modify: `src/vergil_tooling/lib/release/package_index.py` (on resume,
  `gh workflow run publish-index.yml --repo <org>/packages`, then wait)
- Modify: `src/vergil_tooling/lib/release/update_deps_stage.py`,
  `src/vergil_tooling/lib/release/handoff.py` (`OPTIONAL_STAGES` removed)
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--without-deps` allowed
  with `--resume`, recorded in the state)
- Test: `tests/vergil_tooling/test_release_package_index.py`,
  `tests/vergil_tooling/test_release_update_deps_stage.py`,
  `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Produces: `update-deps` failure records itself and fails deferred, like
  `package-index`; `state.without_deps` true skips it.

- [ ] **Step 1: Write the failing tests:**
  `test_package_index_resume_dispatches_then_waits`;
  `test_update_deps_failure_blocks_records_not_install`;
  `test_resume_without_deps_skips_failed_update`;
  `test_without_deps_persists_across_resume`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "make the index wait and the dependency update resumable"`, then
  report-ready.

### Task 8: promote, `finalize` and `install` (vergil-tooling)

Spec §4.4, §4.6.

**Files:**

- Create: `src/vergil_tooling/lib/release/install.py` (from `handoff.py`'s
  `consumer_refresh`; `PHASE = "install"`)
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (the new order;
  `_promote_phase` no longer checks `package-index`; `finalize` replaces
  `close-finalize`, running only `vrg-finalize-pr --cleanup-only`)
- Modify: `src/vergil_tooling/lib/release/finalize.py`
- Modify: `src/vergil_tooling/lib/config.py` (`publish.install`, reading
  `consumer-refresh` with a deprecation warning when `install` is absent;
  both present is an error)
- Test: `tests/vergil_tooling/test_release_install.py` (new),
  `tests/vergil_tooling/test_release_orchestrator.py`,
  `tests/vergil_tooling/test_config.py`

**Interfaces:**

- Produces: the stage order of spec §4.4; `install(ctx) -> None`, which
  needs `promote` done, calls `check_line_on_release(ctx)` (one
  `git ls-remote origin refs/tags/vX.Y refs/tags/vX.Y^{}`, the peeled commit
  equal to the release tag's, else `ReleaseError`), expands `<VERSION>` and
  `<MINOR>`, runs the block, and records `install_done`. No install
  configured is `none configured`, with no warning.

- [ ] **Step 1: Write the failing tests:**
  `test_promote_runs_with_index_deferred`;
  `test_install_runs_with_index_deferred`;
  `test_install_refuses_when_promote_failed`;
  `test_install_refuses_when_line_not_on_release`;
  `test_install_expands_minor`; `test_install_skipped_when_done`;
  `test_config_reads_deprecated_consumer_refresh_with_warning`;
  `test_config_both_keys_is_error`; `test_stage_order_matches_spec`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "decouple promote and install from packaging and rename the install"`,
  then report-ready.

### Task 9: the opt-outs go (vergil-tooling)

Spec §4.8.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--no-promote`,
  `--no-install`), `src/vergil_tooling/bin/vrg_finalize_pr.py`
  (`--no-install`), `src/vergil_tooling/bin/vrg_submit_pr.py`
  (`--no-finalize`, `--no-install`): each parsed hidden and refused with its
  removal message
- Modify: `docs/site/docs/reference/dev/submit-pr.md`,
  `docs/site/docs/reference/dev/finalize-pr.md`,
  `docs/site/docs/reference/dev/release.md`,
  `docs/site/docs/guides/git-workflow.md`,
  `docs/specs/2026-05-29-agent-permission-model-design.md` (the accepted
  gap narrowed to "between finalize runs, or after a finalize run stops
  watching")
- Test: `tests/vergil_tooling/test_vrg_release.py`,
  `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Produces: removal messages of the form `vrg-<tool>: --<flag> was removed;
  <reason>`, the reasons: promote and install are part of every release;
  `vrg-submit-pr` returns once each PR has merged or cannot merge.

- [ ] **Step 1: Write the failing tests:** one
  `test_<flag>_removed_message` per removed flag, each exiting 1 before any
  side effect.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**, and update the docs.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "remove the release and submit opt-outs"`, then report-ready.

### Task 10: `release-records` and `close-tracking` (vergil-tooling)

Spec §4.9.

**Files:**

- Create: `src/vergil_tooling/lib/release/records.py`
- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (both stages)
- Modify: `src/vergil_tooling/lib/release/finalize.py` (`close_tracking`
  posts today's summary and closes)
- Test: `tests/vergil_tooling/test_release_records.py` (new),
  `tests/vergil_tooling/test_release_orchestrator.py`

**Interfaces:**

- Consumes: `StateBlock.issues`, `records_posted` (3a);
  `load_trusted_authors().human`.
- Produces: `marker(repo: str, version: str) -> str`; `record_body(*, repo,
  version, date, release_url, prs: Sequence[tuple[int, str]], install:
  Literal["ran", "none configured"], supersedes: str | None) -> str`, the
  spec §4.9 text; `counting_record(comments: Sequence[Comment], repo, version,
  humans) -> tuple[Comment | None, list[str]]` (a comment counts only when
  its author and last editor are human; others are warnings);
  `post_records(ctx) -> None`.

- [ ] **Step 1: Write the failing tests:** `test_record_body_matches_spec`;
  `test_records_posted_on_closed_and_reopened_issues`;
  `test_records_resume_posts_only_missing`;
  `test_foreign_marker_ignored_with_warning`;
  `test_edited_by_agent_record_does_not_count`;
  `test_records_failure_names_each_issue_and_blocks_close`;
  `test_close_tracking_only_after_all_records`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** Comments are read with author and last editor
  in one GraphQL call per issue, truncation-checked.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "post a Released: record on every issue a release contains"`, then
  report-ready.

### Task 11: `--supersede` (vergil-tooling)

Spec §4.7.

**Files:**

- Create: `src/vergil_tooling/lib/release/supersede.py`
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--supersede X.Y.Z`; not
  with `--resume`)
- Modify: `src/vergil_tooling/lib/release/preflight.py` (the open-release
  check exempts the superseded issue; the `main`-ahead check accepts its
  `superseded_main_commit`)
- Modify: `src/vergil_tooling/lib/release/prepare.py`,
  `src/vergil_tooling/lib/release/records.py` (absorbed issues;
  `Supersedes:` line)
- Test: `tests/vergil_tooling/test_release_supersede.py` (new),
  `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Consumes: `read_state` (3b), `_check_no_open_release` (5),
  `record_body` (10).
- Produces: `check_eligible(state: StateBlock) -> None`, raising
  `ReleaseError` unless the tag and Release exist and the undone stages are
  a subset of `{"cd-publish", "package-index", "update-deps", "install"}`,
  or the first undone stage is `confirm-main` and no tag exists;
  `absorb(old: StateBlock, new_issues) -> tuple[IssueEntry, ...]`;
  `close_superseded(repo, number, new_version) -> None`, after the new
  tracking issue exists.

- [ ] **Step 1: Write the failing tests:**
  `test_supersede_refuses_bare_release`; `test_supersede_eligible_delivery_left`;
  `test_supersede_eligible_untagged_confirm_main`;
  `test_supersede_refuses_merge_release_halt`;
  `test_supersede_absorbs_issues_and_records_supersedes`;
  `test_supersede_accepts_untagged_main_commit`;
  `test_supersede_with_resume_is_usage_error`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "supersede a release that cannot finish"`, then report-ready.

### Task 12: `vrg-release-record check` and the scaffold (vergil-tooling)

Spec §4.10.

**Files:**

- Create: `src/vergil_tooling/bin/vrg_release_record.py` (entry point in
  `pyproject.toml`)
- Modify: `src/vergil_tooling/data/deployment_task_body.md`
- Modify: `docs/site/docs/reference/cli-tools-overview.md` and a new
  `docs/site/docs/reference/dev/release-record.md`
- Test: `tests/vergil_tooling/test_vrg_release_record.py` (new),
  `tests/vergil_tooling/test_vrg_issue_create.py`

**Interfaces:**

- Consumes: `counting_record`, `marker` (10).
- Produces: `vrg-release-record check <owner>/<repo>#<N>`, printing JSON
  `{"task": str, "verdict": "pass" | "fail" | "none", "record": {"version":
  str, "tag": str, "url": str} | null, "ancestry": "pass" | "fail" | null,
  "warnings": [str]}`; exit 0 for `pass`, 3 for `fail` or `none`, 1 for an
  error. The ancestry check: `git ls-remote` the tag, then `git merge-base
  --is-ancestor <merge-sha> <tag-sha>` in a temporary bare fetch of the
  task's repo.

- [ ] **Step 1: Write the failing tests:** `test_check_pass`;
  `test_check_none_when_no_record`; `test_check_fail_when_ancestry_fails`;
  `test_check_ignores_agent_record_with_warning`;
  `test_check_read_failure_exits_1`;
  `test_deployment_scaffold_names_check_command`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** The scaffold's preconditions text: a
  blocked-by development task whose release the deployment depends on is
  confirmed with `vrg-release-record check`; every other precondition is
  checked as before.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "add vrg-release-record check"`, then report-ready.

### Task 13: vergil-tooling release with the resumable release (deployment, .github)

Blocked by 2, 3a, 3b, 4, 5, 6, 7a, 7b, 8, 9, 10, 11 and 12.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Tasks 2 to 12 is published and installed on the host and the VMs, its
  version named in a comment. It is cut with the older tooling, so it posts
  no records (spec §5).
- **Procedure (agent-safe):**
  1. For each of Tasks 2 to 12, check that its PR's merge commit is in the
     release: `vrg-git ls-remote origin refs/tags/<tag>
     'refs/tags/<tag>^{}'`, then `vrg-git merge-base --is-ancestor
     <merge-sha> <tag-sha>` exits 0, after `vrg-git fetch origin tag <tag>`.
  2. `vrg-release --help` lists `--supersede` and not `--no-install`;
     `vrg-release-record --help` exits 0.
  3. List the open `release:` tracking issues in every fleet repo that
     releases with `vrg-release`
     (`vrg-gh issue list --repo <repo> --search "release: in:title" --state open`);
     the list must be empty.
- **Human-run step:** finish by hand and close any tracking issue step 3
  lists, then re-run step 3.
- **SUCCESS:** every check passes, recorded with the version.

### Task 14a: vergil-tooling's install config (vergil-tooling)

Spec §4.6, §5. Blocked by 13.

- **Files:** Modify `vergil.toml`: `[publish].consumer-refresh` becomes
  `[publish].install`, its first command `vrg-tooling-install v<MINOR>`.
- [ ] **Step 1:** Edit, then `vrg-container-run -- vrg-validate`. Expect
  PASS with no deprecation warning.
- [ ] **Step 2:** Commit `vrg-commit --type chore --scope release --message
  "move the install to [publish].install and follow the release line"`,
  then report-ready.

### Task 14b: vergil-claude-plugin's install config (vergil-claude-plugin)

Spec §5. Blocked by 13.

- **Files:** Modify `vergil.toml`: `[publish].consumer-refresh` becomes
  `[publish].install`.
- [ ] **Step 1:** Edit, then `vrg-container-run -- vrg-validate`. Expect
  PASS.
- [ ] **Step 2:** Commit `vrg-commit --type chore --scope release --message
  "move the install to [publish].install"`, then report-ready.

### Task 15: skills read the records (vergil-claude-plugin)

Spec §4.10. Blocked by 13.

**Files:**

- Modify: `skills/issue-deploy/SKILL.md` (the preconditions gate: a
  blocked-by development task whose release is needed is confirmed with
  `vrg-release-record check`; exit 3 is a blocked precondition; the
  ancestry recipe stays as the command's description)
- Modify: `skills/epic-implement/SKILL.md` (release evidence for a
  dependency through the same command)
- Modify: `skills/epic-create/SKILL.md` ("Checking that a release carries
  the changes" names the command first, the ancestry check as its
  cross-check)
- Modify: `docs/repository-standards.md` (`--no-finalize` and
  `--no-install` gone from the usage)

- [ ] **Step 1:** Make the edits; run `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 2:** Commit `vrg-commit --type docs --scope skills --message
  "read release evidence with vrg-release-record check"`, then
  report-ready.

### Task 16: vergil-claude-plugin release with the record skills (deployment, .github)

Blocked by 14b and 15.

- **Precondition:** the vergil-claude-plugin release carrying Tasks 14b
  and 15 is published and installed, its version named in a comment. This
  release is cut with the new tooling, so Tasks 14b and 15 carry
  `Released:` records.
- **Procedure (agent-safe):** `vrg-release-record check` on Tasks 14b and
  15 each passes, with the version named.
- **SUCCESS:** both pass, recorded with the version.

### Task 17: a live release with records (validation, .github)

Blocked by 14a and 16.

- **Precondition (human-attested):** a vergil-tooling release carrying
  Task 14a has been run with the new tooling, its version and tracking
  issue named in a comment.
- **Procedure (agent-safe):**
  1. The tracking issue is locked and closed, and its body holds a state
     block naming every issue the release closes.
  2. `vrg-release-record check vergil-project/vergil-tooling#<14a>` passes.
  3. Every issue in the state block's list has exactly one human-authored
     record for the version.
  4. The records' `Install:` line reads `ran`, and the host's installed
     vergil-tooling is the new version, installed through `v<MINOR>`
     (`vrg-tooling-install`'s log line names the resolved tag).
  5. `git ls-remote` of `refs/tags/v<X.Y>` peels to the release's commit,
     and the dev-container cache's `Install:` line in a `vrg-container-run`
     reports the same release.
- **SUCCESS:** every check passes.

### Task 18: fixture runs

After 13. Filed with:

```bash
fixture-run vergil-project/.github#1440 \
  python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app \
  --validates "the vergil-tooling release with resumable releases (Task 13)" \
  --after vergil-project/.github#<13>
```

Each run's release is then cut with the new tooling, so its validation also
checks the fixture's tracking issue is locked and closed and its issues
carry records.
