# epic-repo-ownership: epic ownership of repos for coordinated releases — implementation plan

- **Epic:** vergil-project/.github#1818
- **Planning issue:** vergil-project/.github#1678 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each implementation task is one PR, run with
> `issue-implement`; operational tasks are run with `issue-deploy` or
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** an epic whose work must ship together across repos takes
ownership of those repos for the window from its group's first submit to
its last release, so that no other release can cut half of it, and gives
the ownership up once every repo is released.

**Architecture:** three phases, as spec §8 orders them. Phase 1 factors
ref-agnostic helpers out of the release lock's ref module, builds the
ownership ref (`refs/vergil/repo-owner`) and its history on them, adds
`vrg-repo-owner` (`take`, `status`, `give-up`, `break`) and makes
`vrg-release` check ownership twice. Phase 2 adds the `coordinated-release`
operational kind, its `Coordinated release:` lines, the runnability rules
in `vrg-epic-audit`, the submit guard, and the plugin's run skill and skill
changes. Phase 3 shows ownership in `vrg-repo-status`, `vrg-queue hoshin`,
hoshin's daily review and the daily agent audit. A live validation runs one
coordinated release across two fixture repos.

**Tech Stack:** Python 3 (vergil-tooling, pytest), git refs pushed through
the release lock's scratch-repository transport, Markdown skills
(vergil-claude-plugin), Zensical site docs.

**Spec:** [`spec.md`](spec.md) in this directory. Section references (§5.2)
are to it. References to the release lock (#1664) are to
`epics/1664-release-lock/spec.md` and `plan.md`; references to batches
(#1672) are to `epics/1672-batch-release-unit/spec.md`.

## Global Constraints

- Ownership state lives only in refs: `refs/vergil/repo-owner` and
  `refs/vergil/repo-owner-history`, on each repo's origin (§5.1). Never an
  issue, a label or a repo variable.
- Taking is a create-only push (`--force-with-lease=<ref>:`); giving up and
  breaking are compare-and-swap deletes against the sha read; history
  appends are compare-and-swap (§5.1). Every write is reconciled by
  re-reading the ref before a failure is reported, as the release lock's
  `refs.write_ref` does today.
- Ownership refs are read with **no** trusted-author check. A ref that
  cannot be read or parsed is **held**, never absent, everywhere it is read
  (§5.1, §6).
- Only `lib/repo_owner/` writes the ownership refs. `vrg-git` refuses every
  push under `refs/vergil/repo-owner`, a delete included (§5.1).
- Names, fixed here so every task agrees (§7 leaves them to the builder):
  the tool `vrg-repo-owner`, the kind and its label `coordinated-release`,
  the run skill `issue-coordinated-release`, the body line
  `Coordinated release: <owner>/<repo>#<N>`, the record file
  `repo-owner.json`, the history file `repo-owner-history.json`.
- **A run's identity** is a `RunIdentity(identity, host, run_dir, token)`:
  `identity` from `vrg-whoami`, `host` from `socket.gethostname()`,
  `run_dir` the absolute path
  `<home main clone>/build/.vergil-state/coordinated-release/<N>/` (`<N>`
  the coordinated release task's number), and `token` a random hex string
  written once to `<run_dir>/run.json` and read back by every later step
  and by a resumed run on the same host. Two identities match only when all
  four fields are equal.
- `vrg-repo-owner take` exit codes: `0` taken; `3` held by another task;
  `4` held by another run of this task, or by this task with another group;
  `5` a ref unreadable; `1` any other failure (a missing main clone, a
  transport error, a failed give-back); `2` usage. `--json` carries the
  same kind as `held_by`: `"task"`, `"run"` or `"unreadable"`. The run
  waits only on `3` (§5.2, §5.5).
- The wait's retry interval is `OWNERSHIP_RETRY_SECS = 300`, in
  `lib/repo_owner/model.py` and nowhere else (§5.5, §7).
- `vrg-release` and `vrg-repo-owner break` are human only, refused when
  `identity_mode.is_agent() or identity_mode.any_agent_signal()`, the guard
  `bin/vrg_release.py` uses today (§5.2, §5.3).
- Every read of a `Coordinated release:` line goes through the trust layer
  (`vrg-issue-read`'s library path, `lib/trust.py`) and fails closed on
  untrusted content, as `Blocked-by:` reads do (§5.4).
- A repo's share of a coordinated group is either exactly one task that is
  in no release group, which counts as an implicit one-member group, or
  exactly one release group of #1672 (#1672 spec §5.9), read only through
  #1672's group tool list command. A share of several tasks must be one
  release group. No task here parses group records itself.
- A share in another org is filed under the primary epic's proxy epic
  there, and its release group, when it has several tasks, lives on that
  proxy epic. This needs #1672 to allow a group on a proxy epic for the
  proxy's own-org tasks ("Notes for `file-tasks`").
- No silent failures: a failed give-back, give-up, history append or issue
  read stops with an error naming what is still held (§6).
- Every vergil-tooling task is test-first and ends with
  `vrg-container-run -- vrg-validate` green and a `vrg-commit`.
- Releases are human actions. A deployment task's precondition is
  `vrg-release-record check` on each task it ships; it never reinstalls
  what the release installed. For the plugin, the left-over step is
  `/reload-plugins` in each running session, a human-run step (plugin
  `issue-deploy/references/release-coverage.md`).

## Review Focus

1. **Two epics racing for overlapping groups.** Epic A takes tooling and
   plugin while epic B takes plugin and actions, at the same moment.
   Expected: each repo has exactly one owner; the loser gives back
   everything it took and holds nothing; neither deadlocks (Task 3,
   `test_overlapping_takes_one_wins_loser_holds_nothing`).
2. **A crash partway through a take, then a resumed run.** The first of two
   repos is taken and the session dies. Expected: a resumed run on the same
   host adopts the first record and takes the second; a run on another host
   gets exit 4 and stops for the human, giving back nothing it does not own
   (Task 3, `test_rerun_same_run_adopts_and_completes`,
   `test_rerun_other_host_exit_4_gives_back_nothing_foreign`).
3. **Ownership taken while a release waits in its pre-check.** A plain
   `vrg-release` passes its first check, waits on develop CD, and meanwhile
   an epic takes the repo and its batch merges. Expected: the second check,
   after the disarm, aborts the release through the release lock's full
   unlock path, naming the owner (Task 5,
   `test_taken_between_checks_aborts_through_unlock`).
4. **A corrupt ownership ref.** The ref points at a commit whose file is not
   JSON. Expected: `take` exits 5, `status` reports it unreadable, a plain
   `vrg-release` fails, the submit guard refuses, and only `break` removes
   it (Task 2, `test_unparseable_owner_ref_reads_as_held`; Task 5,
   `test_unreadable_owner_fails_plain_release`; Task 9,
   `test_guard_refuses_on_unreadable_owner`).
5. **A batch PR that carries one coordinated member among standalone
   tasks**, submitted outside the window. Expected: `vrg-submit-pr` refuses
   the whole PR, naming the member and its coordinated release task (Task
   9, `test_batch_with_one_coordinated_member_refused_unowned`).

---

## Delivery

Every task is filed under the epic, vergil-project/.github#1818, in the repo
its PR lands in. The plan task ID is the `Plan task:` key `file-tasks`
writes. External blockers are named by their full ref; `#1672/<what>` names
a task of epic vergil-project/.github#1672 that does not yet exist (see
"Notes for `file-tasks`").

| Task | Repo | Kind | What | Blocked by |
|---|---|---|---|---|
| 1 | `.github` | task | Spec and plan (vergil-project/.github#1819) | — |
| **Phase 1** | | | | |
| 2 | `vergil-tooling` | task | Ref-agnostic helpers (`lib/ref_store/`) factored out of `lib/release_lock/`; the ownership record, ref and history on them; `vrg-git` refuses pushes to the ownership refs | vergil-project/vergil-tooling#5263, vergil-project/vergil-tooling#5275 |
| 3 | `vergil-tooling` | task | `vrg-repo-owner take`, `give-up` and `status`: run identity, main-clone resolution, all-or-nothing take, staleness | 2 |
| 4 | `vergil-tooling` | task | `vrg-repo-owner break`: human only, history, a comment on the owning task | 3 |
| 5 | `vergil-tooling` | task | `vrg-release --epic --task` and the two ownership checks | 3 |
| 6 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 2, 3, 4 and 5 (the release gate) | 2, 3, 4, 5 |
| **Phase 2** | | | | |
| 7 | `vergil-tooling` | task | The `coordinated-release` kind in every place that lists the kinds; `lib/coordinated_release.py` reads the group; `vrg-issue-create` refuses a link to the task's own group | 3, `#1672/group-list` |
| 8 | `vergil-tooling` | task | `vrg-epic-audit`: a coordinated release task runnable at report-ready of its group, and in its window while it owns the repos; a same-group `Blocked-by:` met at report-ready; the refused link reported | 7 |
| 9 | `vergil-tooling` | task | `vrg-submit-pr` submits a PR with a coordinated member only while its task owns the repo | 7, `#1672/whole-group-rule` |
| 10 | `vergil-tooling` | task | Visibility: owner in `vrg-repo-status`, owned repos in `vrg-queue hoshin`, stale and unreadable ownership in the daily agent audit | 3 |
| 11 | `vergil-tooling` | deployment | A vergil-tooling release carries Tasks 7–10 (the release gate) | 6, 7, 8, 9, 10 |
| 12 | `vergil-claude-plugin` | task | The run skill `issue-coordinated-release` | 11 |
| 13 | `vergil-claude-plugin` | task | `epic-implement` and `issue-implement`: hand-off at report-ready, re-entry, dispatch of the new kind | 11 |
| 14 | `vergil-claude-plugin` | task | `epic-create`: the planning rule and the filing of coordinated release tasks and their lines | 11, `#1672/epic-create-groups` |
| **Phase 3** | | | | |
| 15 | `vergil-claude-plugin` | task | Hoshin's daily review lists owned repos and flags stale ownership | 11 |
| 16 | `vergil-claude-plugin` | deployment | A plugin release carries Tasks 12–15; every running session reloads its plugins | 12, 13, 14, 15 |
| 17 | `vergil-tooling` | validation | A live coordinated release across two fixture repos | 11, 16 |
| 18 | fixtures | fixture run | One run on `python-lib` against the Task 11 release | 11 |
| doc-review | `vergil-tooling` | task | `Documentation review: epic-repo-ownership` | 17, 18 |
| retrospective | `.github` | retrospective | `Retrospective: epic-repo-ownership` | all others |

### Phase exits

- **Phase 1** ends when Task 6 closes: ownership can be taken, shown, given
  up and broken, and `vrg-release` refuses to cut an owned repo for anyone
  but its owner. Nothing yet takes ownership but a human at the shell.
- **Phase 2** ends when Task 16 closes: plans can name coordinated release
  tasks and `epic-implement` runs them.
- **Phase 3** ends with Task 17, the first coordinated release run live.

### Operational tasks

- **Task 6** is the release gate of phase 1. Procedure:
  `vrg-release-record check` exits 0 for Tasks 2, 3, 4 and 5; nothing is
  installed (the release did it). Acceptance: each check passes; the result
  names the release version.
- **Task 11** is the release gate of phase 2's tooling and of Task 10, as
  Task 6 for Tasks 7–10. The plugin tasks need it: they call the new kind,
  the audit's runnability and `vrg-repo-owner`.
- **Task 16** ships the plugin. Precondition: `vrg-release-record check`
  exits 0 for Tasks 12–15. Human-run steps: `/reload-plugins` in each
  running session. Acceptance: the checks pass and the human confirms the
  reload.
- **Task 17** is the live validation, its own section below.
- **Task 18** is a fixture run: `vrg-release` changes, but the change is a
  language-independent read, so one fixture covers the plain (unowned)
  path, and Task 17 covers the owned path on two others:

  ```bash
  fixture-run vergil-project/.github#1818 python-lib \
    --validates "vrg-release's ownership check on an unowned repo, vergil-tooling as released by Task 11" \
    --after <Task 11 ref>
  ```

  Its three tasks are keyed `Plan task: 18 python-lib`.

### Notes for `file-tasks`

- **Built in series (§6).** This epic's tasks are filed only once the
  batch epic's (vergil-project/.github#1672) plan is published and its tasks
  are filed, and its spec still records release groups with the §5.9
  interface and allows a group on a proxy epic for the proxy's own-org
  tasks (its group tool and record check accepting a proxy epic on those
  terms, and `groups_of` returning a proxy task's groups from its proxy).
  If it does not, this spec's group source is revised first, in a new spec
  round, before anything is filed.
- **The `#1672/…` blockers.** Map each to the #1672 task that delivers it,
  by its full ref, and write that ref into the `Blocked-by:` lines:
  - `#1672/group-list`: the group tool's list command and the group record
    format (#1672 spec §8 puts it ahead of its hardening work);
  - `#1672/whole-group-rule`: the whole-group rule on the one PR path
    (#1672 spec §5.2);
  - `#1672/epic-create-groups`: the `epic-create` change that first writes
    release groups (#1672 spec §8).

  If #1672's plan delivers one of these in several tasks, list each.
- **Task 2's `vergil-project/vergil-tooling#5275` blocker** is the release
  lock's phase 2 release gate: refactoring `lib/release_lock/` while #1664
  still changes it would collide, and the three epics are built in series.
- **Advisories.** Two advisories track the planning issue
  (`vrg-security-record list` names them). Before `Tasks filed: yes`,
  retarget each from vergil-project/.github#1678 to the task or tasks that
  fix it, chosen from each advisory's own notes, one after another. All the
  fix tasks are in `vergil-project`, so no advisory moves.
- **No coordinated release task is filed for this epic's own work.** Its
  tooling and plugin changes are ordered by deployment tasks, and nothing
  of them must ship together: every task is releasable on its own.

---

## Phase 1: ownership

### Task 2: Ref helpers, the ownership record and its refs (vergil-tooling)

Spec §5.1.

**Files:**

- Create: `src/vergil_tooling/lib/ref_store/__init__.py`, `transport.py`,
  `history.py` (not `vergil_refs`: `lib/vergil_refs.py` already exists, the
  ecosystem version-pin helpers, and its tests are
  `tests/vergil_tooling/test_vergil_refs.py`; both stay as they are)
- Modify: `src/vergil_tooling/lib/release_lock/refs.py`,
  `src/vergil_tooling/lib/release_lock/history.py`,
  `src/vergil_tooling/lib/release_lock/arming.py` (onto the helpers; no
  behavior change)
- Create: `src/vergil_tooling/lib/repo_owner/__init__.py`, `model.py`,
  `refs.py`, `history.py`
- Modify: `src/vergil_tooling/bin/vrg_git.py` (a `_REPO_OWNER_PREFIX =
  "refs/vergil/repo-owner"` refusal beside `_RELEASE_LOCK_PREFIX`)
- Modify: `tests/vergil_tooling/test_branch_push_guard.py` (adds
  `lib/ref_store/` and `lib/repo_owner/` to the allowed sites, each with
  its reason)
- Test: `tests/vergil_tooling/test_ref_store.py`,
  `test_repo_owner_refs.py`, `test_repo_owner_model.py`, and the existing
  `test_release_lock_*.py` unchanged and passing

**Interfaces:**

- Produces, `ref_store.transport` (moved from `release_lock.refs`,
  re-exported there so callers keep working): `scratch(repo, error)`,
  `list_remote`, `remote_sha`, `fetch`, `read_file`, `build`, `write_ref`,
  each with its current signature. The error classes move with them:
  `ref_store` defines `RefError` and its subclasses `RefReadError` /
  `RefWriteError`, which the moved functions raise. `release_lock.model`
  re-exports those two classes under its own names (aliases, not
  subclasses), and `ReleaseLockError` subclasses `ref_store.RefError`,
  so every existing `except RefReadError`, `except RefWriteError` and
  `except ReleaseLockError` in the release lock and its callers still
  catches what the moved functions raise. `repo_owner` adds subclasses only
  for its own parse errors.
- Produces, `ref_store.history`:

  ```python
  class AppendedHistory:
      def __init__(self, ref: str, filename: str, actions: tuple[str, ...],
                   fields: tuple[str, ...], message_prefix: str) -> None: ...
      def append(self, repo: str, entry: Mapping[str, str | list[str]]) -> None: ...
      def read(self, repo: str) -> list[dict[str, str | list[str]]]: ...
  ```

  `append` refuses an `action` not in *actions* and a field set other than
  *fields*; its retry and buried-entry rules are today's
  `release_lock.history.append`. `release_lock.history` keeps
  `HistoryEntry`, `append` and `read_history`, built on it.
- Produces, `repo_owner.model`:

  ```python
  @dataclass(frozen=True)
  class RunIdentity:
      identity: str
      host: str
      run_dir: str
      token: str

  @dataclass(frozen=True)
  class OwnerRecord:
      epic: str                 # "<owner>/<repo>#<n>", the primary epic
      task: str                 # the coordinated release task
      group: tuple[str, ...]    # full repo names, in take order (sorted)
      run: RunIdentity
      taken_at: str             # ISO 8601 UTC

  def to_json(rec: OwnerRecord) -> str: ...
  def from_json(text: str) -> OwnerRecord: ...   # raises OwnerParseError
  OWNERSHIP_RETRY_SECS = 300
  ```

- Produces, `repo_owner.refs`: `OWNER_REF = "refs/vergil/repo-owner"`,
  `HISTORY_REF = "refs/vergil/repo-owner-history"`;
  `OwnerState(sha: str | None, record: OwnerRecord | None, error: str)` with
  `held` (`sha is not None`); `read_owner(repo) -> OwnerState`,
  `take_owner(repo, rec) -> str | None` (create-only),
  `delete_owner(repo, *, expected: str) -> bool` (CAS).
- Produces, `repo_owner.history`: `append(repo, *, action:
  Literal["take", "give-up", "break", "break-not-applied"], record: str, actor: str, at: str,
  reason: str) -> None` (`record` the record's file text as read, even
  unparsed) and `read_history(repo)`.

- [ ] **Step 1: Failing tests:**
  - `test_release_lock_tests_pass_unchanged` (the existing release-lock
    suites, run as the refactor's guard).
  - `test_unreadable_lock_still_held_after_move` (a fetch failure in the
    moved transport still makes `read_lock` report the lock held).
  - `test_release_lock_error_names_catch_moved_errors` (each release-lock
    `except` class catches what `ref_store` raises).
  - `test_appended_history_refuses_unknown_action`,
    `test_appended_history_concurrent_both_kept`.
  - `test_take_owner_create_only_second_returns_none`.
  - `test_delete_owner_cas_mismatch_keeps_ref`.
  - `test_unparseable_owner_ref_reads_as_held` (`held` true, `record` None,
    `error` names the parse failure).
  - `test_owner_read_has_no_trusted_author_check`.
  - `test_applied_then_failed_push_reconciled_as_success` for take, delete
    and append.
  - `test_model_round_trip`, `test_model_rejects_unsorted_group`,
    `test_model_rejects_unqualified_refs`.
  - `test_vrg_git_refuses_push_to_repo_owner_refs` (a push, a delete and a
    refspec with a wildcard that could match).
- [ ] **Step 2: Run them and see them fail.**
  `vrg-container-run -- pytest tests/vergil_tooling/test_ref_store.py
  tests/vergil_tooling/test_repo_owner_refs.py
  tests/vergil_tooling/test_repo_owner_model.py` → FAIL.
- [ ] **Step 3: Implement.** Move first, run the release-lock suites green,
  then add `repo_owner` on the helpers.
- [ ] **Step 4: Pass; `vrg-container-run -- vrg-validate`; commit.**

### Task 3: `vrg-repo-owner take`, `give-up` and `status` (vergil-tooling)

Spec §5.1 (transport), §5.2, §5.6 (staleness).

**Files:**

- Create: `src/vergil_tooling/lib/repo_owner/run.py`, `clones.py`,
  `take.py`, `stale.py`, `status.py`,
  `src/vergil_tooling/bin/vrg_repo_owner.py`
- Modify: `pyproject.toml` (the `vrg-repo-owner` entry point)
- Test: `tests/vergil_tooling/test_repo_owner_take.py`,
  `test_repo_owner_stale.py`, `test_repo_owner_clones.py`,
  `test_vrg_repo_owner.py`

**Interfaces:**

- Consumes: Task 2's `repo_owner.refs`, `repo_owner.history`,
  `repo_owner.model`.
- Produces, `run.py`: `load_or_create(run_dir: Path) -> RunIdentity`
  (creates `run.json` with a fresh token once; later calls read it) and
  `current(run_dir: Path) -> RunIdentity` (reads only; raises
  `RunDirError` when absent).
- Produces, `clones.py`: `main_clone(repo: str, roots: Sequence[Path] |
  None) -> Path`, built on the fleet's existing clone discovery, not a new
  layout: it finds the clone by its `origin` with `in_flight.find_clones`
  over `live_branches.discover_repos`, with *roots* meaning what they mean
  to `vrg-queue` and `vrg-repo-status` (projects-level roots, any depth of
  `<org>/<repo>` below them; default `live_branches.default_repo_roots()`),
  so one root covers every org. It raises `MissingCloneError` naming the
  repo, and the existing duplicate-clone error for two clones of one repo.
  Every operation resolves every repo first, so a missing clone stops it
  before anything is written (§5.1). The refs themselves are written through Task 2's scratch
  transport, whose `origin` is the repo, as the release lock's are; the
  clone is the one the run submits from (§5.5 step 1).
- Produces, `take.py`:

  ```python
  HeldBy = Literal["task", "run", "unreadable"]

  @dataclass(frozen=True)
  class Held:
      repo: str
      kind: HeldBy
      epic: str          # "" when unreadable
      task: str
      run: RunIdentity | None

  @dataclass(frozen=True)
  class TakeResult:
      taken: tuple[str, ...]      # repos now owned by this run
      held: Held | None           # the first hold met; None on success
      given_back: tuple[str, ...]

  def take(*, epic: str, task: str, repos: Sequence[str], run: RunIdentity) -> TakeResult: ...
  def give_up(*, task: str, repos: Sequence[str] | None, run: RunIdentity) -> tuple[str, ...]: ...
  ```

  `take` walks the repos sorted; a record naming this task, this group and
  this run is adopted; on the first other hold it gives back, by CAS, every
  repo whose record is bound to *run*, and returns the hold. A give-back
  that fails raises `GiveBackError` naming each repo still held. A
  successful `take` writes the repos it took to `<run_dir>/taken.json`
  (`run.dir` of *run*), replacing any earlier copy. `give_up` with *repos*
  `None` gives up the repos `taken.json` lists, raising `RunDirError` when
  it is absent, and checks each record against *run* as for named repos;
  it raises
  `NotBoundError` naming the bound run (and `vrg-repo-owner break`) for a
  record bound to another run, and `ForeignOwnerError` for one naming
  another task. Each take and give-up appends a history entry with
  `actor` the run's identity.
- Produces, `stale.py`: `judge(state: OwnerState) -> Staleness(stale: bool,
  reason: str)`: absent → not stale; unreadable → not stale, reason
  `unreadable`; stale iff the task or the epic is closed. A failed issue
  read raises; it never means stale.
- Produces, the CLI:
  - `vrg-repo-owner take --epic <ref> --task <ref> --run-dir <path> --repo
    <owner>/<repo> [--repo …] [--root DIR …] [--wait] [--json]`, exit
    codes as the Global Constraints give them. Text names, for each hold,
    the repo, its epic, task and run, and `stale` for a stale holder.
    `--wait` retries the whole take every `OWNERSHIP_RETRY_SECS` while the
    result is a hold by another task (exit 3), holding nothing between
    attempts and printing a line each time the holder changes, and returns
    the first other result.
  - `vrg-repo-owner give-up --task <ref> --run-dir <path> [--repo …]
    [--root DIR …]`; exit 0, or 1 naming what is still held.
  - `vrg-repo-owner status [--repo …] [--root DIR …] [--json]`, any
    identity; with no `--repo`, every repo `live_branches.discover_repos`
    finds under the roots whose origin is in an org with a `.github` repo.
    `--root` means the same in every operation, and with no `--root` the
    defaults of `live_branches.default_repo_roots()` apply. JSON: a list of `{repo,
    owned, epic, task, group, run, taken_at, age_secs, stale,
    stale_reason, error}`. Exit 0 on a complete read, 1 if any repo could
    not be read (still printing the rest).

- [ ] **Step 1: Failing tests** (two bare repos as `origin`s, main clones
  under a temp root):
  - `test_take_group_sorted_order_records_group_and_run`.
  - `test_overlapping_takes_one_wins_loser_holds_nothing` (threads racing
    `take` on overlapping groups; afterwards every repo has one owner and
    the loser's `taken` is empty).
  - `test_rerun_same_run_adopts_and_completes`.
  - `test_rerun_other_host_exit_4_gives_back_nothing_foreign`.
  - `test_same_task_other_group_is_held_by_run_kind`.
  - `test_held_by_other_task_exit_3_json_names_epic_task_run`.
  - `test_unreadable_ref_exit_5`.
  - `test_missing_main_clone_takes_nothing_exit_1`.
  - `test_clone_found_across_orgs_under_one_projects_root` (clones laid out
    `<root>/<org>/<repo>` for two orgs; one root resolves both).
  - `test_duplicate_clone_refused`.
  - `test_status_lists_repos_under_projects_root`.
  - `test_wait_retries_only_on_task_hold` (a hold by another run ends the
    wait at once with exit 4).
  - `test_wait_holds_nothing_between_attempts`.
  - `test_give_back_failure_names_repos_still_held` (transport fails on the
    second delete).
  - `test_give_up_refuses_other_run_and_names_break`.
  - `test_give_up_refuses_foreign_task`.
  - `test_give_up_unowned_counts_as_given_up`.
  - `test_give_up_no_repo_reads_taken_json`,
    `test_give_up_no_repo_without_taken_json_fails`.
  - `test_every_take_and_give_up_in_history_with_actor`.
  - `test_stale_when_task_closed`, `test_stale_when_epic_closed`,
    `test_issue_read_failure_raises`.
  - `test_status_json_fields`, `test_status_unreadable_repo_exit_1_lists_rest`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 4: `vrg-repo-owner break` (vergil-tooling)

Spec §5.2 (`break`).

**Files:**

- Create: `src/vergil_tooling/lib/repo_owner/breaking.py`
- Modify: `src/vergil_tooling/bin/vrg_repo_owner.py`
- Test: `tests/vergil_tooling/test_repo_owner_break.py`

**Interfaces:**

- Produces: `vrg-repo-owner break --repo <owner>/<repo> --reason "<why>"
  [--root DIR …]`. Refused before any read under any agent identity or
  signal. In order: read the ref; append a `break` entry with the old
  record's file text (parsed or not) and the reason; comment on the owning
  task, when the record parses, with
  `Ownership of <repo> broken by <identity>: <reason>`; delete the ref by
  CAS. A CAS mismatch deletes nothing: it re-reads the ref, appends a
  `break-not-applied` entry with what is there now, posts a follow-up
  comment on the task saying the break did not apply and naming what is
  there now, and reports it. An unreadable ref is broken like any other,
  with no comment.
- Produces: `breaking.break_owner(repo: str, *, reason: str, actor: str)
  -> None`.

- [ ] **Step 1: Failing tests:** `test_break_refused_under_agent_signal`
  (each of `is_agent` and `any_agent_signal`),
  `test_break_records_history_then_comments_then_deletes`,
  `test_break_unreadable_ref_records_raw_text_no_comment`,
  `test_break_cas_mismatch_deletes_nothing` (and records
  `break-not-applied` in the history and on the task),
  `test_break_requires_reason`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 5: `vrg-release` checks ownership (vergil-tooling)

Spec §5.3. Built on the release lock's pre-check and quiesce
(vergil-project/vergil-tooling#5269) and `--supersede`
(vergil-project/vergil-tooling#5271); Task 2's blocker
(vergil-project/vergil-tooling#5275) covers both.

**Files:**

- Create: `src/vergil_tooling/lib/release/ownership.py`
- Modify: `src/vergil_tooling/bin/vrg_release.py` (`--epic`, `--task`,
  given together or not at all)
- Modify: `src/vergil_tooling/lib/release/preflight.py` (the first check,
  in the pre-check before the lock is taken),
  `src/vergil_tooling/lib/release/orchestrator.py` (the second check, once
  merges are disarmed, #1664 spec §5.3 after step 1.4)
- Modify: `src/vergil_tooling/lib/release/context.py` (the two refs on the
  context; a resume rebuilds nothing from them)
- Test: `tests/vergil_tooling/test_release_ownership.py`,
  `test_release_orchestrator.py`

**Interfaces:**

- Consumes: Task 2's `repo_owner.refs.read_owner`.
- Produces: `ownership.check(repo: str, *, epic: str | None, task: str |
  None) -> None`, raising `OwnershipRefused(ReleaseError)` whose message
  names the owner (or the read error) and tells the human to reconcile
  with it. Rules: with both, pass only if the record names that epic and
  task; with neither, pass only if the repo is unowned; unreadable refuses
  either way.

- [ ] **Step 1: Failing tests:**
  - `test_epic_and_task_must_be_given_together`.
  - `test_plain_release_unowned_runs`.
  - `test_plain_release_owned_fails_naming_owner`.
  - `test_unreadable_owner_fails_plain_release`, and with `--epic`.
  - `test_epic_task_match_runs`, `test_epic_match_other_task_refused`.
  - `test_first_check_before_lock_taken` (an owned repo: the lock ref is
    never written).
  - `test_taken_between_checks_aborts_through_unlock` (the pre-check
    passes, ownership is taken during the CD wait; the second check raises
    and the release lock's unlock path re-arms and deletes the lock).
  - `test_supersede_checks`, `test_resume_does_not_check`.
  - `test_release_never_writes_owner_refs`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 6: Release phase 1 (deployment)

As "Operational tasks" above.

---

## Phase 2: the coordinated release task

### Task 7: The `coordinated-release` kind and its group (vergil-tooling)

Spec §3 (the five places), §5.4.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_issue_create.py`
  (`_OPERATIONAL_KINDS`, the `--kind` choices, `_SCAFFOLD_INTRO`,
  `_SLOT_PLACEHOLDERS`, and for the optional `Release order` section
  `_SECTION_SLOTS`, `_OPTIONAL_SECTIONS` and, if it is framed like
  `Human-run steps`, `_OPTIONAL_SLOT_FRAMES`)
- Create: `src/vergil_tooling/data/coordinated-release_task_body.md` (the
  name `_scaffold_body` reads, `<kind>_task_body.md`)
- Modify: `src/vergil_tooling/lib/queue_view.py` (`_OPERATIONAL_KINDS`),
  `src/vergil_tooling/lib/epics.py` (`_OPERATIONAL_LABELS`),
  `src/vergil_tooling/lib/pr_workflow/batch_rules.py` (the refusal text
  naming the kinds), `src/vergil_tooling/data/labels.json`
- Create: `src/vergil_tooling/lib/coordinated_release.py`
- Test: `tests/vergil_tooling/test_vrg_issue_create.py`,
  `test_coordinated_release.py`, `test_labels.py`

**Interfaces:**

- Consumes: #1672's group tool list command (its library function; the
  name its plan gives).
- Produces: the label `coordinated-release` (`"Coordinated release task:
  submits a cross-repo group inside an ownership window; never
  auto-closed"`). The scaffold's sections: `Intro`, `Preconditions`,
  `Procedure`, `Acceptance`, and `Release order` (optional; one line per
  ordered pair, `<repo> before <repo>: <why>`). The `Procedure` default
  names the run skill.
- Produces, `coordinated_release.py`:

  ```python
  LINE = re.compile(r"^Coordinated release: ([\w.-]+/[\w.-]+#[1-9][0-9]*)$")

  @dataclass(frozen=True)
  class Group:
      task: str                                  # the coordinated release task
      members: tuple[str, ...]                   # every task naming it, sorted
      repos: tuple[str, ...]                     # sorted, one per PR
      by_repo: Mapping[str, tuple[str, ...]]

  def line_of(ref: str) -> str | None: ...       # trusted read; raises UntrustedContentError
  def find_group(epic: str, task: str) -> Group: ...
  def check_group(group: Group) -> None: ...     # raises GroupError
  ```

  `find_group` lists the primary epic's tasks and its proxies' tasks as
  `vrg-epic-audit` does (`epic_audit` / `proxies`), reading each task's
  line through the trust layer. `check_group` accepts a repo's share when
  it is exactly one task in no release group (an implicit one-member
  group), or exactly one release group of #1672, read from the share's own
  epic: the primary epic for its own-org shares, the proxy epic for a share
  in another org. It refuses any other share (several tasks that are not
  one group, a partial group, a task in a group with members outside the
  share), naming the repo and the difference, so each repo has one PR
  (§5.4).
- Produces: `vrg-issue-create --kind coordinated-release` refuses a
  `--blocked-by` naming a task that carries a `Coordinated release:` line,
  naming it. In `file-tasks`' order (the coordinated release task first,
  its members after with their lines) no member has a line yet, so a hit
  means the order was broken; and downstream work that needs a group
  released is blocked by that group's coordinated release task, never by
  its members (§5.4), so no legitimate link is refused. A link added
  later, by an edit, is caught by Task 8's audit.

- [ ] **Step 1: Failing tests:**
  - `test_kind_in_every_operational_list` (one test reading all five
    places, so a sixth forgotten place fails it).
  - `test_coordinated_release_scaffold_sections_and_release_order`.
  - `test_body_file_unknown_section_refused` for the new kind.
  - `test_find_group_reads_primary_and_proxy_tasks`.
  - `test_untrusted_line_fails_closed`.
  - `test_check_group_two_prs_in_one_repo_refused`.
  - `test_check_group_partial_release_group_refused`.
  - `test_check_group_one_member_group_accepted`,
    `test_check_group_single_ungrouped_task_accepted`,
    `test_check_group_several_ungrouped_tasks_refused`.
  - `test_check_group_proxy_share_reads_group_on_proxy` (a share of two
    tasks under a proxy epic, one release group on the proxy: accepted),
    `test_check_group_proxy_single_ungrouped_task_accepted`.
  - `test_coordinated_task_blocked_by_lined_task_refused`.
  - `test_batch_rules_text_names_coordinated_release`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 8: Runnability in `vrg-epic-audit` (vergil-tooling)

Spec §5.4 (report-ready hand-off, same-group `Blocked-by:`, runnable).

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (`_blockers_closed`,
  `operational_status`, `child_status`)
- Create: `src/vergil_tooling/lib/pr_workflow/reported.py`
- Test: `tests/vergil_tooling/test_epic_audit.py`,
  `test_pr_workflow_reported.py`

**Interfaces:**

- Consumes: Task 7's `coordinated_release.find_group`, `line_of`; Task 2's
  `repo_owner.refs.read_owner`.
- Produces, `reported.py`: `reported_ready(repo: str, issue: str) ->
  WorkflowState | None`: the relay state whose `issues` (or `issue`) names
  the task and whose `status` is report-ready; `None` otherwise. It never
  reads the state's `submitted` field: submit records that only in the
  local-file transport, and `vrg-finalize-pr` deletes the relay ref, so the
  relay is no evidence of a submit. `submitted(repo: str, issue: str) ->
  str | None`: the URL of an open PR whose trusted closing references name
  the task (`in_flight`'s `open-pr` signal, through
  `pending_release.trusted_closing_refs`), `None` otherwise; a merged PR
  closes the task, which the audit reads as closed. A read failure raises.
  `reported_ready` lists and fetches `refs/vergil/pr-workflow/*` from a
  scratch repository (Task 2's `ref_store.transport.scratch`), so it
  needs no clone and runs in the CI audit too, and it applies the relay's
  trusted-author rule (`trust.require_trusted_commit`, as
  `GitHubTransport.read` does). A read failure raises.
- Produces, the audit rules:
  - a member is **met** when it is reported ready (`reported_ready`),
    submitted (`submitted`, an open PR closing it) or closed;
  - a coordinated release task is runnable when every member of its group
    is met and every blocker outside its group is closed; otherwise
    blocked, naming the members not met. While Task 2's `read_owner` shows
    any repo of its group owned by this task, it is listed `in-window`
    instead, never blocked, and `epic-implement` dispatches it as runnable,
    so a resumed run always reaches Task 12's gate;
  - a task blocked by a task of its own group (both lines name the same
    coordinated release task) counts that blocker met once it is met;
  - a `Blocked-by:` from a coordinated release task to a member of its own
    group is reported as a drift finding, `coordinated-self-block`, and
    the task is listed `stopped`, never runnable.

- [ ] **Step 1: Failing tests:**
  `test_coordinated_task_blocked_until_all_members_reported`,
  `test_coordinated_task_runnable_when_members_reported`,
  `test_coordinated_task_waits_on_outside_blocker`,
  `test_same_group_blocker_met_at_report_ready`,
  `test_other_group_blocker_still_needs_close`,
  `test_self_block_reported_and_stopped`,
  `test_reported_ready_untrusted_relay_raises`,
  `test_coordinated_task_not_blocked_after_member_submitted`,
  `test_coordinated_task_in_window_while_owned`,
  `test_same_group_blocker_met_once_submitted`,
  `test_submitted_reads_open_pr_closing_task`,
  `test_submitted_ignores_untrusted_closing_reference`,
  `test_member_met_after_relay_ref_deleted` (the relay ref deleted by
  finalize, the open PR still there: met). The submit side is tested
  against the real write paths (`record_submission`, `_delete_relay_ref`),
  not a stub that sets `submitted` on the relay.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 9: The submit guard (vergil-tooling)

Spec §5.5 ("Submission is guarded in the tooling").

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (the check before the
  PR is opened or armed, every mode),
  `src/vergil_tooling/lib/pr_path.py` (the same check in `arm` and
  `push_to_open_pr`, beside #1672's whole-group rule)
- Create: `src/vergil_tooling/lib/coordinated_guard.py`
- Test: `tests/vergil_tooling/test_coordinated_guard.py`,
  `test_vrg_submit_pr.py`

**Interfaces:**

- Consumes: Task 7's `line_of`; Task 2's `read_owner`.
- Produces: `coordinated_guard.check(repo: str, issues: Sequence[str]) ->
  None`, raising `OutsideWindowError` naming each member, its coordinated
  release task, and the repo's current owner (or that it is unowned or
  unreadable). Every issue the PR closes is read; any one carrying a line
  requires the repo's record to name that line's task. Two members naming
  different coordinated release tasks are refused.

- [ ] **Step 1: Failing tests:**
  `test_single_coordinated_task_refused_unowned`,
  `test_batch_with_one_coordinated_member_refused_unowned`,
  `test_submit_allowed_when_owned_by_named_task`,
  `test_owned_by_other_task_refused`,
  `test_guard_refuses_on_unreadable_owner`,
  `test_members_naming_two_tasks_refused`,
  `test_no_coordinated_member_unchanged` (an ordinary PR is not read
  against ownership at all), `test_arm_and_push_paths_run_guard`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 10: Visibility (vergil-tooling)

Spec §5.6. Placed here so it ships in Task 11's release with the rest.

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_repo_status.py` (an `Ownership:`
  section beside `Release lock:`),
  `src/vergil_tooling/lib/queue_view.py` (the hoshin view gains
  `owned_repos`), `src/vergil_tooling/lib/agent_audit.py` (checks
  `stale-ownership` and `unreadable-ownership`, report-only, for the
  audit's own repo)
- Test: `tests/vergil_tooling/test_vrg_repo_status.py`,
  `test_queue_view.py`, `test_agent_audit.py`

**Interfaces:**

- Consumes: Task 3's `repo_owner.status` and `repo_owner.stale.judge`.
- Produces: `vrg-queue hoshin --json` gains `owned_repos`: a list of
  `{repo, epic, task, age_secs, stale, stale_reason, error}`, from
  `status` over the roots it already takes (the same projects-level roots,
  as Task 3's `clones.py` reads them). Text mode prints one line per
  owned repo, `!` before a stale or unreadable one.

- [ ] **Step 1: Failing tests:** `test_repo_status_ownership_section`,
  `test_repo_status_unowned_says_so`, `test_hoshin_json_owned_repos`,
  `test_hoshin_flags_stale`, `test_audit_reports_stale_ownership`,
  `test_audit_reports_unreadable_ownership`,
  `test_audit_never_writes_owner_refs`.
- [ ] **Step 2–4: fail, implement, pass, validate, commit.**

### Task 11: Release phase 2's tooling (deployment)

As "Operational tasks" above.

### Task 12: The run skill `issue-coordinated-release` (vergil-claude-plugin)

Spec §5.5, every step.

**Files:**

- Create: `skills/issue-coordinated-release/SKILL.md`
- Modify: the plugin's skill index and its operational-kind table, where
  the plugin lists `issue-validate` and `issue-deploy`
- Test: the plugin's skill lint (`vrg-container-run -- vrg-validate`)

The skill, in order (each a section):

1. **Preflight and the resumed-run gate.** User mode; read the task with
   `vrg-issue-read`; `run_dir` from the Global Constraints. Read the group
   (`vrg-epic-audit`, which lists its members and their report-ready
   state). If any member's PR is submitted (an open PR whose trusted
   closing references name it, or a merged one, read from GitHub as Task
   8's `submitted` does, never from the relay): when `<run_dir>/handoff.json` exists, its
   `run` equals this run's identity (`vrg-repo-owner`'s `run.current`, all
   four fields), and every repo of the group is owned by that run, go on
   at step 6 (§5.5, the planned hand-off). Otherwise report the repos owned
   and by which run (`vrg-repo-owner status --json`), each PR's state and
   the ownership history, and stop for the human, changing nothing.
2. **Preconditions** (§5.5 step 1): every member reported ready, none
   submitted, each with a frozen worktree at its recorded head in its
   repo's main clone (`vrg-pr-workflow status` in it). A failure stops
   before anything is taken.
3. **Take** (§5.5 step 2): `vrg-repo-owner take --json`. Exit 3: comment on
   the task, once per new holder, naming the epic and task holding each
   repo (a stale holder marked stale, for the human to break), then run
   `take --wait` in the background and go on when it exits. Any exit
   other than 0 and 3 stops for the human. Once
   taken, record each member's line, as read, in `<run_dir>/group.json`.
4. **Submit** (§5.5 step 3): for each repo's PR, re-read each member's
   line and refuse one that changed; `vrg-submit-pr` from its worktree;
   watch each to its merge with `pr-watch` on a failure. A failure that
   cannot be fixed forward stops for the human, ownership held.
5. **Release** (§5.5 step 4): hand the human the paste-ready
   `vrg-release --epic <epic ref> --task <task ref>` for each repo, in the
   task's `Release order` where it has one, as human-run steps. Once every
   member's PR has merged, write `<run_dir>/handoff.json` (the run's
   identity and the time), post a comment on the task naming the commands
   handed over, and end the run `waiting: releases`, recording no outcome.
   The next run is a resumed run that continues at step 6 (step 1).
6. **Confirm** (§5.5 step 5): `vrg-release-record check` on every member,
   exit 0 for each.
7. **Give up and close** (§5.5 step 6): read the group again and compare it
   with `group.json`; a difference stops for the human, ownership held.
   Otherwise `vrg-repo-owner give-up` with a `--repo` for each repo in
   `group.json`, then record `Outcome: SUCCESS` with the results template
   and close the task.

Step 6 maps `vrg-release-record check`'s exit codes as the released tool
defines them. Exit 4 (`none`: no counting record yet, the release not yet
run) ends the run `waiting: releases` again, naming the repos still to
release. Exit 3 (`fail`: a record contradicted by git ancestry, or no
counted closing PR), exit 1 and any other failure stop for the human,
ownership held, the report naming the task and the verdict. Both cases are
among the skill's stop reports.

- [ ] **Step 1: Write the skill** to the outline above, with each command
  as it is run and each stop's exact report.
- [ ] **Step 2: Validate; commit.**

### Task 13: `epic-implement` and `issue-implement` (vergil-claude-plugin)

Spec §5.4 (hand-off at report-ready, re-entry), §5.7 (dispatch).

**Files:**

- Modify: `skills/issue-implement/SKILL.md` (a task carrying a
  `Coordinated release:` line ends at report-ready with the outcome
  `held for coordinated release <ref>`, never submitted)
- Modify: `skills/epic-implement/SKILL.md` (the kind table gains
  `coordinated-release` → `issue-coordinated-release`; that outcome is the
  task's gate passed; on re-entry a reported-ready task carrying the line
  is handed off and shown under its coordinated release task's state,
  never submitted, read from the line; the runnable frontier comes from
  `vrg-epic-audit`, Task 8's rules, a task `in-window` included; a run of
  `issue-coordinated-release` that ends `waiting: releases` is a human-run
  stop: the gate shows the human the commands its comment names, and once the human
  says the releases are done, the task is dispatched again, as a
  deployment's `waiting: human-run steps` is)

- [ ] **Step 1: Edit both skills**; each rule cites the spec section.
- [ ] **Step 2: Validate; commit.**

### Task 14: `epic-create`'s planning rule (vergil-claude-plugin)

Spec §5.7.

**Files:**

- Modify: `skills/epic-create/SKILL.md`: the plan step's rule (a
  coordinated release task for each group of PRs in different repos that
  must ship together, naming its PRs by plan task ID and any real release
  order; a step outside the tooling is its own human-attested task; most
  epics need none); plan pushback holds each group to the smallest set of
  repos, one PR per repo, with dependent same-repo work in one release
  group; `file-tasks` files the coordinated release task first, then
  writes each member's `Coordinated release: <ref>` line, never a
  `Blocked-by:` from it to its own group.

- [ ] **Step 1: Edit the skill.**
- [ ] **Step 2: Validate; commit.**

---

## Phase 3: visibility and the live run

### Task 15: Hoshin shows ownership (vergil-claude-plugin)

Spec §5.6.

**Files:**

- Modify: `skills/hoshin/SKILL.md` (the daily review lists
  `owned_repos` from `vrg-queue hoshin --json`: each repo, epic, task and
  age; stale or unreadable ownership is a finding, with the human's `break`
  command named and never run by the agent)

- [ ] **Step 1: Edit the skill.**
- [ ] **Step 2: Validate; commit.**

### Task 16: Release the plugin (deployment)

As "Operational tasks" above.

### Task 17: A live coordinated release (validation)

- **Intro:** the first coordinated release, run live across two fixture
  repos, proving take, the guard, the release check, the confirm and the
  give-up together.
- **Preconditions:** Tasks 11 and 16 closed; `vrg-release-record check`
  exits 0 for Tasks 7–10 and 12–15; `vergil-fixture-python-app` and
  `vergil-fixture-ts-lib` have main clones and nothing unreleased on
  `develop` (human-attested, since a release of either ships whatever is
  there).
- **Procedure:**
  1. Create a probe development epic in `vergil-project/.github`
     (`vrg-epic-create --title "Epic: coordinated release probe"`), never
     this epic, so `epic-implement` driving vergil-project/.github#1818
     never sees the probe's tasks. File under the probe epic a coordinated
     release task (`--kind coordinated-release`) and one task in each
     fixture (a one-line change to its README), each member carrying the
     task's line.
  2. Implement both members to report-ready; check that `vrg-epic-audit`
     shows the coordinated task blocked until the second is reported, then
     runnable.
  3. Before running it, check that `vrg-submit-pr` refuses one member
     (Task 9's guard).
  4. Run `issue-coordinated-release`. While ownership is held, the human
     runs a plain `vrg-release` in one fixture and sees it refused, naming
     the owner; then runs the two commands the run hands over. The run ends
     `waiting: releases`; once both releases are done, run
     `issue-coordinated-release` again and check that it goes on at
     Confirm.
  5. After the run, check `vrg-repo-owner status` shows both unowned, and
     the history of each holds one `take` and one `give-up` by the run.
  6. Check that the probe epic is closed (the epic rollup closes it once
     every child is closed); if it is still open with every child closed,
     close it by hand.
- **Acceptance:** each check above as stated; the task closed with
  `Outcome: SUCCESS`; both members closed and released; the probe epic
  closed.

### Task 18: Fixture run

As "Operational tasks" above.

### Documentation review and retrospective

- **doc-review** (`vergil-tooling`, `Documentation review:
  epic-repo-ownership`): the tooling site docs for `vrg-repo-owner`,
  `vrg-release --epic --task` and the new kind; the plugin's docs for the
  run skill; `.github`'s operational-task convention. It spawns a per-repo
  task for each repo outside vergil-tooling that needs work.
- **retrospective** (`.github`, `--kind retrospective`, `Retrospective:
  epic-repo-ownership`): terminal, as every epic's.
