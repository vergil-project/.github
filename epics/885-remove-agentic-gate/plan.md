# Install paad from the fork and remove the agentic-review gate Implementation Plan

- **Epic:** vergil-project/.github#885
- **Planning issue:** vergil-project/.github#880 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every box installs paad from the fork's `develop` branch through the
marketplace, the `vrg-vm session` `--plugin-dir` override is gone, and
`issue-implement` is back to implement, validate to green, `report-ready`, so
it runs on cloud VMs.

**Architecture:** Two independent halves. The fork half: the human puts the
fork's `develop` in place and points the host settings at it (D1), a cloud VM
proves the install (V1), then vergil-tooling drops the override (Task 3) and
the human retires its config and the fleet checkout (D2), and the plugin's
paad policy is rewritten (Task 4). The gate half: the plugin drops the gate
(Task 5), vergil-tooling's docs drop its mentions (Task 6), and once a plugin
release carries Task 5 (D3), a cloud VM runs `issue-implement` end to end
(V2).

**Tech Stack:** Markdown skills and docs (vergil-claude-plugin), Python 3.12+
and pytest (vergil-tooling), the paad fork's `make` checks, Claude Code's
plugin marketplace.

**Spec:** `epics/885-remove-agentic-gate/spec.md`

## Global Constraints

- **Standing order (spec §4):** every `issue-implement` run on this epic's
  tasks skips the agentic-review gate and Preflight steps 6 and 7; it
  implements, validates to green and goes straight to `report-ready`. Each
  task body repeats this line, and `epic-implement` passes it in every
  dispatch.
- **Only the human changes the fork.** Agents never run git commands that
  change `wphillipmoore/paad` (`vrg-git` does not stop them); they give the
  human the commands to run (spec §5.5).
- The fleet installs paad from `wphillipmoore/paad` at ref `develop`; the
  plugin id stays `paad@paad` (spec §5.5).
- On `develop`, `preview/paad/.claude-plugin/plugin.json` and the catalog
  entry for `paad` carry no `version` field; `package.json` and
  `plugins/paad` keep upstream's base version; announce lines read
  `<base>-preview` (spec §5.2).
- `develop` rebases onto upstream release tags (`paad--v<X.Y.Z>`) only, never
  `upstream/main` (spec §5.3).
- The push rule: the commit pushed to `develop` is the commit the validation
  gate ran on, from a clean working tree. Pushes touching nothing under
  `preview/paad/` are exempt from the scenario sets (spec §5.4).
- Each removal task starts with a search of vergil-tooling,
  vergil-claude-plugin and `.github` for `plugin-dirs`, `plugin_dirs`,
  `--plugin-dir`, `paad-fleet`, the fleet checkout, `agentic-review` and
  mode 2, and handles every hit outside `epics/`, `CHANGELOG.md`,
  `releases/` and completed plans and specs (spec §7). The file lists below
  are starting points.
- Claude Code's own `claude --plugin-dir <tree>` flag for local development
  stays documented (the plugin `CLAUDE.md`, the README's and site index's
  "Local development"); only the vergil override that set it automatically
  goes.
- Doctrine §1 and §2 section numbers are kept (spec §6.1).
- Releases, `vrg-submit-pr` and merges are human actions. Validation in
  vergil repos is `vrg-container-run -- vrg-validate` only.
- The epic runs as a freeze from D1 until D2 closes: no other fleet work.
  Breakage in between is fixed forward (spec §7).

## Review Focus

1. **Re-registering a marketplace whose repo changed under the same name**
   (`paad` moving from `Ovid/paad` to `wphillipmoore/paad@develop`): the box
   ends up on the fork, or the refresh fails loudly; it never silently stays
   on upstream. Pinned in V1 step 1.
2. **A `version` field that comes back on a rebase:** boxes would silently
   stop picking up pushes. Pinned in D1 step 4 and V1 step 2.
3. **Dangling links into deleted doctrine §3** from any skill or doc: every
   `paad-review-doctrine.md#3` anchor is gone or re-pointed. Pinned in
   Task 5 step 7.
4. **A user's own `--plugin-dir` after `vrg-vm session --`** still reaches
   `claude`, since the manual validation gate relies on it. Pinned in Task 3
   step 1.
5. **A Lima session after the override's config is deleted** loads the
   installed fork paad and lists `paad:paad-delegate`, not upstream paad and
   not the stale fleet tree. Pinned in D2 step 3.

---

## Task order and placement

| # | Task | Repo (where it closes) | Kind | Blocked by |
|---|---|---|---|---|
| D1 | Install paad from the fork | `.github` | deployment | — |
| V1 | The fork install on a cloud VM | `.github` | validation | D1 |
| 3 | Remove the `--plugin-dir` override | vergil-tooling | task | V1 |
| D2 | Install vergil-tooling without the override; retire its config and the fleet checkout | vergil-tooling | deployment | 3 |
| 4 | paad policy and doctrine §1.8 for the fork install | vergil-claude-plugin | task | V1, 5 |
| 5 | Remove the agentic-review gate | vergil-claude-plugin | task | — |
| 6 | Gate mentions in vergil-tooling | vergil-tooling | task | — |
| D3 | Release and install the plugin with the gate removed | vergil-claude-plugin | deployment | 5 |
| V2 | `issue-implement` end to end on a cloud VM | `.github` | validation | D3 |

Task 4 waits for Task 5 only because both edit doctrine §1.8 and
paad-policy §10; landing the removal first keeps the rewrite conflict-free.
D1 and V1 have no fleet PR, since the fork is outside the org; they close by
comment.

The existing bookend tasks stay as filed: the docs task
(vergil-project/.github#886, this plan's PR), the documentation review
(vergil-project/vergil-claude-plugin#916) and the retrospective
(vergil-project/.github#887).

---

### D1: Install paad from the fork

Deployment, carried out by the human with the agent guiding. Spec §5.1–§5.5.
Work in the dev clone `/Users/pmoore/dev/projects/wphillipmoore/paad`. The
agent runs only read-only commands there and gives the human every command
that changes the clone or the remote.

- [ ] **Step 1 (human):** fetch upstream with tags, create `develop` from
  `unattended-mode` and rebase it onto `paad--v2.1.0`, dropping any fleet
  patch 2.1.0 already carries. Expected conflicts are in
  `preview/paad/skills/agentic-review/SKILL.md` (upstream's symlink guard and
  `paad/security` checks).
- [ ] **Step 2 (agent, read-only):** check the tag carries no unreleased
  work: `vrg-git diff paad--v2.1.0:plugins/paad paad--v2.1.0:preview/paad`.
  Expected: the only differences are the `-preview` announce suffix, `internal: true`
  metadata and the `plugin.json` version. Anything else stops D1.
- [ ] **Step 3 (human, agent drafts the edits):** the fleet's version delta,
  as one commit on `develop`:
  - `.claude-plugin/marketplace.json`: the `paad` entry's `source` becomes
    `"./preview/paad"` and its `version` key is removed;
  - `preview/paad/.claude-plugin/plugin.json`: the `version` key is removed;
  - `Makefile` `check-versions`: compare `package.json` with
    `plugins/paad`'s version only; fail if the catalog's `paad` entry or
    `preview/paad`'s manifest carries a `version` key;
  - `Makefile` `check-skill-versions`: when the tree's `plugin.json` has no
    `version`, expect `v<plugins/paad version>-preview` in every announce
    line.
- [ ] **Step 4 (agent, read-only):** run `make test`. Expected: passes. Run
  `grep -c '"version"' preview/paad/.claude-plugin/plugin.json` and
  `python3 -c "import json;print([p.get('version') for p in json.load(open('.claude-plugin/marketplace.json'))['plugins']])"`.
  Expected: `0` and `[None]`. Every `Running paad:` line in
  `preview/paad/skills/*/SKILL.md` reads `v2.1.0-preview`.
- [ ] **Step 5:** run the validation gate (spec §5.4): the #733 scenario sets
  (vergil-project/.github#733, spec §5) against the candidate, loaded with
  `vrg-vm session -- claude --plugin-dir <dev clone>/preview/paad`, in newly
  started sessions, through `issue-validate`'s checklist conventions. For
  the run, the human sets `plugin_dirs = []` in
  `~/.config/vergil/plugin-dirs.toml`, so the old fleet tree is not loaded
  beside the candidate.
  Security results as counts only. Record the commit the gate ran on.
- [ ] **Step 6 (human):** with the working tree clean and HEAD at that
  commit, `git push -u origin develop`.
- [ ] **Step 7 (human):** set the host's `~/.claude/settings.json`
  `extraKnownMarketplaces.paad` to
  `{ "source": { "source": "github", "repo": "wphillipmoore/paad", "ref": "develop" } }`,
  then run `vrg-host plugins` and `vrg-vm update --all`. Expected: on the
  host and every running VM, `paad@paad` installs from the fork.
- [ ] **Step 8:** comment `Outcome: SUCCESS` with the pushed `develop`
  commit and the gate's run summary, and close.

### V1: The fork install on a cloud VM

Validation, blocked by D1. Spec §7 step 2.

- [ ] **Step 1:** refresh a cloud VM (`vrg-vm update` on it). Check
  `claude plugin marketplace list --json` shows `paad` at
  `wphillipmoore/paad` ref `develop`, and `claude plugin list --json` shows
  `paad@paad` at D1's commit. If the refresh refused to re-add the
  marketplace, record the error, have the human remove the registration
  (`claude plugin marketplace remove paad`) and refresh again; record which
  path was needed.
- [ ] **Step 2:** start a session; `paad:paad-delegate` is a listed agent
  type. The human pushes a behaviour-free commit to `develop`
  (`git commit --allow-empty`, which touches no file, so it is exempt from
  the gate and a later rebase drops it). Refresh the VM again.
  Expected: the installed commit is the new one, with no version changed.
- [ ] **Step 3:** comment the outcome with both installed commits and the
  re-add finding (Task 4 records it), and close on SUCCESS.

### Task 3: Remove the `--plugin-dir` override

vergil-tooling, blocked by V1. Spec §7 step 3, §5.5. Standing order applies.

**Files:**

- Delete: `src/vergil_tooling/lib/plugin_dirs.py`,
  `tests/vergil_tooling/test_plugin_dirs.py`
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (the `plugin_dirs_lib` import,
  `_PLUGIN_DIRS_CLOUD_NOTICE`, the `plugin_dirs` parameters of
  `_session_inner` and `_cloud_session`, the load around line 3120)
- Modify: `tests/vergil_tooling/test_vrg_vm.py` (35 override references),
  `tests/vergil_tooling/conftest.py` (`_hermetic_plugin_dirs`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` ("The
  `--plugin-dir` override"), `docs/site/docs/guides/releasing.md`,
  `docs/site/docs/standards/github-issues.md`

**Interfaces:**

- Produces: `_session_inner(...)` and `_cloud_session(target, args)` without
  a `plugin_dirs` parameter; `vrg-vm session` never reads
  `~/.config/vergil/plugin-dirs.toml`.

- [ ] **Step 1: Search** the three repos per Global Constraints and list the
  hits in the task's notes. Add the Review Focus 4 test first:
  `test_session_passes_user_plugin_dir_through`, asserting that
  `vrg-vm session -- claude --plugin-dir /x` builds a launch whose argv
  contains `--plugin-dir /x` exactly once, on Lima and on cloud. Run it;
  expected: passes before and after the removal.
- [ ] **Step 2:** delete the override's tests and the `conftest.py`
  fixture. No test replaces them: once the module is gone nothing reads the
  file.
- [ ] **Step 3:** remove the override from `vrg_vm.py` and delete
  `lib/plugin_dirs.py`. Expected: Step 1's test still passes, and
  `vrg-git grep -nE "plugin_dirs|plugin-dirs" -- src tests` finds nothing.
- [ ] **Step 4:** rewrite the docs: remove the override section and the
  Lima-only paragraph from `cli-tools-overview.md`; in `releasing.md`,
  remove the paragraph saying a Lima session with the override bypasses the
  installed plugin; in `github-issues.md`, remove the override and
  Lima-only text (the gate text is Task 6's).
  The plugin README's Update-section paragraph and the site `index.md`
  sentence on the override are in Task 5, since a vergil-tooling PR cannot
  close plugin changes.
- [ ] **Step 5:** `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 6:** `vrg-commit --type feat --scope vrg-vm --message "remove the --plugin-dir session override"`,
  then `report-ready`.

### D2: Install vergil-tooling without the override; retire its config and the fleet checkout

Deployment, blocked by Task 3. Precondition, attested by the human: a
vergil-tooling release carrying Task 3 is published.

- [ ] **Step 1 (human):** install the release on the host.
- [ ] **Step 2 (human):** delete `~/.config/vergil/plugin-dirs.toml`;
  remove the `paad-fleet` worktree and delete the local `fleet` branch (it
  was never pushed); delete `unattended-mode` locally and on `origin`, since
  `develop` holds its commits. The agent gives the commands; it runs none of
  them.
- [ ] **Step 3:** start a fresh Lima `vrg-vm session`. Expected: no
  `--plugin-dir` in the launch, `claude plugin list --json` shows `paad@paad`
  from `wphillipmoore/paad`, and `paad:paad-delegate` is listed.
- [ ] **Step 4:** comment the outcome and the installed version, and close
  on SUCCESS. The freeze ends here.

### Task 4: paad policy and doctrine §1.8 for the fork install

vergil-claude-plugin, blocked by V1 and Task 5. Spec §5.5. Standing order
applies.

**Files:**

- Modify: `docs/paad-policy.md` §1, §8 (the deviation rows on the installed
  release and the fleet checkout), §9, §10
- Modify: `docs/paad-review-doctrine.md` §1.8
- Modify: `skills/epic-create/SKILL.md` (Design review rounds step 1's fix
  text, which names `vrg-vm session` and the fleet checkout)

- [ ] **Step 1:** search per Global Constraints for `paad-fleet`, `fleet
  checkout`, `fleet` branch, `plugin-dirs`, `Lima only` and `Ovid/paad`
  in the plugin and handle every hit.
- [ ] **Step 2:** rewrite §10 (retitle it "paad from the fork") with
  exactly spec §5.5's six bullets: the install source; the version scheme,
  rebase process (spec §5.3's seven steps, verbatim in substance) and
  validation gate with its push rule and exemption; only the human changes
  the fork; the override, fleet checkout, `fleet` branch, Lima-only
  paragraph and its accepted risk removed; the one reworded accepted risk;
  the delegate required, a missing one meaning a misconfigured box. Add
  V1's re-add finding as the procedure for switching a box's source. Add a
  second accepted risk: only the human pushes to the fork, and nothing
  technical enforces it (`vrg-git` allows `push`, and the clone is on the
  read-write projects mount).
- [ ] **Step 2a:** the retitle changes §10's anchor. Re-point every link to
  `paad-policy.md#10-paad-unattended-mode-and-the-fleet-checkout` in
  vergil-claude-plugin, vergil-tooling and `.github` (outside historical
  records) to the new anchor, including doctrine §1.9's.
- [ ] **Step 3:** make §1, §8 and §9 agree with §10 (paad's version is the
  fork's `develop`, installed through the marketplace).
- [ ] **Step 4:** rewrite doctrine §1.8: the delegate comes from the
  installed paad; the stop message names the host settings entry
  `extraKnownMarketplaces.paad` → `wphillipmoore/paad@develop` and
  `vrg-host plugins` / `vrg-vm update`; no override, fleet checkout or
  "cloud boxes have no delegate". Update `epic-create`'s step 1 fix text
  to the same.
- [ ] **Step 5:** `vrg-git grep -nE "paad-fleet|plugin-dirs|fleet checkout|Lima only|10-paad-unattended-mode-and-the-fleet-checkout" -- docs skills README.md`
  in vergil-claude-plugin, and the same pattern in vergil-tooling's `docs`
  and `src`. Expected: no match outside historical records.
- [ ] **Step 6:** `vrg-container-run -- vrg-validate`; expected: passes.
  `vrg-commit --type docs --scope paad-policy --message "install paad from the fork's develop branch"`,
  then `report-ready`.

### Task 5: Remove the agentic-review gate

vergil-claude-plugin, independent. Spec §6. Standing order applies.

**Files:**

- Modify: `skills/issue-implement/SKILL.md`, `skills/epic-implement/SKILL.md`,
  `skills/epic-retrospective/SKILL.md`
- Modify: `docs/paad-review-doctrine.md` (intro, §1.3, §1.5, §1.6, §1.7,
  §1.8, §1.9, §1.10; delete §3), `docs/paad-policy.md` (the agentic-review
  output rule, the mode 2 deviation row, §10's gate mentions)
- Modify: `hooks/scripts/block-worktree-bypass-write.sh` (header comment),
  `hooks/tests/block-worktree-bypass-write.test.sh` (fixture paths, lines 33
  and 135)
- Modify: `README.md` (the `issue-implement` row; the Update-section
  paragraph on `plugin-dirs.toml`), `docs/site/docs/index.md` (the sentence
  on a Lima session loading a tree with `--plugin-dir`),
  `docs/site/docs/skills/index.md` (the paad review rounds section's
  override and Lima-only text, the `issue-implement` row and section),
  `docs/development/skills-architecture.md` line 349

- [ ] **Step 1:** search per Global Constraints. List any task paused at
  the gate (a trusted `agentic-review round <n>: response` comment with a
  live `Paused:` line) in the fleet's open issues; those are resumed later
  as ordinary runs (spec §6.3).
- [ ] **Step 2: `issue-implement`.** Delete "The agentic-review gate"
  section, Preflight steps 6 and 7 (renumber the rest), step 8's
  resume-a-paused-task line, the Lima-session paragraph, and the gate's
  mentions in the description, overview and creating-issues paragraph.
  Step 5 keeps its trusted-author check, links doctrine §1.2 instead of
  §3.3, and on an untrusted author stops and waits for the human's ruling.
  The skill reads: Preflight, Implement, validate to green, `report-ready`.
- [ ] **Step 3: doctrine.** Before deleting §3, move what kept text
  depends on: the ruling and `Recorded:` line form (§3.5) and the
  `vrg-security-record create` procedure for a `record` ruling (§3.6) go
  into §1.7, written for `issue-implement`'s follow-ups; how rulings arrive
  through `epic-implement`'s dispatch prompt (§3.8) goes into §1.7; every
  other reference to §3 (the trusted-author link to §3.3, and the links in
  §1.2–§1.10) points at §1.2, §1.6, §1.7 or §2.7, or goes with the mode 2
  text around it. Then delete §3. In §1: drop §1.3's `agentic-review`
  column; rename §1.5's `agentic-review` row to `issue-implement`,
  `build/.vergil-state/issue-implement/<N>/`, holding a `record` ruling's
  `security-<i>.md`; drop mode 2 handling in §1.6–§1.10 and §1.7's "inside
  the agentic-review gate" branch, keeping the approval rule and the
  `Awaiting a ruling` list; the intro describes mode 1 design review only.
  Keep §1 and §2 numbering.
- [ ] **Step 4:** `epic-implement`: remove the paused-at-the-gate report
  list and any dispatch text about gate pauses and `fix` confirmations.
  `epic-retrospective`: remove the step reading `agentic-review summary` or
  round comments.
- [ ] **Step 5:** paad-policy: remove the agentic-review output rule, the
  three-round-cap row and §10's gate mentions.
- [ ] **Step 6:** hook: reword the header comment and rename the test
  fixtures to `build/.vergil-state/issue-implement/9001/1/...`. Run the hook
  test through `vrg-validate`; expected: passes with the logic unchanged.
- [ ] **Step 7:** docs per Files. Then check Review Focus 3:
  `vrg-git grep -nE "paad-review-doctrine\.md#3|agentic-review gate|mode 2" -- . ':!CHANGELOG.md' ':!releases' ':!docs/specs' ':!docs/plans'`,
  and inside the doctrine
  `vrg-git grep -nE "\]\(#3[0-9]|§3\.[0-9]" -- docs/paad-review-doctrine.md`.
  Expected: no match from either.
- [ ] **Step 8:** list any `build/.vergil-state/agentic-review/` directory
  in the fleet's main clones, once, in the final report, for the human to
  delete.
- [ ] **Step 9:** `vrg-container-run -- vrg-validate`; expected: passes.
  `vrg-commit --type feat --scope issue-implement --message "remove the agentic-review gate"`,
  then `report-ready`.

### Task 6: Gate mentions in vergil-tooling

vergil-tooling, independent. Spec §6.2, §7 step 6. Standing order applies.

**Files:**

- Modify: `docs/site/docs/standards/github-issues.md` (the gate section),
  `docs/site/docs/guides/security-records.md` (the policy exception at
  lines 15–18 and the "Agentic review, during `issue-implement`" section
  are reworded to what `issue-implement` keeps: a security follow-up it
  finds stays private until the human rules, a `record` ruling files it
  through `vrg-security-record`, and its details sit in
  `build/.vergil-state/issue-implement/<N>/`, linking doctrine §1.7; line
  53's review rounds become `epic-create`'s only),
  `docs/site/docs/guides/account-setup.md` ("used first by the agentic
  review gate")
- Modify: `src/vergil_tooling/bin/vrg_git.py:87` and
  `tests/vergil_tooling/test_gitignore.py:36` (comments only: the
  `symbolic-ref` allowance and `paad/code-reviews/` serve any paad skill run)

- [ ] **Step 1:** search per Global Constraints for `agentic-review`,
  `agentic review` and mode 2 in vergil-tooling; handle every hit not
  Task 3's.
- [ ] **Step 2:** edit the files above; no behaviour changes. Then
  `vrg-git grep -nE "paad-review-doctrine\.md#3|agentic-review gate|agentic review gate" -- . ':!CHANGELOG.md' ':!releases' ':!docs/specs' ':!docs/plans'`.
  Expected: no match.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`; expected: passes.
  `vrg-commit --type docs --scope paad --message "drop the agentic-review gate from the docs"`,
  then `report-ready`.

### D3: Release and install the plugin with the gate removed

Deployment, blocked by Task 5. Precondition, attested by the human: a
vergil-claude-plugin release carrying Task 5 is published and the floating
tag points at it. Run `vrg-host plugins` and `vrg-vm update --all`; record
the installed version. Check a new session's `issue-implement` has no "The
agentic-review gate" section.

### V2: `issue-implement` end to end on a cloud VM

Validation, blocked by D3. Spec §7 step 7. On a cloud VM, run
`issue-implement` on a real open task (Task 6, if still open, or any small
fleet task). Expected: Preflight passes with no delegate check, no state
directory check and no gate; the run validates to green and reaches
`report-ready`. Record the task, the VM and the report-ready ref; close on
SUCCESS.
