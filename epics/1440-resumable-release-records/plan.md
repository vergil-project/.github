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
  `issue-deploy` and `epic-create`, and its docs.
- **The config switch** (`[publish].install`, `v<MINOR>`) lands in each
  repo's `vergil.toml` only once the tooling that reads it is installed,
  because a release is run by the installed tooling, not the one it ships.
- **Operational tasks:** an early release of the resolver fix on its own;
  one deployment of the new vergil-tooling, whose cut is preceded by
  clearing the fleet's open tracking issues; one of the plugin; a
  live-release validation; and fixture runs.

**Tech Stack:** Python 3 and pytest (vergil-tooling), Markdown skills
(vergil-claude-plugin), Markdown documents (.github).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Release freeze.** No vergil-tooling release is cut from `develop`
  from the merge of Task 3a until Task 13's release is cut, which needs
  every one of Tasks 3a to 12 merged. Task 2d's release of
  Task 2 alone comes before it. The human enforces it: Task 3a's
  report-ready notes say its merge opens the freeze, and Task 13 records
  the cut that ends it. An urgent vergil-tooling fix during the window waits
  for Task 13, unless the human lifts the freeze deliberately.
  (Mechanizing freezes is vergil-project/.github#1453.)
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
   Task 10 (`test_records_resume_posts_only_missing`).
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
- **2d** (.github, deployment): the vergil-tooling release carrying Task 2
  alone, installed, before the freeze opens. Blocked by 2.
- **3a** (vergil-tooling): the state block module and the tracking issue's
  Contents section. Blocked by 2d (its merge opens the freeze).
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
  records its failure; `--without-deps` on resume. Blocked by 4.
- **8** (vergil-tooling): promote decoupled; `finalize` split from the
  close; the per-stage hold rule and `OPTIONAL_STAGES` removed; `install`
  renamed, gated on `promote` and checking `vX.Y`; the `[publish].install`
  key and `<MINOR>`; the hand-run remedy messages removed. Blocked by 2, 4,
  7a and 7b.
- **9a** (vergil-tooling): `vrg-release --no-promote` and `--no-install`
  removed. Blocked by 8.
- **9b** (vergil-tooling): `vrg-finalize-pr --no-install`, and
  `vrg-submit-pr --no-finalize` and `--no-install`, removed; `vrg-submit-pr`
  always finishes; their docs. Blocked by 9a.
- **10** (vergil-tooling): `release-records` and `close-tracking`. Blocked
  by 6, 7a, 7b and 8.
- **11** (vergil-tooling): `--supersede`. Blocked by 5 and 10.
- **12** (vergil-tooling): `vrg-release-record check` and the deployment
  scaffold. Blocked by 10.
- **13** (.github, deployment): the fleet's open tracking issues cleared
  with the older tooling, then the vergil-tooling release carrying 3a to 12
  cut and installed, ending the freeze; the fixture runs and Task 19 filed
  with its version. Blocked by 3a, 3b, 4, 5, 6, 7a, 7b, 8, 9a, 9b, 10, 11
  and 12.

### Config, skills and validation

- **14a** (vergil-tooling): vergil-tooling's `vergil.toml` moves to
  `[publish].install` with `vrg-tooling-install v<MINOR>`. Blocked by 13.
- **14b** (vergil-claude-plugin): vergil-claude-plugin's `vergil.toml` moves
  to `[publish].install`. Blocked by 13.
- **15** (vergil-claude-plugin): `issue-deploy` and `epic-create` use
  `vrg-release-record check`; `repository-standards.md` drops
  `--no-finalize`. Blocked by 13.
- **16** (.github, deployment): the vergil-claude-plugin release carrying
  14b and 15, installed. Blocked by 14b and 15.
- **17** (.github, validation): a live vergil-tooling release with the new
  tooling: locked issue, records, install through `v<MINOR>`, the check
  command. Blocked by 14a and 16.
- **18** (fixture runs, all six fixtures): filed with `fixture-run` by Task
  13, once its release version is known.
- **20** (vergil-tooling): the deprecated `[publish].consumer-refresh`
  fallback removed, one minor version after Task 13's release. Blocked by
  14a and 14b.
- **19** (.github, validation): each fixture run's release left a locked,
  closed tracking issue and a record on every issue it closed. Filed by
  Task 13, blocked by the six fixture runs' release tasks.

### Bookends

- **Follow-on brainstorm** (.github, #1444). Blocked by 17, so it starts
  once the records have run on a real release.
- **Documentation review** (vergil-tooling, #4837). Blocked by 2, 3a to
  12, 14a, 14b, 15 and 20. Its sweep covers the docs repo's
  release and workflow pages, and spawns a docs-repo task for them.
- **Retrospective** (.github, #1442). Blocked by everything else.

Notes:

- **Merged is not deployed.** The config switch (14a, 14b) waits for the
  deployment (13): a release is run by the installed tooling, which must
  already read `[publish].install` and expand `<MINOR>`. Until then the
  deprecated `consumer-refresh` key keeps every repo installing.
- **The first release with the new tooling** is the one after 13. The
  releases that ship 2 (Task 2d) and 3a to 12 (Task 13) are cut by the
  older tooling and post no records (spec §5).
- **The freeze** opens when 3a merges and ends with Task 13's cut, whatever
  order 3a to 12 merge in.
- **Filing.** Every task above but 18 and 19 is filed at epic-create's
  filing step, in its repo, linked under #1440, with its `Blocked-by:`
  refs. Task 13 files 18 (with `fixture-run`, naming its version) and 19
  (with `vrg-issue-create --kind validation`, blocked by the six release
  tasks `fixture-run` prints), as the plan's seeding approves. The security advisories
  tracking the planning issue are retargeted then, one at a time, all in
  vergil-project: GHSA-9rj7-jmfj-cjqq and GHSA-826g-5hcc-9p76 to Task 3b;
  GHSA-r67r-gh5h-7j92 to Tasks 10 and 12; GHSA-phxw-fpvx-rm39 to Tasks 12
  and 15; GHSA-m5jx-7295-476w to Tasks 3b, 10 and 12; GHSA-j675-33vm-gh52
  to Task 6; GHSA-4g9g-g7vg-c62v and GHSA-32p4-93jj-q8w5 to Tasks 6 and 12;
  GHSA-7r62-g8v4-h3wx to Task 3b.

---

### Task 2: `vX.Y` follows its tag (vergil-tooling)

Spec §4.6.

**Files:**

- Modify: `src/vergil_tooling/lib/tooling_install.py` (`resolve`,
  `install_release`, module docstring)
- Modify: `src/vergil_tooling/lib/tooling_release.py` (docstring: a line
  installs the release its tag names), `src/vergil_tooling/lib/vm_packages.py`
  (docstring)
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
  accepts `classify_ref(tag) in {"line", "exact"}`; a line is resolved with
  `resolve()` first, and the exact tag is fetched, installed and reported
  (`Installed.tag` is `vX.Y.Z`).

- [ ] **Step 1: Write the failing tests:**
  `test_resolve_line_follows_tag_not_newest` (listing: `v2.1` peels to the
  commit of `v2.1.300`, while `v2.1.301` exists; result `tag == "v2.1.300"`);
  `test_resolve_line_without_tag_raises` (`v3.2` absent: raises naming the
  ref, never falling back to a patch); `test_resolve_line_commit_without_release_tag_raises`;
  `test_install_release_line_installs_and_reports_exact_tag` (output names
  `vX.Y.Z`, not `vX.Y`); `test_vrg_tooling_install_help_names_line`;
  `test_resolve_tooling_tag_follows_demoted_line` (in
  `test_tooling_release.py`) and `test_vm_resolve_follows_demoted_line` (in
  `test_vm_packages.py`), each with `v2.1` peeling to an older patch than
  the newest.
- [ ] **Step 2:** Run `vrg-container-run -- pytest
  tests/vergil_tooling/test_tooling_install.py -k line`. Expect FAIL.
- [ ] **Step 3: Implement.** One `ls_remote(GIT_URL, "refs/tags/vX.Y*")`
  already returns the `vX.Y` tag and its peeled line; `_tag_shas` gives the
  commit. Exact pins are unchanged.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type fix --scope tooling-install
  --message "resolve vX.Y through its rolling tag"`, then report-ready.

### Task 2d: vergil-tooling release with the resolver fix (deployment, .github)

Blocked by 2.

- **Precondition (human-attested):** the vergil-tooling release carrying
  Task 2, and no task of 3a to 12, is published and installed on the host
  and the VMs, its version named in a comment.
- **Procedure (agent-safe):**
  1. Task 2's PR merge commit is in the release: `vrg-git fetch origin tag
     <tag>`, `vrg-git ls-remote origin refs/tags/<tag> 'refs/tags/<tag>^{}'`,
     then `vrg-git merge-base --is-ancestor <merge-sha> <tag-sha>` exits 0.
  2. `vrg-tooling-install --help` names `vX.Y`.
- **SUCCESS:** both pass, recorded with the version. Task 3a may then merge,
  which opens the freeze.

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
  tag_commit: str | None (the commit the tag peels to, written by
  `confirm-main`),
  main_cd_run: str | None, develop_cd_run: str | None,
  failed_jobs: tuple[FailedJob, ...], index_listed: bool, without_deps: bool,
  done: tuple[str, ...], install_done: bool, records_posted: tuple[str, ...],
  supersedes: str | None, superseded_main_commit: str | None,
  failed_release_run: str | None)` (`failed_release_run` is the CD release
  run that `confirm-main` saw fail);
  `parse_state(body: str) -> StateBlock` raising `StateBlockError` (missing,
  duplicated, bad JSON, unknown or missing key, wrong type);
  `render_state(state: StateBlock) -> str`; `upsert_state(body: str, state:
  StateBlock) -> str`.
- Produces, in `tracking.py`: `read_state(ctx) -> StateBlock` (body read,
  then `parse_state`, then the version check against the title);
  `write_state(ctx, state) -> None` (read-modify-write of the body).
  Task 3b inserts the guard into both `read_state` and `write_state`'s own
  read.

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
  call is made after a breach); `test_write_state_stops_on_edit_between_read_and_write`
  (a foreign body edit after a guarded read: the write is not made).
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

- Modify: `src/vergil_tooling/lib/release/orchestrator.py` (one wrapper
  around every stage records `done` and the stage's facts in the state; a
  resumed run skips done stages; `teardown-worktree` becomes `fail_fast`)
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
  StateBlock, stages: Sequence[str]) -> str | None`, which ignores
  `ALWAYS_RUN = ("audit", "preflight", "publish-status")` (they run on every
  run and are never recorded done); `verify_facts(ctx, state) -> None` (the
  PRs are merged; `main_cd_run`'s head SHA is `release_pr`'s merge commit;
  the tag peels to `state.tag_commit`, which is that merge commit or a
  single-parent freeze commit on top of it, by `promote`'s merged-work rule;
  `develop_cd_run`'s head SHA is `bump_pr`'s merge commit; a mismatch
  raises `ReleaseError` naming the fact). Each stage
  writes its facts through `write_state`; later tasks add stages under the
  same wrapper.

- [ ] **Step 1: Write the failing tests:**
  `test_resume_after_finalize_skips_branch_stages` (state `done` through
  `finalize`: no worktree is created, `prepare` is not called);
  `test_resume_after_teardown_does_not_adopt_worktree`;
  `test_failed_teardown_stops_the_run`;
  `test_resume_at_merge_release_adopts_worktree`;
  `test_resume_runs_only_host_auth_config_checks`;
  `test_verify_facts_tag_moved_raises` (the tag now peels to a commit
  other than `tag_commit`); `test_verify_facts_freeze_commit_tag_passes`;
  `test_verify_facts_develop_run_matches_back_merge`;
  `test_first_undone_all_done_is_none`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** The skip is in the wrapper: a stage in
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
  the released repo only, sorted by number, each with every closing PR. A
  closing reference counts only when it comes from the PR body's text and
  every identity in the body's history (its author and every editor in
  `userContentEdits`) is in the trusted set (`human` and `agents` of
  `load_trusted_authors()`). A reference linked by hand (a `ConnectedEvent`)
  counts only when its actor is trusted. Any other PR or reference
  contributes no issues and is named in a warning. Produces
  `trusted_closing_refs(pr: Mapping[str, object], trusted: Sequence[str])
  -> tuple[tuple[int, ...], list[str]]`, which Task 12 reuses.

- [ ] **Step 1: Write the failing tests:** `test_issue_list_from_feature_prs`;
  `test_issue_list_skips_deps_release_automation`;
  `test_issue_list_ignores_other_repo_refs`;
  `test_issue_list_dedupes_issue_closed_twice`;
  `test_issue_list_excludes_pr_edited_by_untrusted_identity` (warning names
  the PR); `test_issue_list_excludes_earlier_untrusted_edit` (an untrusted
  edit followed by a trusted one); `test_issue_list_excludes_untrusted_manual_link`;
  `test_closing_refs_truncated_raises`; `test_closing_refs_failure_raises`;
  `test_preflight_prints_issue_list`.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** `closingIssuesReferences(first: 20,
  userLinkedOnly: false)` with each node's `number` and `repository {
  nameWithOwner }`, the PR's `body`, `author`, `userContentEdits` and
  `ConnectedEvent` timeline items, in the same query, every connection
  checked with `_complete_nodes`. A reference counts as from the body when
  a closing line in the body names it, matched with the sanctioned keywords
  of `lib/linkage.py` (`Close`, `Closes`, `Closed`), never a list of its
  own. Test: `test_issue_list_counts_closed_keyword` (`Closed #N`).
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
  `test_confirm_main_resume_reruns_ci_then_cd`;
  `test_deferred_publish_failures_derived_from_state`;
  `test_confirm_main_records_failed_release_run` (a failed CD release job
  writes `failed_release_run` before the stage fails).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** `ReleaseContext.deferred_publish_failures`
  becomes a read-only property derived from `state.failed_jobs` and
  `state.index_listed`; `confirm.py` and `package_index` write the state
  instead of appending to a list.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "deliver failed CD publish jobs on resume"`, then report-ready.

### Task 7b: `package-index` and `update-deps` resumable (vergil-tooling)

Spec §4.5.

**Files:**

- Modify: `src/vergil_tooling/lib/release/package_index.py` (on resume,
  `gh workflow run publish-index.yml --repo <org>/packages`, then wait)
- Modify: `src/vergil_tooling/lib/release/update_deps_stage.py`
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--without-deps` allowed
  with `--resume`, recorded in the state)
- Test: `tests/vergil_tooling/test_release_package_index.py`,
  `tests/vergil_tooling/test_release_update_deps_stage.py`,
  `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Produces: `update-deps` failure records itself in the state and fails
  deferred, like `package-index`; `state.without_deps` true skips it.
  `OPTIONAL_STAGES` stays until Task 8 replaces it with the hold rule.

- [ ] **Step 1: Write the failing tests:**
  `test_package_index_resume_dispatches_then_waits`;
  `test_update_deps_failure_recorded_in_state`;
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
- Modify: `src/vergil_tooling/lib/release/handoff.py` (`OPTIONAL_STAGES`
  and `blocking_failures` replaced by the hold rule)
- Modify: `src/vergil_tooling/lib/release/tracking.py` (delete
  `publish_deferred_remedy`, `comment_publish_deferred` and
  `promote_pending`); `orchestrator.py`'s `_publish_status_stage` and
  `_close_finalize_outcome` name only `vrg-release --resume` or
  `vrg-release --supersede X.Y.Z`
- Modify, for the rename (every `consumer-refresh`/`consumer_refresh`
  name): `src/vergil_tooling/bin/vrg_release.py`,
  `src/vergil_tooling/bin/vrg_host.py`,
  `src/vergil_tooling/bin/vrg_tooling_install.py`,
  `src/vergil_tooling/lib/host_status.py`, `src/vergil_tooling/lib/progress.py`,
  `src/vergil_tooling/lib/release/context.py`,
  `src/vergil_tooling/lib/release/package_index.py`,
  `src/vergil_tooling/lib/tooling_install.py`,
  `src/vergil_tooling/data/host_dependencies.toml`; tests
  `test_github_config_lib.py`, `test_vrg_github_repo_config.py`,
  `test_vrg_host.py`, and `test_consumer_refresh.py` and
  `test_release_handoff.py`, which move with the stage.
  `PublishConfig.consumer_refresh` is renamed `install`.
- Modify: `src/vergil_tooling/lib/config.py` (`publish.install`, reading
  `consumer-refresh` with a deprecation warning when `install` is absent;
  both present is an error)
- Test: `tests/vergil_tooling/test_release_install.py` (new),
  `tests/vergil_tooling/test_release_orchestrator.py`,
  `tests/vergil_tooling/test_config.py`

**Interfaces:**

- Produces: the stage order of spec §4.4; `holds(stage: str, failed:
  Sequence[str]) -> list[str]`, the failed stages that hold *stage* back:
  for `promote` and `finalize`, those through `confirm-develop`; for
  `install`, those and
  `promote`; for `release-records` and `close-tracking`, every failed stage;
  `install(ctx) -> None`, which
  needs `promote` done, calls `check_line_on_release(ctx)` (one
  `git ls-remote origin refs/tags/vX.Y refs/tags/vX.Y^{}`, the peeled commit
  equal to the release tag's, else `ReleaseError`), expands `<VERSION>` and
  `<MINOR>`, runs the block, and records `install_done`. No install
  configured is `none configured`, with no warning.

- [ ] **Step 1: Write the failing tests:**
  `test_promote_runs_with_index_deferred`;
  `test_install_runs_with_index_deferred`;
  `test_finalize_and_install_run_after_failed_cd_publish`;
  `test_promote_held_by_failed_confirm_develop`;
  `test_update_deps_failure_does_not_hold_install`;
  `test_publish_status_names_only_resume_or_supersede`;
  `test_no_output_names_hand_run_commands` (no run output contains
  `gh run rerun`, `gh workflow run` or `vrg-promote`);
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

### Task 9a: `vrg-release`'s opt-outs go (vergil-tooling)

Spec §4.8. Blocked by 8.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--no-promote`,
  `--no-install`, each parsed hidden and refused with its removal message)
- Modify: `docs/site/docs/reference/dev/release.md`
- Test: `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Produces: removal messages of the form `vrg-release: --<flag> was
  removed; promote and install are part of every release`. `vrg-finalize-pr
  --release --no-install` and `vrg-submit-pr --release --no-install` still
  pass `--no-install` on until Task 9b, so each fails with this message in
  between; no release is cut in between (the freeze).

- [ ] **Step 1: Write the failing tests:** `test_no_promote_removed_message`,
  `test_no_install_removed_message`, each exiting 1 before any side effect.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**, and update the doc.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "remove vrg-release --no-promote and --no-install"`, then report-ready.

### Task 9b: the submit and finalize opt-outs go (vergil-tooling)

Spec §4.8. Blocked by 9a; it ships in Task 13's release.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (`--no-install`),
  `src/vergil_tooling/bin/vrg_submit_pr.py` (`--no-finalize`,
  `--no-install`): each parsed hidden and refused with its removal message
- Modify: `docs/site/docs/reference/dev/submit-pr.md`,
  `docs/site/docs/reference/dev/finalize-pr.md`,
  `docs/site/docs/guides/git-workflow.md`,
  `docs/specs/2026-05-29-agent-permission-model-design.md` (the accepted
  gap narrowed to "between finalize runs, or after a finalize run stops
  watching")
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py`,
  `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Produces: removal messages of the form `vrg-<tool>: --<flag> was removed;
  <reason>`, the reasons: install is part of every release; `vrg-submit-pr`
  returns once each PR has merged or cannot merge.

- [ ] **Step 1: Write the failing tests:** one
  `test_<flag>_removed_message` per removed flag, each exiting 1 before any
  side effect.
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement**, and update the docs.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope release --message
  "remove the submit and finalize opt-outs"`, then report-ready.

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
  `test_close_tracking_only_after_all_records`;
  `test_update_deps_failure_holds_records`;
  `test_records_comment_read_paginates` (150 comments, the record on page
  2); `test_records_comment_page_failure_raises`;
  `test_no_deferred_vergil_tooling_imports` (walks the modules reachable
  from `release-records` and `close-tracking` and fails on any
  function-level `vergil_tooling` import).
- [ ] **Step 2:** Run them. Expect FAIL.
- [ ] **Step 3: Implement.** Comments are read with author and last editor
  through a paginated GraphQL read per issue; any failed page raises.
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
  `superseded_main_commit`; on the untagged path, the release version is
  the superseded version's next patch, `X.Y.(Z+1)`, unless `minor` or
  `major` is given)
- Modify: `src/vergil_tooling/lib/release/prepare.py`,
  `src/vergil_tooling/lib/release/records.py` (absorbed issues;
  `Supersedes:` line)
- Test: `tests/vergil_tooling/test_release_supersede.py` (new),
  `tests/vergil_tooling/test_vrg_release.py`

**Interfaces:**

- Consumes: `read_state` (3b), `_check_no_open_release` (5),
  `record_body` (10).
- Produces: `check_eligible(state: StateBlock) -> None`, raising
  `ReleaseError` unless either the tag and Release exist and every stage
  before `release-records` that is not done is in `{"cd-publish",
  "package-index", "update-deps", "install"}`, or the first undone stage is
  `confirm-main`, no tag exists and `failed_release_run` is set;
  `absorb(old: StateBlock, new_issues) -> tuple[IssueEntry, ...]`;
  `close_superseded(repo, number, new_version) -> None`, called after the
  new tracking issue exists, which also removes an untagged superseded
  release's release worktree and its local and remote `release/X.Y.Z`
  branch.

- [ ] **Step 1: Write the failing tests:**
  `test_supersede_refuses_bare_release`; `test_supersede_eligible_delivery_left`;
  `test_supersede_eligible_untagged_confirm_main`;
  `test_supersede_untagged_takes_next_patch_and_passes_branch_check`;
  `test_supersede_untagged_removes_worktree_and_branch`;
  `test_supersede_refuses_confirm_main_halt_without_failed_release_run`;
  `test_supersede_refuses_merge_release_halt`. The eligibility fixtures are
  state blocks written by a simulated run of the pipeline, not built by
  hand;
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
  "warnings": [str]}`; exit 0 for `pass`, 3 for `fail`, 4 for `none`, 1 for
  an error. The record used is the newest counting one; any others are
  listed in `warnings`. The closing PRs are the task's closing PRs that pass
  Task 6's `trusted_closing_refs`. The ancestry check takes the merge
  commit of every such PR from GitHub (never from the record), and each must
  be an ancestor of the record's tag: `git ls-remote` the tag, then `git
  merge-base --is-ancestor <merge-sha> <tag-sha>` in a temporary bare fetch
  of the task's repo. Comments are read paginated, as in Task 10.

- [ ] **Step 1: Write the failing tests:** `test_check_pass`;
  `test_check_none_when_no_record_exits_4`;
  `test_check_fail_when_ancestry_fails_exits_3`;
  `test_check_two_closing_prs_both_must_be_ancestors`;
  `test_check_two_versions_uses_newest_lists_other`;
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

Blocked by 3a, 3b, 4, 5, 6, 7a, 7b, 8, 9a, 9b, 10, 11 and 12.

- **Before the cut (agent-safe, then human-run):** list the open `release:`
  tracking issues in every fleet repo that releases with `vrg-release`
  (`vrg-gh issue list --repo <repo> --search "release: in:title" --state
  open`). The human finishes each with the installed older tooling: a
  halted release with `vrg-release --resume`; a release the older tooling
  left complete with a deferred publish by the steps its deferral comment
  gives (re-run the named jobs or the index workflow, `vrg-promote` if the
  comment says so, then close the issue). The listing is re-run until it is
  empty.
- **Precondition (human-attested):** the freeze held (no vergil-tooling
  release since Task 2d's); every one of Tasks 3a to 12 is merged; and the
  release carrying them is then cut, published and installed on the host
  and the VMs, its version named in a comment. That
  cut ends the freeze. It is cut with the older tooling, so it posts no
  records (spec §5).
- **Procedure (agent-safe):**
  1. For each of Tasks 3a to 12, check that its PR's merge
     commit is in the release: `vrg-git fetch origin tag <tag>`, `vrg-git
     ls-remote origin refs/tags/<tag> 'refs/tags/<tag>^{}'`, then `vrg-git
     merge-base --is-ancestor <merge-sha> <tag-sha>` exits 0.
  2. `vrg-release --help` lists `--supersede` and not `--no-install`;
     `vrg-submit-pr --help` does not list `--no-finalize`;
     `vrg-release-record --help` exits 0.
  3. Re-run the tracking-issue listing: still empty.
  4. File the fixture runs (Task 18) with the version, then Task 19 blocked
     by the six release tasks `fixture-run` prints.
- **SUCCESS:** every check passes and both filings are done, recorded with
  the version and the filed refs.

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
  `vrg-release-record check`: exit 0 passes; exit 3 (`fail`) is a blocked
  precondition; exit 4 (`none`, a release from before the switch or from a
  repo outside `vrg-release`) falls back to the existing ancestry recipe,
  which stays in the skill; one worked example for each)
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

Filed by Task 13, step 4, once its release version is known:

```bash
fixture-run vergil-project/.github#1440 \
  python-lib,python-app,rust-lib,rust-app,ts-lib,ts-app \
  --validates "vergil-tooling v<version from Task 13>" \
  --after vergil-project/.github#<13>
```

### Task 19: fixture releases carry records (validation, .github)

Filed by Task 13, step 4, with `vrg-issue-create --epic
vergil-project/.github#1440 --repo vergil-project/.github --kind validation
--title "Validate: fixture releases carry release records"`, blocked by the
six release tasks `fixture-run` printed.

- **Precondition:** each fixture run's release task is closed, its version
  and tracking issue named in its closing comment.
- **Procedure (agent-safe):** for each fixture, its release tracking issue
  is locked and closed; `vrg-release-record check` passes on the fixture's
  change task; every issue in the tracking issue's state block carries
  exactly one human-authored record for the version.
- **SUCCESS:** every check passes for all six fixtures.

### Task 20: the deprecated install key goes (vergil-tooling)

Spec §4.6. Blocked by 14a and 14b; cut it into the first minor release after
Task 13's.

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (the `consumer-refresh` key is
  an unknown key again, refused like any other)
- Test: `tests/vergil_tooling/test_config.py`

- [ ] **Step 1: Write the failing test:**
  `test_config_refuses_removed_consumer_refresh_key`.
- [ ] **Step 2:** Run it. Expect FAIL.
- [ ] **Step 3: Implement**, deleting the fallback and its warning.
- [ ] **Step 4:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expect PASS.
- [ ] **Step 5:** Commit `vrg-commit --type feat --scope config --message
  "remove the deprecated consumer-refresh key"`, then report-ready.
