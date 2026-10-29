# paad unattended mode Implementation Plan

- **Epic:** vergil-project/.github#699
- **Planning issue:** vergil-project/.github#698 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an unattended mode to paad's pushback, alignment and
agentic-review, a caller-dispatched `paad-delegate` agent that never asks,
writes nothing and returns a structured result. Build it on a fork, run it
across the vergil fleet from a dedicated preview checkout, and switch vergil's
review doctrine to it, keeping today's path as a fallback.

**Architecture:** The paad change is prose plus one lint: a new agent
definition (the contract), one standard paragraph and one unattended section
per supported skill, conditional wording in the shared configuration and
security paragraphs, and a `check_unattended.py` lint. vergil-tooling gains a
`--plugin-dir` override in `vrg-vm session`, read from a host config file.
vergil-claude-plugin's doctrine gains the delegate path beside today's path.

**Tech Stack:** paad's Makefile checks and Python 3 scripts (stdlib only, with
`--self-test`); vergil-tooling Python 3 with pytest; Markdown doctrine.

**Spec:** [`epics/699-paad-unattended-mode/spec.md`](spec.md)

## Global Constraints

- **paad work follows paad's workflow, not vergil's** (spec 7). All skill and
  agent edits go in `preview/paad/`, never `plugins/paad/`. `make export &&
  make test` passes before each commit. A changelog entry of 1–3 lines goes
  under `[Unreleased]`. No version bump, no `make release`, no `make promote`:
  releases are the author's.
- **Nothing vergil-specific in the paad repo:** no vergil names, no vergil
  configuration, no change to paad's tooling for vergil's sake. vergil
  helpers live outside it.
- **paad's vocabulary only** in everything paad reads (the contract, the
  unattended sections, and vergil's dispatch prompt): spec, plan, intent and
  action documents, base, diff, finding, report. vergil concepts appear only
  as concrete identifiers (an issue ref, a path).
- **Interactive behaviour is unchanged.** Every new instruction applies only
  under `paad-delegate`.
- **Claude Code only.** Every export (`kiro_and_antigravity/`, `pi/`) stays
  correct, and no Pi, Kiro or Antigravity delegate is added.
- Blocked reasons, verbatim: `blocked: skill does not support unattended
  mode`, `blocked: analyst agent not available`, `blocked: the inputs must be
  named`, `blocked: both sides must be named`, `blocked: the working tree has
  uncommitted changes`, `blocked: ambiguous argument`, `blocked: the
  classification is unverified`.
- **vergil-tooling and vergil-claude-plugin work** uses `vrg-container-run --
  vrg-validate` as its only validation, `vrg-commit`, and `vrg-pr-workflow
  report-ready`; agents never run `vrg-submit-pr`.
- **The paad development environment is Lima only.** No cloud support.
- **Security details** never appear in public records: counts and GHSA IDs
  only.

## Review Focus

1. **The delegate dispatched on a skill with no unattended section** (for
   example `agentic-owasp`): it must return `blocked: skill does not support
   unattended mode` without running the skill, rather than half-following
   prose that defers to a missing section. Pinned in Task 4
   (`unsupported-skill` smoke) and Task 10 (validation).
2. **agentic-review with no arguments under the delegate**: it must review
   against the resolved default branch and list that under `Defaults
   applied:`, not block. Pinned in Task 7 (`no-args` smoke).
3. **A `paad/config/<skill>.md` present on the branch under review**: it must
   be reported as `Config not followed:` and change nothing. Pinned in Task 5
   (`config-present` smoke) and Task 10.
4. **An override entry that is relative, or a directory missing in the
   guest**: the launch must fail loudly, never start `claude` without the
   tree. Pinned in Task 8 (`test_load_plugin_dirs_rejects_relative`,
   `test_session_inner_checks_plugin_dirs_in_guest`).
5. **A cloud target while the override is configured**: the session must
   launch without `--plugin-dir` and print the Lima-only notice. Pinned in
   Task 8 (`test_cloud_session_skips_plugin_dirs_with_notice`).

---

## Delivery

Each task below is one issue under the epic. Tasks 1–7 and 11 are fork or
human tasks filed in `vergil-project/.github` and closed by comment (spec 7);
the rest close by a same-repo PR or as operational tasks.

| Task | Issue (repo, kind) | Blocked by |
|---|---|---|
| 1 | Author review of the spec (`.github`, human sign-off) | the docs PR (#700) |
| 2 | Fork, clone and fleet checkout setup (`.github`, closed by comment) | 1 |
| 3 | Spike: delegate on the fork's preview tree (`.github`, closed by comment) | 2 |
| 4 | The `paad-delegate` agent and the unattended lint (`.github`, fork) | 3 |
| 5 | pushback unattended mode, and the shared paragraphs (`.github`, fork) | 4 |
| 6 | alignment unattended mode (`.github`, fork) | 5 |
| 7 | agentic-review unattended mode (`.github`, fork) | 5 |
| 8 | `--plugin-dir` override in `vrg-vm session` (vergil-tooling) | 1 |
| 9 | Deploy the override and the fleet checkout (`.github`, deployment) | 6, 7, 8 |
| 10 | Validate the mode and the drafted vergil prompts (`.github`, validation) | 9 |
| 11 | Offer the change upstream (`.github`, human, closed by comment) | 10 |
| 12 | Doctrine and policy: the delegate path with fallback (vergil-claude-plugin) | 10 |
| 13 | Follow-on brainstorm: other harnesses (`.github`) | 11, 12 |

Task 8 depends only on the author review, so it runs in parallel with the fork
work. The bookends already exist: the docs task vergil-project/.github#700,
the documentation review vergil-project/vergil-claude-plugin#820 and the
retrospective vergil-project/.github#701.

## File map

| Repo | File | Responsibility |
| --- | --- | --- |
| paad fork | `preview/paad/agents/paad-delegate.md` | The contract (Task 4) |
| paad fork | `scripts/check_unattended.py` | The new lint, with `--self-test` (Tasks 4, 5) |
| paad fork | `Makefile` | `check-unattended` in `tree-checks`; its self-test in `self-tests` (Task 4) |
| paad fork | `preview/paad/skills/*/SKILL.md` | Conditional configuration paragraph in every skill (Task 5) |
| paad fork | `scripts/check_security.py` | The conditional shared security paragraph (Task 5) |
| paad fork | `preview/paad/skills/pushback/SKILL.md` | Unattended paragraph and section (Task 5) |
| paad fork | `preview/paad/skills/alignment/SKILL.md` | Unattended paragraph and section (Task 6) |
| paad fork | `preview/paad/skills/agentic-review/SKILL.md` | Unattended paragraph and section, `Intent:` input (Task 7) |
| paad fork | `preview/paad/skills/paad-help/SKILL.md` | The mode documented, Claude Code only (Tasks 4–7) |
| paad fork | `scripts/convert_skills.py` | Exports stay correct with the new agent and paragraph (Task 4) |
| paad fork | `CHANGELOG.md` | `[Unreleased]` entry (Task 4, extended by 5–7) |
| vergil-tooling | `src/vergil_tooling/lib/plugin_dirs.py` | Load the override config (Task 8) |
| vergil-tooling | `src/vergil_tooling/bin/vrg_vm.py` | Pass `--plugin-dir` on Lima; notice on cloud (Task 8) |
| vergil-tooling | `tests/vergil_tooling/test_plugin_dirs.py`, `tests/vergil_tooling/test_vrg_vm.py` | Tests (Task 8) |
| vergil-tooling | `docs/site/docs/reference/cli-tools-overview.md` | The override documented (Task 8) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` | Delegate path, fallback, common core (Task 12) |
| vergil-claude-plugin | `docs/paad-policy.md` | Fork, override, deviations retired (Task 12) |
| vergil-claude-plugin | `skills/epic-create/SKILL.md`, `skills/issue-implement/SKILL.md` | Dispatch per the doctrine (Task 12) |

## The structured result (fixed here, used by Tasks 4–7, 10 and 12)

Spec 9 leaves the syntax to the builder. Later tasks parse it, so this plan
fixes it. Lines in order; `<…>` are values:

```text
Running paad:<skill> v<version>
Config: <path>                       (only when a caller-named file was followed)
Config not followed: <path>          (one line per file found and not followed)
Status: complete | blocked: <reason>
Agent types dispatched: <type>[, <type>…] | none
Defaults applied: none | then one "- <interaction point>: <default>" line each
Findings: <n>
### F<i> <severity> — <location>
<the skill's native finding fields, one "<Field>: <value>" line each>
Discards: <k>
- <one line each>
<skill-specific fields>
=== paad security section ===
Security: <k>
### S<i> <severity> — <location>
<native finding fields>
Security discards: <k>
=== end paad security section ===
Files written: none
```

Native finding fields: pushback and alignment `Claim:`, `Options:` (ranked
list), `Recommendation:`; agentic-review `Symbol:`, `Excerpt:`, `Fix:`,
`Confidence:`, `Scope:` (`in scope`, `out-of-scope bug`, `out-of-scope
addition`), `Bug class:`. agentic-review's skill-specific fields are its
existing counts line, `Agents dispatched:` (lenses), `Intent sources:` and
`Verifier warnings:`. A blocked result stops after `Agent types dispatched:`
and ends with `Files written: none`.

---

### Task 1: Author review of the spec (human sign-off)

**Files:** none (issue comments; a follow-up docs PR in `.github` if the
feedback changes the spec or plan).

- [ ] **Step 1:** The human sends Curtis Poe the published spec and the
  analysis (planning issue vergil-project/.github#698), with the §4.10
  conflicts list and the §4.9 proposals as the agenda.
- [ ] **Step 2:** Record a paraphrased summary of the author's feedback on
  this issue: what he accepts, what he wants changed, and the PR split he
  prefers.
- [ ] **Step 3:** If the feedback changes the design, open a follow-up docs
  task under the epic and fold the changes into `spec.md` and `plan.md` in
  its PR. Tasks 2–7 wait for that PR to merge.
- [ ] **Step 4:** The human signs off in a comment (`Author review: signed
  off`), and closes the issue.

### Task 2: Fork, clone and fleet checkout setup

**Files:** none in any repo. Records go on the issue.

**Interfaces:**
- Produces: the fork `wphillipmoore/paad`; the clone
  `~/dev/projects/wphillipmoore/paad` (`origin` the fork, `upstream`
  `Ovid/paad`, pull only); the fleet worktree
  `~/dev/projects/wphillipmoore/paad-fleet` on local branch `fleet`, created
  with `git worktree add` from the clone. Later tasks use these paths.

- [ ] **Step 1 (human):** Fork `Ovid/paad` to `wphillipmoore/paad` on GitHub.
- [ ] **Step 2:** Clone it to `~/dev/projects/wphillipmoore/paad`, add
  `upstream` (`https://github.com/Ovid/paad.git`), and set `upstream`'s push
  URL to `DISABLED` so nothing can be pushed there.
- [ ] **Step 3:** Settle how agents run git in the clone (spec 7). From
  inside the clone, run `vrg-git status`, `vrg-git fetch upstream` and
  `vrg-git log -1`. Record which work. If the hook
  guard or `vrg-git` refuses a non-vergil repo, record the refusal verbatim
  and the rule the fork tasks follow instead (the human runs those commands
  with `!`).
- [ ] **Step 4:** Create the fleet worktree on branch `fleet` at `main`, and
  record its path and commit.
- [ ] **Step 5:** Verify: `make export && make test` passes in the clone on
  `main` (paad's own baseline). Record the output's last line (`All checks
  passed.`). Close the issue with the comment.

### Task 3: Spike: the delegate on the fork's preview tree

Settles the facts later tasks rely on before any skill text is written.

**Files:** a throwaway branch `spike/delegate` in the clone, never pushed.

- [ ] **Step 1:** On `spike/delegate`, add a minimal
  `preview/paad/agents/paad-delegate.md` (tools `Read, Grep, Glob, Bash,
  Skill, Agent`; body: run the named skill, return its final answer
  unchanged).
- [ ] **Step 2:** From a logged-in session started with `claude --plugin-dir
  ~/dev/projects/wphillipmoore/paad/preview/paad`, dispatch
  `paad:paad-delegate` to run `paad:rethink` on a one-line question. Record:
  the delegate loads; it invokes the skill; the skill's `paad-analyst`
  dispatches work; the announce line reads `v<X.Y.Z>-preview`.
- [ ] **Step 3:** Record which parts of the plugin a running session re-reads
  on `/reload-plugins` (agent definitions, skills) by editing the delegate's
  body and reloading.
- [ ] **Step 4:** Record whether `make check-export-dryrun
  TREE=preview/paad` passes with the new agent file present, and if not, the
  exact error. Task 4 handles it.
- [ ] **Step 5:** Delete the branch. Close the issue with the recorded facts.

### Task 4: The `paad-delegate` agent and the unattended lint

**Files:**
- Create: `preview/paad/agents/paad-delegate.md`
- Create: `scripts/check_unattended.py`
- Modify: `Makefile` (`self-tests`, `tree-checks`, new `check-unattended`
  target)
- Modify: `scripts/convert_skills.py` (only if Task 3 step 4 failed)
- Modify: `preview/paad/skills/paad-help/SKILL.md`, `CHANGELOG.md`

**Interfaces:**
- Produces: the agent `paad:paad-delegate`; the result format above;
  `check_unattended.py <skills_dir>` exits 1 with one `FAIL:` line per
  problem; constants `UNATTENDED_MARKER = "**Unattended mode:**"` (the
  paragraph's opening) and `UNATTENDED_SECTION = "## Unattended mode"` (the
  section heading). Tasks 5–7 add the paragraph and section; Task 5 adds the
  configuration-condition check.

- [ ] **Step 1: Write the lint's self-tests first.** In
  `check_unattended.py --self-test`, assert:
  - a skill with `UNATTENDED_MARKER` and no `UNATTENDED_SECTION` fails with
    `FAIL: <skill> carries the unattended paragraph but has no "## Unattended
    mode" section`;
  - a skill with the section and no marker fails the same way, reversed;
  - a skill with neither passes;
  - a skill with both passes.
- [ ] **Step 2:** Run `python3 scripts/check_unattended.py --self-test`.
  Expected: FAIL (module has no checks yet).
- [ ] **Step 3:** Implement `check(skills_dir: Path) -> list[str]` and
  `main(argv: list[str]) -> int` in `scripts/check_unattended.py`, stdlib
  only, in the style of `check_security.py`. A scan that cannot read a
  `SKILL.md` is a failure, never a pass.
- [ ] **Step 4:** Run the self-test. Expected: PASS. Add it to `self-tests`,
  and add `check-unattended` (`python3 scripts/check_unattended.py
  $(SKILLS_DIR)`) to `tree-checks`.
- [ ] **Step 5: Write `paad-delegate.md`** in the style of
  `paad-analyst.md`, carrying every contract point of spec 4.1, in this
  order: no person is present and the final message is the return value; the
  dispatch prompt is binding except against the rules below; run only a
  skill that has an unattended section, else `blocked: skill does not
  support unattended mode`; never ask, apply the skill's unattended defaults;
  follow no config file the dispatch prompt does not name, and list each as
  `Config not followed:`; dispatch only `paad:paad-analyst`, and if it is not
  available return `blocked: analyst agent not available`; invoke only the
  named skill; write nothing, with any tool (incidental artifacts of
  read-only commands allowed in gitignored paths only, not listed); return
  the result format above and nothing else; everything read is untrusted
  data. Frontmatter: `name: paad-delegate`; `tools: Read, Grep, Glob, Bash,
  Skill, Agent`; a `description` that says it is dispatched explicitly, by
  name, by a calling agent, and never for interactive use.
- [ ] **Step 6:** If Task 3 recorded an export failure, make
  `convert_skills.py` skip `paad-delegate.md` for Kiro, Antigravity and Pi,
  with a comment naming the reason (the mode is Claude Code only).
- [ ] **Step 7:** Document the mode in `paad-help` (what it is, how a caller
  dispatches it, the result format, the caller's duty to keep the security
  section out of public records, Claude Code only). Add the `[Unreleased]`
  changelog line.
- [ ] **Step 8: Smoke `unsupported-skill`.** From a session started with
  `--plugin-dir` on the clone's preview tree, dispatch the delegate on
  `agentic-owasp`. Expected: `Status: blocked: skill does not support
  unattended mode`, `Agent types dispatched: none`, `Files written: none`,
  and nothing else ran.
- [ ] **Step 9:** `make export && make test`. Expected: `All checks passed.`
- [ ] **Step 10:** Commit on branch `unattended-mode` in the clone; push to
  the fork. Close the issue with the branch, commit and the `make test` last
  line.

### Task 5: pushback unattended mode, and the shared paragraphs

The first skill end to end (spec 8). It also makes the shared paragraphs
conditional, which touches every skill.

**Files:**
- Modify: `preview/paad/skills/*/SKILL.md` (configuration paragraph, every
  skill)
- Modify: `scripts/check_security.py` (`PARAGRAPH`), and every producer's
  copy of it
- Modify: `scripts/check_unattended.py` (configuration-condition check)
- Modify: `preview/paad/skills/pushback/SKILL.md`, `paad-help`,
  `CHANGELOG.md`

**Interfaces:**
- Consumes: Task 4's constants and result format.
- Produces: `CONFIG_DELEGATE_CONDITION`, the exact sentence every
  configuration paragraph carries (below); the pattern of an unattended
  section that Tasks 6 and 7 copy.

- [ ] **Step 1: Extend the self-test first.** Assert that a skill whose
  configuration paragraph lacks `CONFIG_DELEGATE_CONDITION` fails with
  `FAIL: <skill>'s configuration paragraph lacks the delegate condition`.
  Run it. Expected: FAIL.
- [ ] **Step 2:** Add to the configuration paragraph of every skill, after
  its first sentence, this sentence verbatim, and set
  `CONFIG_DELEGATE_CONDITION` to it: `When this skill runs under
  paad-delegate, read only a config file the dispatch prompt names, and list
  each other config file found as not followed.` Implement the check. Run
  the self-test and `make tree-checks TREE=preview/paad`. Expected: PASS.
- [ ] **Step 3:** Make the shared security paragraph conditional: under
  `paad-delegate`, it defers to the skill's unattended section, and writes
  nothing under `paad/security/`. Update `PARAGRAPH` in `check_security.py`
  and every producer's copy together. `make check-security` must pass.
  Interactive text and behaviour are otherwise unchanged.
- [ ] **Step 4: Add pushback's unattended paragraph** (opening with
  `UNATTENDED_MARKER`, next to the announce, configuration and security
  paragraphs) **and its `## Unattended mode` section**, listing:
  - every interaction point with its default, from spec 4.3's pushback,
    "pushback, alignment" and "all three" rows;
  - the native finding fields (`Claim:`, `Options:`, `Recommendation:`);
  - the steps skipped: every edit, the report file, and every
    `paad/security/` write;
  - security classification under the delegate with paad's shared
    definition, matches going to the security section (spec 4.2).

  The paragraph is worded so it still reads correctly where no delegate
  exists (an `npx skills`, Kiro, Antigravity or Pi install): it applies only
  "when this skill runs under paad-delegate".
- [ ] **Step 5:** Show the unattended branch in pushback's digraph where it
  changes the flow. `make check-digraphs` must pass.
- [ ] **Step 6: Smoke `pushback-plain` and `config-present`.** Under the
  delegate, run pushback on a small fixture spec in a scratch git repo, once
  plain and once with `paad/config/pushback.md` present. Expected: every
  result line in order; `Status: complete`; findings carry `Claim:`,
  `Options:`, `Recommendation:`; the second run lists `Config not followed:
  paad/config/pushback.md` and the same findings; `Files written: none`;
  `git status --porcelain` empty in the scratch repo.
- [ ] **Step 7: Smoke `pushback-interactive`.** Run pushback interactively
  from the preview tree. Expected: it asks, walks findings, and offers the
  update or the report, as in 2.0.2.
- [ ] **Step 8:** Update `paad-help` and the changelog line. `make export &&
  make test`. Expected: `All checks passed.` Commit, push, close with branch,
  commit and `make test` last line.

### Task 6: alignment unattended mode

**Files:** Modify `preview/paad/skills/alignment/SKILL.md`, `paad-help`,
`CHANGELOG.md`.

**Interfaces:** Consumes Task 5's section pattern and constants.

- [ ] **Step 1:** Add alignment's unattended paragraph and section, as in Task
  5 step 4, from spec 4.3's alignment, "pushback, alignment" and "all three"
  rows. alignment's section carries paad's shared security definition and
  routes matches to the security section. Interactive alignment keeps no
  security routing (that is a proposal, spec 4.9). Do not register alignment
  in `PRODUCERS`.
- [ ] **Step 2:** Update the digraph. `make check-digraphs` passes.
- [ ] **Step 3: Smoke `alignment-one-side` and `alignment-tdd`.** Under the
  delegate: with only a spec named, expected `Status: blocked: both sides must
  be named`; with a spec and a plan whose one code task is not
  red/green/refactor, expected a `Minor` finding for that task and the plan
  file unchanged.
- [ ] **Step 4:** Interactive smoke as in Task 5 step 7. `make export && make
  test`. Commit, push, close as in Task 5.

### Task 7: agentic-review unattended mode

**Files:** Modify `preview/paad/skills/agentic-review/SKILL.md` (and its
`references/` only if a dispatch prompt there must name the delegate rule),
`paad-help`, `CHANGELOG.md`.

**Interfaces:** Consumes Task 5's pattern. Produces the `Intent:` input: a
line `Intent: <path>[, <path>…]` in the dispatch prompt; Spec Compliance reads
those files as explicit spec files ahead of its other sources.

- [ ] **Step 1:** Add the unattended paragraph and section from spec 4.3's
  agentic-review and "all three" rows: no arguments reviews against the
  default branch and is listed under `Defaults applied:`; uncommitted changes
  and an ambiguous argument block; no report file, backlog read or backlog
  write; large diffs use the full 12-dispatch form, and only an actual
  dispatch failure blocks; a missing verifier token blocks. Native fields as
  in the result format above.
- [ ] **Step 2:** Under the delegate, the "If the analyst agent is not
  available … dispatch a general-purpose subagent instead" paragraph does not
  apply (spec 4.10): the section says so.
- [ ] **Step 3:** Document the `Intent:` input in the section and in
  `paad-help`. Interactive arguments are unchanged.
- [ ] **Step 4:** Update the digraph. `make check-digraphs` and `make
  check-dispatch-sites` pass.
- [ ] **Step 5: Smoke `no-args`, `intent` and `dirty`.** In a scratch repo with
  a branch carrying one known bug: no arguments gives `Status: complete`, the
  bug found, and `- no arguments: reviewed against the default branch` under
  `Defaults applied:`; `Intent: spec.md` appears in `Intent sources:`; with an
  uncommitted change, `Status: blocked: the working tree has uncommitted
  changes`. In every run `paad/code-reviews/` and `paad/security/` are not
  created.
- [ ] **Step 6:** Interactive smoke. `make export && make test`. Commit, push,
  close as in Task 5.

### Task 8: `--plugin-dir` override in `vrg-vm session`

**Files:**
- Create: `src/vergil_tooling/lib/plugin_dirs.py`
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`_session_inner`,
  `_cmd_session`, `_cloud_session`)
- Test: `tests/vergil_tooling/test_plugin_dirs.py`,
  `tests/vergil_tooling/test_vrg_vm.py`
- Modify: `docs/site/docs/reference/cli-tools-overview.md`

**Interfaces:**
- Produces: `default_plugin_dirs_path() -> Path` (`~/.config/vergil/plugin-dirs.toml`);
  `load_plugin_dirs(path: Path | None = None) -> tuple[str, ...]`;
  `class PluginDirsError(ValueError)`; `_session_inner(args, identity_name,
  rel_path, model, plugin_dirs: tuple[str, ...] = ()) -> str`.
- File format: `plugin_dirs = ["/abs/path", …]`, the only key.

- [ ] **Step 1: Write the failing tests** in `test_plugin_dirs.py`:
  - `test_load_plugin_dirs_missing_file_is_empty`: no file gives `()`;
  - `test_load_plugin_dirs_reads_list`: `plugin_dirs = ["/a/preview/paad"]`
    gives `("/a/preview/paad",)`;
  - `test_load_plugin_dirs_rejects_relative`: `["preview/paad"]` raises
    `PluginDirsError` naming the entry;
  - `test_load_plugin_dirs_rejects_malformed`: invalid TOML, a non-list, a
    non-string entry, or an unknown key each raise `PluginDirsError`.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`. Expected: the new
  tests FAIL (module missing).
- [ ] **Step 3:** Implement `lib/plugin_dirs.py` in the style of
  `lib/host_pins.py` (`tomllib`, a loud error, a missing file meaning
  "none").
- [ ] **Step 4: Write the failing session tests** in `test_vrg_vm.py`:
  - `test_session_inner_adds_plugin_dirs`: with `plugin_dirs=("/a",)`, the
    resolver's passthrough contains `--plugin-dir /a`, after `--model`;
  - `test_session_inner_checks_plugin_dirs_in_guest`: the inner command
    tests each directory with `[ -d … ]` before `exec`, printing `ERROR:
    plugin directory not found in the guest: /a` and exiting 1;
  - `test_session_inner_raw_override_ignores_plugin_dirs`: a `-- bash`
    override gets no flag;
  - `test_cmd_session_loads_plugin_dirs`: the Lima path passes the loaded
    tuple, and a `PluginDirsError` returns 1 with the message on stderr;
  - `test_cloud_session_skips_plugin_dirs_with_notice`: with entries
    configured, the cloud path passes `()` and prints, verbatim, `NOTE: the
    --plugin-dir override is supported only on macOS with Lima VMs; this
    cloud session runs the installed plugins.`
- [ ] **Step 5:** Implement. Run `vrg-container-run -- vrg-validate
  --all-versions`. Expected: PASS, with 100% branch coverage.
- [ ] **Step 6:** Document the file, its format, the Lima-only rule and how to
  turn it off in `cli-tools-overview.md` under `vrg-vm session`.
- [ ] **Step 7:** Commit with `vrg-commit`, hand off with `vrg-pr-workflow
  report-ready`.

### Task 9: Deploy the override and the fleet checkout (deployment)

**Preconditions:** Tasks 6, 7 and 8 closed; a vergil-tooling release
carrying Task 8 is installed on the host (human-attested).

- [ ] **Step 1:** In the fleet worktree, move `fleet` to the
  `unattended-mode` branch's head; record the commit.
- [ ] **Step 2:** Write `~/.config/vergil/plugin-dirs.toml` with
  `plugin_dirs =
  ["/Users/pmoore/dev/projects/wphillipmoore/paad-fleet/preview/paad"]`.
- [ ] **Step 3:** Start a Lima session with `vrg-vm session`. In it, confirm
  `paad:paad-delegate` is listed as an agent type and `paad:paad-help`
  announces `-preview`.
- [ ] **Step 4:** Temporarily point the file at a missing directory; confirm
  the launch fails with the guest error; restore it.
- [ ] **Step 5:** Record `Outcome: SUCCESS` with the evidence and close.

### Task 10: Validate the mode and the drafted vergil prompts (validation)

**Preconditions:** Task 9 closed.

- [ ] **Step 1: Draft the prompts** for both modes per spec 4.8: the skill and
  arguments; document roles (mode 1) or `Intent:` (mode 2); the working
  directory; the review-the-whole-artifact sentence; the no-anchoring rule by
  concrete identifier; the security self-test and widened definition. Review
  the drafts for vergil vocabulary (none may remain except as identifiers).
  Record both drafts on the issue.
- [ ] **Step 2:** Run every set in spec 5 with the drafts, from Lima sessions
  using the override: mode 1 plain and seeded runs; mode 2 on two recorded
  commits, each twice; planted dispatch (judged from the session
  transcripts' subagent spawns, not the result line); unsupported skill;
  seeded access-control gap; planted config; one interactive run of each
  changed skill.
- [ ] **Step 3:** Judge each against spec 5's "cleanly". Record per run: the
  status, the field parse, the fleet checkout commit, `Files written:`, and
  the before and after state of the worktree, its ignored `paad/`
  directories, the starting directory's `paad/` and the temporary
  directories.
- [ ] **Step 4:** Record `Outcome: SUCCESS` or `Outcome: FAILURE`. On failure,
  file fix tasks on the fork under the epic and leave this open.

### Task 11: Offer the change upstream (human)

- [ ] **Step 1:** Open the PR(s) from fork branches to `Ovid/paad`, split as
  Task 1 recorded, with Task 10's evidence and the two spec 4.9 proposals.
- [ ] **Step 2:** Record the PR links on the issue and close it. Upstream's
  decision is the author's; no later task waits on it.

### Task 12: Doctrine and policy: the delegate path with fallback

**Files:** Modify `docs/paad-review-doctrine.md`, `docs/paad-policy.md`,
`skills/epic-create/SKILL.md`, `skills/issue-implement/SKILL.md` in
vergil-claude-plugin.

- [ ] **Step 1:** Restructure the doctrine per spec 4.8: the shared procedure
  (output checks, screening, security handling, the no-anchoring rule) moves
  into the common core (§1); each mode's section keeps only what differs.
- [ ] **Step 2:** Add the delegate path with Task 10's validated prompts, and
  the selection rule: use it when `paad:paad-delegate` is loaded, otherwise
  run today's path and record the deviation in the round comment or task
  record. The output checks for the delegate path are spec 4.8's list,
  including `-preview` as an expected announce and `Agent types dispatched:
  paad:paad-analyst`.
- [ ] **Step 3:** Add mode 2 screening of every ordinary finding and discard
  before the findings comment is posted, on both paths.
- [ ] **Step 4:** Keep every item under spec 4.8's "These go away" in the
  fallback path, marked for retirement once a released paad carries the
  mode.
- [ ] **Step 5:** paad-policy records the override, how it is turned on and
  off, the Lima-only rule, the fallback, and the deviations register rows the
  delegate path retires.
- [ ] **Step 6:** Point the two skills' dispatch steps at the doctrine's
  selection rule. Run `vrg-container-run -- vrg-validate`. Commit and hand
  off with `report-ready`.

### Task 13: Follow-on brainstorm: other harnesses

- [ ] **Step 1:** Run `superpowers:brainstorming` on whether unattended mode
  applies to Kiro, Antigravity or Pi, given each harness's subagent support.
- [ ] **Step 2:** Record the outcome on the issue (a new planning issue, or
  "not worth doing" with the reason) for the retrospective's §5. Close it.
