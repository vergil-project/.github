# Steering-file and Claude settings review — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Write down which file owns each Claude Code setting and
instruction. Then clean up every steering and settings file in the fleet to
match, keep them matching with the nightly config audit, and add a
`steering-audit` skill for the periodic review that needs judgment.

**Architecture:**

- **The model.** A new vergil-tooling site page holds the ownership model.
- **The audit.** vergil-tooling's managed templates
  (`data/claude_md_consumer.md`, `data/claude_settings.json`) get a new,
  smaller shape. For one release, the config audit
  (`lib/repo_config.py`) accepts both the old and the new shape, so no
  repo's nightly run goes red while the fleet is swept.
- **The sweeps.** Each repo then gets one sweep PR.
- **The tightening.** A final audit change removes the bridge and adds the
  200-line budget and the AGENTS.md rule.
- **The skill.** The new `steering-audit` skill lives in
  vergil-claude-plugin.
- **The user-level files** are reviewed with that skill and applied by the
  human on the host.

**Tech Stack:** Python 3.12+ (pytest, `importlib.resources`), Markdown
(skills, site docs, steering files), JSON settings, Claude Code
`/doctor prompt-audit` and `/paad:pushback`.

**Spec:** `epics/558-steering-settings-review/spec.md`
(vergil-project/.github#558)

**Shape:** this is an epic-level plan. Each task below is **one issue and one
PR in the repo named in its heading**, worked in a worktree of that repo.
Paths are relative to that repo's root. Deployment and validation tasks are
operational: they have no code PR and close on `Outcome: SUCCESS`.

## Global Constraints

- **CLAUDE.md line budget:** each repo's `CLAUDE.md` has at most **200
  lines**, counted as `len(text.splitlines())`, including the template region.
  Don't move content into `@imports` to meet it.
- **Template region budget:** `data/claude_md_consumer.md` is at most **60
  lines**.
- **Settings template:** `data/claude_settings.json` becomes exactly this,
  and each repo's `.claude/settings.json` must equal it after JSON parsing:

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
    "attribution": { "commit": "", "pr": "" }
  }
  ```

- **No `enabledPlugins` or `extraKnownMarketplaces` in repo settings.** They
  live only in the user-level `~/.claude/settings.json`.
- **Bridge warning text** (exact):
  - `.claude/settings.json matches the previous template; replace it with the current template (epic vergil-project/.github#558).`
  - `CLAUDE.md template region matches the previous template; replace it with the current template (epic vergil-project/.github#558).`
- **Personal files stay off GitHub.** Drafts of `~/.claude/CLAUDE.md` and
  `~/.claude/settings.json` are never posted in issues, PR bodies,
  report-ready notes or commits. Record only "reviewed and applied on
  <date>" and a neutral summary of the kinds of changes.
- **No memory dependency.** `steering-audit` doesn't read, write or mention
  memory files. `memory-audit` is not changed.
- **Placement law.** Each task is filed in the repo of its heading.
- **Release model.** Both audit changes ship as **v2.1 patch releases**.
  Every consuming repo pins the rolling `v2.1` tag, so a release reaches all
  repos on the next nightly config audit, with no pin bump. Releases
  (`vrg-release`) are human actions; agents only attest that they happened.
- **Platform facts** are cited from `https://code.claude.com/docs/en/` pages
  and re-verified, with the verification date recorded, whenever the model
  page changes.

## Decided content

**Template region** (`data/claude_md_consumer.md`, at most 60 lines). These
sections, in this order, with these line caps:

| Section | Lines | Content |
|---|---|---|
| Memory | ≤ 5 | Never write memory files without explicit human approval: propose, suggest a destination, wait. (Moved from the global file. The memory mechanism is experimental.) |
| Worktrees | ≤ 12 | Sessions start at the project root; one worktree per issue at `.worktrees/issue-<N>-<slug>/`; the main worktree is read-only. Link to `vergil-tooling/docs/specs/worktree-convention.md` for the rest. |
| Git and GitHub | ≤ 4 | `vrg-git` and `vrg-gh` only; raw `git` and `gh` are blocked by the hook guard. |
| Validation | ≤ 4 | `vrg-container-run -- vrg-validate` is the only validation command. |
| PR handoff | ≤ 14 | Agents never run `vrg-submit-pr`. They hand off with `vrg-pr-workflow report-ready`. After report-ready the branch is frozen; `vrg-pr-workflow unfreeze` is the only way to lift the freeze. Report-ready text is public. (Condensed from vergil-tooling's "Identity modes and PR submission".) |
| Engineering principles | ≤ 6 | Correctness over cost over speed; never ship known-broken code. No silent failures: no swallowed exceptions or fallbacks that hide errors. (Moved from the global file.) |
| paad | ≤ 3 | paad usage policy and deviations: `vergil-claude-plugin/docs/paad-policy.md`. |

**Global `~/.claude/CLAUDE.md` classification** (agent's call, for review in
alignment):

| Section | Disposition |
|---|---|
| Voice-to-text input | keep (personal input method) |
| Memory management | delete: moves to the template region |
| High-stakes recommendations — sources are load-bearing | keep (personal working style) |
| Correctness over cost | delete: moves to the template region |
| No silent failures | delete: moves to the template region |
| Superpowers visual companion — always enabled | keep (personal tool preference) |

## Review Focus

1. **Key order and formatting.** A repo `.claude/settings.json` whose keys
   are in a different order or use different whitespace must still pass. The
   comparison is on parsed JSON. Owned by Task 2.
2. **Legacy repos during the bridge.** A repo still in the legacy
   (no-marker) form, carrying the **previous** template verbatim, must pass
   with the warning, not fail. Five repos are in that state on release day.
   Owned by Task 2.
3. **New repos.** A repo scaffolded by `vrg-repo-init` after Task 2 must
   pass the audit with no warning. Owned by Task 2.
4. **`vrg-update-deps` on the new shape.** A version bump on a repo whose
   settings have no marketplace entry must leave `.claude/settings.json`
   unchanged. Owned by Task 2.
5. **Line-budget edges.** A `CLAUDE.md` of exactly 200 lines passes, 201
   fails, and a trailing newline doesn't count as an extra line. Owned by
   Task 7.

---

### Task 1: Ownership model page (vergil-tooling, implementation)

**Files:**
- Create: `docs/site/docs/guides/claude-config-ownership.md`
- Modify: `docs/site/mkdocs.yml` (nav, under "Identity & Permissions", after "Permission Model")
- Modify: `docs/site/docs/guides/permission-model.md`, `docs/site/docs/reference/config-audit.md`, `docs/site/docs/guides/consuming-repo-setup.md` (one link each)
- Modify: `src/vergil_tooling/lib/vm_guest.py:401-403,447-452` (comments only)

**Interfaces:**
- Produces: the page anchor `claude-config-ownership.md`. Task 4's skill
  and the sweeps (Task 5) cite it as "the ownership model".

- [ ] **Step 1: Re-verify the platform facts.** Fetch the docs pages listed in
  spec §3 and confirm each of the 10 facts. Any that changed go back to the
  human before writing. The page records the URLs and the date.
- [ ] **Step 2: Write the page.** Sections: the placement rule (spec §5.1,
  quoted); the ownership table (spec §5.1); the platform facts with
  citations; unused scopes and why (`settings.local.json`, managed settings);
  the recurring review (the nightly audit's rules and `steering-audit`'s two
  scopes, run at least quarterly and on any Claude Code release that changes
  settings or instruction-file behavior).
- [ ] **Step 3: Correct the `vm_guest.py` comments.** Say what changelog
  2.1.195 actually states (project-only enabled plugins need explicit install
  consent), and that the reconcile step stays until Task 9 shows whether
  user-enabled plugins install on their own. No code change.
- [ ] **Step 4: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: pass (markdownlint covers the page; `mkdocs` nav resolves).
- [ ] **Step 5: Commit** with `vrg-commit --type docs --scope claude-config`.

### Task 2: New templates and the bridged audit (vergil-tooling, implementation)

**Files:**
- Modify: `src/vergil_tooling/data/claude_settings.json` (to the Global Constraints shape)
- Create: `src/vergil_tooling/data/claude_settings_previous.json` (today's repo file: the old template **with** `"ref": "main"` seeded)
- Modify: `src/vergil_tooling/data/claude_md_consumer.md` (to the Decided content)
- Create: `src/vergil_tooling/data/claude_md_consumer_previous.md` (today's template, byte for byte)
- Modify: `src/vergil_tooling/lib/repo_config.py` (`_check_claude_md`, `_check_claude_settings`; delete `_check_marketplace_ref`, `_check_settings_section`, `_check_attribution`)
- Modify: `src/vergil_tooling/lib/repo_init.py:322-332` (`render_claude_settings` stops seeding the ref)
- Modify: `CLAUDE.md` (template region replaced; the "Identity modes and PR submission" body section reduced to what the template doesn't cover), `.claude/settings.json` (to the new shape)
- Modify: `docs/site/docs/reference/config-audit.md` (settings and CLAUDE.md check rows; the bridge)
- Test: `tests/vergil_tooling/test_repo_config.py`, `tests/vergil_tooling/test_repo_init.py`, `tests/vergil_tooling/test_update_deps_vergil_eco.py`

**Interfaces:**
- Produces:
  - `_load_template(previous: bool = False) -> str`
  - `_load_settings_template(previous: bool = False) -> dict[str, Any]`
  - `_check_claude_settings(repo_root, items, warnings)` and
    `_check_claude_md(repo_root, items, warnings)`, which now append the
    bridge warnings.
  - `audit_local_config` passes `warnings` to `_check_claude_md`.
- Note: vergil-tooling's own nightly audit runs from source, so this repo's
  `CLAUDE.md` and settings change in **this** PR.

- [ ] **Step 1: Write the failing tests.**
  - `test_settings_current_template_passes_clean`: a file equal to the new
    template gives no items and no warnings.
  - `test_settings_key_order_irrelevant`: the same content with keys
    reordered and other whitespace passes clean.
  - `test_settings_previous_template_warns`: the previous file passes, with
    exactly the settings bridge warning.
  - `test_settings_other_shape_fails`: the new template plus
    `enabledPlugins` gives one `local.claude_settings` item.
  - `test_claude_md_region_previous_warns` and
    `test_claude_md_legacy_previous_warns`: the previous template, inside
    markers and verbatim without markers respectively, passes with the
    CLAUDE.md bridge warning.
  - `test_claude_md_region_current_clean`.
  - `test_render_claude_settings_equals_template`: in `test_repo_init.py`,
    the rendered JSON equals the new template and has no
    `extraKnownMarketplaces`.
  - `test_repo_init_output_audits_clean`: a scaffolded repo gives no items
    and no warnings.
  - `test_normalize_claude_ref_noop_without_marketplace`: in
    `test_update_deps_vergil_eco.py`, the function returns `None` and leaves
    the file byte-identical.
- [ ] **Step 2: Run the tests and see them fail.**
  `vrg-container-run -- uv run pytest tests/vergil_tooling/test_repo_config.py tests/vergil_tooling/test_repo_init.py tests/vergil_tooling/test_update_deps_vergil_eco.py -q`
  Expected: the new tests FAIL.
- [ ] **Step 3: Write the data files.** Create the two `_previous` files from
  the current content first, then rewrite the two templates.
- [ ] **Step 4: Implement the bridge.**
  - **Settings:** parsed equality with the current template is clean; with
    the previous one, a warning; anything else is one `DiffItem`, whose
    `expected` names the current template and whose `actual` lists the
    differing top-level keys.
  - **CLAUDE.md:** the same three-way result for the marked region (diffed
    against the current template) and for the legacy verbatim match.
- [ ] **Step 5: Update this repo's own files.** Replace this repo's
  `CLAUDE.md` region and `.claude/settings.json`. Remove the body text that
  the template region now covers. The "Cloud session prompt contract" stays
  in this repo's body: it is mostly human-side, and its agent-relevant part
  (stop at report-ready; report-ready text is public) is in the template's
  PR-handoff section.
- [ ] **Step 6: Run the tests and see them pass.** Then run
  `vrg-container-run -- vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 7: Refactor.** Move the shared `importlib.resources` read into
  `_read_data_file(name: str) -> str`. `_load_template` and
  `_load_settings_template` become thin wrappers that only pick the file
  name (current or `_previous`) and, for settings, parse the JSON. Check
  for duplication between the settings and CLAUDE.md three-way checks.
  Re-run the tests: PASS.
- [ ] **Step 8: Commit** with `vrg-commit --type feat --scope repo-config`.

### Task 3: Release the bridged audit (vergil-tooling, deployment)

Blocked by Task 1 and Task 2.

- **Precondition (human-attested):** a v2.1 patch release containing Task 2
  is published, and the rolling `v2.1` tag points to it.
- **Procedure:** after the next nightly run, check each consuming repo's
  `ops.yml` config-audit job. Alternatively, run `vrg-github-repo-config
  audit` from each repo's checkout with the released tooling installed.
  Expected: exit 0, with the bridge warning(s) and no `DiffItem`.
- **Acceptance:** all eight repos exit 0. A red result is a FAILURE, and the
  task stays open.

### Task 4: `steering-audit` skill (vergil-claude-plugin, implementation)

Blocked by Task 1.

**Files:**
- Create: `skills/steering-audit/SKILL.md`
- Modify: `README.md` (skills table), `docs/site/docs/skills/index.md` (table row and a `## steering-audit` section), `CLAUDE.md` if it lists the skills

**Interfaces:**
- Consumes: the ownership model page (Task 1).
- Produces: the invocations `/vergil:steering-audit` (repo scope) and
  `/vergil:steering-audit user`.

- [ ] **Step 1: Write the SKILL.md.**
  - **Frontmatter:** `name: steering-audit`. The description has trigger
    phrases ("audit steering files", "review CLAUDE.md", "review Claude
    settings", "steering audit") and says that memory is not in scope.
  - **Workflow:**
    1. **Inventory.** In repo scope: `CLAUDE.md` with its line count,
       `.claude/rules/`, `.claude/settings.json`, and any `AGENTS.md`. In
       user scope: `~/.claude/CLAUDE.md` and `~/.claude/settings.json`.
    2. **Re-verify.** Check the model page's platform facts against the live
       docs, and report any drift first.
    3. **Mechanical check.** Run `vrg-github-repo-config audit` (repo scope
       only).
    4. **Review.** Run `/doctor prompt-audit <path>` on each steering file.
       If a skill cannot invoke it, tell the human to run it and paste the
       report. Then run `/paad:pushback <path>`.
    5. **Walk the findings one at a time.** Each finding gets one
       disposition: keep, edit, move to a named managed home (template
       region, plugin, repo docs, `.claude/rules/`), delete, or file to
       intake. Wait for the human's decision on each.
    6. **Apply.** Repo-scope edits go through the normal worktree and PR
       flow. User-scope drafts are shown in the session or written to a
       local scratch file, never to GitHub, and the human applies them.
  - **Constraints:** user scope runs once per cycle; drafts of user-level
    files stay off GitHub; there is no memory access; every finding ends
    fixed, in intake, or dropped by the human with a reason (umbrella
    §2.4).
- [ ] **Step 2: Add the catalog entries.**
- [ ] **Step 3: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: PASS (this includes the plugin's `validate-custom` skill checks).
- [ ] **Step 4: Commit** with `vrg-commit --type feat --scope skills`.

### Task 5a–5i: Per-repo steering sweep (one task per repo, implementation)

One task in each repo below. Each is blocked by Task 4, and each except
vergil-tooling is blocked by Task 3.

| Task | Repo | `CLAUDE.md` now | Extra work |
|---|---|---|---|
| 5a | vergil-tooling | 554 | delete `AGENTS.md`; move "Architecture" catalogues and the "Consumption Model" detail into site docs and link them |
| 5b | vergil-claude-plugin | 375 | adopt markers; delete `AGENTS.md` |
| 5c | vergil-containers | 307 | adopt markers |
| 5d | vergil-actions | 234 | adopt markers; delete `AGENTS.md` (its `standards-and-conventions` instructions are obsolete) |
| 5e | .github | 158 | adopt markers |
| 5f | docs | 128 | — |
| 5g | vergil-vm | 124 | adopt markers |
| 5h | vergil-python | 120 | — |
| 5i | packages | 115 | — |

**Files (each repo):** `CLAUDE.md`, `.claude/settings.json`, `AGENTS.md`
(where listed), and the repo's docs pages that receive moved material.

Steps (each repo):

- [ ] **Step 1: Run the review.** Run `/vergil:steering-audit` in the repo
  and take its findings to the human (the skill's workflow).
- [ ] **Step 2: Replace the managed parts.** Put the current template in the
  marked region (adding markers where listed), and set
  `.claude/settings.json` to the template.
- [ ] **Step 3: Trim the body to the ownership model.**
  - Delete any fleet policy the template already states.
  - Move reference material into the repo's docs and link it.
  - Move part-of-repo guidance to `.claude/rules/` with `paths` frontmatter.
  - Before deleting an `AGENTS.md`, move anything in it that is still true
    and not already in `CLAUDE.md`.
- [ ] **Step 4: Review again.** Run `/vergil:steering-audit` on the
  trimmed files (the spec's "before and after" review) and resolve its
  findings with the human.
- [ ] **Step 5: Check the result.** `wc -l CLAUDE.md` must show at most
  200, and `vrg-github-repo-config audit` must exit 0 with **no** bridge
  warning. Run `vrg-container-run -- vrg-validate`; expected: PASS.
- [ ] **Step 6: Commit** with `vrg-commit --type docs --scope claude-md`.
  The report-ready notes list where each moved section went, and **every
  instruction deleted** together with where it is still covered (template
  region, hook guard, skill or docs). This is the evidence for the spec's
  "no lost instruction" anti-goal.

### Task 6: User-level review and apply (.github, validation)

Blocked by Task 3, Task 4, and **every one of Tasks 5a–5i**.

- **Precondition:** every repo's `CLAUDE.md` carries the new template
  region, which has the Memory and Engineering principles sections, so
  removing those sections from the global file leaves no repo without them.
- **Procedure:**
  1. The agent runs `/vergil:steering-audit user` with the human, applying
     the Decided-content classification and fixing the user settings:
     - keep the single `vergil-marketplace` definition;
     - fix or remove the macOS-only `statusLine`;
     - the human decides on the `AskUserQuestion` deny hook.
  2. The human applies both files on the host. They reach the VMs on the
     next session.
  3. In a fresh VM session, the agent confirms that `~/.claude/CLAUDE.md`
     no longer holds the three moved sections and that `settings.json` has
     no `/opt/homebrew` path.
- **Results comment:** `Outcome: SUCCESS`, the date it was applied, and a
  neutral summary of the kinds of changes. No file content.

### Task 7: Tighten the audit (vergil-tooling, implementation)

Blocked by every Task 5 task.

**Files:**
- Delete: `src/vergil_tooling/data/claude_settings_previous.json`, `src/vergil_tooling/data/claude_md_consumer_previous.md`
- Modify: `src/vergil_tooling/lib/repo_config.py`:
  - remove the bridge and the legacy no-marker path;
  - add `_check_claude_md_length(repo_root: Path, items: list[DiffItem]) -> None` (limit `CLAUDE_MD_MAX_LINES = 200`, field `local.claude_md_length`);
  - add `_check_agents_md(repo_root: Path, items: list[DiffItem]) -> None` (field `local.agents_md`; passes when there is no `AGENTS.md`, or when `CLAUDE.md` has a line that is exactly `@AGENTS.md`);
  - wire both into `audit_local_config`.
- Modify: `src/vergil_tooling/lib/update_deps/updaters/vergil_eco.py` (delete `normalize_claude_ref` and its calls), `src/vergil_tooling/lib/vergil_refs.py` (delete `EXPECTED_MARKETPLACE_REF` and `is_deprecated_marketplace_ref` if nothing else uses them; check with grep)
- Modify: `docs/site/docs/reference/config-audit.md`
- Test: `tests/vergil_tooling/test_repo_config.py`, `tests/vergil_tooling/test_update_deps_vergil_eco.py`

- [ ] **Step 1: Write the failing tests.**
  - `test_claude_md_200_lines_passes`, `test_claude_md_201_lines_fails`,
    `test_claude_md_trailing_newline_not_counted` (200 lines plus a final
    `\n` passes).
  - `test_agents_md_absent_passes`, `test_agents_md_imported_passes`,
    `test_agents_md_unimported_fails`.
  - `test_settings_previous_template_now_fails`,
    `test_claude_md_legacy_form_fails` (expected `well-formed template
    markers`).
- [ ] **Step 2: Run the tests and see them fail.**
- [ ] **Step 3: Implement.** Remove the dead tests that cover the deleted
  code.
- [ ] **Step 4: Run the tests and see them pass.** Then run
  `vrg-container-run -- vrg-validate`; expected PASS at 100% coverage.
- [ ] **Step 5: Refactor.** Remove imports and constants left unused
  (`grep -rn 'EXPECTED_MARKETPLACE_REF\|is_deprecated_marketplace_ref\|MARKETPLACE_NAME' src tests`
  should show only live uses). Collapse `_load_template(previous=...)`
  back to the single current file. Re-run validation: PASS.
- [ ] **Step 6: Release gate.** Note in the report-ready notes that this
  PR must not be released until every Task 5 sweep has merged, because a
  v2.1 patch takes effect fleet-wide that night.
- [ ] **Step 7: Commit** with `vrg-commit --type feat --scope repo-config`.

### Task 8: Release the tightened audit (vergil-tooling, deployment)

Blocked by Task 7.

- **Precondition (human-attested):** a v2.1 patch release containing
  Task 7 is published, and the rolling `v2.1` tag points to it.
- **Procedure:** after the next nightly run, check every repo's
  config-audit job, as in Task 3. Expected: exit 0 with **no warnings**.

### Task 9: Cold-rebuild check on an agent VM (.github, validation)

Blocked by Task 6 and Task 8.

- **Procedure:** cold-rebuild one agent VM, then check:
  - `claude plugin list --json` includes `vergil`, `paad` and `superpowers`,
    all enabled;
  - a raw `git status` in any repo is blocked by the guard hook;
  - a commit and a PR body drafted by Claude have no `Co-Authored-By`
    trailer and no attribution line;
  - in each repo, `/memory` shows that repo's `CLAUDE.md` loaded and no
    `AGENTS.md` exists.
- **Also record (no pass or fail):** whether the plugins were installed
  before `vm_guest`'s reconcile step ran, which tells us whether
  user-enabled plugins install on their own (spec §3.8). If the reconcile
  step turns out to be redundant, file an intake item to remove it.
- **Acceptance:** every check passes.

### Bookends (already filed)

- Docs task vergil-project/.github#559: this plan and the spec.
- Documentation review vergil-project/vergil-tooling#3720: runs after Task 9.
- Retrospective vergil-project/.github#560: last.

## Dependency summary

| Task | Blocked by |
|---|---|
| 1, 2 | — |
| 3 (deploy) | 1, 2 |
| 4 | 1 |
| 5a (vergil-tooling) | 2, 4 |
| 5b–5i | 3, 4 |
| 6 (validation) | 3, 4, 5a–5i |
| 7 | 5a–5i |
| 8 (deploy) | 7 |
| 9 (validation) | 6, 8 |
| docs review #3720 | 9 |
| retrospective #560 | everything else |

Tasks 1 and 2, Task 4, and the sweeps run in parallel wherever their
blockers allow.

## Self-Review

- **Spec coverage:**

  | Spec section | Task(s) |
  |---|---|
  | §5.1 model | 1 |
  | §5.2 template, markers, AGENTS.md, body trims | 2, 5 |
  | §5.2 global file | 6 |
  | §5.3 template and audit | 2 |
  | §5.3 bridge | 2, 7 |
  | §5.3 repo files | 5 |
  | §5.3 user file | 6 |
  | §5.3 `vm_guest` | 1, 9 |
  | §5.4 | 9 |
  | §5.5 mechanical | 2, 7 |
  | §5.5 skill | 4 |
  | §6 ordering | dependency summary |
  | §8 operational tasks | 3, 8, 9 |

  Gap fixed: the spec's memory-policy section moving out of the global file
  needed a home, so it is the template's Memory section.
- **Types:** `_load_template`, `_load_settings_template`,
  `_check_claude_md` and `_check_claude_settings` keep the same names
  across Tasks 2 and 7.
- **Proportion:** about 1.2× the spec. Code bodies are left to the
  implementer; only the settings JSON and the warning text are pinned,
  because the spec fixes them.
