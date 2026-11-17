# Machine-readable tool output — implementation plan

- **Epic:** vergil-project/.github#1268
- **Planning issue:** vergil-project/vergil-tooling#4108 (review history is in
  its comments)
- **Status:** not planned; kept as a reference analysis

**Not carried out.** No task in this plan is filed, and none should be worked
from it as written. See the note at the top of [`spec.md`](spec.md) for why,
and for the standalone tasks that carry the security fixes and the
going-forward principle. Read it as a worked breakdown: where each kind of
site is, which helper and test would pin it, and which traps (error shapes,
case, exit codes, caller contracts) a conversion must handle.

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Deployment tasks are worked
> with `issue-deploy`, validation tasks with `issue-validate`. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** every site in scope reads a tool's machine-readable output, or,
where the tool has none, a fixed locale and an anchored match. Each fix is
pinned by a test that feeds in the input the old code misread.

**Architecture:** one task per spec section (§7.2 to §7.10), each adding the
section's shared helper and converting every matching site, split into
lettered sub-tasks where a section is too big for one PR or meets epic
vergil-project/.github#1218. Two security-fix tasks carry the advisories
(spec §8). Four releases deliver the work: an early vergil-tooling release for
the cloud VM fixes, a vergil-tooling release that completes the release gate,
a vergil-claude-plugin release and a vergil-vm release. A fresh-VM validation
and fixture runs follow.

**Tech Stack:** Python 3 and pytest (vergil-tooling); bash hooks and their
`hooks/tests/*.test.sh` suites (vergil-claude-plugin); cloud-init templates,
provision scripts and `tests/check-*.sh` (vergil-vm); `git`, `gh`,
cloud-init, apt, rpm, dpkg, brew, uv, gcloud and az as the tools read.

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **Re-verify first** (spec §7.1). Every code task starts by re-reading its
  sites against the current `origin/develop`. It drops a site that is already
  fixed or gone, adds a matching site the survey missed, and lists both in its
  PR body.
- **Settle the open claims in the task that needs them** (spec §7.1):
  `gh --paginate` concatenation (Task 6), `az` exit code for a missing
  resource (Task 7), `loginctl -o json` support (Task 8a, only if it reads
  `loginctl` as JSON). Each is settled from the tool's documentation or a real
  run, and the answer is recorded in the PR body.
- **Tests with the input that broke** (spec §7.1). Each fixed site has a test
  whose input is the case the old parse misread.
- **No silent failure** (spec §5). A machine-readable read that does not parse
  raises, naming the tool and what was expected. No new `except: return`
  default, no new `|| true`, no new `2>/dev/null` that hides a failure.
- **Ordinary input unchanged** (spec §5). Apart from the defect fixed,
  behaviour for ordinary values stays the same. The only deliberate behaviour
  change is Task 10's worktree enforcement.
- **Retries are never narrowed on a guess** (spec §7.5). A retry pattern that
  cannot be tied to a recorded transient failure is kept and flagged, never
  dropped.
- **`lib/pr_merge.py` is out of scope** (spec §4), and `github.pr_state`
  keeps its signature and return values (spec §7.11).
- **Security text stays neutral** (spec §8). Tasks 9 and 11 name their
  advisories only by GHSA ID, in their title, body, commits, PR and
  report-ready notes. Task 8a, which also closes an advisory, names it only by
  GHSA ID in the same places and describes its fix no further than spec §7.8
  does; so does Task D3 for the advisory it closes.
- **Refactor before commit.** Every code task ends with a refactor pass over
  what it touched, then a re-run of `vrg-container-run -- vrg-validate`,
  before its commit step. The tasks below do not repeat this step.
- **Process.** Validation runs only through `vrg-container-run --
  vrg-validate` in the task repo's worktree. Commit with `vrg-commit`; hand
  off with `vrg-pr-workflow report-ready`. Releases are human-gated; no agent
  cuts one.

## Review Focus

1. **A path git prints with a space, a newline, a quote or a leading
   character.** Every `-z` read returns it byte for byte, and the dirt preview
   shows the first path whole. Pinned in Task 2
   (`test_dirt_preview_keeps_first_character`,
   `test_status_z_path_with_newline`).
2. **A resource GitHub lists before it can be read, reported through
   GraphQL.** `retry_known_resource` still retries it until it resolves.
   Pinned in Task 4a (`test_known_resource_retries_graphql_not_found`).
3. **An unrelated failure whose text contains a retry word** (a path or
   message containing `timeout` or `eof`). It is not retried, and a write is
   never retried on it. Pinned in Task 5
   (`test_retry_word_in_unrelated_failure_not_retried`).
4. **A cloud-init boot whose only recoverable errors are deprecations.** The
   VM is ready, and the warning is reported. Pinned in Task 7
   (`test_deprecation_only_status_is_ready`) and Task 12
   (`check-readiness-probe.sh` deprecation case).
5. **A host whose locale is not English.** apt queries still find the
   candidate version. Pinned in Task 8a (`test_apt_query_runs_under_c_locale`).

---

## Delivery table

| Task | Repo | Kind | What | Blocked by |
| --- | --- | --- | --- | --- |
| 1 | .github | task (#1269) | Spec and plan (this PR) | — |
| 2 | vergil-tooling | task | §7.2 git file lists and paths | — |
| 3 | vergil-tooling | task | §7.3 git ref and format queries | 2 |
| 3b | vergil-tooling | task | §7.2/§7.3 items in functions #1218 rewrites | 2, 3, 9, vergil-tooling#4567, vergil-tooling#4573 |
| 4a | vergil-tooling | task | §7.4 the gh error helper; not-found and retry_known_resource | — |
| 4b | vergil-tooling | task | §7.4 the other classifiers, auth exit code, check rollup | 4a |
| 5 | vergil-tooling | task | §7.5 the retry classifier | 4a |
| 6 | vergil-tooling | task | §7.6 structured gh write results | 4a |
| 7 | vergil-tooling | task | §7.7 cloud VM tools | — |
| 8a | vergil-tooling | task | §7.8 host and VM guest tools | 7 |
| 8b | vergil-tooling | task | §7.8 package tools and container-cache clean-all | — |
| 9 | vergil-tooling | task | Security fixes (GHSA IDs below) | — |
| 11 | vergil-claude-plugin | task | Security fixes (GHSA IDs below) | — |
| 10 | vergil-claude-plugin | task | §7.9 hook parsing; worktree convention always enforced | 11 |
| 12 | vergil-vm | task | §7.10 readiness probe, profile checks, tests | — |
| D1 | .github | deployment | vergil-tooling release carrying Task 7, installed | 7 |
| D2 | .github | deployment | vergil-tooling release completing the gate, installed | D1, 2–6, 8a, 8b, 9 |
| D3 | .github | deployment | vergil-claude-plugin release (Tasks 10, 11), installed | 10, 11 |
| D4 | .github | deployment | vergil-vm release (Task 12) | 12 |
| V1a | .github | validation | Fresh cloud VM: vrg-vm's cloud-init gate (Task 7) | D1 |
| V1b | .github | validation | Fresh Lima VM: the template readiness probe (Task 12) | D1, D4 |
| F | fixtures | fixture runs | python-app, python-lib, rust-app, rust-lib, ts-lib, ts-app | D2 |
| — | vergil-tooling | bookend (#4604) | Documentation review | 2–12 |
| — | .github | bookend (#1270) | Retrospective | all |

Notes on the table:

- **Ordering by shared files.** Blocked-by edges keep apart the tasks that
  rewrite the same functions in these files. Other tasks that touch a shared
  file in different functions (Tasks 3, 4a and 4b in `lib/github.py`,
  `lib/repo_init.py`, `lib/release/merge.py` and `lib/upstream_refresh.py`;
  Tasks 4b and 6 in `bin/vrg_docs_stage.py`) run in parallel, and the PR that
  lands second rebases:
  - `lib/retry.py`: Task 4a, then Task 5;
  - `bin/vrg_vm.py`: Task 7, then Task 8a;
  - `lib/git.py` and the release modules: Task 2, then Task 3.
- **Security-fix tasks** (Tasks 9 and 11) go first by default and are blocked
  by nothing; any task that meets one of them rebases onto it. The table
  records their edges without reasons (spec §8).
- **#1218 dependencies** (spec §7.11). Task 3b holds every item in a function
  #1218 rewrites: `lib/release/preflight.py` `_check_branch_and_tree` (#1218
  Task 9, vergil-tooling#4567) and `bin/vrg_finalize_pr.py` (#1218 Task 12b,
  vergil-tooling#4573). Task 2 and Task 3 skip those items.
- **D1 is optional in effect**: if every gated task lands before Task 7, D1
  and D2 collapse into one release, and D1's issue closes on D2's evidence.
- **The advisories tracked by #4108** are thirteen: the six from the survey
  that spec §8 names, and seven recorded during design review. They are
  retargeted when the tasks are filed:
  - GHSA-rp4r-x254-p53h, GHSA-f4j7-x3f8-59gv and GHSA-j7c6-37h8-hjvg to
    Task 9;
  - GHSA-cgp7-q74r-m6vh, GHSA-8mqc-j6w7-2fxp and GHSA-94p8-hq6c-5r6h to
    Task 11;
  - GHSA-ch78-r3p7-2wmx to Task 8a;
  - GHSA-r374-jh25-85gc, GHSA-j526-gg2w-49jw, GHSA-983m-g4hq-523v,
    GHSA-c5h5-5x82-3cqv and GHSA-gc6p-jpjc-69hv, whose
    whole remedy is the published design (spec §8 and this plan), to the
    documentation task (#1269);
  - GHSA-3368-cr4q-cp5q, whose remedy is the plugin release reaching
    installed machines, to Task D3.

---

### Task 2: §7.2 git file lists and paths (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/git.py` (new helper; `diff_workflow_files`,
  `file_at`), `src/vergil_tooling/lib/worktrees.py` (`_describe_dirt`,
  `_newest_mtime`, `worktree list` reads),
  `src/vergil_tooling/lib/update_deps/updaters/images.py`,
  `src/vergil_tooling/lib/release/bump.py`,
  `src/vergil_tooling/lib/release/conflict.py`,
  `src/vergil_tooling/lib/changelog.py`
- Test: `tests/vergil_tooling/test_git.py`, `test_worktrees.py`,
  `test_changelog.py`, and the existing tests of each modified module

**Interfaces:**

- Produces, in `vergil_tooling.lib.git`:
  - `read_raw(*args: str) -> str`: like `read_output`, same environment and
    retry handling, but returns stdout unstripped;
  - `read_records(*args: str) -> list[str]`: runs `read_raw` and splits on
    NUL, dropping only the one empty string after the final terminator. The
    caller passes `-z`.

- [ ] **Step 1: Write the failing tests:**
  - `test_read_raw_keeps_leading_and_trailing_whitespace`;
  - `test_read_records_splits_on_nul_and_keeps_newlines` (`"a b\0c\nd\0"` →
    `["a b", "c\nd"]`);
  - `test_dirt_preview_keeps_first_character` (a first porcelain record whose
    status is a space and an `M`, for `src/x.py` → preview names `src/x.py`);
  - `test_status_z_path_with_newline`;
  - `test_status_z_rename_record_order` (`-z` gives the new path first, then
    the old one as its own record);
  - `test_ls_files_z_path_with_space` for `_newest_mtime`, `images.py` and
    `bump.py`;
  - `test_diff_name_only_z_path_with_space` for `diff_workflow_files`,
    `bump.py` and `conflict.py`;
  - `test_worktree_list_z_path_with_space`;
  - `test_file_at_keeps_trailing_newline` and
    `test_changelog_keeps_leading_whitespace`.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement** the two helpers, then convert each site:
  `status --porcelain -z`, `ls-files -z`, `diff --name-only -z`,
  `worktree list --porcelain -z` (git 2.36 or later; the pinned Ubuntu 24.04
  ships 2.43), and `read_raw` for `changelog.py` and `file_at`. Leave the
  `bin/vrg_finalize_pr.py` items to Task 3b.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope git --message "read
  git paths as NUL-separated, unstripped records"`, then report-ready.

### Task 3: §7.3 git ref and format queries (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/git.py` (`merged_branches`,
  `local_branches`, `branch_upstreams`), `src/vergil_tooling/bin/vrg_git.py`
  (`_upstream_is_integration_branch`), `src/vergil_tooling/lib/release/merge.py`,
  `src/vergil_tooling/lib/release/preflight.py` (outside
  `_check_branch_and_tree`), `src/vergil_tooling/lib/repo_init.py`,
  `src/vergil_tooling/bin/vrg_reword.py`, `src/vergil_tooling/lib/github.py`
  (`remote -v`), `src/vergil_tooling/lib/upstream_refresh.py`
- Test: the existing tests of each modified module

**Interfaces:**

- Consumes: Task 2's `read_records`.
- Produces: `git.current_branch_ref() -> str | None`, the full ref from
  `symbolic-ref -q HEAD` (`None` when detached), for Task 3b.

- [ ] **Step 1: Write the failing tests:**
  - `test_branch_lists_use_lstrip_2` (a branch named `heads/x` and a tag named
    `x` side by side; each branch is listed once, by its own name);
  - `test_upstream_remote_with_slash` (remote `team/fork`, upstream branch
    `develop` → remote `team/fork`, ref `refs/heads/develop`);
  - `test_current_branch_ref_detached_is_none`;
  - `test_release_compares_full_ref` (merge.py and preflight.py);
  - `test_repo_init_step_subject_anchored` (a subject that only contains the
    step name does not match);
  - `test_reword_accepts_abbreviated_pick`;
  - `test_remote_v_split_on_tab` (a URL path with a space);
  - `test_upstream_refresh_pins_diff_config` (with `diff.noprefix=true`,
    `diff.mnemonicPrefix=true` and `diff.suppressBlankEmpty=true` set in a
    test repo's config, the parsed patch and hunks are unchanged).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement** the conversions as spec §7.3 lists them. In
  `upstream_refresh.py`, pass `-c diff.noprefix=false
  -c diff.mnemonicPrefix=false -c diff.suppressBlankEmpty=false
  --src-prefix=a/ --dst-prefix=b/` (git's defaults, which the parser expects)
  on each diff it parses. Leave `_check_branch_and_tree` and `bin/vrg_finalize_pr.py` to
  Task 3b.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope git --message "query
  refs and upstreams through exact git formats"`, then report-ready.

### Task 3b: items in functions #1218 rewrites (vergil-tooling)

Blocked by Tasks 2, 3 and 9, vergil-tooling#4567 and vergil-tooling#4573.

**Files:**

- Modify: `src/vergil_tooling/lib/release/preflight.py`
  (`_check_branch_and_tree`), `src/vergil_tooling/bin/vrg_finalize_pr.py`
- Test: `tests/vergil_tooling/test_release_preflight.py`,
  `test_vrg_finalize_pr.py`

**Interfaces:**

- Consumes: Task 2's `read_records`, Task 3's `current_branch_ref`.

- [ ] **Step 1: Re-read** both files as #1218 left them and list the sites
  that still read text: the current-branch comparison, the file display, the
  `status --porcelain` reads and the `branch --list <name>` existence test.
- [ ] **Step 2: Write the failing tests:**
  - `test_preflight_compares_full_ref`;
  - `test_finalize_branch_exists_is_not_a_glob` (a branch `feature/a*` exists
    and `feature/ab` does not; the existence test for `feature/ab` is false);
  - `test_finalize_status_z_path_with_space`.
- [ ] **Step 3: Implement**: `show-ref --verify --quiet refs/heads/<b>` for
  existence, `-z` reads, and `current_branch_ref` for the comparison.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope finalize-pr --message
  "read branch and status output in release preflight and finalize exactly"`,
  then report-ready.

### Task 4a: the gh error helper; not-found and retry_known_resource (vergil-tooling)

**Files:**

- Create: `src/vergil_tooling/lib/gh_errors.py`
- Modify: `src/vergil_tooling/lib/retry.py` (`is_not_found`,
  `_NOT_FOUND_PATTERNS`), and each `retry_known_resource` caller that needs a
  conversion: `lib/release/merge.py`, `lib/release/tracking.py`
  (`read_issue_body`), `lib/release/confirm.py`,
  `lib/release/package_index.py`, `lib/promote.py`,
  `lib/upstream_refresh.py`, `lib/automation_identity.py`
- Test: `tests/vergil_tooling/test_gh_errors.py`, `test_retry.py`, and the
  callers' tests

**Interfaces:**

- Produces, in `vergil_tooling.lib.gh_errors`:
  - `ErrorKind = Literal["not_found", "unprocessable", "auth", "rate_limited", "server", "other"]`;
  - `rest_status(result: subprocess.CalledProcessError | subprocess.CompletedProcess[str]) -> int | None`:
    the HTTP status from a `gh api -i` response's status line, or from the
    JSON error body. It takes a completed process too, for
    `github.is_http_404`, whose `gh api -i` call returns its non-zero exit
    as an answer;
  - `graphql_error_types(exc: subprocess.CalledProcessError) -> list[str]`:
    the `type` of each entry in the response's `errors` list;
  - `porcelain_status(exc: subprocess.CalledProcessError) -> int | None`: the
    status from an anchored match on the stderr line of the one subcommand
    left on text, the streamed `gh run watch`. Its pattern is built from that
    subcommand's recorded real 404 output, not an assumed shape;
  - `classify(exc: subprocess.CalledProcessError) -> ErrorKind`, which tries
    the three in that order. A failure none of them can read is `other`,
    never a guess.
- Produces: `retry.is_not_found(exc) -> bool` is `classify(exc) ==
  "not_found"`. `retry_known_resource`'s signature is unchanged.
- `github.pr_state` keeps its signature and return values.

- [ ] **Step 1: Write the failing tests** in `test_gh_errors.py`, from
  recorded real `gh` output kept as fixtures under
  `tests/vergil_tooling/data/gh_errors/`:
  - `test_rest_404_is_not_found`;
  - `test_graphql_not_found_type` (`{"errors":[{"type":"NOT_FOUND",…}]}`);
  - `test_run_watch_404_line` (the recorded `gh run watch` 404 output);
  - `test_404_in_unrelated_message_is_other` (a body that mentions `404` in
    passing);
  - `test_unreadable_failure_is_other`.
- [ ] **Step 2: Write the failing tests** in `test_retry.py`:
  - `test_known_resource_retries_graphql_not_found`;
  - `test_known_resource_retries_rest_404`;
  - `test_known_resource_retries_run_watch_404` (streamed `gh run watch`);
  - `test_read_issue_body_retries_lag_404` and
    `test_known_pr_state_retries_lag_404` (each against a recorded `gh` 404
    that then resolves).
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 4: Implement** the module and point `is_not_found` at it. Convert
  each `retry_known_resource` caller to a form with a structured error:
  - `gh release view` (`confirm.py`) → `gh api
    repos/{owner}/{repo}/releases/tags/{tag}`;
  - `gh run view` (`confirm.py`, `package_index.py`) → `gh api
    repos/{owner}/{repo}/actions/runs/{id}`, and its jobs from
    `…/runs/{id}/jobs` through `github.read_json_paginated`, with the field
    names the callers read mapped from the REST response;
  - GraphQL-backed porcelain reports a missing resource only as stderr text
    (`GraphQL: Could not resolve to a …`), never as a structured `errors`
    list, so every such read under `retry_known_resource` moves to
    `gh api graphql` (or the REST equivalent):
    - `gh issue view … --json body` in `lib/release/tracking.py`
      `read_issue_body`;
    - `gh pr view` in `lib/release/merge.py` `known_pr_state`, converted in
      `merge.py` itself; `github.pr_state` stays unchanged, since
      `lib/pr_merge.py` also calls it;
  - `gh run watch` stays on text, through `porcelain_status`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS, with
  the release tests unchanged.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope github --message
  "classify gh failures by REST status, GraphQL error type or HTTP line"`,
  then report-ready.

### Task 4b: the other classifiers, auth exit code, check rollup (vergil-tooling)

Blocked by Task 4a.

**Files:**

- Modify: `src/vergil_tooling/lib/remote_config.py` (`is_not_found`),
  `src/vergil_tooling/lib/github.py` (`is_repo_not_found`, `is_http_404`,
  `_is_missing_token_error`, the "no checks reported" reads),
  `src/vergil_tooling/lib/github_config.py`,
  `src/vergil_tooling/lib/dependabot_alerts.py`,
  `src/vergil_tooling/lib/rulings/draft_ref.py`,
  `src/vergil_tooling/lib/repo_init.py`, `src/vergil_tooling/lib/ref_refusal.py`,
  `src/vergil_tooling/bin/vrg_docs_stage.py`
- Test: the existing tests of each modified module

**Interfaces:**

- Consumes: Task 4a's `classify`, `rest_status`, `graphql_error_types`.
- Produces: `github.is_missing_auth(exc: subprocess.CalledProcessError) -> bool`,
  true on `gh` exit code 4.

- [ ] **Step 1: Write the failing tests:**
  - `test_remote_config_404_in_body_is_not_not_found`;
  - `test_repo_not_found_from_graphql_type` (`is_repo_not_found` on the
    converted `gh api graphql` call's `NOT_FOUND` error);
  - `test_repo_visibility_values_unchanged` (`PUBLIC`, `PRIVATE` and
    `INTERNAL` come back exactly as before, and an unknown value raises);
  - `test_422_from_status_not_text` (`github_config`, `ref_refusal`,
    `repo_init`);
  - `test_missing_auth_is_exit_code_4` (and exit code 1 with the word
    `token` in the message is not);
  - `test_no_checks_read_from_status_check_rollup` (an empty
    `statusCheckRollup`).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement** each site on the Task 4a helper. A site whose
  subcommand reports a missing resource only as text is converted the same
  way as in Task 4a. `gh repo view --json visibility` in `repo_visibility`
  becomes `gh api graphql` on `repository(owner:, name:) { visibility }`:
  GraphQL's `visibility` is already `PUBLIC`/`PRIVATE`/`INTERNAL`, which
  `is_public`, `security_records.advisory_home`, `vrg-dependabot-audit` and
  epic homing compare exactly, and a missing repo arrives as a `NOT_FOUND`
  entry in the `errors` list, which `graphql_error_types` reads.
  `repo_visibility` raises on any other value.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope github --message
  "read gh not-found, unprocessable and auth failures from status and exit
  code"`, then report-ready.

### Task 5: §7.5 the retry classifier (vergil-tooling)

Blocked by Task 4a.

**Files:**

- Modify: `src/vergil_tooling/lib/retry.py` (`_RETRYABLE_PATTERNS`,
  `_READ_RETRYABLE_PATTERNS`, `is_retryable`, `is_retryable_read`)
- Create: `tests/vergil_tooling/data/retry/` (recorded failures, one file
  each, named for the pattern and the incident or issue it came from)
- Test: `tests/vergil_tooling/test_retry.py`

- [ ] **Step 1: Collect** a recorded real failure for each pattern from the
  issues and incidents the pattern's comment cites (for example #2835 for the
  502 phrasing). Each fixture holds stdout and stderr separately, with the
  stream of each line taken from a real capture or from the tool's documented
  output stream, and names its source. A pattern with no recorded failure,
  or whose stream cannot be established, keeps its current matching (wording
  and stdout-plus-stderr) and is listed in the PR body as flagged. For the
  bare `timeout` and `eof` patterns, which cite no incident, look beyond the
  pattern comments too: CI logs and the incident issues #2835 and #3148.
- [ ] **Step 2: Write the failing tests:**
  - `test_each_pattern_matches_its_recorded_failure` (parametrized over the
    data directory);
  - `test_retry_word_in_unrelated_failure_not_retried` (stderr naming a file
    `timeout_eof.py`, and stdout containing `timeout`);
  - `test_stdout_only_match_not_retried`.

  These two tests, and Review Focus 3, cover only patterns that pass Step 1.
  If `timeout` or `eof` stays flagged, the tests use the word of a converted
  pattern instead, and the PR body records that Review Focus 3 is still open
  for the flagged word.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 4: Implement**: both classifiers read stderr only for every
  pattern whose fixture establishes stderr, and each such pattern becomes an
  anchored or word-bounded regular expression. Flagged patterns keep their
  current matching.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope retry --message
  "match retryable failures on stderr with anchored patterns"`, then
  report-ready.

### Task 6: §7.6 structured gh write results (vergil-tooling)

Blocked by Task 4a.

**Files:**

- Modify: `src/vergil_tooling/lib/intake_route.py` (issue transfer),
  `src/vergil_tooling/lib/release/tracking.py` (issue creation),
  `src/vergil_tooling/bin/vrg_docs_stage.py` (release listing)
- Test: `test_intake_route.py`, `test_release_tracking.py`,
  `test_vrg_docs_stage.py`

**Interfaces:**

- Consumes: Task 4a's `classify`; `github.read_json_paginated`.

- [ ] **Step 1: Settle** how `gh --paginate` concatenates pages for the
  release listing, and record it in the PR body.
- [ ] **Step 2: Write the failing tests:**
  - `test_transfer_reads_new_issue_from_mutation` (the `transferIssue`
    response's `issue.number` and `issue.url`);
  - `test_tracking_issue_create_reads_json_response`;
  - `test_docs_stage_release_listing_spans_pages`.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 4: Implement** the three conversions. `lib/pr_merge.py` is not
  touched.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope github --message
  "read gh write results from their JSON responses"`, then report-ready.

### Task 7: §7.7 cloud VM tools (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/vm_cloud.py` (readiness),
  `src/vergil_tooling/bin/vrg_vm.py` (gcloud and az existence),
  `src/vergil_tooling/lib/vm_provider.py` (gcloud project), the `tofu version`
  parse
- Test: `test_vm_cloud.py`, `test_vrg_vm.py`, `test_vm_provider.py`

**Interfaces:**

- Produces, in `lib/vm_cloud.py`:
  `CloudInitResult(ready: bool, status: str, warnings: list[str], errors: list[str])`
  and `parse_cloud_init_status(stdout: str, returncode: int) -> CloudInitResult`,
  for Task 12 to mirror.

- [ ] **Step 1: Settle** the `az` exit code for a missing resource, and record
  it in the PR body.
- [ ] **Step 2: Write the failing tests** (spec §7.7's definition of
  degraded):
  - `test_clean_status_is_ready`;
  - `test_error_status_is_not_ready`;
  - `test_error_level_recoverable_error_is_not_ready` (`status: done`, exit
    2, an ERROR entry);
  - `test_deprecation_only_status_is_ready` (exit 2, only DEPRECATED and
    WARNING entries, each reported in `warnings`);
  - `test_unparseable_status_raises`;
  - `test_gcloud_missing_instance_from_empty_list`;
  - `test_az_missing_resource_from_exit_code`;
  - `test_gcloud_project_from_value_format`;
  - `test_tofu_prerelease_version_accepted` (`v1.10.0-rc1`).
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 4: Implement** on `cloud-init status --format json`,
  `gcloud … list --filter=… --format=json`, the `az` exit code and
  `--format=value(core.project)`.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope vm --message "judge
  cloud VM readiness and resources from JSON and exit codes"`, then
  report-ready.

### Task 8a: §7.8 host and VM guest tools (vergil-tooling)

Blocked by Task 7.

**Files:**

- Modify: `src/vergil_tooling/lib/vm_packages.py` (apt queries,
  `_remove_uv_copy`), `src/vergil_tooling/lib/host_methods.py` (brew),
  `src/vergil_tooling/lib/vm_guest.py` (`get_tooling_version`, the occupancy
  script, `vm_probe`), `src/vergil_tooling/bin/vrg_vm.py` (`_probe_running`)
- Test: `test_vm_packages.py`, `test_host_methods.py`, `test_vm_guest.py`,
  `test_vrg_vm.py`

**Interfaces:**

- Produces: `vm_guest.ProbeError(RuntimeError)`, raised by `vm_probe` on a run
  failure or a parse failure. `vm_probe`'s return type is unchanged.
  `_probe_running` returns `Mapping[str, ProbeResult | ProbeError]`.

- [ ] **Step 1: Write the failing tests:**
  - `test_apt_query_runs_under_c_locale`;
  - `test_brew_installed_version_from_json_v2`;
  - `test_uv_tool_list_reader_anchored` (for `get_tooling_version` and
    `_remove_uv_copy`: a line `vergil-tooling-extra v1.0` is not
    vergil-tooling);
  - `test_uv_tool_list_bad_version_raises`;
  - `test_tooling_version_uv_absent_is_checked` (no `uv` on the guest is an
    explicit, tested absence that falls through to the deb version) and
    `test_tooling_version_uv_failure_raises` (a `uv tool list` that fails for
    any other reason raises, as does a failing `dpkg-query`);
  - `test_probe_raises_on_run_failure` and `test_probe_raises_on_parse_failure`;
  - `test_occupancy_script_fails_when_loginctl_fails` (a stub `loginctl` that
    exits 1 makes the script exit non-zero);
  - `test_vm_list_shows_error_marker_for_one_vm` (one VM's probe raises; its
    AGENTS, HUMANS and SPEC columns show `error`, and the other VMs' rows list
    normally);
  - `test_vm_list_spec_column_on_failed_probe` (a present dedicated VM whose
    probe raises shows `error` in SPEC, never `ok` or `NEEDS-REBUILD`).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement**: `LC_ALL=C` on the apt queries; `brew info
  --json=v2`; the `uv tool list` readers on the strict parse
  `host_methods` already uses; `get_tooling_version` without its
  `2>/dev/null`, `except CalledProcessError: pass` and `|| true` masks,
  checking for `uv` with `command -v` first; the occupancy script without
  `2>/dev/null` and `|| echo ""` on its `loginctl` calls, exiting non-zero
  when one fails; `vm_occupancy`'s docstring updated, since its "(0, 0) on
  failure" contract no longer holds;
  `_probe_running` catching `ProbeError` per VM and storing it as that VM's
  entry in the mapping it returns, so `_occupancy` and `_present_spec_state`
  both see the failure and show `error`, never treating the VM as not
  running.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope vm --message "read
  host and guest tool output strictly and surface probe failures per VM"`,
  then report-ready.

### Task 8b: §7.8 package tools and container-cache clean-all (vergil-tooling)

**Files:**

- Modify: `src/vergil_tooling/lib/package/python_builder.py`,
  `src/vergil_tooling/lib/package/install_test.py`, the rpm requires and
  `rpmkeys --checksig` reads in `src/vergil_tooling/lib/package/`,
  `src/vergil_tooling/bin/vrg_container_cache.py` (`clean-all`)
- Test: the existing tests of each modified module

- [ ] **Step 1: Write the failing tests:**
  - `test_rpm_requires_from_query_format`;
  - `test_rpmkeys_checksig_under_c_locale`;
  - `test_dpkg_version_with_epoch` (`1:2.3-1` compares by epoch first);
  - `test_install_file_lists_clean` (`dpkg -L` and `rpm -ql` lists hold only
    paths: no blank lines, no `package diverts others to:` lines);
  - `test_clean_all_matches_exact_cache_tag` (an image tagged `a--b` that is
    not a cache tag is kept);
  - `test_clean_all_removes_legacy_cache_tag` (an image in the pre-#4162
    `<base_tag>--<branch>--<hash>` shape is still removed).
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement**: `rpm -qp --qf`, `LC_ALL=C` for `rpmkeys`, the
  epoch in the version compare, and `clean-all` matching through
  `lib/container_cache.py`'s existing cache-ref parser, which accepts both
  `_CACHE_REF_RE` and `_LEGACY_CACHE_REF_RE`.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope package --message
  "read package metadata through query formats and a fixed locale"`, then
  report-ready.

### Task 9: security fixes (vergil-tooling)

Not blocked.

- **Advisories:** GHSA-rp4r-x254-p53h, GHSA-f4j7-x3f8-59gv,
  GHSA-j7c6-37h8-hjvg.
- **Design:** each advisory records its own fix. The implementer reads them
  with `vrg-security-record show <GHSA-id>` in the terminal only.
- **Tests:** one test per advisory, with the input its advisory names.
- **Public text:** the issue, commits, PR and report-ready notes name the
  advisories by GHSA ID only (spec §8).
- **Split:** into lettered sub-tasks if the three do not fit one PR.

### Task 11: security fixes (vergil-claude-plugin)

- **Advisories:** GHSA-cgp7-q74r-m6vh, GHSA-8mqc-j6w7-2fxp,
  GHSA-94p8-hq6c-5r6h.
- **Design, tests, public text and split:** as for Task 9; the tests go in
  `hooks/tests/`.

### Task 10: §7.9 hook parsing; worktree convention always enforced (vergil-claude-plugin)

Blocked by Task 11.

**Files:**

- Modify: `hooks/scripts/detect-deprecation-warnings.sh`,
  `hooks/scripts/capture-merge-review-result.sh`,
  `scripts/bin/validate-custom`, `hooks/scripts/block-protected-branch-work.sh`,
  `hooks/scripts/block-worktree-bypass-write.sh`
- Test: `hooks/tests/capture-merge-review-result.test.sh`,
  `hooks/tests/block-worktree-bypass-write.test.sh`,
  `hooks/tests/hook-modes.test.sh`, and three new suites:
  `hooks/tests/detect-deprecation-warnings.test.sh`,
  `hooks/tests/validate-custom.test.sh` and
  `hooks/tests/block-protected-branch-work.test.sh` (the plugin has no tests
  of `validate-custom` or of that hook's worktree enforcement today)

- [ ] **Step 1: Write the failing tests:**
  - deprecation hook: an object-shaped `tool_response` is read, through
    `lib/hook-input.sh`;
  - capture hook: the hash is taken from stdin (no file name in the
    `sha256sum` output), and input that is not a single JSON object is
    refused;
  - `validate-custom`: a path containing a glob character is read literally,
    and anything other than exactly one `ls-files -z --stage` record fails;
  - both worktree hooks: in a repo with `vergil.toml` and **no**
    `.worktrees/` line in `.gitignore`, a commit or write from the main tree
    is denied; in a repo without `vergil.toml`, both hooks allow it;
  - hook tests read verdicts with `jq -e`, not by grepping jq's
    pretty-printed output.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`; expect the new
  tests to fail.
- [ ] **Step 3: Implement.** In both worktree hooks, remove the `.worktrees/`
  grep and the main/develop fallback, and enforce the convention wherever
  `is_managed_repo` holds. Update each hook's header comment to say so.
- [ ] **Step 4: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 5: Commit** `vrg-commit --type fix --scope hooks --message
  "parse hook input strictly and always enforce worktrees in a Vergil repo"`,
  then report-ready.

### Task 12: §7.10 readiness probe, profile checks, tests (vergil-vm)

**Files:**

- Create: `templates/probe/readiness-judge.sh`, outside `templates/provision/`
  so the provision-manifest check does not apply to it. It defines one shell
  function, `judge_readiness`, which reads the `cloud-init status --format
  json` output on stdin and returns 0 for ready, non-zero otherwise, printing
  each warning. Also create `tests/check-readiness-probe.sh`, which sources the
  file directly.
- Modify: `templates/agent.yaml.skel`. The readiness probe pulls the function
  in through `@@INCLUDE` at the top of its script, before the `bash -c`, and
  exports it with `export -f judge_readiness`. Inside the loop it captures the
  status explicitly, with no pipe, so `pipefail` cannot turn cloud-init's exit
  code 2 on a deprecation-only boot into a failure:
  `status_json=$(cloud-init status --format json) || cloud_init_rc=$?`. It
  then judges from the JSON alone; the recorded exit code is logged, not
  swallowed. Also modify `templates/agent.yaml` (regenerated with
  `scripts/build-template.sh`, so `tests/check-template-generation.sh` still
  passes), `templates/provision/40-profile.sh`, the `tests/check-*.sh` scripts
  that grep `dpkg -l`, `nerdctl images`, `ss`, the login shell or `git config`

**Interfaces:**

- Mirrors Task 7's definition of degraded (spec §7.7).

- [ ] **Step 1: Write the failing tests** in `tests/check-readiness-probe.sh`,
  sourcing `templates/probe/readiness-judge.sh` and feeding `judge_readiness`
  recorded `cloud-init status --format json` output on stdin:
  clean → ready; error → not ready; an ERROR-level recoverable error → not
  ready; deprecation-only → ready, with the warning printed; buildkitd and
  containerd matched on their exact arguments or unit state (a process whose
  command line only contains `buildkitd` does not count).
- [ ] **Step 2: Write the failing profile checks:** a virtual package is
  found by `apt-get install -s`; an installed vagrant plugin is found from
  `plugins.json` (or by fixed-string match), and one whose name is a prefix of
  another is not.
- [ ] **Step 3: Run** `vrg-container-run -- vrg-validate`; expect the new
  checks to fail.
- [ ] **Step 4: Implement**, and convert the test scripts:
  `dpkg-query -W -f='${db:Status-Status}'`, `nerdctl image inspect`, an exact
  `ss` port match, an exact login-shell comparison, `git config --get-all`
  with a whole-line match.
- [ ] **Step 5: Run** `vrg-container-run -- vrg-validate`; expect PASS.
- [ ] **Step 6: Commit** `vrg-commit --type fix --scope readiness --message
  "judge VM readiness from cloud-init JSON and exact process checks"`, then
  report-ready.

### Task D1: vergil-tooling release carrying Task 7, installed (deployment, .github)

Blocked by Task 7.

- **Precondition (human-attested):** a vergil-tooling release containing
  Task 7 is published, its version named in a comment.
- **Human-run steps:** `vrg-host upgrade vergil-tooling` on the host.
- **Procedure (agent-safe):** `vrg-vm update --all`; then on one VM,
  `dpkg-query -W -f='${Version}' vergil-tooling` names the release.
- **SUCCESS:** the release is installed on the host and every VM, with the
  version recorded.

### Task D2: vergil-tooling release completing the gate, installed (deployment, .github)

Blocked by D1 and Tasks 2–6, 8a, 8b and 9.

- **Precondition (human-attested):** a vergil-tooling release containing
  Tasks 2–6, 8a, 8b and 9 is published, its version named in a comment.
- **Human-run steps:** `vrg-host upgrade vergil-tooling` on the host.
- **Procedure (agent-safe):**
  1. `vrg-vm update --all`.
  2. `vrg-vm list` shows every running VM's occupancy (no `error` marker on a
     healthy VM).
  3. The release notes list each task.
- **SUCCESS:** all three hold, with the version recorded.

### Task D3: vergil-claude-plugin release, installed (deployment, .github)

Blocked by Tasks 10 and 11.

- **Precondition (human-attested):** a vergil-claude-plugin release containing
  Tasks 10 and 11 is published, its version named in a comment.
- **Human-run steps:** `vrg-host plugins` on the host.
- **Procedure (agent-safe):**
  1. `vrg-vm update --all`.
  2. Neither installed `block-protected-branch-work.sh` nor
     `block-worktree-bypass-write.sh` contains the exact string
     `grep -qxF '.worktrees/'` (the `.worktrees/` case pattern, comments and
     deny message stay).
  3. In a scratch repo with `vergil.toml` and no `.worktrees/` line, run each
     installed hook directly on a hook-input JSON: a main-tree commit for
     `block-protected-branch-work.sh`, a main-tree file write for
     `block-worktree-bypass-write.sh`. Each prints a
     `permissionDecision` of `deny`, checked with `jq -e`. (A raw `git commit`
     is not a valid check: `block-raw-git-commit.sh` denies it regardless.)
- **SUCCESS:** all three hold, with the version recorded.

### Task D4: vergil-vm release (deployment, .github)

Blocked by Task 12.

- **Precondition (human-attested):** a vergil-vm release containing Task 12 is
  published, its version named in a comment.
- **Procedure (agent-safe):** the release's generated templates contain the
  JSON readiness probe.
- **SUCCESS:** that holds, with the version recorded.

### Tasks V1a and V1b: fresh VMs come up ready (validation, .github)

Two validations, one per backend, since each readiness check runs on only one:
**V1a** exercises `vrg-vm`'s cloud-init gate in `lib/vm_cloud.py` (Task 7) on
a cloud VM, blocked by D1. **V1b** exercises the readiness probe in
`templates/agent.yaml.skel` (Task 12) on a Lima VM, blocked by D1 and D4.
Each runs the same procedure on its backend.

- **Precondition (machine probe):** its blocking deployment tasks are closed
  with `Outcome: SUCCESS`.
- **Procedure:**
  1. Build a fresh agent VM on the backend with the released vergil-tooling
     (and, for V1b, vergil-vm). Record that it comes up ready, and the full
     `cloud-init status --format json` output, including what
     `recoverable_errors` holds.
  2. On a scratch VM on the same backend, force a recoverable ERROR in
     cloud-init (for example, a failing `runcmd` entry). Record that the
     backend's readiness check reports it as not ready.
  3. If step 1's `recoverable_errors` holds WARNING or DEPRECATED entries,
     record that the VM was still judged ready and the warnings were printed.
- **SUCCESS:** steps 1 and 2 behave as stated, and step 3's result is recorded.

### Fixture runs (F)

Seed with `fixture-run vergil-project/.github#1268 python-app,python-lib,rust-app,rust-lib,ts-lib,ts-app --validates "vergil-tooling release completing epic #1268's gate (machine-readable tool output)" --after <D2 ref>`, once D2 is filed. The change reaches the release tooling, the retry and not-found classifiers and the package builders that every fixture's release uses (spec §7.11).
