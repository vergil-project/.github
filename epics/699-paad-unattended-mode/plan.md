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
definition (the contract), one unattended paragraph (carrying the overrides
of paad's shared configuration and security paragraphs, which stay
unchanged) and one unattended section per supported skill, and a
`check_unattended.py` lint. vergil-tooling gains a
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
- **paad's shared paragraphs and checks are not changed.** The configuration
  and security paragraphs stay as they are in every skill, and so do
  `check-config` and `scripts/check_security.py`. The delegate's overrides
  live in each supported skill's own unattended paragraph. `make test` checks
  both trees, and `plugins/paad` can change only at a release, so a change to
  a shared paragraph could never pass on the shipped tree.
- **No skill outside `PRODUCERS` and `READERS` names `paad/security/`.** New
  text says "writes no security file", not the path.
- **Paths are absolute:** `/Users/pmoore/dev/projects/wphillipmoore/paad` (the
  clone) and `/Users/pmoore/dev/projects/wphillipmoore/paad-fleet` (the fleet
  worktree). They are the same on the Mac and in the Lima VMs; `~` is not.
- **Sessions are the human's.** Starting a `claude --plugin-dir` session,
  `/reload-plugins` and answering interactive runs are human steps. The agent
  prepares the fixtures and the exact prompt, and checks the pasted results
  against the expected values. Steps marked **(human)** below.
- **Claude Code only.** Every export (`kiro_and_antigravity/`, `pi/`) stays
  correct, and no Pi, Kiro or Antigravity delegate is added.
- Fixed blocked reasons, verbatim: `blocked: skill does not support
  unattended mode`, `blocked: analyst agent not available`, `blocked: the
  inputs must be named`, `blocked: both sides must be named`, `blocked: the
  working tree has uncommitted changes`, `blocked: ambiguous argument`,
  `blocked: the classification is unverified`. Any other stop at which a
  skill cannot proceed safely returns `blocked: <reason>` in its own words,
  never a guess (spec 4.1). Callers treat every `blocked:` alike.
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
   prose that defers to a missing section. Pinned in Task 3
   (`unsupported-skill` smoke) and Task 9 (validation).
2. **agentic-review with no arguments under the delegate**: it must review
   against the resolved default branch and list that under `Defaults
   applied:`, not block. Pinned in Task 6 (`no-args` smoke).
3. **A `paad/config/<skill>.md` present on the branch under review**: it must
   be reported as `Config not followed:` and change nothing. Pinned in Task 4
   (`config-present` smoke, step 5) and Task 9.
4. **An override entry that is relative, or a directory missing in the
   guest**: the launch must fail loudly, never start `claude` without the
   tree. Pinned in Task 7 (`test_load_plugin_dirs_rejects_relative`,
   `test_session_inner_checks_plugin_dirs_in_guest`).
5. **A cloud target while the override is configured**: the session must
   launch without `--plugin-dir` and print the Lima-only notice. Pinned in
   Task 7 (`test_cloud_session_skips_plugin_dirs_with_notice`).

---

## Delivery

Each task below is one issue under the epic. Tasks 1–6, 11 and 12 are fork,
brainstorm or human tasks filed in `vergil-project/.github` and closed by
comment (spec 7); the rest close by a same-repo PR or as operational tasks.

| Task | Issue (repo, kind) | Blocked by |
|---|---|---|
| 1 | Recreate the fork, the clone and the fleet checkout (`.github`, closed by comment) | — |
| 2 | Spike: delegate on the fork's preview tree (`.github`, closed by comment) | 1 |
| 3 | The `paad-delegate` agent and the unattended lint (`.github`, fork) | 2 |
| 4 | pushback unattended mode (`.github`, fork) | 3 |
| 5 | alignment unattended mode (`.github`, fork) | 4 |
| 6 | agentic-review unattended mode (`.github`, fork) | 4 |
| 7 | `--plugin-dir` override in `vrg-vm session` (vergil-tooling) | — |
| 8 | Deploy the override and the fleet checkout (`.github`, deployment) | 5, 6, 7 |
| 9 | Validate the mode and the drafted vergil prompts (`.github`, validation) | 8 |
| 10 | Doctrine and policy: the delegate path with fallback (vergil-claude-plugin) | 9 |
| 11 | Follow-on brainstorm: other harnesses (`.github`) | 9 |
| 12 | Decide: offer the change upstream, or try a different idea (`.github`, human) | 10, 11 |

Task 7 runs in parallel with the fork work. Nothing waits on the paad author:
the human shares progress with him informally, and the upstream conversation
is Task 12, the last task before the retrospective (spec 4.9). The bookends
already exist: the docs task vergil-project/.github#700, the documentation
review vergil-project/vergil-claude-plugin#820 and the retrospective
vergil-project/.github#701.

## File map

| Repo | File | Responsibility |
| --- | --- | --- |
| paad fork | `preview/paad/agents/paad-delegate.md` | The contract (Task 3) |
| paad fork | `scripts/check_unattended.py` | The new lint, with `--self-test` (Tasks 3, 4) |
| paad fork | `Makefile` | `check-unattended` in `tree-checks`; its self-test in `self-tests` (Task 3) |
| paad fork | `preview/paad/skills/pushback/SKILL.md` | Unattended paragraph and section (Task 4) |
| paad fork | `preview/paad/skills/alignment/SKILL.md` | Unattended paragraph and section (Task 5) |
| paad fork | `preview/paad/skills/agentic-review/SKILL.md` | Unattended paragraph and section, `Intent:` input (Task 6) |
| paad fork | `preview/paad/skills/paad-help/SKILL.md` | The mode documented, Claude Code only (Tasks 3–6) |
| paad fork | `scripts/convert_skills.py` | Exports stay correct with the new agent and sections (Tasks 3–6, only if a dry run fails) |
| paad fork | `CHANGELOG.md` | `[Unreleased]` entry (Task 3, extended by 4–6) |
| vergil-tooling | `src/vergil_tooling/lib/plugin_dirs.py` | Load the override config (Task 7) |
| vergil-tooling | `src/vergil_tooling/bin/vrg_vm.py` | Pass `--plugin-dir` on Lima; notice on cloud (Task 7) |
| vergil-tooling | `tests/vergil_tooling/test_plugin_dirs.py`, `tests/vergil_tooling/test_vrg_vm.py` | Tests (Task 7) |
| vergil-tooling | `docs/site/docs/reference/cli-tools-overview.md` | The override documented (Task 7) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` | Delegate path, fallback, common core (Task 10) |
| vergil-claude-plugin | `docs/paad-policy.md` | Fork, override, deviations retired (Task 10) |
| vergil-claude-plugin | `skills/epic-create/SKILL.md`, `skills/issue-implement/SKILL.md` | Dispatch per the doctrine (Task 10) |

## The structured result (fixed here, used by Tasks 3–6, 9 and 10)

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
existing counts line, `Agents dispatched:` (lenses), `Intent sources
consulted:` and `Verifier warnings:`, under paad's own names. A blocked result
stops after `Agent types dispatched:` and ends with `Files written: none`.

`Agent types dispatched:` per skill: pushback and alignment dispatch no
subagents, so `none` is expected and any listed type must be
`paad:paad-analyst`; agentic-review must list `paad:paad-analyst`, because
its six lenses dispatch, and nothing else.

---

### Task 1: Recreate the fork, the clone and the fleet checkout

An earlier fork and clone exist from April 2026: a stale `main` with no
`preview/` tree, carrying two non-upstream commits (`2781839`, a vergil
`st-config.toml`, and its revert `5a9b72d`). They are deleted and recreated
so the fork starts equal to upstream.

**Files:** none in any repo. Records go on the issue.

**Interfaces:**
- Produces: the fork `wphillipmoore/paad`; the clone
  `/Users/pmoore/dev/projects/wphillipmoore/paad` (`origin` the fork,
  `upstream` `Ovid/paad`, pull only); the fleet worktree
  `/Users/pmoore/dev/projects/wphillipmoore/paad-fleet` on local branch
  `fleet`, created with `git worktree add` from the clone.

- [ ] **Step 1 (human):** Delete the GitHub repo `wphillipmoore/paad` and the
  folder `/Users/pmoore/dev/projects/wphillipmoore/paad`. Deleting a repo
  cannot be undone.
- [ ] **Step 2 (human):** Fork `Ovid/paad` to `wphillipmoore/paad` on GitHub.
- [ ] **Step 3:** Clone the fork to
  `/Users/pmoore/dev/projects/wphillipmoore/paad`. **(human)** Add `upstream`
  (`https://github.com/Ovid/paad.git`) and set its push URL to `DISABLED`
  with `!`, since `vrg-git remote` is denied.
- [ ] **Step 4:** Settle how agents run git in the clone (spec 7). From
  inside it, run `vrg-git status`, `vrg-git fetch upstream` and `vrg-git log
  -1`. Record which work. Record any refusal verbatim, and the rule the fork
  tasks follow instead (the human runs those commands with `!`).
- [ ] **Step 5:** Record that `main` equals `upstream/main` (same commit).
- [ ] **Step 6:** Create the fleet worktree on branch `fleet` at `main`, and
  record its path and commit.
- [ ] **Step 7:** Verify: `make export && make test` passes in the clone on
  `main` (paad's own baseline). Record the output's last line (`All checks
  passed.`). Close the issue with the comment.

### Task 2: Spike: the delegate on the fork's preview tree

Settles the facts later tasks rely on before any skill text is written.

**Files:** a throwaway branch `spike/delegate` in the clone, never pushed.

- [ ] **Step 1:** On `spike/delegate`, add a minimal
  `preview/paad/agents/paad-delegate.md` (tools `Read, Grep, Glob, Bash,
  Skill, Agent`; body: run the named skill, return its final answer
  unchanged).
- [ ] **Step 2 (human):** In a session started with `claude --plugin-dir
  /Users/pmoore/dev/projects/wphillipmoore/paad/preview/paad`, dispatch
  `paad:paad-delegate` to run `paad:rethink` on a one-line question, with the
  prompt the agent prepared. The agent records from the pasted result: the
  delegate loads; it invokes the skill; the skill's `paad-analyst` dispatches
  work; the announce line reads `v<X.Y.Z>-preview`.
- [ ] **Step 3 (human):** Edit the delegate's body, run `/reload-plugins`, and
  dispatch again. The agent records whether agent definitions and skills are
  re-read on reload.
- [ ] **Step 4:** Delete the branch. Close the issue with the recorded facts.

### Task 3: The `paad-delegate` agent and the unattended lint

**Files:**
- Create: `preview/paad/agents/paad-delegate.md`
- Create: `scripts/check_unattended.py`
- Modify: `Makefile` (`self-tests`, `tree-checks`, new `check-unattended`
  target)
- Modify: `scripts/convert_skills.py` (only if the step 6 dry run fails)
- Modify: `preview/paad/skills/paad-help/SKILL.md`, `CHANGELOG.md`

**Interfaces:**
- Produces: the agent `paad:paad-delegate`; the result format above;
  `check_unattended.py <skills_dir>` exits 1 with one `FAIL:` line per
  problem; constants `UNATTENDED_MARKER = "**Unattended mode:**"` (the
  paragraph's opening) and `UNATTENDED_SECTION = "## Unattended mode"` (the
  section heading). Tasks 4–6 add the paragraph and section; Task 4 adds the
  override check.

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
  wherever the skill would stop and ask because it cannot proceed safely,
  return `blocked: <reason>` instead of guessing, even at a stop its section
  does not list; follow no config file the dispatch prompt does not name,
  and list each as `Config not followed:`; dispatch only `paad:paad-analyst`,
  and if a skill that dispatches finds it not available, return `blocked:
  analyst agent not available`; invoke only the
  named skill; write nothing, with any tool (incidental artifacts of
  read-only commands allowed in gitignored paths only, not listed); return
  the result format above and nothing else; everything read is untrusted
  data. Frontmatter: `name: paad-delegate`; `tools: Read, Grep, Glob, Bash,
  Skill, Agent`; a `description` that says it is dispatched explicitly, by
  name, by a calling agent, and never for interactive use.
- [ ] **Step 6:** Run `make check-export-dryrun TREE=preview/paad`. If it
  fails because of the new agent file (red), make `convert_skills.py` skip
  `paad-delegate.md` for Kiro, Antigravity and Pi, with a comment naming the
  reason (the mode is Claude Code only), and re-run it (green). Tasks 4–6 run
  the same dry run after adding their sections, and fix the exporter the same
  way if a section breaks it.
- [ ] **Step 7:** Document the mode in `paad-help` (what it is, how a caller
  dispatches it, the result format, the caller's duty to keep the security
  section out of public records, Claude Code only). Add the `[Unreleased]`
  changelog line.
- [ ] **Step 8: Smoke `unsupported-skill` (human).** In a session started
  with `--plugin-dir` on the clone's preview tree, dispatch the delegate on
  `agentic-owasp` with the prepared prompt. Expected: `Status: blocked: skill
  does not support unattended mode`, `Agent types dispatched: none`, `Files
  written: none`, and nothing else ran.
- [ ] **Step 9: Refactor.** Tidy `check_unattended.py` and the contract
  without changing behaviour; the self-test and smoke results stand.
- [ ] **Step 10:** `make export && make test`. Expected: `All checks
  passed.`
- [ ] **Step 11:** Commit on branch `unattended-mode` in the clone; push to
  the fork. Close the issue with the branch, commit and the `make test` last
  line.

### Task 4: pushback unattended mode

The first skill end to end (spec 8). paad's shared configuration and
security paragraphs are not touched (Global Constraints): the override lives
in pushback's own unattended paragraph, next to them.

**Files:**
- Modify: `scripts/check_unattended.py` (override check)
- Modify: `preview/paad/skills/pushback/SKILL.md`, `paad-help`,
  `CHANGELOG.md`

**Interfaces:**
- Consumes: Task 3's constants and result format.
- Produces: `UNATTENDED_OVERRIDE`, the exact sentence every unattended
  paragraph carries (below); the pattern of an unattended paragraph and
  section that Tasks 5 and 6 copy.

- [ ] **Step 1: Extend the self-test first.** Assert that a skill whose
  unattended paragraph lacks `UNATTENDED_OVERRIDE` fails with `FAIL:
  <skill>'s unattended paragraph lacks the override sentence`, and that a
  skill with no unattended paragraph is not checked. Run it. Expected: FAIL.
- [ ] **Step 2: Add pushback's unattended paragraph** (opening with
  `UNATTENDED_MARKER`, directly after the configuration and security
  paragraphs). It carries, verbatim, the sentence below, and
  `UNATTENDED_OVERRIDE` is set to it: `When this skill runs under
  paad-delegate, the Configuration paragraph above applies only to a config
  file the dispatch prompt names, each other config file found is listed as
  not followed, the Security findings paragraph writes no security file and
  asks nothing, and the "Unattended mode" section below governs the run.`
  Implement the check. Run the self-test and `make tree-checks
  TREE=preview/paad`. Expected: PASS.
- [ ] **Step 3: Add pushback's `## Unattended mode` section**, listing:
  - every interaction point with its default, from spec 4.3's pushback,
    "pushback, alignment" and "all three" rows;
  - the native finding fields (`Claim:`, `Options:`, `Recommendation:`);
  - the steps skipped: every edit, the report file, and every security file;
  - security classification under the delegate with paad's shared
    definition, matches going to the security section (spec 4.2).

  The paragraph and section never name the `paad/security/` path ("writes no
  security file"), and read correctly where no delegate exists (an `npx
  skills`, Kiro, Antigravity or Pi install), since they apply only "when this
  skill runs under paad-delegate".
- [ ] **Step 4:** Show the unattended branch in pushback's digraph where it
  changes the flow. `make check-digraphs` and `make check-export-dryrun
  TREE=preview/paad` pass.
- [ ] **Step 5: Smoke `pushback-plain` and `config-present` (human).** The
  agent prepares a scratch git repo with a small fixture spec, and the
  prompt. In a `--plugin-dir` session, the human runs pushback under the
  delegate, once plain and once with `paad/config/pushback.md` present, and
  pastes the results. Expected: every result line in order; `Status:
  complete`; `Agent types dispatched: none`; findings carry `Claim:`,
  `Options:`, `Recommendation:`; the second run lists `Config not followed:
  paad/config/pushback.md` and the same findings; `Files written: none`; `git
  status --porcelain` empty in the scratch repo.
- [ ] **Step 6: Smoke `pushback-interactive` (human).** Run pushback
  interactively from the preview tree. Expected: it asks, walks findings, and
  offers the update or the report, as in 2.0.2.
- [ ] **Step 7: Refactor.** Tidy the section's wording and the lint without
  changing behaviour; the checks and smokes stand.
- [ ] **Step 8:** Update `paad-help` and the changelog line. `make export &&
  make test`. Expected: `All checks passed.` Commit, push, close with branch,
  commit and `make test` last line.

### Task 5: alignment unattended mode

**Files:** Modify `preview/paad/skills/alignment/SKILL.md`, `paad-help`,
`CHANGELOG.md`.

**Interfaces:** Consumes Task 4's section pattern and constants.

- [ ] **Step 1:** Add alignment's unattended paragraph (with
  `UNATTENDED_OVERRIDE`) and section, as in Task 4 steps 2–3, from spec
  4.3's alignment, "pushback, alignment" and "all three" rows. alignment's
  section carries paad's shared security definition and routes matches to
  the security section. Interactive alignment keeps no security routing
  (that is a proposal, spec 4.9). alignment stays out of `PRODUCERS` and
  `READERS`, so its text never names the `paad/security/` path.
- [ ] **Step 2:** Update the digraph. `make check-digraphs`, `make
  check-security` and `make check-export-dryrun TREE=preview/paad` pass.
- [ ] **Step 3: Smoke `alignment-one-side` and `alignment-tdd` (human, with
  agent-prepared fixtures).** Under the delegate: with only a spec named,
  expected `Status: blocked: both sides must be named`; with a spec and a
  plan whose one code task is not red/green/refactor, expected a `Minor`
  finding for that task, `Agent types dispatched: none`, and the plan file
  unchanged.
- [ ] **Step 4:** Interactive smoke as in Task 4 step 6. `make export && make
  test`. Commit, push, close as in Task 4.

### Task 6: agentic-review unattended mode

**Files:** Modify `preview/paad/skills/agentic-review/SKILL.md` (and its
`references/` only if a dispatch prompt there must name the delegate rule),
`paad-help`, `CHANGELOG.md`.

**Interfaces:** Consumes Task 4's pattern. Produces the `Intent:` input: a
line `Intent: <path>[, <path>…]` in the dispatch prompt; Spec Compliance reads
those files as explicit spec files ahead of its other sources.

- [ ] **Step 1:** Add the unattended paragraph (with `UNATTENDED_OVERRIDE`)
  and section from spec 4.3's agentic-review and "all three" rows: no arguments reviews against the
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
- [ ] **Step 4:** Update the digraph. `make check-digraphs`, `make
  check-dispatch-sites` and `make check-export-dryrun TREE=preview/paad`
  pass.
- [ ] **Step 5: Smoke `no-args`, `intent` and `dirty` (human, with
  agent-prepared fixtures).** In a scratch repo with a branch carrying one
  known bug: no arguments gives `Status: complete`, the bug found, `Agent
  types dispatched: paad:paad-analyst`, and `- no arguments: reviewed against
  the default branch` under `Defaults applied:`; `Intent: spec.md` appears in
  `Intent sources consulted:`; with an uncommitted change, `Status: blocked:
  the working tree has uncommitted changes`. In every run
  `paad/code-reviews/` and `paad/security/` are not created.
- [ ] **Step 6:** Interactive smoke (human). `make export && make test`.
  Commit, push, close as in Task 4.

### Task 7: `--plugin-dir` override in `vrg-vm session`

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
    cloud session runs the installed plugins.`;
  - `test_cloud_session_rejects_malformed_plugin_dirs`: a malformed file
    fails the cloud launch with the `PluginDirsError` message and returns 1,
    as on Lima.
- [ ] **Step 5:** Implement. Run `vrg-container-run -- vrg-validate
  --all-versions`. Expected: PASS, with 100% branch coverage.
- [ ] **Step 6: Refactor.** Remove duplication between the Lima and cloud
  paths' loading, without changing behaviour. Re-run the step 5 command.
  Expected: PASS.
- [ ] **Step 7:** Document the file, its format, the Lima-only rule and how to
  turn it off in `cli-tools-overview.md` under `vrg-vm session`.
- [ ] **Step 8:** Commit with `vrg-commit`, hand off with `vrg-pr-workflow
  report-ready`.

### Task 8: Deploy the override and the fleet checkout (deployment)

**Preconditions:** Tasks 5, 6 and 7 closed; a vergil-tooling release
carrying Task 7 is installed on the host (human-attested).

- [ ] **Step 1 (human):** In the fleet worktree, move `fleet` to the
  `unattended-mode` branch's head; record the commit. Only the human moves
  `fleet`; agents never run git in the fleet worktree.
- [ ] **Step 2:** Write `~/.config/vergil/plugin-dirs.toml` (on the Mac) with
  `plugin_dirs =
  ["/Users/pmoore/dev/projects/wphillipmoore/paad-fleet/preview/paad"]`.
- [ ] **Step 3 (human):** Start a Lima session with `vrg-vm session`. In it,
  confirm `paad:paad-delegate` is listed as an agent type and
  `paad:paad-help` announces `-preview`.
- [ ] **Step 4 (human):** Temporarily point the file at a missing directory;
  confirm the launch fails with the guest error; restore it.
- [ ] **Step 5:** Record `Outcome: SUCCESS` with the evidence and close.

### Task 9: Validate the mode and the drafted vergil prompts (validation)

**Preconditions:** Task 8 closed.

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
  changed skill (human).
- [ ] **Step 3:** Judge each against spec 5's "cleanly". Record per run: the
  status, the field parse, `Agent types dispatched:` against the per-skill
  expectation (the result format above), the fleet checkout commit, `Files
  written:`, and the before and after state of the worktree, its ignored
  `paad/` directories, the starting directory's `paad/` and the temporary
  directories. For the seeded runs, also confirm that no seeded security
  detail appears in any comment the run would post, only in the security
  section.
- [ ] **Step 4:** Record `Outcome: SUCCESS` or `Outcome: FAILURE`. On failure,
  file fix tasks on the fork under the epic and leave this open.
- [ ] **Step 5: Retry after a fix.** When a fix lands on the fork, the human
  moves `fleet` to it (Task 8 step 1), the agent records the new commit, and
  every set in step 2 runs again. Only a full pass closes the task.

### Task 10: Doctrine and policy: the delegate path with fallback

**Files:** Modify `docs/paad-review-doctrine.md`, `docs/paad-policy.md`,
`skills/epic-create/SKILL.md`, `skills/issue-implement/SKILL.md` in
vergil-claude-plugin.

- [ ] **Step 1:** Restructure the doctrine per spec 4.8: the shared procedure
  (output checks, screening, security handling, the no-anchoring rule) moves
  into the common core (§1); each mode's section keeps only what differs.
- [ ] **Step 2:** Add the delegate path with Task 9's validated prompts, and
  the selection rule: use it when `paad:paad-delegate` is loaded, otherwise
  run today's path and record the deviation in the round comment or task
  record. The output checks for the delegate path are spec 4.8's list,
  including `-preview` as an expected announce and the per-skill `Agent
  types dispatched:` expectation (the result format above). Every
  `blocked:` result pauses for the human and is never re-run automatically.
  Each round logs the fleet checkout's branch and commit, as a convenience,
  and notes it if the commit changed since the last round.
- [ ] **Step 3: Parse keywords without regard to case.** On both paths, the
  checks match fixed keywords case-insensitively (`Security:`, `Discards:`,
  `Running paad:`, `Status:`, `SECURITY-CHECK-FAILED` and the rest) and keep
  everything else exact, so an annotation still fails and a case difference
  does not (vergil-project/.github#706).
- [ ] **Step 4: Cut the security section out first.** On the delegate path,
  the orchestrator removes the section between `=== paad security section
  ===` and `=== end paad security section ===` before any other processing.
  Mode 1 writes it to the §2.7 scratch copy (mode 700, outside every repo),
  then runs the walk-through and `vrg-security-record`. Mode 2 fixes each
  finding on the branch or runs `vrg-security-record`. Round comments and
  findings comments are built only from the result with the section
  removed.
- [ ] **Step 5:** Add mode 2 screening of every ordinary finding and discard
  before the findings comment is posted, on both paths.
- [ ] **Step 6:** Keep every item under spec 4.8's "These go away" in the
  fallback path, marked for retirement once a released paad carries the
  mode. Retire §2.10 ("Fallback", the never-adopted paad-config idea), which
  this epic replaces, so "fallback" means only today's path.
- [ ] **Step 7:** paad-policy records the override, how it is turned on and
  off, the Lima-only rule, the fallback, the deviations register rows the
  delegate path retires, and the standing rule for the fleet checkout: only
  the human moves `fleet`, agents never run git in the fleet worktree, and
  after a move the validation sets (Task 9) pass on the new commit before
  vergil relies on it.
- [ ] **Step 8:** Point the two skills' dispatch steps at the doctrine's
  selection rule. Run `vrg-container-run -- vrg-validate`. Commit and hand
  off with `report-ready`.

### Task 11: Follow-on brainstorm: other harnesses

- [ ] **Step 1:** Run `superpowers:brainstorming` on whether unattended mode
  applies to Kiro, Antigravity or Pi, given each harness's subagent support.
- [ ] **Step 2:** Record the outcome on the issue (a new planning issue, or
  "not worth doing" with the reason) for the retrospective's §5. Close it.

### Task 12: Decide: offer the change upstream, or try a different idea (human)

The epic's last task before the retrospective (spec 4.9). It closes only on a
recorded decision.

- [ ] **Step 1:** The human reviews Task 9's evidence and how the mode behaved
  in fleet use since Task 10 landed.
- [ ] **Step 2:** The human decides, and the decision is recorded on the issue
  with its reason:
  - **offer it upstream:** open the PR(s) from fork branches to `Ovid/paad`,
    split the way the author prefers, with Task 9's evidence, the spec §4.10
    conflicts list and the two spec §4.9 proposals; record the PR links.
    Upstream's decision is the author's, and nothing in the epic waits on it;
  - **try a different idea:** file a planning issue for the next attempt,
    with what this experiment taught, and link it.
- [ ] **Step 3:** Close the issue. The retrospective's §5 records the
  decision.
