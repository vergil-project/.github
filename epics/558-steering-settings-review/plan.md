# Steering-file and Claude settings review — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Write down which file owns each Claude Code setting and
instruction. Then clean up every steering and settings file in the managed
fleet to match, keep them matching with the nightly config audit and the
agent chokepoints, and add a `steering-audit` skill for the periodic review
that needs judgment.

**Architecture:**

- **The model.** A new vergil-tooling site page holds the ownership model.
- **The audit.** vergil-tooling's managed templates
  (`data/claude_md_consumer.md`, `data/claude_settings.json`) get a new,
  smaller shape. Until the bridge-removal PR merges, the config audit
  (`lib/repo_config.py`) accepts both the old and the new shape, so no
  managed repo's nightly run goes red while the fleet is swept.
- **The local-settings guard.** A new library check, called from the agent
  chokepoints, refuses to proceed while a `.claude/settings.local.json` that
  Claude Code would load holds anything other than `{}`.
- **The plugin.** vergil-claude-plugin gains `steering-audit` and loses the
  deprecated `memory-init`, `memory-audit` and `handoff` skills, and is
  released so agents load the new skill.
- **The sweeps.** Each managed repo (16, in three orgs) gets one sweep PR.
- **The tightening.** A final audit change removes the bridge and adds the
  200-line budget and the no-AGENTS.md rule.
- **The user-level files** are reviewed with `steering-audit user` and
  applied by the human on the host.

**Tech Stack:** Python 3.12+ (pytest, `importlib.resources`), Markdown
(skills, site docs, steering files), JSON settings, git plumbing
(`git rev-parse`), `/paad:pushback`.

**Spec:** `epics/558-steering-settings-review/spec.md`
(vergil-project/.github#558)

**Shape:** this is an epic-level plan. Each task below is **one issue and one
PR in the repo named in its heading**, worked in a worktree of that repo.
Paths are relative to that repo's root. Deployment and validation tasks are
operational: they have no code PR and close on `Outcome: SUCCESS`.

## Global Constraints

- **The managed fleet** is every repo that pins `vergil = "v2.1"` and whose
  `.github/workflows/ops.yml` calls `ops-github-config`. It is found by a
  scan, not a hand-kept list. On 2026-10-27 it is 16 repos (Tasks 5a–5p).
- **Org scope.** In scope: `vergil-project`, `logical-minds-foundry`,
  `mnemosys-project`. On hold: `mq-rest-admin-project`, `diogenes-project`,
  `vergils-nemesis`; nothing in them is changed, and each gets one ad-hoc
  tracking issue (see Task filing). Every other repo or checkout is ignored.
  `v2.0` repos in the in-scope orgs are outside the managed fleet.
- **CLAUDE.md line budget:** each managed repo's `CLAUDE.md` has at most
  **200 lines**, counted as `len(text.splitlines())`, including the template
  region. Don't move content into `@imports` to meet it.
- **Template region budget:** `data/claude_md_consumer.md` is at most **60
  lines**.
- **Settings template:** `data/claude_settings.json` becomes exactly this,
  and each managed repo's `.claude/settings.json` must equal it after JSON
  parsing:

  ```json
  {
    "permissions": { "allow": ["Bash(vrg-*)"] },
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hooks": [
            { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard.sh" }
          ]
        }
      ]
    },
    "enabledPlugins": { "vergil@vergil-marketplace": true },
    "attribution": { "commit": "", "pr": "" }
  }
  ```

- **No `extraKnownMarketplaces` in repo settings.** It lives only in the
  user-level `~/.claude/settings.json`.
- **Local settings:** `.claude/settings.local.json` stays gitignored (the
  managed `.gitignore` fence is unchanged). A copy that Claude Code may load
  must be absent or parse to exactly `{}`.
- **Bridge warning text** (exact):
  - `.claude/settings.json matches the previous template; replace it with the current template (epic vergil-project/.github#558).`
  - `CLAUDE.md template region matches the previous template; replace it with the current template (epic vergil-project/.github#558).`
- **Local-settings refusal text** (exact first line):
  `refusing: <path> must be absent or exactly {} — move any rule it holds into the repo template or your user settings, then delete it (epic vergil-project/.github#558).`
  For an agent identity the refusal has a second line, exactly:
  `Agents: stop and ask the human.` A human identity gets the first line
  only, with `warning:` in place of `refusing:`, and the tool proceeds.
- **Personal files stay off GitHub, and the agent never writes them.** The
  content of `~/.claude/CLAUDE.md` and `~/.claude/settings.json`, and
  user-scope review findings, are never posted in issues, intake, PR bodies,
  report-ready notes or commits, and the agent writes no edit, draft or
  note of them. Record only `reviewed and applied on <date>`, section
  counts, and the names of sections that move to a public home. The names
  and content of sections that stay are never recorded.
- **No memory dependency.** `steering-audit` doesn't read, write or mention
  memory files.
- **Placement law.** Each task is filed in the repo of its heading. Tasks in
  logical-minds-foundry and mnemosys-project are filed under that repo's own
  ad-hoc epic and are never linked to this epic; the gate task (5-gate)
  tracks them by text.
- **Release model.** Audit changes ship as **v2.1 patch releases**, which
  reach every managed repo on the next nightly audit, with no pin bump.
  Releases happen several times a day, so the gate is on **merging**: a PR
  whose precondition is unmet must not merge. Releases (`vrg-release`, the
  plugin release) are human actions; agents only attest that they happened.
- **vergil-tooling audits from source.** Every audit-change PR in
  vergil-tooling updates vergil-tooling's own `CLAUDE.md` region and
  `.claude/settings.json` in the same PR.
- **Platform facts** are cited from `https://code.claude.com/docs/en/` pages
  and re-verified, with the verification date recorded, whenever the model
  page changes.

## Decided content

**Template region** (`data/claude_md_consumer.md`, at most 60 lines). These
sections, in this order, with these line caps:

| Section | Lines | Content |
|---|---|---|
| Memory | ≤ 3 | Never write memory files without explicit human approval. |
| Worktrees | ≤ 12 | Sessions start at the project root; one worktree per issue at `.worktrees/issue-<N>-<slug>/`; the main worktree is read-only. Link to the GitHub URL of `vergil-tooling/docs/specs/worktree-convention.md` for the rest, and one line saying parallel sub-agents are briefed with that spec's "Agent prompt contract" (`https://github.com/vergil-project/vergil-tooling/blob/develop/docs/specs/worktree-convention.md#agent-prompt-contract`). |
| Git and GitHub | ≤ 4 | `vrg-git` and `vrg-gh` only; raw `git` and `gh` are blocked by the hook guard. |
| Validation | ≤ 4 | `vrg-container-run -- vrg-validate` is the only validation command. |
| PR handoff | ≤ 15 | Agents never run `vrg-submit-pr`. They hand off with `vrg-pr-workflow report-ready`. After report-ready the branch is frozen; `vrg-pr-workflow unfreeze` is the only way to lift the freeze. Report-ready title, summary and notes are public (the relay ref): no secrets. |
| Local settings | ≤ 2 | `.claude/settings.local.json` must stay absent or `{}`; the agent tools refuse otherwise. |
| Engineering principles | ≤ 6 | Correctness over cost over speed; never ship known-broken code. No silent failures: no swallowed exceptions or fallbacks that hide errors. |
| paad | ≤ 3 | paad usage policy and deviations: `https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-policy.md`. |

**Sections moving from `~/.claude/CLAUDE.md` to the template** (spec §5.2
step 1; removed on the host in Task 6):

| Section | Disposition |
|---|---|
| Memory management | move: the template's Memory rule replaces it |
| Correctness over cost | move: template region |
| No silent failures | move: template region |

The classification is **complete**: 6 sections, 3 move (listed above),
3 keep, 0 delete. The keep sections are not listed here. Task 6 applies this
classification and does not re-classify; anything new its review turns up
becomes an intake item, and the section stays on the host until then.

## Review Focus

1. **Key order and formatting.** A repo `.claude/settings.json` whose keys
   are in a different order or use different whitespace must still pass. The
   comparison is on parsed JSON. Owned by Task 2.
2. **Legacy repos during the bridge.** A repo still in the legacy
   (no-marker) form, carrying the **previous** template verbatim, must pass
   with the warning, not fail. Owned by Task 2.
3. **Absent local settings.** In a repo with no `.claude/settings.local.json`
   anywhere (every unswept and out-of-scope repo), every chokepoint must
   proceed silently. Owned by Task 2b.
4. **The main-checkout copy inside the container.** The container mounts the
   parent `.git` but not the main checkout's working tree, so the
   main-checkout copy looks absent from inside it. The guard therefore never
   runs inside a container: `vrg-container-run` checks every copy on the
   host or VM before starting one. Owned by Task 2b.
5. **A human running the agent chokepoints.** `vrg-release` and
   `vrg-update-deps` run `vrg-container-run`, and a human may run
   `vrg-commit`. With a non-empty local-settings file, a human identity gets
   a warning and the tool proceeds; only agent identities are refused.
   Owned by Task 2b.
6. **Line-budget edges.** A `CLAUDE.md` of exactly 200 lines passes, 201
   fails, and a trailing newline doesn't count as an extra line. Owned by
   Task 7.

---

### Task 1: Ownership model page (vergil-tooling, implementation)

**Files:**

- Create: `docs/site/docs/guides/claude-config-ownership.md`
- Modify: `docs/site/mkdocs.yml` (nav, under "Identity & Permissions", after "Permission Model")
- Modify: `docs/site/docs/guides/permission-model.md`, `docs/site/docs/reference/config-audit.md`, `docs/site/docs/guides/consuming-repo-setup.md` (one link each)
- Modify: `src/vergil_tooling/lib/vm_guest.py:401-403,447-452` and `tests/vergil_tooling/test_vm_guest.py:814` (comments only)

**Interfaces:**

- Produces: the page `claude-config-ownership.md`. Task 4's skill and the
  sweeps (Task 5) cite it as "the ownership model".

- [ ] **Step 1: Re-verify the platform facts.** Fetch the docs pages listed in
  spec §3 and confirm each of the 11 facts. Any that changed go back to the
  human before writing. The page records the URLs and the date.
- [ ] **Step 2: Write the page.**
  - **Contents:** the placement rule and ownership table (spec §5.1); the
    platform facts with citations; unused scopes and why (managed settings);
    the local-settings rule, where it is enforced, and its known limit (a
    repo whose root, `.git` or `.claude` isn't owned by the user: Claude
    Code then keeps the file elsewhere, and the guard does not check it);
    the recurring review
    (the nightly audit's rules and `steering-audit`'s two scopes, run at
    least quarterly and on any Claude Code release that changes settings or
    instruction-file behavior).
  - **Optional extra:** `/doctor prompt-audit` is mentioned as a check the
    human can run by hand.
- [ ] **Step 3: Correct the `vm_guest.py` and `test_vm_guest.py` comments.** Say what changelog
  2.1.195 actually states (project-only enabled plugins need explicit install
  consent), and that the reconcile step stays until Task 9 shows whether
  user-enabled plugins install on their own. No code change.
- [ ] **Step 4: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: PASS.
- [ ] **Step 5: Commit** with `vrg-commit --type docs --scope claude-config`.

### Task 2: New templates and the bridged audit (vergil-tooling, implementation)

Blocked by Task 1 (the template text cites the model page) and Task 2b
(the template's Local settings line says the agent tools refuse).

**Files:**

- Modify: `src/vergil_tooling/data/claude_settings.json` (to the Global Constraints shape)
- Create: `src/vergil_tooling/data/claude_settings_previous.json` (today's repo file: the old template **with** `"ref": "main"` seeded)
- Modify: `src/vergil_tooling/data/claude_md_consumer.md` (to the Decided content)
- Create: `src/vergil_tooling/data/claude_md_consumer_previous.md` (today's template, byte for byte)
- Modify: `src/vergil_tooling/lib/repo_config.py` (`_check_claude_md`, `_check_claude_settings`; delete `_check_marketplace_ref`, `_check_settings_section`, `_check_attribution`)
- Modify: `src/vergil_tooling/lib/repo_init.py:322-332` (`render_claude_settings` stops seeding the ref) and `repo_init.py:423-447` (`render_claude_md` stops writing a `## Validation` section outside the markers; the template states it)
- Modify: `CLAUDE.md` (template region replaced; body text the region now covers removed; the "Cloud session prompt contract" stays in the body), `.claude/settings.json` (to the new shape)
- Modify: `docs/site/docs/reference/config-audit.md` (settings and CLAUDE.md check rows; the bridge)
- Test: `tests/vergil_tooling/test_repo_config.py`, `tests/vergil_tooling/test_repo_init.py`, `tests/vergil_tooling/test_update_deps_vergil_eco.py`

**Interfaces:**

- Produces:
  - `_read_data_file(name: str) -> str`
  - `_load_template(previous: bool = False) -> str`
  - `_load_settings_template(previous: bool = False) -> dict[str, Any]`
  - `_check_claude_settings(repo_root, items, warnings)` and
    `_check_claude_md(repo_root, items, warnings)`, which append the bridge
    warnings; `audit_local_config` passes `warnings` to both.

- [ ] **Step 1: Write the failing tests.**
  - `test_settings_current_template_passes_clean`
  - `test_settings_key_order_irrelevant` (reordered keys and whitespace pass
    clean)
  - `test_settings_previous_template_warns` (exactly the settings bridge
    warning)
  - `test_settings_other_shape_fails` (the new template plus
    `extraKnownMarketplaces` gives one `local.claude_settings` item)
  - `test_claude_md_region_previous_warns`,
    `test_claude_md_legacy_previous_warns`,
    `test_claude_md_region_current_clean`
  - `test_render_claude_md_has_no_body_validation_section` (in
    `test_repo_init.py`)
  - `test_render_claude_settings_equals_template` and
    `test_repo_init_output_audits_clean` (in `test_repo_init.py`)
  - `test_normalize_claude_ref_noop_without_marketplace` (in
    `test_update_deps_vergil_eco.py`: returns `None`, file byte-identical)
- [ ] **Step 2: Run the tests and see them fail.**
  `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_config.py tests/vergil_tooling/test_repo_init.py tests/vergil_tooling/test_update_deps_vergil_eco.py -q`
- [ ] **Step 3: Write the data files.** Create the two `_previous` files from
  the current content first, then rewrite the two templates.
- [ ] **Step 4: Implement the bridge.** Parsed equality with the current
  template is clean; with the previous one, a warning; anything else is one
  `DiffItem`. The same three-way result applies to the CLAUDE.md marked
  region and to the legacy verbatim match.
- [ ] **Step 5: Update this repo's own files.**
- [ ] **Step 6: Run the tests and see them pass,** then
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 7: Refactor.** `_load_template` and `_load_settings_template`
  become thin wrappers over `_read_data_file`; remove duplication between the
  two three-way checks. Re-run the tests: PASS.
- [ ] **Step 8: Account for every deleted instruction.** For the template
  diff and vergil-tooling's trimmed body, list each instruction removed and
  where it is still covered: the new template, a named skill and line, or a
  site page. An instruction with no remaining home stays in the template
  (within its budget) or moves to a home in this PR. Instructions to check
  include the `! <command>` escape, the per-tool worktree path rules,
  "linkage defaults to `Ref`", the expected `.github/workflows/**` push
  refusal, and the explicit push at the recorded `head_sha`. The list goes
  in the report-ready notes, and the reviewer checks it against the spec §4
  anti-goal.
- [ ] **Step 9: Commit** with `vrg-commit --type feat --scope repo-config`.

### Task 2b: Local-settings guard (vergil-tooling, implementation)

No blockers. An absent file passes, so this is safe to merge at any time.

**Files:**

- Create: `src/vergil_tooling/lib/local_settings.py`
- Modify: `src/vergil_tooling/bin/vrg_commit.py`, `src/vergil_tooling/bin/vrg_pr_workflow.py` (report-ready), `src/vergil_tooling/bin/vrg_container_run.py`
- Modify: `src/vergil_tooling/lib/identity_mode.py` (add `any_agent_signal() -> bool`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (one line per tool)
- Test: `tests/vergil_tooling/test_local_settings.py`, `tests/vergil_tooling/test_identity_mode.py`, plus one refusal test per caller in its existing test file

**Interfaces:**

- Produces:
  - `local_settings_paths(cwd: Path) -> tuple[list[Path], list[str]]`. It
    returns the candidate paths and a list of "not checked" notes:
    - the worktree root's `.claude/settings.local.json`;
    - the main checkout root's copy, resolved from `git rev-parse
      --git-common-dir`;
    - the invoking directory's copy and each ancestor's copy up to the
      worktree root (a session started in a subdirectory, or after `/cd`).
    - Inside an in-scope repo, a `git rev-parse` failure raises with git's
      error. It is never a pass.
    - Known limit, recorded on the model page (Task 1): when the repo root,
      its `.git` or `.claude` isn't owned by the user, Claude Code keeps the
      file beside `.claude/settings.json` instead; not checked.
  - `find_violations(paths: list[Path]) -> list[Path]`. A path violates if
    the file exists and does not parse to exactly `{}`. Unparseable JSON
    violates.
  - `REFUSAL_TEMPLATE` and `WARNING_TEMPLATE`: the Global Constraints texts.
  - `IN_SCOPE_ORGS = ("vergil-project", "logical-minds-foundry",
    "mnemosys-project")`, with a comment that this is the epic's explicit
    org scope (Global Constraints).
  - `in_scope(cwd: Path) -> bool`: True when the repo's `origin` remote
    owner is in `IN_SCOPE_ORGS`. Not a git repo, no `origin`, or another
    owner gives False.
  - `identity_mode.any_agent_signal() -> bool`: True when **any** resolution
    signal implies an agent (the `identity-mode` file, `app.pem`,
    `VRG_APP_ID`, or `VRG_IDENTITY_MODE` naming an agent mode). A
    `VRG_IDENTITY_MODE=human` value cannot override a provisioned agent
    signal. `resolve()` itself is unchanged; the wider resolver question is
    vergil-project/.github#593.
  - `enforce(cwd: Path) -> int`: the one call site pattern. Out of scope:
    returns 0 silently. In scope: an agent (`any_agent_signal()`)
    **refuses** (returns 1); otherwise it **warns** and returns 0.
- Callers, all through `enforce`, all on the host or VM, never inside a
  container: `vrg-commit`, `report-ready`, and `vrg-container-run` (before
  starting the container, so it covers every containerised run, including
  `vrg-validate`).
- Not callers:
  - `vrg-validate` refuses to run outside a dev container
    (`vrg_validate.py:98`), and inside one it can see neither the main
    checkout's working tree nor the identity signals; `vrg-container-run`
    has already checked every copy. CI runs `vrg-validate` directly in job
    containers, which hold no local file, so CI needs no check.
  - `vrg-hook-guard`: its stderr is never shown on exit 0, and a local file
    that disables hooks disables it too.

- [ ] **Step 1: Write the failing tests.**
  - `test_absent_passes`, `test_empty_object_passes` (`{}`, `{ }\n`)
  - `test_key_in_worktree_copy_violates`
  - `test_key_in_main_checkout_copy_violates_from_worktree` (a real
    `git worktree` in `tmp_path`)
  - `test_unparseable_violates`
  - `test_subdirectory_copy_violates`
  - `test_rev_parse_failure_raises`
  - `test_out_of_scope_org_is_silent`, `test_no_origin_is_silent`,
    `test_not_a_git_repo_is_silent`
  - `test_env_human_on_agent_vm_still_refuses` (mode file says `user`,
    `VRG_IDENTITY_MODE=human`) and `test_no_agent_signal_warns`
  - `test_any_agent_signal_*` (in `test_identity_mode.py`, one per signal)
  - per host-side caller: `test_<tool>_refuses_for_agent` (exit 1, refusal
    text on the first line, `Agents: stop and ask the human.` on the second)
    and `test_<tool>_warns_for_human` (exit 0, warning text)
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement** the library, then wire the callers.
- [ ] **Step 4: Run the tests and see them pass,** then
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 5: Refactor.** Every caller goes through `enforce`; no caller
  re-implements path resolution or the identity rule.
- [ ] **Step 6: Commit** with `vrg-commit --type feat --scope local-settings`.

### Task 3: Release the bridged audit (vergil-tooling, deployment)

Blocked by Task 2, Task 2b and vergil-project/.github#588 (an ad-hoc fix
that makes `.github`'s legacy template match pass again; its audit has
failed since 2026-10-25).

- **Precondition (human-attested):** a v2.1 patch release containing Tasks 2
  and 2b is published, and the rolling `v2.1` tag points to it.
- **Precondition (before that release):** no stray local settings in the
  in-scope orgs. On the host and on each agent VM, list every
  `.claude/settings.local.json` under the checkouts of the three in-scope
  orgs (main roots and `.worktrees/*`), for example
  `find <projects-dir>/{vergil-project,logical-minds-foundry,mnemosys-project} -name settings.local.json -path '*/.claude/*'`,
  and delete each one or reduce it to `{}`. The agent may run the scan on
  the VMs; the human clears the host. Other orgs are not scanned.
- **Procedure:** run the fleet scan. File one sweep task (5j–5p, plus any
  repo the scan adds) in each cross-org repo's own ad-hoc epic, with its
  prerequisites as text. After the next nightly run, check every managed
  repo's `ops.yml` config-audit job.
- **Acceptance:** every managed repo's audit shows its local section
  compliant apart from the bridge warning(s): no local `DiffItem`. Remote
  (GitHub settings) drift and credential errors are reported separately and
  don't fail this task.

### Task 4: `steering-audit` skill (vergil-claude-plugin, implementation)

Blocked by Task 1.

**Files:**

- Create: `skills/steering-audit/SKILL.md`
- Modify: `README.md` (skills table), `docs/site/docs/skills/index.md` (table row and a `## steering-audit` section), `CLAUDE.md` if it lists the skills
- Modify: `scripts/bin/validate-custom` (add the skill frontmatter check)

**Interfaces:**

- Consumes: the ownership model page (Task 1).
- Produces: `/vergil:steering-audit` (repo scope) and
  `/vergil:steering-audit user`.

- [ ] **Step 1: Write the SKILL.md.**
  - **Frontmatter:** `name: steering-audit`. The description has trigger
    phrases ("audit steering files", "review CLAUDE.md", "review Claude
    settings", "steering audit").
  - **Workflow:**
    1. **Inventory.** In repo scope: `CLAUDE.md` with its line count,
       `.claude/rules/`, `.claude/settings.json`, any `AGENTS.md`, and the
       local-settings copies. In user scope: `~/.claude/CLAUDE.md` and
       `~/.claude/settings.json`.
    2. **Re-verify.** Check the model page's platform facts against the live
       docs, and report any drift first.
    3. **Mechanical checks.**
       - In repo scope, run `vrg-github-repo-config audit`.
       - In both scopes: every path named in a steering file exists (URLs
         are not paths and are skipped), every named command resolves, the
         line count, and the ownership rules.
    4. **Review.** Run `/paad:pushback <path>` on each steering file, with
       these directives pinned verbatim in the SKILL.md: collect the
       findings and return them; do not present them one at a time; do not
       edit the reviewed file; write no report and no other file.
    5. **Walk the findings one at a time.** Each finding gets one
       disposition: keep, edit, move to a named managed home (template
       region, plugin, repo docs, `.claude/rules/`), delete, or (repo scope
       only) file to intake. Wait for the human's decision on each.
    6. **Apply.** Repo-scope edits go through the normal worktree and PR
       flow. In user scope the skill applies nothing (below).
  - **User-scope constraints:** the user files have no change control, so
    user scope is **read-only and interactive only**. The skill writes no
    file of any kind (no edits, no drafts, no reports, no notes); all
    feedback stays in the session. The human decides every change and makes
    it on the host. It runs once per cycle; findings never go to GitHub or
    intake; the public record is the date and counts only.
  - **No memory access.**
- [ ] **Step 2: Add the catalog entries.**
- [ ] **Step 3: Add the frontmatter check.** `validate-custom` today runs
  only the hook tests. Add a check that every `skills/*/SKILL.md` has YAML
  frontmatter with a non-empty `name` equal to its directory name and a
  non-empty `description`; any failure exits non-zero naming the file.
  Requested by the human (spec §5.5); wider plugin validation is intake
  item vergil-project/.github#587.
- [ ] **Step 4: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: PASS, including the new frontmatter check.
- [ ] **Step 5: Commit** with `vrg-commit --type feat --scope skills`.

### Task 4b: Remove the deprecated skills (vergil-claude-plugin, implementation)

No blockers.

**Files:**

- Delete: `skills/memory-init/`, `skills/memory-audit/`, `skills/handoff/`
- Modify:
  - `README.md` (skills table rows);
  - `docs/site/docs/skills/index.md` (table rows and the three sections);
  - `CLAUDE.md`: the handoff mention in the body only. The memory-skill list is template text in a legacy (no-marker) file, and editing it fails the audit until 5b; 5b replaces it.
  - `docs/paad-policy.md` §5 (`vergil:handoff` is removed; `paad:handoff` is the one to use) and §7 (remove the deviation row naming `vergil:handoff`);
  - the "`/vergil:handoff` remains the recovery net" lines in `skills/issue-deploy/SKILL.md`, `skills/issue-validate/SKILL.md`, `skills/issue-validate/references/operational-task-lifecycle.md`, `skills/issue-implement/SKILL.md`, `skills/epic-implement/SKILL.md` and `skills/epic-retrospective/SKILL.md`;
  - `skills/epic-implement/SKILL.md:58`, which dispatches sub-agents using the "Agent prompt contract" in `CLAUDE.md`: the new template no longer carries the contract, so point it at `https://github.com/vergil-project/vergil-tooling/blob/develop/docs/specs/worktree-convention.md#agent-prompt-contract`;
  - the "Front-Loaded Judgment, Trusted Execution" doctrine references ("see `CLAUDE.md`") in `skills/issue-implement/SKILL.md:21`, `skills/issue-deploy/SKILL.md:33` and `skills/issue-validate/SKILL.md:33`: point them at `https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/site/docs/skills/index.md#execution-doctrine-front-loaded-judgment-trusted-execution`.

- [ ] **Step 1: Delete and edit.** Each "recovery net" line becomes: the
  issue's comments are the recovery record, and `paad:handoff` is available
  for session continuity. Repoint `epic-implement`'s contract reference and
  the three doctrine references.
- [ ] **Step 2: Check for leftovers.**
  `vrg-git grep -n 'memory-init\|memory-audit\|vergil:handoff'` should show
  only history, which is left as is: `CHANGELOG.md`, `releases/`, `paad/`,
  and dated files under `docs/plans/` and `docs/specs/` (including
  `docs/specs/figures/`), plus the template text in `CLAUDE.md` (left for
  5b). Under `skills/`, `vrg-git grep -n 'prompt contract" in'` shows no
  hit, and every hit of `vrg-git grep -n 'Front-Loaded Judgment' -- skills/`
  is read to confirm it points at the URL (the old wording splits "see" and
  `` `CLAUDE.md` `` across lines, so a line grep can't prove it).
- [ ] **Step 3: Validate.** `vrg-container-run -- vrg-validate`. Expected:
  PASS.
- [ ] **Step 4: Commit** with `vrg-commit --type feat --scope skills`.
- [ ] **Step 5: Note on #561.** After merge, comment on
  vergil-project/.github#561 that the skills were removed, and that
  `vrg-vm`'s memory projection is left for it.

### Task 4c: Release the plugin (vergil-claude-plugin, deployment)

Blocked by Task 4 and Task 4b.

- **Precondition (human-attested):** a vergil-claude-plugin release
  containing Tasks 4 and 4b is published to the `main` marketplace ref.
- **Procedure:** in a fresh agent VM session, confirm that
  `/vergil:steering-audit` is listed, and that `memory-init`, `memory-audit`
  and `vergil:handoff` are not.
- **Acceptance:** both checks pass.

### Task 5a–5p: Per-repo steering sweep (one task per managed repo, implementation)

Each sweep is blocked by Tasks 1, 3 and 4c. vergil-tooling (5a) needs Task 2
rather than Task 3, because its audit runs from source; 5a runs
`uv run vrg-github-repo-config audit`. For 5b–5p, the VM's installed
vergil-tooling must be at or after the Task 3 release (`vrg-vm update`, then
check the version) before Step 1. Cross-org sweeps
(5j–5p) are filed under their own repo's ad-hoc epic, and their bodies list
the prerequisites as text.

| Task | Repo | `CLAUDE.md` now | Extra work |
|---|---|---|---|
| 5a | vergil-project/vergil-tooling | 554 | delete `AGENTS.md`; move the "Architecture" catalogues and "Consumption Model" detail into site docs and link them |
| 5b | vergil-project/vergil-claude-plugin | 375 | adopt markers; delete `AGENTS.md` |
| 5c | vergil-project/vergil-containers | 307 | adopt markers |
| 5d | vergil-project/vergil-actions | 234 | adopt markers; delete `AGENTS.md` |
| 5e | vergil-project/.github | 158 | adopt markers; keep the "Archived epics — do not read" section, outside the marked region |
| 5f | vergil-project/docs | 128 | — |
| 5g | vergil-project/vergil-vm | 124 | adopt markers |
| 5h | vergil-project/vergil-python | 120 | — |
| 5i | vergil-project/packages | 115 | — |
| 5j–5n | logical-minds-foundry/{docs, mq-gateway-replay-lab, mq-protocol-gateway, mq-resiliency-lab-for-linux, mq-resiliency-observability} | measured at sweep time | adopt markers if legacy |
| 5o–5p | mnemosys-project/{docs, melete} | measured at sweep time | adopt markers if legacy |

**Files (each repo):** `CLAUDE.md`, `.claude/settings.json`, `AGENTS.md`
(where present), and the docs pages that receive moved material.

Steps (each repo):

- [ ] **Step 1: Run the review.** Run `/vergil:steering-audit` in the repo
  and take its findings to the human.
- [ ] **Step 2: Replace the managed parts.** Put the current template in the
  marked region (adding markers where needed), and set
  `.claude/settings.json` to the template.
- [ ] **Step 3: Trim the body to the ownership model.** Delete fleet policy
  the template states; move reference material into docs and link it; move
  part-of-repo guidance to `.claude/rules/`. Before deleting an `AGENTS.md`,
  move anything in it that is still true.
- [ ] **Step 4: Review again.** Run `/vergil:steering-audit` on the trimmed
  files and resolve its findings with the human.
- [ ] **Step 5: Check the result.** `wc -l CLAUDE.md` shows at most 200;
  `vrg-github-repo-config audit` prints `local: compliant` with no
  `WARNING:` line under it (its exit status also counts remote drift and
  credential errors the sweep didn't cause, so it is not the test; report
  any remote finding in the notes); `vrg-container-run -- vrg-validate`
  passes.
- [ ] **Step 6: Commit** with `vrg-commit --type docs --scope claude-md`. The
  report-ready notes list where each moved section went, and **every
  instruction deleted** together with where it is still covered.

### Task 5-gate: Cross-org sweeps closed (.github, validation)

Blocked by Task 3 (it is filed then).

- **Precondition:** a fleet scan run on the day of the check.
- **Procedure:** confirm that every managed repo outside vergil-project has
  a closed sweep task (5j–5p, plus any repo the scan adds), and that its
  nightly audit's local section is compliant with no bridge warning.
- **Acceptance:** all closed and green. Task 7 is blocked by this gate.

### Task 6: Apply the user-level changes (.github, validation)

Blocked by Task 4c, Tasks 5a–5i and Task 5-gate.

- **Precondition:** every managed repo's `CLAUDE.md` carries the new template
  region, so removing the moved sections from the global file leaves no
  repo without them.
- **Procedure:**
  1. Run `/vergil:steering-audit user` with the human (read-only; in the
     session only). Apply the Decided classification: confirm the three
     moved sections are covered by the template, and take anything new to
     intake rather than re-classifying. Review the user settings:
     - keep the single `vergil-marketplace` definition;
     - fix or remove the macOS-only `statusLine`;
     - the human decides on the `AskUserQuestion` deny hook.
  2. The human makes every change to both files on the host. The agent
     writes nothing.
  3. In a fresh VM session, confirm that `~/.claude/CLAUDE.md` no longer
     holds the three moved sections, and that `settings.json` has no
     `/opt/homebrew` path.
- **Results comment:** `Outcome: SUCCESS`, the date it was applied, and
  counts only. No file content, no findings.

### Task 7: Tighten the audit (vergil-tooling, implementation)

Blocked by Tasks 5a–5i and Task 5-gate. **Must not merge before them**: the
next v2.1 patch release takes it fleet-wide that night.

**Files:**

- Delete: `src/vergil_tooling/data/claude_settings_previous.json`, `src/vergil_tooling/data/claude_md_consumer_previous.md`
- Modify: `src/vergil_tooling/lib/repo_config.py`:
  - remove the bridge and the legacy no-marker path;
  - add `_check_claude_md_length(repo_root: Path, items: list[DiffItem]) -> None` (`CLAUDE_MD_MAX_LINES = 200`, field `local.claude_md_length`);
  - add `_check_agents_md(repo_root: Path, items: list[DiffItem]) -> None` (field `local.agents_md`; fails whenever `AGENTS.md` exists);
  - wire both into `audit_local_config`.
- Modify: `src/vergil_tooling/lib/update_deps/updaters/vergil_eco.py` (delete `normalize_claude_ref` and its calls), `src/vergil_tooling/lib/vergil_refs.py` (delete `EXPECTED_MARKETPLACE_REF` and `is_deprecated_marketplace_ref` if nothing else uses them)
- Modify: `docs/site/docs/reference/config-audit.md`
- Test: `tests/vergil_tooling/test_repo_config.py`, `tests/vergil_tooling/test_update_deps_vergil_eco.py`, `tests/vergil_tooling/test_vergil_refs.py` (its tests of the deleted constants go with them)

- [ ] **Step 1: Write the failing tests.**
  - `test_claude_md_200_lines_passes`, `test_claude_md_201_lines_fails`,
    `test_claude_md_trailing_newline_not_counted`
  - `test_agents_md_absent_passes`, `test_agents_md_present_fails`
  - `test_settings_previous_template_now_fails`,
    `test_claude_md_legacy_form_fails`
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement,** removing the tests that cover the deleted code.
- [ ] **Step 4: Run the tests and see them pass,** then
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 5: Refactor.** Remove imports and constants left unused; collapse
  `_load_template(previous=...)` back to the single current file.
- [ ] **Step 6: Commit** with `vrg-commit --type feat --scope repo-config`.
  The report-ready notes repeat the merge gate.

### Task 8: Release the tightened audit (vergil-tooling, deployment)

Blocked by Task 7.

- **Precondition (human-attested):** a v2.1 patch release containing Task 7
  is published, and the rolling `v2.1` tag points to it.
- **Procedure:** after the next nightly run, check every managed repo's
  config-audit job.
- **Acceptance:** every one's local section is compliant with **no
  warnings**. Remote drift is reported separately.

### Task 9: Cold-rebuild check on an agent VM (.github, validation)

Blocked by Task 6 and Task 8.

- **Procedure:** cold-rebuild one agent VM, then check:
  - when the **first session starts, without `/reload-plugins`**,
    `claude plugin list --json` includes `vergil`, `paad` and `superpowers`,
    all enabled;
  - the `vergil-marketplace` source is at `ref` `main`, both in
    `~/.claude/settings.json` and in `known_marketplaces.json`. If not,
    `vm_guest`'s `marketplace add` (no ref) has overwritten it: file a fix
    task for `vm_guest` under this epic; this check stays failed until it
    lands;
  - a raw `git status` in any repo is blocked by the guard hook;
  - an agent commit has no `Co-Authored-By` trailer, and a `report-ready`
    summary and notes carry no attribution line;
  - in each repo, `/memory` shows that repo's `CLAUDE.md` loaded, and no
    `AGENTS.md` exists;
  - putting a key into the **main checkout's** `.claude/settings.local.json`
    makes `vrg-commit` in a worktree refuse; deleting the file lets it
    proceed.
- **Also record (no pass or fail):** whether settings alone install the
  plugins (spec §3.8). The rebuild always runs `vm_guest`'s reconcile step
  (`vrg_vm.py` `_rebuild_stages`), so probe it separately after the checks
  above: uninstall `vergil`, `paad` and `superpowers` **and** remove both
  marketplaces (`claude plugin marketplace remove`), leaving the settings
  files as they are. Start a fresh session without `/reload-plugins`, and
  record `claude plugin list --json` at start and again after 2 minutes.
  Only if all three came back is the reconcile step redundant; then file an
  intake item to remove it. Finish with `vrg-vm update` so the VM is left
  reconciled.
- **Acceptance:** every check passes.

### Task filing: on-hold org tracking issues

At task filing, file one ad-hoc issue in each on-hold org
(`mq-rest-admin-project`, `diogenes-project`, `vergils-nemesis`), in that
org's own ad-hoc epic, never linked to this epic. Each says: when the org is
revived, bring it into the steering and settings model (template region,
`.claude/settings.json`, the local-settings guard's `IN_SCOPE_ORGS`, and the
nightly config audit), following this epic's spec. Nothing in those orgs is
changed now.

### Bookends (already filed)

- Docs task vergil-project/.github#559: this plan and the spec.
- Documentation review vergil-project/vergil-tooling#3720: runs after Task 9.
- Retrospective vergil-project/.github#560: last.

## Dependency summary

| Task | Blocked by |
|---|---|
| 1, 2b, 4b | — |
| 2 | 1, 2b |
| 3 (deploy) | 2, 2b, .github#588 |
| 4 | 1 |
| 4c (deploy) | 4, 4b |
| 5a (vergil-tooling) | 1, 2, 4c |
| 5b–5p | 1, 3, 4c |
| 5-gate (validation) | 3 (checks 5j–5p by text) |
| 6 (validation) | 4c, 5a–5i, 5-gate |
| 7 | 5a–5i, 5-gate |
| 8 (deploy) | 7 |
| 9 (validation) | 6, 8 |
| docs review #3720 | 9 |
| retrospective #560 | everything else |

## Self-Review

- **Spec coverage:**

  | Spec section | Task(s) |
  |---|---|
  | §2.1a managed fleet | Global Constraints, 5j–5p, 5-gate |
  | §5.1 model | 1 |
  | §5.2 template, markers, AGENTS.md, body trims | 2, 5 |
  | §5.2 deprecated skills | 4b |
  | §5.2 global file (classify, then apply) | Decided content, 6 |
  | §5.3 template and audit, bridge, vergil-tooling from source | 2, 7 |
  | §5.3 local settings file | 2b |
  | §5.3 release model, merge gate | Global Constraints, 3, 7, 8 |
  | §5.3 user file | 6 |
  | §5.3 `vm_guest` | 1, 9 |
  | §5.4 | 9 |
  | §5.5 mechanical | 2, 7 |
  | §5.5 skill (both scopes) | 4 |
  | §6 ordering, cross-org | dependency summary, 5-gate |
  | §8 operational tasks | 3, 4c, 5-gate, 8, 9 |

- **Types:** `_load_template`, `_load_settings_template`,
  `_check_claude_md` and `_check_claude_settings` keep their names across
  Tasks 2 and 7. `local_settings_paths` and `find_violations` are defined
  once (Task 2b) and only called elsewhere.
- **Proportion:** about the same length as the spec. Code bodies are left to
  the implementer; only the settings JSON and the warning and refusal text
  are pinned, because the spec fixes them.
