# Machine-readable tool output — design

- **Epic:** vergil-project/.github#1268
- **Planning issue:** vergil-project/vergil-tooling#4108 (review history is in
  its comments)
- **Status:** draft

## 1. Intent

Vergil runs many command-line tools and reads what they print: `git`, `gh`,
cloud-init, apt, rpm and dpkg, brew, uv, gcloud and az. Wherever such a tool
offers a form of its output designed for parsing, vergil should read that form
and not the text meant for a person at a terminal. The forms in question are:

- NUL-separated records (`git … -z`);
- exact format strings (`git for-each-ref --format`, `--format=value(…)`);
- JSON modes (`gh --json`, `cloud-init status --format json`,
  `brew info --json=v2`);
- HTTP status codes, read from the response and not from the message text;
- exit codes;
- a fixed locale (`LC_ALL=C`) where a tool offers nothing better.

This epic fixes the sites that read human-readable text today, in
vergil-tooling, vergil-claude-plugin and vergil-vm.

## 2. Why

Text written for people is fragile to parse:

- values that contain spaces, newlines, quotes or leading characters are split
  or trimmed wrongly;
- the wording and layout change between tool versions;
- translated locales change the words a parser looks for;
- a substring test such as "the output contains `404`" matches unrelated
  text.

Each of these costs correctness, and some cost robustness against crafted
input: a branch name, path, commit subject or error body is controlled by
whoever wrote it. The machine-readable forms exist to remove exactly these
problems, and are usually simpler to handle.

An ad-hoc run on the planning issue surveyed every vergil-tooling `src/` file
that runs a tool (about 155 files), the plugin's `hooks/` and `scripts/` and
vergil-vm's `scripts/`, `templates/`, `opentofu/` and `tests/`. Most sites
already use a machine-readable form. About 70 vergil-tooling sites across about
45 files do not, and the plugin and vergil-vm have a short list each. The
survey is the input to this design; its site list is in the planning issue's
comments. The sites that matter most:

- `lib/worktrees.py` reads `git status --porcelain` through a helper that
  strips the whole output, so the first character of the first path is lost in
  the dirt preview.
- `lib/vm_cloud.py` reads `cloud-init status --long` as text. On cloud-init
  23.4 and later a degraded boot shows `status: done` with exit code 2, and is
  accepted as ready.
- `lib/retry.py` retries on bare substrings such as `eof` and `timeout` across
  stdout and stderr combined, so an unrelated failure, a write included, can
  be retried.
- `lib/remote_config.py` `is_not_found` is true for any `404` anywhere in the
  output.
- `lib/vm_packages.py` parses apt's `Candidate:` line, which is translated, with
  no fixed locale.

## 3. Goals

- Every site in scope reads a machine-readable form, or, where the tool has
  none, a fixed locale and an anchored match.
- Each fix is proven by a test that feeds in the input the old code misread.
- Each pattern is handled once, by a shared helper, not re-solved at every
  site.

## 4. Non-goals

- **Machine-readable output from vergil's own CLIs.** `vrg-issue-create`
  output, the `vrg-validate` stage report and the `vrg-container-run` banner
  are parsed by other vergil code, but changing them changes contracts that
  other repos consume. That work needs its own design and is planned as
  vergil-project/.github#1271.
- **A central layer for reading tool output.** Most sites are already correct;
  routing every tool call through one module is a refactor the problem does not
  call for.
- **Rework of sites already using a machine-readable form**, and style-only
  changes to them.
- **The merge path in `lib/pr_merge.py`**, which epic
  vergil-project/.github#1218 owns (§7.6).

## 5. Anti-goals

- A fix that changes what a command does for ordinary input. Apart from the
  defects being fixed, behaviour for ordinary values stays the same.
- A narrower retry classifier that stops retrying failures that are genuinely
  transient. Tightening retries must not trade a wrong retry for a missed one
  (§7.4).
- A fallback that hides a parse failure. Where a machine-readable read fails to
  parse, the code raises; it never returns a default such as `(0, 0)` or an
  empty list.

## 6. Approaches

1. **By pattern group (chosen).** Each group adds its shared helper and
   converts every site that matches it, in one reviewable PR. A reviewer checks
   one helper and its uses together, and a group can be rolled back on its own.
2. **By file or module.** Fix every problem in a file at once. This gives fewer
   merge conflicts, but each PR mixes unrelated reasoning (git paths, HTTP
   errors, locale), which makes review and rollback harder. Not chosen.
3. **One central output layer.** Route every tool call through one parsing
   module. Far bigger than the problem, since most sites are already right.
   Not chosen.

## 7. Design

### 7.1 Shared rules

- **Re-verify first.** The survey is a snapshot. Each task re-reads its sites
  against the current code before changing them, drops a site that is already
  fixed or gone, and adds a matching site the survey missed. For example,
  `lib/retry.py` gained `_READ_RETRYABLE_PATTERNS` (#4119) beside
  `_RETRYABLE_PATTERNS`, and both are in scope for §7.5.
- **Settle open claims in the task that needs them.** The survey was unsure of
  these; the task that depends on each confirms it from the tool's
  documentation or a real run before relying on it:
  - how `gh --paginate` concatenates pages, for the `vrg_docs_stage` release
    listing (§7.6);
  - that `az` exits 3 for a missing resource (§7.7);
  - which systemd versions support `loginctl -o json`, wherever a task reads
    `loginctl`.

  A claim that turns out false changes the fix, not the goal: the task uses
  whatever form the tool really offers.
- **Tests with the input that broke.** Each fixed site gets a test whose input
  is the case the old parse misread: a path with a space, newline or leading
  character; a branch name with a slash; a translated apt line; a degraded
  cloud-init status; an error body that mentions `404` in passing.
- **No silent failure.** A machine-readable read that does not parse raises
  with the tool's name and what was expected.

### 7.2 Git file lists and paths

- Add a `git` read helper that returns output unstripped, for `-z`, porcelain
  and blob output.
- Read with `-z` and split on NUL:
  - `status --porcelain` in `lib/worktrees.py`;
  - `ls-files` in `worktrees._newest_mtime`,
    `lib/update_deps/updaters/images.py` and `lib/release/bump.py`;
  - `diff --name-only` in `git.diff_workflow_files`, `lib/release/bump.py` and
    `lib/release/conflict.py`;
  - `worktree list --porcelain`;
  - the file display in `bin/vrg_finalize_pr.py`.
- Use the unstripped read in `lib/changelog.py` and `git.file_at`, where
  leading or trailing whitespace is content.

### 7.3 Git ref and format queries

- `git.merged_branches`, `local_branches` and `branch_upstreams`: use
  `for-each-ref --format=%(refname:lstrip=2) refs/heads` in place of
  `%(refname:short)`, which can be ambiguous.
- `vrg_git._upstream_is_integration_branch`: use `%(upstream:remotename)` and
  `%(upstream:remoteref)` in place of splitting on `/`.
- `lib/release/merge.py` and `lib/release/preflight.py`: compare the current
  branch with `symbolic-ref -q HEAD` or `branch --show-current`, not
  `rev-parse --abbrev-ref HEAD`.
- `repo_init` step detection: `log --format=%s` with an anchored pattern, not
  `--oneline`.
- `vrg_reword`: accept the abbreviated `p` todo command.
- `vrg_finalize_pr`: test that a branch exists with `show-ref --verify`, not
  `branch --list <name>`, which treats the name as a glob.
- `lib/github.py`: split `remote -v` on the tab.
- `lib/upstream_refresh.py`: pin the diff prefixes and
  `diff.suppressBlankEmpty` on the command line, so user git config cannot
  change the patch and hunk text it parses.

### 7.4 GitHub error classification

- Add one helper that classifies a failed `gh` call by the kind of call, since
  `gh` reports errors in two shapes:
  - REST calls (`gh api` on a REST path): the HTTP status, read from
    `gh api -i` or from the JSON error body;
  - GraphQL-backed calls (`gh pr view`, `gh issue view`, `gh repo view`,
    `gh api graphql`): these answer HTTP 200 and report a missing resource
    only in the response's `errors` list, so the helper reads the GraphQL
    error `type` (`NOT_FOUND`). A caller that cannot get the structured
    errors from its subcommand is converted to `gh api graphql` or to the
    equivalent REST call.
- `retry.retry_known_resource` and every caller of `retry.is_not_found` are in
  scope (among them `lib/release/merge.py`, `lib/release/confirm.py`,
  `lib/release/package_index.py`, `lib/promote.py`,
  `lib/upstream_refresh.py` and `lib/automation_identity.py`). Each still
  recognises a missing resource in both shapes, so the retry that rides out
  GitHub's list-before-readable lag (#3137) keeps working.
- Use the helper in place of the substring matches for `404`, `not found` and
  `422`:
  - `remote_config.is_not_found`;
  - `retry.is_not_found`;
  - `github.is_repo_not_found` and `is_http_404`;
  - `lib/github_config.py`;
  - `lib/dependabot_alerts.py`;
  - `lib/rulings/draft_ref.py`;
  - `lib/repo_init.py`;
  - `lib/ref_refusal.py`.
- Detect missing authentication from `gh`'s exit code 4, in
  `bin/vrg_docs_stage.py` and `github._is_missing_token_error`.
- Read the "no checks reported" case from `statusCheckRollup`.

### 7.5 The retry classifier

- `_RETRYABLE_PATTERNS` and `_READ_RETRYABLE_PATTERNS` in `lib/retry.py` match
  stderr only, with anchored or word-bounded patterns.
- Each pattern is tested against recorded real failures: it must match the
  transient failure it exists for, and must not match a recorded unrelated
  failure.
- Kept separate from §7.4 because it changes retry behaviour across the fleet.
  A pattern that cannot be tied to a recorded transient failure is kept as it
  is and flagged, never dropped on a guess (§5).

### 7.6 GitHub write results

Read structured results in place of regexes over `gh`'s messages:

- `lib/intake_route.py` issue transfer: the GraphQL `transferIssue` mutation,
  reading the new issue from its response.
- Issue creation in `lib/release/tracking.py`: `gh api` POST, reading the JSON
  response.
- The `vrg_docs_stage` release listing: `read_json_paginated`.

This depends on §7.4, whose helper it uses for error cases.

`lib/pr_merge.py` is out of scope. Its merge-rejection handling tells apart a
branch behind base, a branch-policy block and a moved head from the merge
endpoint's reply, and the PR's state cannot replace that signal (#2856). The
merge path is being reworked by epic vergil-project/.github#1218 (parallel PR
merging), which owns any change to how those rejections are read.

### 7.7 Cloud VM tools

- `lib/vm_cloud.py`: read `cloud-init status --format json` together with the
  exit code. A degraded result is a failure, not ready.
- `bin/vrg_vm.py`: decide that a gcloud instance or disk is missing from
  `list --filter … --format=json`, and that an az resource is missing from its
  exit code, not from `not found` text.
- `lib/vm_provider.py`: read the gcloud project with
  `--format=value(core.project)`.
- The `tofu version` parse accepts prerelease versions.

### 7.8 Guest, host and package tools

- `lib/vm_packages.py`: run the apt queries under `LC_ALL=C`.
- `lib/host_methods.py`: `brew info --json=v2` in place of
  `brew list --versions`.
- `uv tool list` has no machine-readable form (uv 0.13.7), so it stays text
  under the fixed-locale-and-anchored-match rule (§3). `lib/host_methods.py`
  already parses it strictly and raises on an unexpected line; the readers in
  `lib/vm_guest.py` (`get_tooling_version`) and `lib/vm_packages.py`
  (`_remove_uv_copy`) are brought to the same standard: an anchored match on
  the tool's header line, a version field that must parse, and a raise on a
  listed line that does not.
- `lib/vm_guest.py` session probe (`vm_probe`): raise on both a failure to run
  and a failure to parse, in place of returning `(0, 0)`. `vrg-vm list` probes
  every running VM in parallel, so it catches the error per VM and shows an
  explicit error marker for that VM's occupancy; the other rows still list.
- Package queries in `lib/package/`: `rpm -qp --qf` for requires; `LC_ALL=C`
  for `rpmkeys --checksig`; the epoch in the `dpkg-query` version in
  `python_builder.py`; clean `dpkg -L` and `rpm -ql` file lists in
  `install_test.py`.
- `vrg-container-cache clean-all`: match the exact cache tag. Today it removes
  any image whose tag contains `--`.

### 7.9 vergil-claude-plugin

- `detect-deprecation-warnings.sh`: use the shared `hook-input.sh` library, and
  handle an object-shaped `tool_response`.
- `capture-merge-review-result.sh`: hash from stdin so no file name appears in
  the `sha256sum` output, and check that its input is a single JSON object.
- `validate-custom`: run `git ls-files -z --stage` with a literal pathspec, and
  require exactly one record.
- **Vergil-repo detection.** `block-protected-branch-work.sh` and
  `block-worktree-bypass-write.sh` decide whether to enforce the worktree
  convention by grepping the repo-root `.gitignore` for an exact `.worktrees/`
  line. That line is an indirect signal: it says the repo ignores git
  worktrees, not that it uses Vergil. Both hooks instead decide that a repo is
  a Vergil repo by the presence of `vergil.toml` at the main worktree's root,
  through one shared helper in `hooks/scripts/lib/`. This is a deliberate
  change of signal, not only of parsing; every Vergil repo carries both, so
  enforcement covers the same repos.
- Hook tests read verdicts with `jq -e`, not by grepping jq's pretty-printed
  output.

### 7.10 vergil-vm

- The readiness probe in `templates/agent.yaml.skel`:
  - read `cloud-init status --format json` and treat a degraded result as a
    failure, matching §7.7;
  - match the buildkitd and containerd processes on exact arguments or unit
    state, not `pgrep -f` substrings.
- `40-profile.sh`:
  - check package availability with `apt-get install -s`, since
    `apt-cache madison` misses virtual packages;
  - check installed vagrant plugins from `plugins.json`, or with a
    fixed-string match.
- Tests:
  - `dpkg-query -W -f='${db:Status-Status}'` in place of `dpkg -l | grep ^ii`;
  - `nerdctl image inspect` in place of grepping `nerdctl images`;
  - an exact `ss` port match;
  - an exact login-shell comparison;
  - `git config --get-all` with a whole-line match.

### 7.11 Delivery shape

- One task per section from §7.2 to §7.10, each one PR, filed in the repo its
  PR lands in. A section too big for one reviewable PR is split into lettered
  sub-tasks, each with what it is blocked by.
- §7.6 is blocked by §7.4. The rest are independent of each other.
- The release gate follows what the release path calls, not only the files
  under `lib/release/`. §7.2 to §7.6 and §7.8 all change code a release runs:
  release tooling, the not-found and retry classifiers every release step
  uses, release tracking, and the package builders and install tests. So the
  plan carries one vergil-tooling release after all of them, and fixture runs
  on the fixtures the release path affects, each blocked by that release.
- §7.7 and §7.10 change how a VM is judged ready, so the plan carries a
  validation that builds a fresh VM with the released vergil-tooling and
  vergil-vm and confirms it comes up ready, and that a degraded cloud-init run
  is reported as a failure.

## 8. Security

The planning issue records six security advisories in the org's `.github`
advisories, three against vergil-tooling and three against
vergil-claude-plugin. None of them is fixed by §7.

- **Dedicated tasks.** The advisories are fixed by their own security-fix
  tasks, separate from the §7 sections: one in vergil-tooling for its three
  and one in vergil-claude-plugin for its three, each split into lettered
  sub-tasks if it is too big for one PR. Each advisory is retargeted to the
  task or tasks that fix it once they are filed, so their PRs close it.
- **Design lives in the advisory.** Each advisory records its own fix. A
  security-fix task's public title and body name it only by its GHSA IDs, and
  say nothing of the weakness, the site or the fix.
- **Nothing in public text.** No advisory detail appears in this spec, the
  plan, a task, a commit message, a PR or the report-ready notes.
- **Release gate.** The vergil-tooling security-fix task is part of the
  release gate in §7.11, since its sites are in the release and branch-handling
  path.

## 9. Left to the builder

- The exact names and signatures of the shared helpers.
- Whether a site is better served by a fixed locale or an exit code where the
  tool offers both and neither is JSON.
- The split of an oversized section into sub-tasks, where the plan does not
  already make it.
