# Report-only Dependabot Alerting — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every fleet repo has Dependabot alerts on (no Dependabot PRs ever), a canonical PR-suppressed `.github/dependabot.yml`, and a nightly ops job that fails on any open alert.

**Architecture:** vergil-tooling gains a settings flip, a `vrg-dependabot-audit` command, and a `lib/dependabot.py` renderer shared by scaffolding, a single-repo sync CLI, a fleet-sweep target, a `vrg-validate` limit-0 guard, and (release B) a local repo-config audit check. vergil-actions adds a `dependabot-audit` job to the shared `ops-github-config.yml`. Rollout is sequenced by operational tasks.

**Tech Stack:** Python 3.12+ (vergil-tooling, pytest, `gh` via `lib/github.py`), GitHub Actions reusable workflows (vergil-actions), PyYAML.

**Spec:** `epics/420-dependabot-alerting/spec.md`

## Global Constraints

- Dependabot must never open a PR: every `updates` entry carries `open-pull-requests-limit: 0`, and the repo setting Dependabot security updates stays `"disabled"`.
- Alerts are on for **every** repo, public and private: `vulnerability_alerts=True`.
- The audit fails on **any** open alert at any severity. No threshold, no severity filter, no in-repo suppression. Only `state=open` counts, so dismissal on GitHub is the sole escape hatch.
- Exit codes: `0` no open alerts; `1` at least one open alert; `2` could not check. "Could not check" is never a pass.
- **Public** repo logs carry only counts by severity plus `https://github.com/<repo>/security/dependabot`, never package names, manifest paths, or GHSA IDs. If visibility can't be resolved, the public format is used.
- `dependabot.yml` is **exact match** against one canonical render. No local additions.
- The nightly missing/non-canonical `dependabot.yml` check ships in **release B**, after every sweep PR has merged.
- All git goes through `vrg-git`, commits through `vrg-commit`, validation through `vrg-container-run -- vrg-validate`. Agents stop at `vrg-pr-workflow report-ready`.

## Review Focus

1. **The App token lacks "Dependabot alerts: Read", or the installation hasn't accepted it.** The audit must exit 2 with a message naming the missing permission, not 1 and not 0. `cli_main` maps auth errors to exit 1, so the audit must catch `GitHubAPIError` / auth errors itself (precedent: `vrg_github_repo_config.py` lines 64 and 83). Test: `test_audit_permission_denied_exits_2`.
2. **Alerts disabled on the repo.** The API returns 403 "Dependabot alerts are disabled for this repository." That must exit 2 with "Dependabot alerts are disabled for <repo>". Test: `test_audit_alerts_disabled_exits_2`.
3. **More than 100 open alerts.** Every page must count. Use `read_json_paginated`, never a bare `read_json`. Test: `test_audit_counts_all_pages`.
4. **A public repo with alerts.** The output must contain no package name or GHSA ID. Test: `test_audit_public_repo_counts_only`.
5. **A `dependabot.yml` entry with no `open-pull-requests-limit` key.** It must fail the `vrg-validate` guard, because Dependabot's default is 5. Test: `test_dependabot_guard_missing_limit_fails`.

---

## Epic task map

File each task in the repo where its PR lands, under `vergil-project/.github#420` (`vrg-issue-create --epic vergil-project/.github#420 --repo …`):

| Plan part | Repo | Kind | Blocked-by |
|---|---|---|---|
| Part 1: alerts side (settings flip + `vrg-dependabot-audit`) | vergil-tooling | task | — |
| Part 2: file side (renderer, sync CLI, scaffolding, validate guard, fleet target) | vergil-tooling | task | — |
| Part 3: `dependabot-audit` ops job | vergil-actions | task | Part 1 |
| Deploy: App permission walk-through (also confirms **Administration: Read & write** on each App, which `apply` needs) | vergil-actions | deployment | **exists: vergil-actions#999** |
| Deploy: fleet `diff` + `apply` (same day as release A) | vergil-tooling | deployment | Part 1 |
| Deploy: `dependabot.yml` fleet sweep | vergil-tooling | deployment | Part 2 |
| Part 4: release-B local audit check | vergil-tooling | task | Deploy: sweep |
| Validate: end-to-end fleet check + Scorecard | vergil-actions | validation | Part 3, #999, Deploy: apply, Part 4 |

The Part 3 **PR** is blocked only by Part 1, because merging it is harmless. The vergil-actions **release** that ships it has three preconditions: release A published, #999 done, and the fleet `apply` deployment done. Otherwise every repo with alerts still off would exit 2 the next night (spec §6).

Releases A (Parts 1+2), B (Part 4), and the vergil-actions release (Part 3) are **human-gated preconditions**, attested in the deployment tasks, never cut by an agent. Release A is followed by the `apply` deployment the same day (accepted red window, spec §6).

---

## Part 1 — Alerts side (vergil-tooling)

### Task 1.1: Flip desired security settings to alerts on

**Files:**
- Modify: `src/vergil_tooling/lib/github_config.py` (`desired_security_settings`, ~line 147)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**
- Produces: `desired_security_settings(*, ghas: bool) -> DesiredSecuritySettings` with `vulnerability_alerts=True` regardless of `ghas`.

- [ ] **Step 1: Update the existing tests.** Every `assert s.vulnerability_alerts is False` for the `desired_security_settings` results becomes `is True`, for both the `ghas=True` and `ghas=False` cases. `dependabot_security_updates == "disabled"` stays. Add `test_desired_security_settings_alerts_on_without_ghas`: `desired_security_settings(ghas=False).vulnerability_alerts is True`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_github_config_lib.py -k security -v`. Expected: FAIL on the `vulnerability_alerts` asserts.
- [ ] **Step 3: Implement.** Set `vulnerability_alerts=True` in `desired_security_settings`. Search `tests/` for other fixtures that build an expected desired state with alerts False, such as apply/diff tests, and update only those that represent *desired* state, not *observed* state.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope repo-config --message "enable Dependabot alerts in the desired security posture"`

### Task 1.2: `vrg-dependabot-audit`

**Files:**
- Create: `src/vergil_tooling/lib/dependabot_alerts.py`
- Create: `src/vergil_tooling/bin/vrg_dependabot_audit.py`
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-dependabot-audit = "vergil_tooling.bin.vrg_dependabot_audit:main"`)
- Create: `docs/site/docs/reference/dependabot-audit.md`, plus a row in `docs/site/docs/reference/cli-tools-overview.md` and an entry in the mkdocs nav if the nav is explicit
- Test: `tests/vergil_tooling/test_dependabot_alerts.py`, `tests/vergil_tooling/test_vrg_dependabot_audit.py`

**Interfaces:**
- Produces (lib):
  - `@dataclass(frozen=True) class Alert: severity: str; ecosystem: str; package: str; manifest_path: str; ghsa_id: str; html_url: str`
  - `class AuditUnavailable(Exception)`: its message is the human diagnostic.
  - `fetch_open_alerts(repo: str) -> list[Alert]`: calls `github.read_json_paginated(f"repos/{repo}/dependabot/alerts?state=open")`, maps each item from `security_advisory.severity`, `dependency.package.ecosystem`, `dependency.package.name`, `dependency.manifest_path`, `security_advisory.ghsa_id`, `html_url`, and translates `GitHubAPIError` / `MissingGitHubTokenError` into `AuditUnavailable` via `classify_error`.
  - `classify_error(repo: str, exc: subprocess.CalledProcessError) -> str`: inspects `exc.stderr`. If it contains `"disabled"` (case-insensitive), it returns `"Dependabot alerts are disabled for {repo}"`. If it contains `403` or `Resource not accessible`, it returns `"token lacks the 'Dependabot alerts: Read' permission for {repo}"`. If it contains `404`, it returns `"repository {repo} not found or not accessible"`. Otherwise it returns `"could not read Dependabot alerts for {repo}: {stderr first line}"`.
  - `SEVERITY_ORDER = ("critical", "high", "medium", "low")`
  - `render_report(repo: str, alerts: list[Alert], *, public: bool) -> str`
- Produces (CLI): `vrg-dependabot-audit --repo OWNER/REPO` (required). Exit 0/1/2 per Global Constraints.

- [ ] **Step 1: Write the failing lib tests** in `test_dependabot_alerts.py`, patching `vergil_tooling.lib.dependabot_alerts.github.read_json_paginated`:
  - `test_fetch_maps_fields`: one raw alert dict yields `Alert(severity="high", ecosystem="pip", package="requests", manifest_path="uv.lock", ghsa_id="GHSA-xxxx-yyyy-zzzz", html_url="https://github.com/o/r/security/dependabot/1")`. Assert the endpoint argument is `"repos/o/r/dependabot/alerts?state=open"`.
  - `test_audit_counts_all_pages`: the mock returns 150 items, and `len(fetch_open_alerts("o/r")) == 150`.
  - `test_audit_alerts_disabled_exits_2` (lib half): `GitHubAPIError` with stderr `"HTTP 403: Dependabot alerts are disabled for this repository."` raises `AuditUnavailable` with message `"Dependabot alerts are disabled for o/r"`.
  - `test_audit_permission_denied_exits_2` (lib half): stderr `"HTTP 403: Resource not accessible by integration"` raises a message containing `"Dependabot alerts: Read"`.
  - `test_render_private_itemizes`: two alerts (low, critical). Critical's line comes first, each line contains its package, manifest path, GHSA ID and URL, and the totals line reads `2 open Dependabot alerts: 1 critical, 0 high, 0 medium, 1 low`.
  - `test_audit_public_repo_counts_only`: same alerts with `public=True`. The output contains the totals line and `https://github.com/o/r/security/dependabot`, and contains neither `requests` nor `GHSA-`.
  - `test_render_clean`: no alerts renders `o/r: no open Dependabot alerts`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_dependabot_alerts.py -v`. Expected: FAIL (module missing).
- [ ] **Step 3: Implement `lib/dependabot_alerts.py`** with the signatures above. Sort itemized lines by `SEVERITY_ORDER` index, then `package`. An unknown severity string sorts last and is still counted. The totals line is always printed.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Write the failing CLI tests** in `test_vrg_dependabot_audit.py`, patching `fetch_open_alerts` and `github.repo_visibility`:
  - `test_cli_clean_exits_0`
  - `test_cli_alerts_exit_1` (private): stdout carries the itemized report.
  - `test_audit_alerts_disabled_exits_2` (CLI half): `AuditUnavailable("Dependabot alerts are disabled for o/r")` makes the command exit 2, with stderr `vrg-dependabot-audit: Dependabot alerts are disabled for o/r`.
  - `test_audit_permission_denied_exits_2` (CLI half): an auth error raised from `fetch_open_alerts` (for example `MissingGitHubTokenError`) makes the command exit **2**, not 1.
  - `test_cli_visibility_unresolvable_uses_public_format`: `repo_visibility` raises `GitHubAPIError`. The command still exits 1 for alerts, prints a stderr warning `could not resolve visibility for o/r; using the public (counts-only) format`, and its stdout contains no GHSA ID.
- [ ] **Step 6: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_dependabot_audit.py -v`. Expected: FAIL.
- [ ] **Step 7: Implement `bin/vrg_dependabot_audit.py`.** `main(argv: list[str] | None = None) -> int`. Wrap the body in its own `try/except` that turns `AuditUnavailable` and the auth error types into exit 2, the same way `vrg_github_repo_config.py` keeps auth failures off exit 1. Add the `pyproject.toml` script entry.
- [ ] **Step 8: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 9: Write the reference page** `docs/site/docs/reference/dependabot-audit.md`: purpose, exit codes, public vs private output, dismissal as the escape hatch, and the required App permission. Add the `cli-tools-overview.md` row.
- [ ] **Step 10: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green, 100% coverage.
- [ ] **Step 11: Commit.** `vrg-commit --type feat --scope dependabot --message "add vrg-dependabot-audit, failing on any open Dependabot alert"`

---

## Part 2 — File side (vergil-tooling)

### Task 2.1: Canonical renderer and scaffolding

**Files:**
- Create: `src/vergil_tooling/lib/dependabot.py`
- Modify: `src/vergil_tooling/lib/repo_init.py` (config-files step, next to the `.gitignore` write around line 1309)
- Test: `tests/vergil_tooling/test_dependabot.py`, `tests/vergil_tooling/test_repo_init.py`

**Interfaces:**
- Produces:
  - `DEPENDABOT_PATH = Path(".github/dependabot.yml")`
  - `LANGUAGE_ECOSYSTEMS: dict[str, str] = {"python": "uv", "go": "gomod", "rust": "cargo", "java": "maven", "typescript": "npm"}`. cpp and anything else map to nothing. Java is `maven` only, because the fleet's Java uses Maven (`pom.xml`/`mvnw`). Gradle is YAGNI.
  - `render(lang: str | None) -> str`: the canonical file text.
  - `resolve_language(repo_root: Path) -> str | None`: `declared_primary_language(repo_root)`, with `ConfigError` → `None`, mirroring `repo_config._gitignore_language`.
  - `check(text: str | None, lang: str | None) -> list[str]`: a list of reasons; empty means compliant. `None` → `["missing .github/dependabot.yml"]`. Any text other than `render(lang)` → `["differs from the canonical render"]`.
  - `limit_violations(text: str) -> list[str]`: parses YAML and returns one reason per `updates` entry whose `open-pull-requests-limit` is missing or not `0`, naming the entry's `package-ecosystem`. A parse error returns `["unparseable: <error>"]`.

- [ ] **Step 1: Write the failing tests** in `test_dependabot.py`:
  - `test_render_python_exact`: `render("python")` equals exactly:

    ```yaml
    # Managed by vergil-tooling (epic vergil-project/.github#420). Do not edit.
    # Report-only: Dependabot alerts are on; Dependabot never opens PRs here.
    # open-pull-requests-limit: 0 on every entry is enforced by vrg-validate
    # and the nightly repo-config audit.
    version: 2
    updates:
      - package-ecosystem: "github-actions"
        directory: "/"
        schedule:
          interval: "weekly"
        open-pull-requests-limit: 0
      - package-ecosystem: "uv"
        directory: "/"
        schedule:
          interval: "weekly"
        open-pull-requests-limit: 0
    ```

  - `test_render_no_language_actions_only`: `render(None)` and `render("cpp")` contain only the `github-actions` entry.
  - `test_render_every_mapped_language`: for each key of `LANGUAGE_ECOSYSTEMS`, `limit_violations(render(lang)) == []`, and the ecosystem appears.
  - `test_check_missing`, `test_check_differs`, `test_check_canonical`.
  - `test_dependabot_guard_missing_limit_fails`: an entry with no limit key gives one violation naming its ecosystem.
  - `test_limit_nonzero_fails`: `open-pull-requests-limit: 5` gives a violation.
  - In `test_repo_init.py`: `test_config_files_writes_dependabot_yml`. After the config-files step, `.github/dependabot.yml` equals `dependabot.render(<ctx primary language>)`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_dependabot.py tests/vergil_tooling/test_repo_init.py -v`. Expected: FAIL.
- [ ] **Step 3: Implement** `lib/dependabot.py`. Build `render` as text, not via `yaml.dump`, so the exact bytes are stable. Wire `repo_init` to write `render(ctx.primary_language)` alongside `.gitignore`, creating `.github/` if needed.
- [ ] **Step 4: Verify the ecosystem keys** against <https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file> (`package-ecosystem` values). If `uv` isn't accepted, change the mapping to `pip` and the fixture to match, and note the evidence in the PR notes.
- [ ] **Step 5: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope dependabot --message "render the canonical report-only dependabot.yml and scaffold it in new repos"`

### Task 2.2: `vrg-validate` limit-0 guard

**Files:**
- Modify: `src/vergil_tooling/bin/validate_common.py` (`main`, after the yamllint block)
- Test: `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**
- Consumes: `dependabot.limit_violations(text: str) -> list[str]`, `dependabot.DEPENDABOT_PATH`.

- [ ] **Step 1: Write the failing tests**:
  - `test_validate_dependabot_absent_skips`: no file means the step doesn't run and doesn't fail.
  - `test_validate_dependabot_limit_zero_passes`.
  - `test_validate_dependabot_nonzero_fails`: returns 1 and prints `dependabot.yml: <reason>` on stderr.
  - `test_validate_dependabot_unparseable_fails`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_validate_common.py -k dependabot -v`. Expected: FAIL.
- [ ] **Step 3: Implement.** If `repo_root / DEPENDABOT_PATH` is a file, print `Running: dependabot-limit (.github/dependabot.yml)`, call `limit_violations`, print each reason to stderr, and return 1 if there are any. The step checks **only** the limit, not exact match, and never requires the file to exist.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope validate --message "reject any dependabot.yml entry that could open a PR"`

### Task 2.3: Single-repo sync CLI and fleet-sweep target

**Files:**
- Create: `src/vergil_tooling/bin/vrg_dependabot_sync.py`
- Modify: `src/vergil_tooling/bin/vrg_fleet_sync.py`
- Modify: `pyproject.toml` (`vrg-dependabot-sync = "vergil_tooling.bin.vrg_dependabot_sync:main"`)
- Test: `tests/vergil_tooling/test_vrg_dependabot_sync.py`, plus the existing fleet-sync tests (find with `grep -rl vrg_fleet_sync tests/`)

**Interfaces:**
- Consumes: `dependabot.render`, `dependabot.check`, `dependabot.resolve_language`, `DEPENDABOT_PATH`.
- Produces:
  - `vrg-dependabot-sync [--repo PATH] [--check | --write]`, mirroring `vrg-gitignore-sync`. Exit 0 ok, 1 `--check` drift, 2 usage error. `--write` prints `already in sync` when the file is unchanged, else `wrote .github/dependabot.yml (<lang or none>)`.
  - `vrg-fleet-sync --target {gitignore,dependabot}` (default `gitignore`, so existing invocations are unchanged).

- [ ] **Step 1: Write the failing tests**:
  - Sync CLI: `test_sync_check_missing_exits_1`, `test_sync_write_creates_file`, `test_sync_write_idempotent_prints_already_in_sync`, `test_sync_write_overwrites_noncanonical` (exact match: local additions are replaced).
  - Fleet: `test_fleet_target_default_is_gitignore`, a regression test that the existing behaviour is unchanged. `test_fleet_target_dependabot_spec`: the `SweepSpec` has `branch_slug="dependabot-config"`, `commit_scope="dependabot"`, `commit_type="chore"`, `epic="vergil-project/.github#420"`, and title `chore(dependabot): add the report-only dependabot.yml`. Its applicator invokes `["vrg-dependabot-sync", "--write", "--repo", <worktree>]` and maps `already in sync` to `changed=False`.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_vrg_dependabot_sync.py <fleet test file> -v`. Expected: FAIL.
- [ ] **Step 3: Implement.** The sync CLI is a near-copy of `vrg_gitignore_sync.py`'s shape. In `vrg_fleet_sync.py`, move the current constants into a per-target table `{target: (SweepSpec fields, applicator)}`. Factor out a shared `_run_sync_applicator(argv: list[str], worktree: Path) -> AppResult` from `_gitignore_applicator`, and parameterize the help text. Leave everything else unchanged. One hard-coded second target is all this task adds; config-driven discovery stays follow-on #328.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope dependabot --message "sync dependabot.yml per repo and across the fleet"`

---

## Part 3 — Ops job (vergil-actions)

### Task 3.1: `dependabot-audit` job in `ops-github-config.yml`

**Files:**
- Modify: `.github/workflows/ops-github-config.yml`
- Modify: the vergil-actions site docs for the App permission table (`docs/site/docs/configuration.md`, `APP_CLIENT_ID` section), adding the row `Dependabot alerts | Read-only | Nightly dependabot-audit (ops)`.

**Interfaces:**
- Consumes: `vrg-dependabot-audit --repo OWNER/REPO` (Part 1; must be in a released vergil-tooling).
- Release preconditions (human-attested, not merge blockers): release A published, vergil-actions#999 done, and the fleet `apply` deployment done.

- [ ] **Step 1: Add the job**, modelled on `github-config` (same `runs-on`, `container`, checkout, and `./actions/shared/setup/vergil`). The token step uses `actions/create-github-app-token@v3` with `app-id`/`private-key` from secrets, no `owner`/`repositories` (so it's scoped to the calling repo only), and `permission-vulnerability-alerts: read`, with a comment in the style of `branch-sweep` explaining the narrowest scope and why `GITHUB_TOKEN` can't be used. Final step: `run: vrg-dependabot-audit --repo "${{ github.repository }}"` with `GH_TOKEN` from the token step. Leave the workflow-level `permissions: contents: read` unchanged, and change no consumer `ops.yml`.
- [ ] **Step 2: Confirm the input name** `permission-vulnerability-alerts` against the `actions/create-github-app-token` README at the pinned major version. A wrong name fails token creation loudly, which is acceptable but wastes a release.
- [ ] **Step 3: Update the App permission table** in `configuration.md`, and add a short paragraph saying the ops App needs "Dependabot alerts: Read" and each installation must accept it.
- [ ] **Step 4: Validate.** Run `vrg-container-run -- vrg-validate` (actionlint and yamllint cover the workflow). Expected: green.
- [ ] **Step 5: Commit.** `vrg-commit --type feat --scope ops --message "fail the nightly ops run on any open Dependabot alert"`

---

## Part 4 — Release-B local audit check (vergil-tooling)

**Blocked-by the sweep deployment.** Start only when every sweep PR has merged.

### Task 4.1: `_check_dependabot_yml` in the local repo-config audit

**Files:**
- Modify: `src/vergil_tooling/lib/repo_config.py` (new `_check_dependabot_yml`, called from `audit_local_config` after `_check_gitignore`)
- Modify: `docs/site/docs/reference/config-audit.md` (document the `local.dependabot` item)
- Test: `tests/vergil_tooling/test_repo_config.py`

**Interfaces:**
- Consumes: `dependabot.check(text: str | None, lang: str | None) -> list[str]`, `dependabot.resolve_language`, `DEPENDABOT_PATH`.
- Produces: a `DiffItem(field="local.dependabot", expected="canonical report-only dependabot.yml", actual=<reason>)` per reason.

- [ ] **Step 1: Write the failing tests**: `test_audit_dependabot_missing`, `test_audit_dependabot_noncanonical`, `test_audit_dependabot_canonical_passes`. Also update any existing `audit_local_config` fixture that builds a fully compliant repo, so it writes the canonical file.
- [ ] **Step 2: Run them and confirm they fail.** Run `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_config.py -k dependabot -v`. Expected: FAIL.
- [ ] **Step 3: Implement** `_check_dependabot_yml(repo_root: Path, items: list[DiffItem]) -> None`, mirroring `_check_gitignore`: read the file if present, else `None`, then append one `DiffItem` per reason.
- [ ] **Step 4: Run them and confirm they pass.** Same command. Expected: PASS.
- [ ] **Step 5: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: green.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope repo-config --message "audit dependabot.yml against the canonical render nightly"`

---

## Operational tasks (filed at step 9; bodies follow the stamped scaffold)

- **Deploy: fleet `diff` + `apply`** (vergil-tooling, `deployment`). Precondition: release A published (human-attested). The human runs `vrg-github-repo-config diff --repo <r>` for every fleet repo and reviews drift unrelated to alerts, then runs a scripted `apply` loop on the **same day** as release A. Acceptance: a `diff` shows no alerts drift on any repo, and the next nightly config audit shows no alerts drift.
- **Deploy: `dependabot.yml` sweep** (vergil-tooling, `deployment`). Precondition: release A published. The agent runs `vrg-fleet-sync --target dependabot --repos <every fleet clone>` (dry run first). The human submits and merges each resulting PR. Acceptance: `vrg-dependabot-sync --check` exits 0 on every repo's `develop`.
- **Validate: end-to-end** (vergil-actions, `validation`). Preconditions: the vergil-actions release with Part 3, #999, the `apply` deployment, and release B, all done. Run `workflow_dispatch` of `ops.yml` on one repo per account in #999's inventory, plus one repo known to have open alerts. Acceptance: exit 0 on clean repos and exit 1 on the alerting repo; no exit 2 anywhere; the public repo's log has counts only; the `local.dependabot` audit is clean; no PR authored by `dependabot[bot]` exists in any fleet repo (`is:pr author:app/dependabot` org search); and Scorecard's Dependency-Update-Tool check passes on one public repo (<https://scorecard.dev/viewer/?uri=github.com/vergil-project/vergil-tooling> or a `scorecard` CLI run).
