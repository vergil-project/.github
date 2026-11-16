# Upstream-refresh PRs merge themselves Implementation Plan

- **Epic:** vergil-project/.github#1168
- **Planning issue:** vergil-project/.github#1067 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** each task is one PR, implemented with the
> `issue-implement` skill (operational tasks with `issue-deploy` or
> `issue-validate`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** the nightly upstream-refresh job in vergil-containers and
vergil-python carries its PR through to a merge into `develop` as the org
automation App, and succeeds only when the PR has merged or there was
nothing to change.

**Architecture:** vergil-tooling gains an App token source that mints,
renews and revokes repo-scoped installation tokens in-process. It is wired
into the shared `gh` and `git` call layer, so every call starts with a
current token. `vrg-upstream-refresh` splits into a `prepare` half, which
runs the hook with no secrets and writes a patch, and a `publish` half,
which holds the key. Publish applies the patch on the recorded `develop`
commit, checks it against the repo's path fence, and opens the PR through
the mechanized-PR building block, which waits until it merges. vergil-actions'
`ops-upstream-refresh.yml` becomes two declarative jobs. Each consuming repo
declares its allowlist.

**Tech stack:** Python 3 (vergil-tooling `src/vergil_tooling`, pytest under
`tests/vergil_tooling`), `cryptography` for RS256 signing, the GitHub REST
API, and GitHub Actions YAML (vergil-actions, vergil-containers,
vergil-python).

**Spec:** [`epics/1168-upstream-refresh-merge/spec.md`](spec.md)

## Global constraints

- **The key boundary** (spec §6): the App's private key never shares a
  runner with the work (the repo hook and what it loads). The `prepare`
  job holds no App credentials. The `publish` job never runs the hook and
  never runs git in a tree prepare touched.
- The key and every token are never in argv, a log line, an exception
  message or a file. App API calls that carry a JWT or a token directly
  (installation lookup, mint, revoke) are made in-process with
  `urllib.request`, never through `gh`.
- The key and client ID reach publish in `VRG_AUTOMATION_PRIVATE_KEY` and
  `VRG_AUTOMATION_CLIENT_ID`. They are set only on the publish step, and
  removed from `os.environ` at startup. They are never `VRG_APP_ID` or
  `VRG_PRIVATE_KEY_PATH`.
- Upstream-refresh's token permissions: `contents: write`,
  `pull_requests: write`, `workflows: write`, `metadata: read`, limited to
  the one target repo.
- Renewal interval: 1800 seconds (renew once the token is older). The
  publish deadline: 7200 seconds from the tool's own start. The publish
  job's `timeout-minutes: 140` (the 120-minute deadline, plus 15 minutes
  for the steps before the tool, plus a 5-minute margin).
- The job name is `upstream-refresh`, and its branches are
  `automation/upstream-refresh/<run-id>`. The PR title is
  `chore(upstream): refresh upstream inputs`.
- Exit codes: 0 means merged, or nothing to change. 2 means not merged,
  naming the PR and the reason. 1 is any other failure, with the reason,
  including a usage error (argparse's own exit 2 is mapped to 1).
- The path fence: the allowlist is read only from publish's own `develop`
  clone (`[upstream-refresh].paths`), and the touched set comes from the
  applied tree, never from prepare's meta. `.github/**` is always refused.
  In `vergil.toml`, only `upstream-version` and `summary` of existing
  `[[package]]` entries may change. A new entry must be a series copy of an
  existing one, and a removed entry is allowed. A repo with no `paths`
  refuses every
  patch.
- With no token source registered, `lib/github.py` and `lib/git.py` behave
  exactly as today. `github-config`, `vrg-release`, `vrg-finalize-pr` and
  human runs are unchanged.
- No silent failures: every failure is a non-zero exit with the reason
  named.
- The only validation command is `vrg-container-run -- vrg-validate`. Every
  "run the tests" step below means that command.
- Commits use `vrg-commit`, and handoff uses `vrg-pr-workflow
  report-ready`. Changes under `.github/workflows/**` are pushed by the
  human's `vrg-submit-pr`.
- A red nightly upstream-refresh run between the vergil-tooling release
  (D1) and the vergil-actions release (D2) is accepted (spec §6.4).

## Review focus

1. **A renewal falls due in the middle of a long `wait_for_checks` poll.**
   Expected: the next `gh` call in that poll carries the renewed token, and
   the replaced one is revoked. Test in Task 2.
2. **A patch that touches `.github/workflows/ci.yml`, even when the
   allowlist has `**`.** Expected: refused before any push, naming the path
   and the rule. Test in Task 3a.
3. **A `vergil.toml` patch that changes `[[package]]` and
   `[dependencies].vergil` together.** Expected: refused, naming
   `[dependencies]`. Test in Task 3a.
4. **`develop` moved between prepare and publish.** Expected: the PR's
   commit is parented on the recorded commit. A recorded commit that is no
   longer on `develop` is refused. Test in Task 3b.
5. **The first revocation's response is lost.** Expected: the retry's 401
   counts as revoked, and a merged run still exits 0. Test in Task 1.

---

## Delivery

| Task | Repo | What | Blocked by |
| --- | --- | --- | --- |
| 1 | vergil-tooling | The App token source (`lib/app_token.py`) | — |
| 2 | vergil-tooling | Token source in every `gh` and `git` call | 1 |
| 3a | vergil-tooling | `prepare`, `paths` and the path fence | — |
| 3b | vergil-tooling | `publish`, CLI modes, tooling docs | 1, 2, 3a |
| D1 | vergil-tooling (deployment) | Release vergil-tooling | 3b |
| C1 | vergil-containers | The allowlist, comment and doc updates | D1 |
| P1 | vergil-python | The allowlist and comment updates | D1 |
| A1 | vergil-actions | Workflow as prepare and publish; docs | 3b |
| D2 | vergil-actions (deploy) | Release; close legacy PRs | D1, A1, C1, P1 |
| V1 | vergil-actions (validation) | Live validation in both repos | D2 |

The bookends already exist: the documentation review
vergil-project/vergil-tooling#4444, blocked by V1, and the retrospective
vergil-project/.github#1170, last.

C1 and P1 wait for D1 because the released tooling must accept the new
`[upstream-refresh].paths` key before a consumer's `vergil.toml` uses it.
D2 waits for C1 and P1 because a repo with no allowlist refuses every
patch.

**No fixture run.** The change touches only `ops-upstream-refresh.yml`,
which the fixture repos do not call, and `vrg-upstream-refresh`, which they
do not configure. The new `cryptography` dependency is covered by
vergil-tooling's own CI and packaging.

Advisory hand-off: each advisory tracked by the planning issue is
retargeted to the task or tasks that fix it.

| Task | Advisories |
| --- | --- |
| 1 | GHSA-42q4-pvfr-wm7g, GHSA-2pqp-hwp9-93c9 |
| 2 | GHSA-2pqp-hwp9-93c9 |
| 3a | GHSA-jc62-q496-8pc2, GHSA-vq8h-h7qr-4v7q |
| 3a | GHSA-fvr8-fc3j-8h46, GHSA-jhm6-x586-6965 |
| 3b | GHSA-rp3h-jvjv-3jw7, GHSA-vv3j-95jg-7m6v, GHSA-jp84-c599-qqvv |
| C1 | GHSA-jhm6-x586-6965, GHSA-rxw3-347f-hfhr |
| A1 | GHSA-rp3h-jvjv-3jw7, GHSA-pfp8-v4q6-vxc8 |

### Task 1: The App token source

**Files:**

- Create: `src/vergil_tooling/lib/app_token.py`
- Modify: `pyproject.toml` (add `cryptography` to `dependencies`, and to any
  packaging dependency list that mirrors it)
- Test: `tests/vergil_tooling/test_app_token.py`

**Interfaces:**

- Produces:
  - `@dataclass(frozen=True) class AppCredentials: client_id: str;
    private_key: str`, with a `__repr__` that never shows the key.
  - `read_credentials_from_env() -> AppCredentials | None`: pops
    `VRG_AUTOMATION_CLIENT_ID` and `VRG_AUTOMATION_PRIVATE_KEY` from
    `os.environ`, and returns `None` when neither was set. Exactly one set
    raises `AppTokenError`.
  - `UPSTREAM_REFRESH_PERMISSIONS: Mapping[str, str] = {"contents":
    "write", "pull_requests": "write", "workflows": "write", "metadata":
    "read"}`.
  - `class AppTokenError(RuntimeError)`.
  - `class AppTokenSource`, a context manager:
    `__init__(self, creds: AppCredentials, repo: str, permissions:
    Mapping[str, str], *, expected_slug: str, renew_after_secs: int = 1800,
    clock: Callable[[], float] = time.monotonic)`; `token(self) -> str`
    (the current token, minted on first use and renewed once older than
    `renew_after_secs`); `close(self) -> None` (revokes the current token;
    idempotent). `__exit__` calls `close`.
- Consumes: `automation_identity.verify_token_is_app` semantics (a login
  of `<slug>[bot]`), done here with the new token in a scoped environment.

- [ ] **Step 1: Write the failing tests**, with the App endpoints faked at
  the `urllib.request` boundary and a fake clock.
  - `test_first_token_is_scoped_to_repo_and_permissions`: the mint request
    body has `repositories: [<name>]` and exactly the four permissions.
  - `test_jwt_signed_in_process_rs256`: the JWT verifies against the
    test key's public half. No subprocess is started (patch
    `subprocess.run` and `subprocess.Popen` to fail).
  - `test_app_slug_mismatch_fails`: `GET /app` returns a different slug,
    which raises `AppTokenError` naming both slugs.
  - `test_renews_after_interval_and_revokes_old`: at a clock of 1801
    seconds, `token()` returns a new token, and the old one got
    `DELETE /installation/token`.
  - `test_no_renewal_before_interval`
  - `test_revoke_retry_401_counts_as_revoked`: the first DELETE raises a
    connection reset, and the retry gets 401. `close()` returns normally.
  - `test_revoke_401_on_first_attempt_fails`
  - `test_revoke_5xx_retried_then_fails_naming_step`
  - `test_close_is_idempotent`
  - `test_key_absent_from_repr_errors_and_logs`: the key's PEM body
    appears in no `repr`, no exception message and no captured output.
  - `test_read_credentials_pops_both_vars` and
    `test_read_credentials_one_var_only_raises`.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** The JWT has `iss` set to the client ID,
  `iat` 60 seconds back and `exp` 9 minutes on. The installation comes
  from `GET /repos/{repo}/installation`. Each new token's login is checked
  with GraphQL `viewer { login }`, against `<expected_slug>[bot]`.
  Revocation retries network errors and 5xx only, three attempts with
  backoff. A 401 on a retry is "already revoked".
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope automation`).

### Task 2: Route the token source through every `gh` and `git` call

**Files:**

- Modify: `src/vergil_tooling/lib/github.py` (`_gh_env`,
  `_run_with_retry`), `src/vergil_tooling/lib/git.py` (`_git_env` and its
  retry loop), `src/vergil_tooling/lib/retry.py`
- Test: `tests/vergil_tooling/test_github.py`, `tests/vergil_tooling/test_git.py`

**Interfaces:**

- Produces:
  - `class TokenProvider(Protocol): def token(self) -> str` in
    `lib/github.py`.
  - `github.set_token_source(source: TokenProvider | None) -> None`, and a
    `github.token_source(source)` context manager that sets the source and
    restores the previous one.
  - `retry.run_with_retry(*args, env_factory: Callable[[], dict[str, str] |
    None] | None = None, **kwargs)`: when given, the environment is rebuilt
    for every attempt.
- Consumes: `AppTokenSource.token()` (Task 1), through the protocol only.

- [ ] **Step 1: Write the failing tests.**
  - `test_gh_env_uses_registered_source_per_call`: two calls whose source
    returns `t1`, then `t2`, run with `GH_TOKEN=t1`, then `GH_TOKEN=t2`.
  - `test_gh_retry_after_401_rebuilds_env`: the first attempt gets 401
    with `t1`, and the retry runs with `t2`.
  - `test_git_remote_call_uses_source_header`: `git push` gets
    `GIT_CONFIG_VALUE_0` built from the source's current token (the
    `_git_auth_env` form). No git config file is written.
  - `test_wait_for_checks_poll_picks_up_renewal`: a source renewed between
    two polls inside one `github.wait_for_checks` call gives the second
    poll the new token.
  - `test_source_and_app_mode_configured_refuses`: a registered source
    with `VRG_APP_ID` or `VRG_PRIVATE_KEY_PATH` set raises, naming the
    variable.
  - `test_no_source_env_unchanged`: with no source, `_gh_env` and
    `_git_env` return exactly what they return today, which the existing
    tests already pin.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Keep the source in a module-level
  `ContextVar`. `_gh_env` and `_git_env` check it first. `_run_with_retry`
  and git's retry loop pass an `env_factory` in place of a fixed `env`.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope automation`).

### Task 3a: `prepare`, the `paths` config key and the path fence

**Files:**

- Modify: `src/vergil_tooling/lib/config.py` (`UpstreamRefreshConfig`
  gains `paths: tuple[PathRule, ...] = ()`, with `@dataclass(frozen=True)
  class PathRule: glob: str; lines: str | None = None`. Each `paths` entry
  is either a string (a glob, with no line rule) or a table
  `{ path = "<glob>", lines = "<regex>" }`. The allowed keys of
  `upstream-refresh` gain `paths`), `src/vergil_tooling/lib/upstream_refresh.py`
  (rewrite), `src/vergil_tooling/lib/develop_clone.py` (`clone_develop`
  gains `token: str | None = None`, used through `git._git_auth_env`)
- Test: `tests/vergil_tooling/test_upstream_refresh.py`,
  `tests/vergil_tooling/test_config.py`, `tests/vergil_tooling/test_develop_clone.py`

**Interfaces:**

- Produces (in `lib/upstream_refresh.py`):
  - `PATCH_FILE = "changes.patch"` and `META_FILE = "meta.json"`. The meta
    holds `{"repo": str, "base_sha": str, "paths": [str], "notes": [str]}`.
  - `@dataclass(frozen=True) class Prepared: repo: str; base_sha: str;
    paths: list[str]; notes: list[str]`.
  - `prepare(repo: str, out_dir: Path, *, token: str | None = None) ->
    Prepared`: clones `develop` (with `token` when given), reads
    `[upstream-refresh]` from the clone, runs the hook there, refuses
    untracked files, and writes `PATCH_FILE` (`git diff --binary`) and
    `META_FILE`. With no change it writes meta with `paths: []` and no
    patch.
  - `read_prepared(in_dir: Path) -> tuple[Prepared, bytes | None]`, which
    validates the meta's shape and refuses anything else.
  - `run_hook(tree: Path, hook: str, report: Path) -> list[str]`, now
    started with a scrubbed environment: no `GH_TOKEN`, `GITHUB_TOKEN`,
    `VRG_AUTOMATION_*`, `VRG_APP_ID` or `VRG_PRIVATE_KEY_PATH`.
  - `fence_violations(rules: UpstreamRefreshConfig, paths: Sequence[str],
    changed_lines: Mapping[str, Sequence[str]], vergil_toml_before: str |
    None, vergil_toml_after: str | None) ->
    list[str]`: one message per violation, each naming the path or table
    and the rule. An empty list means allowed. Its `paths` argument is
    always the set the caller computed from the applied tree, never one
    read from prepare's output. `changed_lines` maps each touched path to
    its removed and added lines (from `git diff -U0 <base_sha>` in the
    applied clone). A path whose first matching rule has a `lines` regex
    is allowed only when every changed line matches it as a whole line
    (`re.fullmatch`). A binary, added or deleted file under such a rule is
    refused.
  - `render_body(notes: list[str], paths: list[str]) -> str`, whose intro
    now says the job merges the PR itself once CI passes.
  - Removed: `REFRESH_BRANCH`, `sync`, `open_refresh_prs`,
    `scratch_worktree`, `ensure_clean_on_base`, the push and close helpers.
- Consumes: `develop_clone.clone_develop`, `config.read_config`.

- [ ] **Step 1: Write the failing tests.**
  - `test_prepare_writes_patch_and_meta`: against a local bare repo
    standing in for `origin`, the patch applies with `git apply` on the
    recorded `base_sha`.
  - `test_prepare_no_change_writes_empty_paths`
  - `test_prepare_refuses_untracked_files`
  - `test_hook_env_has_no_tokens_or_key`: the hook prints its
    environment, and none of the scrubbed variables appears.
  - `test_hook_read_from_clone_config`
  - `test_fence_allows_listed_globs`
  - `test_fence_refuses_path_outside_allowlist`
  - `test_fence_refuses_github_dir_even_with_star_star`: `paths = ["**"]`,
    patch path `.github/workflows/ci.yml`.
  - `test_fence_no_paths_refuses_everything`
  - `test_fence_vergil_toml_version_and_summary_allowed`: an existing
    `[[package]]` entry's `upstream-version` and `summary` change.
  - `test_fence_vergil_toml_smoke_change_refused`
  - `test_fence_vergil_toml_build_command_change_refused`: a change to
    `staged.build-command`.
  - `test_fence_vergil_toml_series_copy_allowed`: a new
    `vergil-python3.15` entry that equals the `vergil-python3.14` entry
    with `3.14` replaced by `3.15` in `name`, `smoke` and `staged`, with
    its own `upstream-version` and `summary`.
  - `test_fence_vergil_toml_new_entry_extra_command_refused`: the same new
    entry, but its `smoke` has an extra `; curl …`, is refused, naming the
    entry and `smoke`.
  - `test_fence_vergil_toml_new_entry_changed_build_command_refused`
  - `test_fence_vergil_toml_new_entry_unrelated_name_refused`: a new entry
    whose name differs by more than a series token.
  - `test_fence_vergil_toml_removed_package_allowed`
  - `test_fence_vergil_toml_dependencies_change_refused`: `[[package]]`
    and `[dependencies].vergil` both changed; the message names
    `[dependencies]`.
  - `test_fence_vergil_toml_upstream_refresh_change_refused`
  - `test_read_prepared_rejects_bad_meta`: a missing key, a non-hex SHA,
    or a path with `..`.
  - `test_fence_line_rule_allows_version_lines`: in a
    `docker/common/x.dockerfile` covered by
    `lines = '^ARG [A-Z0-9_]+_VERSION=\d+(\.\d+)*$'`, only an
    `ARG FOO_VERSION=1.2.3` line changes.
  - `test_fence_line_rule_refuses_value_outside_format`: the same rule
    refuses `ARG X_VERSION=1.2.3/../x` and `ARG X_VERSION=1.2.3 # x`. A
    line rule matches the whole line (`re.fullmatch`).
  - `test_fence_line_rule_refuses_run_line_change`: the same file with a
    changed `RUN` line is refused, naming the file and the line rule.
  - `test_fence_line_rule_refuses_added_file`
  - `test_config_accepts_paths_key` (strings and `{ path, lines }` tables),
    `test_config_rejects_bad_line_regex` and
    `test_config_rejects_unknown_key`.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Match globs with `PurePosixPath.full_match`,
  or `fnmatch` with `**` support. Parse `vergil.toml` before and after with
  `tomllib`:
  - outside `package`, the two must be equal;
  - entries are matched by `name`. Within an existing entry, only
    `upstream-version` and `summary` may differ, so `smoke`, `staged` and
    every other key must be equal;
  - a removed entry is allowed;
  - a new entry, which vergil-python's hook adds when a new CPython series
    appears, is allowed only as a **series copy** of an existing entry E:
    - E's `name` and the new `name` must be equal except for one
      `\d+\.\d+` series token (old in E, new in the new entry);
    - replacing every occurrence of the old token with the new one in
      each of E's string values, `upstream-version` and `summary` aside,
      must give exactly the new entry's values, with the same keys and
      nested tables;
    - `upstream-version` and `summary` are free.

  Each difference is a violation, naming the entry and the key. The human's
  local run follows exactly the same rules and the same merge flow as the
  automation; there is no separate attended path.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope upstream-refresh`).

### Task 3b: `publish`, the CLI modes and the tooling docs

**Files:**

- Modify: `src/vergil_tooling/bin/vrg_upstream_refresh.py` (rewrite),
  `src/vergil_tooling/lib/upstream_refresh.py` (add `publish`),
  `src/vergil_tooling/lib/mechanized_pr.py` (`open_sync_pr` gains
  `deadline_secs: int | None = None`; `None` keeps today's rule; its
  module docstring drops "cannot be renewed inside the job" and describes
  the token source), `README.md` (the "one PR for a human to merge" line),
  `docs/site/docs/reference/upstream-refresh-config.md`,
  `docs/site/docs/reference/dependency-update-config.md`,
  `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_vrg_upstream_refresh.py`,
  `tests/vergil_tooling/test_upstream_refresh.py`,
  `tests/vergil_tooling/test_mechanized_pr.py`

**Interfaces:**

- Produces:
  - `publish(repo: str, in_dir: Path, *, deadline_secs: int, bot:
    tuple[str, str] | None) -> mechanized_pr.SyncResult`, where `bot` is
    `(name, email)` for the commit author and committer, and `None` for a
    human run.
  - The CLI:
    - `vrg-upstream-refresh prepare --out DIR [--repo R]`;
    - `vrg-upstream-refresh publish --from DIR [--repo R]`;
    - `vrg-upstream-refresh [--repo R] [--dry-run [--rules-from
      PATH]]`, the combined default.

    `--dry-run` runs prepare into a temporary directory, applies the patch
    to a second clone exactly as publish does, runs `fence_violations` on
    it, and prints the PR body and every violation. It pushes nothing, and
    exits 1 if any violation is found. The fence rules come from the
    `develop` tip, or from the `vergil.toml` named by `--rules-from`, so a
    branch can test its new `paths` before they merge. `--rules-from` is
    accepted only with `--dry-run`.
- Consumes: Tasks 1, 2 and 3a, and `mechanized_pr.open_sync_pr`.

- [ ] **Step 1: Write the failing tests.**
  - `test_publish_applies_on_recorded_base`: `develop` has moved by one
    commit. The PR commit's parent is `base_sha`.
  - `test_publish_refuses_base_not_on_develop`: the compare API says
    `diverged`, or the commit is unknown.
  - `test_publish_refuses_patch_that_does_not_apply`
  - `test_publish_refuses_extra_untracked_paths`
  - `test_publish_fence_sees_tracked_change_missing_from_meta`: the patch
    modifies `.github/workflows/ci.yml`, and the meta leaves it out. The
    run is refused for the mismatch and the fence names the path.
  - `test_publish_touched_set_must_equal_meta_paths`: a meta that lists a
    path the patch does not touch is refused.
  - `test_publish_fence_reads_rules_from_develop_tip`: the recorded base's
    `vergil.toml` has a looser `paths` than the tip, and the tip's rules
    apply. A meta carrying its own rules has no effect.
  - `test_publish_fence_refusal_pushes_nothing`
  - `test_publish_empty_paths_closes_earlier_pr_in_sync`
  - `test_publish_sets_bot_author_and_committer`
  - `test_prepare_refuses_when_key_present`
  - `test_actions_default_mode_refused`
  - `test_actions_publish_without_creds_refused`
  - `test_publish_refuses_app_mode_configured`
  - `test_publish_mints_before_identity_resolution`: with only the
    credential variables and `VRG_AUTOMATION_APP` set, the mode resolves
    to `automation` through the source's first token.
  - `test_human_run_keeps_env_deadline`: with no token source,
    `open_sync_pr` gets `deadline_secs=None`, so
    `$VRG_MECHANIZED_PR_DEADLINE` still applies.
  - `test_publish_deadline_counts_from_tool_start`: after 600 seconds of
    clone and apply on a fake clock, `open_sync_pr` gets
    `deadline_secs=6600`.
  - `test_sigterm_revokes_and_exits_nonzero`, including a signal right
    after the first mint, during identity resolution.
  - `test_exit_codes`: merged gives 0, in sync gives 0, `PRNotMergedError`
    gives 2 with the URL printed, a fence refusal gives 1, and a usage
    error (for example the removed `--base`) gives 1, not argparse's 2.
  - `test_log_lines`: closed, opened or in sync, each renewal and
    revocation with a time, and how the wait ended. No token appears.
  - `test_agent_modes_refused`: `prepare`, `publish` and the combined
    mode are refused under each agent mode.
  - `test_agent_dry_run_allowed`: `--dry-run` (with or without
    `--rules-from`) runs under an agent mode, registers no token source,
    and makes no write call. Patch `github` and `git` push helpers to
    fail.
  - `test_dry_run_reports_fence_violations`: with `--rules-from` naming a
    file whose `paths` omits a touched file, the dry run prints the
    violation, pushes nothing and exits 1. `--rules-from` without
    `--dry-run` is a usage error (exit 1).
  - `test_human_default_runs_prepare_then_publish_without_source`: outside
    Actions, the combined mode runs both halves with the human's ambient
    credentials, registers no token source, and keeps the human's commit
    identity.
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.**
  - The publish flow, in order:
    1. read the credentials;
    2. refuse App mode;
    3. build the `AppTokenSource` with `expected_slug` from
       `automation_identity.env_slug()`;
    4. install the SIGINT and SIGTERM handlers, which `close()` and exit
       130 or 143, before anything can mint a token;
    5. register the source (`github.token_source`);
    6. resolve the identity, which mints the first token;
    7. clone `develop`, and read the fence rules (`[upstream-refresh]`)
       from that tip before anything else. Then fetch and check out
       `base_sha` detached, and `git apply`;
    8. compute the touched set from the clone itself, with
       `git status --porcelain=v1 -z --untracked-files=all --no-renames`:
       added, modified, deleted, mode-changed and untracked files. Refuse
       unless it equals the meta's `paths` exactly. Pass that set, never
       the meta's, to `fence_violations`, with the changed lines from
       `git diff -U0 <base_sha>` in the clone, the `vergil.toml` "before"
       taken from `base_sha` and "after" from the working tree. The meta's
       `paths` are used only for the PR body;
    9. set the clone's `user.name` and `user.email` from `bot`;
    10. `open_sync_pr(..., deadline_secs=7200 - elapsed)` when a token
        source is registered, and `deadline_secs=None` for a human run.
  - The bot email is `<id>+<slug>[bot]@users.noreply.github.com`, with
    `<id>` from `GET /users/<slug>[bot]`.
  - Update the three docs pages, the module docstring and `--help`. The
    human merge goes, and prepare, publish, the fence and `paths` are
    described.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope upstream-refresh`).

### Task D1 (deployment): Release vergil-tooling

Kind `deployment`, filed in vergil-tooling and blocked by 3b. Its
precondition is human-attested: vergil-tooling is released on the `v2.1`
line with Tasks 1, 2, 3a and 3b.

- **Procedure:** confirm the released version installs through
  `setup/vergil`, and that `vrg-upstream-refresh --help` shows the
  `prepare` and `publish` modes.
- **Acceptance:**
  - the release is recorded;
  - the next nightly upstream-refresh in either repo either fails at the
    new tool's refusal, or has not yet run. Either is expected (spec §6.4).

### Task C1: vergil-containers: the allowlist, comment and doc updates

**Files:** Modify `vergil.toml` (`[upstream-refresh]` gains `paths`, and
the comment loses "a human merges it"), `.github/workflows/ops.yml` (the
`upstream-refresh` job comment, and the `language-lifecycle` comments that
describe its token and pushes "as upstream-refresh's"),
`docs/site/docs/operations/version-management.md` (the human merge gate
goes), and `docker/pins/report_exposure.py` with its test expectations
(the "never auto-merges" note, and the pointer to an open `upstream-refresh`
PR, which now names the `automation/upstream-refresh/` PRs), and the
`docker/pins/workflow_images.py` module docstring (it says the refresh
cannot edit workflows because its token has no `workflows` permission;
the reason is now the path fence, where `.github/**` is always refused).

- [ ] **Step 1:** Derive `paths` from what the hook writes *at
  implementation time*. Read `docker/pins/upstream-refresh.sh` and each
  script it runs on `develop` then, not this plan's snapshot, which was
  taken on 2026-11-16:
  - `resolve_latest.py`: the `docker/common/*` fragments, as one entry
    `{ path = "docker/common/*",
    lines = '^ARG [A-Z0-9_]+_VERSION=\d+(\.\d+)*$' }`, and
    `docker/pins/workflow-images.txt` (added by vergil-containers#919), as
    `{ path = "docker/pins/workflow-images.txt",
    lines = '^[A-Z0-9_]+_VERSION=\d+(\.\d+)*$' }`;
  - `resolve_toolchains.py`: `docker/pins/toolchains.yml`;
  - `generate.py`: `docker/versions/matrix.json` only;
  - `generate_catalog.py`: `docker/pins/CATALOG.md`.

  As of the snapshot, that is the whole list. Data files are listed as
  exact paths. `generate.py`
  renders more (`window.json`, the `docker/build.sh` smoke table, the
  `Dockerfile.template` `ARG` defaults, the README and docs version tables,
  and two `.github/workflows/` package lists), but those come from
  `manifest.yml`, which the hook never edits. They are deliberately
  excluded, and a comment in `vergil.toml` says so. PR CI's
  `generate.py --check` still catches drift. No entry may cover the hook
  script or its resolvers and generators (`docker/pins/*.sh`,
  `docker/pins/*.py`, `docker/versions/*.py`).
- [ ] **Step 1b: The content checks the fence relies on** (spec §6.2).
  - Confirm that `generate.py --check` and `check_pins.py` run in PR CI,
    and that the jobs running them are required status checks on
    `develop` (the repo's ruleset, as `vrg-github-repo-config audit`
    reports it). Record the check names in the `vergil.toml` comment
    beside `paths`. If either is not required, stop and raise it: the
    fence must not ship without it.
  - Where `toolchains.yml`'s format allows a value-format line rule (a
    version and a hex hash per line), give its entry one. Otherwise record
    why not.
- [ ] **Step 2:** Check it: run `vrg-upstream-refresh --dry-run
  --rules-from vergil.toml` locally, and confirm it reports no violation.
  If upstream has nothing new that day, there is no change to check.
  Record that, and rerun the check on a later day before handoff, or use a
  past refresh PR's diff.
- [ ] **Step 3: Run the tests** (`vrg-validate` checks `vergil.toml`
  against the released tooling).
- [ ] **Step 4: Commit** (`vrg-commit --type chore --scope upstream-refresh`).

### Task P1: vergil-python: the allowlist and comment updates

**Files:** Modify `vergil.toml` (`[upstream-refresh]` gains
`paths = ["runtime.toml", "vergil.toml"]`, and the comment is updated) and
`.github/workflows/ops.yml` (the `upstream-refresh` job comment loses
"Nothing is auto-merged"). Check `docs/runtime-package-pattern.md`, and
update it if it describes the human merge.

- [ ] **Step 1:** Edit as above.
- [ ] **Step 2:** Run `vrg-upstream-refresh --dry-run --rules-from
  vergil.toml` locally, and confirm it reports no violation, as in C1.
- [ ] **Step 3: Run the tests.**
- [ ] **Step 4: Commit** (`vrg-commit --type chore --scope upstream-refresh`).

### Task A1: `ops-upstream-refresh.yml` as prepare and publish

**Files:**

- Modify: `.github/workflows/ops-upstream-refresh.yml` (rewrite),
  `docs/site/docs/workflows/ops-upstream-refresh.md`,
  `docs/site/docs/workflows/index.md` ("one human-merged PR"),
  `docs/site/docs/configuration.md` (the upstream-refresh permission rows:
  the tool mints its own token, with the four permissions, and holds the
  key only in publish. They also state that a cancel which does not reach
  the publish step inside its container counts as a hard kill: the last
  token is then left to expire within its hour, on a runner that never ran
  the work)
- Create: `actions/ops/upstream-refresh/tests/key-boundary.test.sh`, with
  git mode 100755, and a step in `.github/workflows/smoke-setup-vergil.yml`
  that runs it (or its `validate-custom` entry, if it needs only `yq` and
  `jq`). `scripts/bin/check-test-suites-run` fails any `*.test.sh` that
  nothing runs.
- Modify: `actions/ops/github-config/tests/automation-env.test.sh` (the
  fleet-wide rule for where `VRG_AUTOMATION_APP` may be set), its section
  in `docs/site/docs/workflows/ops-github-config.md`, and its summary in
  `smoke-setup-vergil.yml`

**Interfaces:**

- Consumes: the CLI modes of Task 3b. Each caller's `secrets:` stay
  `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.

- [ ] **Step 1: Write the failing test** `key-boundary.test.sh`, which
  parses the workflow and asserts:
  - `secrets.APP_CLIENT_ID` and `secrets.APP_PRIVATE_KEY` appear only in
    the `env:` of the publish job's run step;
  - the prepare job has no `secrets.` reference and has
    `permissions: contents: read`;
  - `VRG_AUTOMATION_APP` is set on no job-level `env:`;
  - each job's first step is the `develop` guard;
  - there is no `if:` on any step;
  - both jobs set `timeout-minutes` (prepare 30, publish 140);
  - `create-github-app-token` and `gh auth setup-git` are absent.

  It runs where `actions/ops/github-config/tests/automation-env.test.sh`
  runs today.
- [ ] **Step 1b: Extend `automation-env.test.sh`'s rule.** A step that sets
  `VRG_AUTOMATION_APP` is accepted in one of two forms:
  - `GH_TOKEN` from an earlier token step, as today;
  - `VRG_AUTOMATION_CLIENT_ID` and `VRG_AUTOMATION_PRIVATE_KEY` from
    `secrets.*`, with no `GH_TOKEN`.

  Anything else is still a violation. Add fixture cases
  `credential_form_accepted`, `credential_form_missing_key_rejected` and
  `credential_form_with_gh_token_rejected`. Its `case_real_workflows` then
  passes on the rewritten `ops-upstream-refresh.yml`.
- [ ] **Step 2: Run the tests and see it fail.**
- [ ] **Step 3: Rewrite the workflow** as spec §6.3 lays it out:
  - **`prepare`:**
    - `timeout-minutes: 30`, so a hung hook cannot hold the concurrency
      group;
    - the guard;
    - checkout of `ref: develop` with `persist-credentials: false`;
    - install;
    - `vrg-upstream-refresh prepare --out "$RUNNER_TEMP/upstream-refresh"
      --repo "$REPO"`, with `GH_TOKEN: ${{ github.token }}` (read-only)
      on that step;
    - `actions/upload-artifact@v7`, named `upstream-refresh-prepare`.
  - **`publish`:**
    - `needs: prepare`, and `timeout-minutes: 140`;
    - the guard;
    - checkout and install;
    - `actions/download-artifact@v7`;
    - `vrg-upstream-refresh publish --from <dir> --repo "$REPO"`, with
      `VRG_AUTOMATION_APP`, `VRG_AUTOMATION_CLIENT_ID` and
      `VRG_AUTOMATION_PRIVATE_KEY` set on that step only.
  - Workflow-level: `concurrency: group: ${{ github.repository
    }}/upstream-refresh`, `cancel-in-progress: false`. Both jobs keep
    today's container input. Rewrite the header comment.
- [ ] **Step 3b: Check the callers' `permissions:`.** Compare
  vergil-containers' and vergil-python's `ops.yml` `upstream-refresh` jobs
  with what the two new jobs need (`contents: read` for prepare's
  `GITHUB_TOKEN`; nothing for publish, whose writes use the tool's own
  token). Record the result in the PR body. If a caller needs a change,
  it goes into C1 or P1.
- [ ] **Step 4: Run the tests and see them pass.**
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope ops`). The human's
  `vrg-submit-pr` pushes it.

### Task D2 (deployment): Release vergil-actions; close the legacy PRs

Kind `deployment`, filed in vergil-actions and blocked by D1, A1, C1 and
P1. Its precondition is human-attested: vergil-actions is released with A1,
and `@v2.1` points at it.

- **Procedure:**
  1. Before the next nightly run, close every open PR on the head
     `upstream-refresh` or `upstream-refresh-*` in vergil-containers and
     vergil-python. Use the comment "Superseded: upstream-refresh now
     opens a per-run PR on `automation/upstream-refresh/` and merges it
     itself (epic vergil-project/.github#1168)", and delete each branch.
  2. Record the PRs closed.
- **Acceptance:** neither repo has an open PR or a branch named
  `upstream-refresh*`.

### Task V1 (validation): Live validation in both repos

Kind `validation`, filed in vergil-actions and blocked by D2.

- **Procedure:** dispatch `ops.yml` in vergil-python, then in
  vergil-containers, or let the night run. Repeat a repo whose run had
  nothing to change, until one run opens a PR.
- **Acceptance (spec §7):**
  - **vergil-python:** the run exits 0. Either it is in sync, or its PR on
    `automation/upstream-refresh/<run-id>`, authored by
    `<org>-automation[bot]`, merged within the run.
  - **vergil-containers:** the run is green, and its publish log shows, in
    order: a renewal, the revocation of the replaced token, successful
    polls, and the merge, seen by a poll. A branch update, if one happened,
    is recorded as extra evidence.
  - **The key boundary, from the logs:** prepare's log shows no App token
    minted; publish's log shows the hook never ran.
  - **The guard:** a dispatch from a non-`develop` branch fails at
    prepare's guard step. Publish is skipped (`needs: prepare`), so no App
    token is minted. Publish's own guard is defence in depth, and A1's
    `key-boundary.test.sh` already proves it is there.
  - No open PR or branch named `upstream-refresh*` remains in either repo.
