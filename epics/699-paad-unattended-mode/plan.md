# paad unattended mode Implementation Plan

- **Epic:** vergil-project/.github#699
- **Planning issue:** vergil-project/.github#698 (review history is in its
  comments)
- **Status:** approved (amended by vergil-project/.github#764)

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
- **paad's "one phrasing" rule.** New skill text says "the analyst" and "the
  delegate" in prose; literal agent names (`paad:paad-analyst`,
  `paad:paad-delegate`) appear only in the delegate's contract and in
  `subagent_type` dispatch lines, so nothing dangling reaches the exports.
- **A running conflicts list.** Each fork task that meets paad text the
  delegate contradicts, beyond spec 4.10's list, appends it (paad text, what
  the delegate needs, how they coexist) to one comment on the epic,
  vergil-project/.github#699, titled `paad prose conflicts found during the
  build`. Task 12 sends spec 4.10 plus that list.
- **Paths are absolute:** `/Users/pmoore/dev/projects/wphillipmoore/paad` (the
  clone) and `/Users/pmoore/dev/projects/wphillipmoore/paad-fleet` (the fleet
  worktree). They are the same on the Mac and in the Lima VMs; `~` is not.
- **Git writes in the fork are the human's.** The hook guard blocks raw `git`,
  `vrg-git` allows `worktree add` only under a repo's `.worktrees/`, and
  `vrg-commit` writes vergil's conventional style, not paad's
  `<skill>: <message>`. So cloning, creating the fleet worktree, committing
  and pushing are human steps run with `!`. Agents edit files, run `make
  export && make test` once Task 14 has given the agent VM `make` (until
  then the human runs it on the Mac), and prepare the commit message in
  paad's style. They
  use `vrg-git` only for read-only commands Task 1 shows to work. "Commit and
  push" below means this rule.
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

1. **The delegate dispatched on a skill not on its supported list** (for
   example `agentic-owasp`): it must return `blocked: skill does not support
   unattended mode` before invoking anything, rather than loading the skill,
   whose own text says to proceed at once. Pinned in Task 3
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

Each task below is one issue under the epic. Tasks 1–6, 11, 12 and 15 are
fork, brainstorm or human tasks filed in `vergil-project/.github` and closed
by comment (spec 7); the rest close by a same-repo PR or as operational
tasks. Tasks 1–7 are delivered: Tasks 1–6 by comment, with the fork commits
`7f4eb0b`, `8bd2bf2`, `353f66c` and `abf879f` on `unattended-mode`, and Task
7 by vergil-project/vergil-tooling#3843, released in vergil-tooling 2.1.279.

| Task | Issue (repo, kind) | Blocked by |
|---|---|---|
| 1 | Recreate the fork, the clone and the fleet checkout (`.github`, closed by comment) | — |
| 2 | Spike: delegate on the fork's preview tree (`.github`, closed by comment) | 1 |
| 3 | The `paad-delegate` agent and the unattended lint (`.github`, fork) | 2 |
| 4 | pushback unattended mode (`.github`, fork) | 3 |
| 5 | alignment unattended mode (`.github`, fork) | 4 |
| 6 | agentic-review unattended mode (`.github`, fork) | 5 |
| 7 | `--plugin-dir` override in `vrg-vm session` (vergil-tooling) | — |
| 8 | Deploy the override and the fleet checkout (`.github`, deployment) | 5, 6, 7 |
| 9 | Validate the mode and the drafted vergil prompts (`.github`, validation) | 8, 13a |
| 10 | Doctrine and policy: the delegate path with fallback (vergil-claude-plugin) | 9, 13a, 13b, 13c |
| 11 | Follow-on brainstorm: other harnesses (`.github`) | 9 |
| 12 | Decide: offer the change upstream, or try a different idea (`.github`, human) | 10, 11 |
| 13a | The state directory and its guards (vergil-claude-plugin#829) | 16, and a vergil-tooling release carrying it, synced to the consumer repos |
| 13b | The mode 2 loop (vergil-claude-plugin#849, also delivering #848 and #838) | 13a |
| 13c | Creating an issue needs approval (vergil-claude-plugin, issue filed with this amendment) | 13b |
| 14 | `make` on the agent VM (vergil-vm#376) | — |
| 15 | Report paad-side findings to the paad author privately (`.github`#768, human) | — |
| 16 | The `build/` exception in the worktree convention and the consumer `CLAUDE.md` (vergil-tooling, issue filed with this amendment) | — |

Task 7 runs in parallel with the fork work. Tasks 14, 15 and 16 can start at
once, and 13a follows 16, so the convention states the exception before the
hook and doctrine use it. Task 13 works on today's path and does not wait for
the delegate. Task 10 then restructures the doctrine on top of 13a and 13b.
13a also carries vergil-project/vergil-claude-plugin#829, a security finding
in the gate's own rules that is tracked privately. Task 15 is not blocked:
the human sends the private report to the paad author at once, because a bug
in paad is worth reporting whatever Task 12 decides. When the spec and plan
merge, the agent:

- updates the bodies of vergil-claude-plugin#849 and #848 to state the rules
  as spec 4.11 settles them, and removes #848's `Blocked-by` on #827, so the
  gate's requirements match what 13b builds;
- removes #768's `Blocked-by` on #735;
- closes vergil-project/vergil-claude-plugin#846 as not planned: the
  proposed exclusion is declined, because the approval gate makes it
  unnecessary (spec 4.8);
- files the issues for 13c and 16.

The documentation review (vergil-project/vergil-claude-plugin#820) has one
specific requirement from this epic. It hunts across every repo for wording
that calls the main worktree strictly read-only, and states the real rule:
the main clone's `build/` is a valid place for ignored scratch, as the MQ
lab's shared live-lab state already uses it, and `build/.vergil-state/` is
the namespace for agent state. Nothing waits on the paad author: the
human shares progress with the author informally, and the upstream conversation
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
| vergil-claude-plugin | `docs/paad-review-doctrine.md` §2.3 and §2.4 (mode 1 check fixes), §2.7 (scratch-copy location), §3.1 and §3.6 (the private directory and its deletion), §3.4 (hash, seal, pre-dispatch checks, sweep); `hooks/scripts/block-worktree-bypass-write.sh` and its test in `hooks/tests/` | The state directory, its guards, mode 1 check fixes and the write exception (Task 13a) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` §1 (the security definition), §2.3 (the widened self-test removed), §2.7 (screening pointed at §1), §3.1, §3.3, §3.5–§3.8; `skills/issue-implement/SKILL.md`; `skills/epic-implement/SKILL.md` | The mode 2 loop (Task 13b) |
| vergil-claude-plugin | `docs/paad-review-doctrine.md` (the approval default); `skills/issue-implement/SKILL.md`; `skills/epic-implement/SKILL.md`; `skills/issue-deploy/SKILL.md`; `skills/issue-validate/SKILL.md` and its lifecycle reference; `skills/epic-retrospective/SKILL.md` | Creating an issue needs approval (Task 13c) |
| vergil-tooling | `docs/specs/worktree-convention.md`; `src/vergil_tooling/data/claude_md_consumer.md` | The `build/` exception to the read-only main worktree (Task 16) |
| vergil-vm | `templates/provision/10-base.sh`, `tests/test_tools.sh` | `make` provisioned and checked (Task 14) |

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
consulted:` and `Verifier warnings:`, under paad's own names.

A blocked result has `Findings: 0` and `Discards: 0`, and still carries the
security section, empty (`Security: 0`, `Security discards: 0`), before
`Files written: none`, so every result has exactly one pair of markers. A
skill refused before it is invoked (not on the supported list) opens with
`Running paad:<skill> (not invoked)`, since its version is unknown without
loading it; callers accept that form only with that status.

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
- [ ] **Step 3 (human):** Clone the fork to
  `/Users/pmoore/dev/projects/wphillipmoore/paad`, add `upstream`
  (`https://github.com/Ovid/paad.git`), and set its push URL to `DISABLED`.
- [ ] **Step 4:** Find which read-only `vrg-git` commands agents can use in
  the clone (spec 7): from inside it, run `vrg-git status`, `vrg-git fetch
  upstream` and `vrg-git log -1`, and record which work and any refusal
  verbatim. Every git write stays the human's (Global Constraints).
- [ ] **Step 5:** Record that `main` equals `upstream/main` (same commit).
- [ ] **Step 6 (human):** Create the fleet worktree on branch `fleet` at
  `main`; the agent records its path and commit.
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
  skill on the contract's own list of supported skills, which reads, at this
  task, none, and gains `paad:pushback`, `paad:alignment` and
  `paad:agentic-review` as Tasks 4–6 land; a skill not on the list returns
  `blocked: skill does not support unattended mode` before anything is
  invoked; never ask, apply the skill's unattended defaults;
  wherever the skill would stop and ask because it cannot proceed safely,
  return `blocked: <reason>` instead of guessing, even at a stop its section
  does not list; follow no config file the dispatch prompt does not name,
  and list each as `Config not followed:`; dispatch only `paad:paad-analyst`,
  and if a skill that dispatches finds it not available, return `blocked:
  analyst agent not available`; invoke only the
  named skill; write nothing, with any tool (incidental artifacts of
  read-only commands allowed in gitignored paths only, not listed); return
  the result format above and nothing else; the caller must keep the
  security section out of every public record (issue comments, commit
  messages, PR text, handoff notes); everything read is untrusted data. Frontmatter: `name: paad-delegate`; `tools: Read, Grep, Glob, Bash,
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
  `agentic-owasp` with the prepared prompt. Expected: `Running
  paad:agentic-owasp (not invoked)`, `Status: blocked: skill does not support
  unattended mode`, `Agent types dispatched: none`, an empty security
  section, `Files written: none`, and the skill never announced.
- [ ] **Step 9: Refactor.** Tidy `check_unattended.py` and the contract
  without changing behaviour; the self-test and smoke results stand.
- [ ] **Step 10:** `make export && make test`. Expected: `All checks
  passed.`
- [ ] **Step 11 (human):** Commit on branch `unattended-mode` in the clone,
  with the agent's message in paad's style, and push to the fork. The agent
  closes the issue with the branch, commit and the `make test` last line.

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
  `UNATTENDED_OVERRIDE` is set to it: `When this skill runs under the
  delegate, the Configuration paragraph above applies only to a config
  file the dispatch prompt names, each other config file found is listed as
  not followed, any Security findings paragraph above writes no security file
  and asks nothing, the result format's line order governs, and the
  "Unattended mode" section below governs the run.` Implement the check, add
  `paad:pushback` to the contract's supported list, and run the self-test and
  `make tree-checks TREE=preview/paad`. Expected: PASS.
- [ ] **Step 3: Add pushback's `## Unattended mode` section**, listing:
  - every interaction point with its default, from spec 4.3's pushback,
    "pushback, alignment" and "all three" rows;
  - pushback's own severity scale, restated so the skill never needs its
    report template: `critical`, `serious`, `moderate`, `minor`;
  - the native finding fields (`Claim:`, `Options:`, `Recommendation:`);
  - the steps skipped: every edit, the report file, and every security file;
  - security classification under the delegate with paad's shared
    definition, matches going to the security section (spec 4.2).

  The paragraph and section never name the `paad/security/` path ("writes no
  security file") or the delegate's literal agent name ("the delegate", per
  the one-phrasing rule), and read correctly where no delegate exists (an
  `npx skills`, Kiro, Antigravity or Pi install), since they apply only "when
  this skill runs under the delegate".
- [ ] **Step 4:** Show the unattended branch in pushback's digraph where it
  changes the flow. `make check-digraphs` and `make check-export-dryrun
  TREE=preview/paad` pass.
- [ ] **Step 5: Smoke `pushback-plain` and `config-present` (human).** The
  agent prepares a scratch git repo with a small fixture spec, and the
  prompt. In a `--plugin-dir` session, the human runs pushback under the
  delegate, once plain and once with `paad/config/pushback.md` present, and
  pastes the results. Expected: every result line in order; `Status:
  complete`; `Agent types dispatched: none`; every severity is one of
  `critical`, `serious`, `moderate`, `minor`; findings carry `Claim:`,
  `Options:`, `Recommendation:`; the second run lists `Config not followed:
  paad/config/pushback.md` and the same findings; `Files written: none`; `git
  status --porcelain` empty in the scratch repo.
- [ ] **Step 6: Smoke `pushback-interactive` (human).** Run pushback
  interactively from the preview tree. Expected: it asks, walks findings, and
  offers the update or the report, as in 2.0.2.
- [ ] **Step 7: Refactor.** Tidy the section's wording and the lint without
  changing behaviour; the checks and smokes stand.
- [ ] **Step 8:** Update `paad-help` and the changelog line. `make export &&
  make test`. Expected: `All checks passed.` Commit and push (human), and
  close with branch, commit and `make test` last line, as in Task 3.

### Task 5: alignment unattended mode

**Files:** Modify `preview/paad/skills/alignment/SKILL.md`, `paad-help`,
`CHANGELOG.md`.

**Interfaces:** Consumes Task 4's section pattern and constants.

- [ ] **Step 1:** Add alignment's unattended paragraph (with
  `UNATTENDED_OVERRIDE`) and section, as in Task 4 steps 2–3, from spec
  4.3's alignment, "pushback, alignment" and "all three" rows, restating
  alignment's scale (`Critical`, `Important`, `Minor`), and add
  `paad:alignment` to the contract's supported list. alignment's section
  carries paad's shared security definition and routes matches to the
  security section. Interactive alignment keeps no security routing
  (that is a proposal, spec 4.9). alignment stays out of `PRODUCERS` and
  `READERS`, so its text never names the `paad/security/` path.
- [ ] **Step 2:** Update the digraph. `make check-digraphs`, `make
  check-security` and `make check-export-dryrun TREE=preview/paad` pass.
- [ ] **Step 3: Smoke `alignment-one-side` and `alignment-tdd` (human, with
  agent-prepared fixtures).** Under the delegate: with only a spec named,
  expected `Status: blocked: both sides must be named`; with a spec and a
  plan whose one code task is not red/green/refactor, expected a `Minor`
  finding for that task, every severity one of `Critical`, `Important`,
  `Minor`, `Agent types dispatched: none`, and the plan file unchanged.
- [ ] **Step 4:** Interactive smoke as in Task 4 step 6. `make export && make
  test`. Commit and push (human), and close as in Task 4.

### Task 6: agentic-review unattended mode

**Files:** Modify `preview/paad/skills/agentic-review/SKILL.md`,
`preview/paad/skills/agentic-review/references/spec-compliance.md`,
`preview/paad/skills/agentic-review/references/verifier.md`, `paad-help`,
`CHANGELOG.md`.

**Interfaces:** Consumes Task 4's pattern. Produces the `Intent:` input: a
line `Intent: <path>[, <path>…]` in the dispatch prompt; Spec Compliance reads
those files as explicit spec files ahead of its other sources.

- [ ] **Step 1:** Add the unattended paragraph (with `UNATTENDED_OVERRIDE`)
  and section from spec 4.3's agentic-review and "all three" rows, and add
  `paad:agentic-review` to the contract's supported list. The section
  restates agentic-review's scale (`Critical`, `Important`, `Suggestion`);
  no arguments reviews against the default branch and is listed under
  `Defaults applied:`; uncommitted changes
  and an ambiguous argument block; no report file, backlog read or backlog
  write; large diffs use the full 12-dispatch form, and only an actual
  dispatch failure blocks; a missing verifier token blocks. Native fields as
  in the result format above.
- [ ] **Step 2:** Under the delegate, the "If the analyst agent is not
  available … dispatch a general-purpose subagent instead" paragraph does not
  apply (spec 4.10): the section says so.
- [ ] **Step 3: Wire in `Intent:`.** Under the delegate, the orchestrator
  passes the `Intent:` paths into the Spec Compliance dispatch as explicit
  spec files, and `references/spec-compliance.md`'s intent-source priority
  gains a delegate-only first item for them, ahead of `$ARGUMENTS`. Document
  the input in the section and in `paad-help`. Interactive arguments are
  unchanged. Add the reference-file change to the running conflicts list.
- [ ] **Step 3a: An empty backlog slice.** Under the delegate, the verifier
  is given an explicitly empty backlog slice, and its update, mint and
  migrate directives are dropped (`references/verifier.md` says so for the
  delegate). Add it to the running conflicts list.
- [ ] **Step 4:** Update the digraph. `make check-digraphs`, `make
  check-dispatch-sites` and `make check-export-dryrun TREE=preview/paad`
  pass.
- [ ] **Step 5: Smoke `no-args`, `intent`, `dirty` and `security` (human,
  with agent-prepared fixtures).** For `security`, a branch carries one
  planted security bug in the diff and one in surrounding code (an
  out-of-scope bug): both appear in the security section and nowhere else,
  and the ordinary findings carry neither. For the rest, in a scratch repo
  with a branch carrying one known bug: no arguments gives `Status: complete`, the bug found, `Agent
  types dispatched: paad:paad-analyst`, and `- no arguments: reviewed against
  the default branch` under `Defaults applied:`; `Intent: spec.md` appears in
  `Intent sources consulted:`; with an uncommitted change, `Status: blocked:
  the working tree has uncommitted changes`. In every run
  `paad/code-reviews/` and `paad/security/` are not created, and every
  severity is one of `Critical`, `Important`, `Suggestion`.
- [ ] **Step 6:** Interactive smoke (human). `make export && make test`.
  Commit and push (human), and close as in Task 4.

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
  "none"). Run `vrg-container-run -- vrg-validate`. Expected: the loader
  tests PASS.
- [ ] **Step 4: Write the failing session tests** in `test_vrg_vm.py`, then
  run `vrg-container-run -- vrg-validate`. Expected: these tests FAIL; a test
  that already passes is rewritten until it fails for the missing behaviour:
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

This first move is itself the deployment under test (spec 4.7): Task 9
validates it with the override on. The new text applies only under the
delegate, and Tasks 4–6's interactive smokes showed the skills unchanged
otherwise. The gate applies to every later move.

- [ ] **Step 1 (human):** In the fleet worktree, move `fleet` to the
  `unattended-mode` branch's head; record the commit. Only the human moves
  `fleet`; agents never run git in the fleet worktree.
- [ ] **Step 2 (human):** Create `~/.config/vergil/plugin-dirs.toml` on the
  Mac, by hand, with the contents the agent supplies: `plugin_dirs =
  ["/Users/pmoore/dev/projects/wphillipmoore/paad-fleet/preview/paad"]`.
  `vrg-vm session` reads it on the Mac; the VMs never need it, and agents in
  a VM cannot reach the Mac's `~/.config`.
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
  concrete identifier. Security uses paad's own definition only, with no
  vergil self-test and no exclusion (spec 4.8). Review
  the drafts for vergil vocabulary (none may remain except as identifiers).
  Record both drafts on the issue.
- [ ] **Step 2:** Run every set in spec 5 with the drafts, from Lima sessions
  using the override: mode 1 plain and seeded runs; mode 2 on two recorded
  commits, each twice; mode 2 seeded, on a branch with a planted security
  bug in scope and one out of scope; planted dispatch (judged from the
  session transcripts' subagent spawns, not the result line); unsupported
  skill; seeded access-control gap; a non-security finding on the same
  symbol as a planted security bug (it must arrive inside the security
  section); `intent-bad-path` (an `Intent:` path with a disallowed character,
  and one that does not exist, each expected to give `Status: blocked: …`);
  planted config; one interactive run of
  each changed skill (human); and one interactive session given a plain
  "review my spec" request (human), for the auto-selection check.
- [ ] **Step 2a: Handle every run's security section by hand.** The runs use
  real material, so a result can carry real security findings before Task
  10 writes the procedure down. For every unattended run: cut the security
  section out first, under spec 4.8's well-formedness rule; keep it in the
  mode's private location under the main clone's `build/.vergil-state/`
  (`design-review/` for mode 1, `agentic-review/` for mode 2, as Task 13a
  defines them); screen the ordinary findings and discards. Present each
  real finding to the human in the terminal, as spec 4.11 does. On an
  approving ruling it is fixed or recorded with `vrg-security-record`, as
  the human directs; a rejected one is dropped with no record. Seeded
  findings are never recorded. Everything recorded on this public issue is
  counts only.
- [ ] **Step 3:** Judge each unattended run against spec 5's "cleanly", and
  each interactive run against spec 5's interactive criteria: it behaves as
  in 2.0.2 (asks, walks findings, offers the update or the report, routes
  security as today), shows no unattended field, and `paad-delegate` is not
  chosen automatically, judged from the auto-selection session's transcript
  (no delegate started). Record per unattended run: the status; that the
  reviewer asked nothing, judged from the session transcript (no question
  turns from the delegate); the field parse; `Agent types dispatched:`
  against the per-skill expectation (the result format above); the fleet
  commit recorded at the last move; `Files written:`; and the before and
  after state of the worktree, its ignored `paad/` directories, the starting
  directory's `paad/` and the temporary directories. For the seeded runs,
  also confirm that no seeded security detail appears in any comment the run
  would post, only in the security section.
- [ ] **Step 4:** Record `Outcome: SUCCESS` or `Outcome: FAILURE`. On failure,
  file fix tasks for the fork in `vergil-project/.github` under the epic
  (spec 7), and leave this open.
- [ ] **Step 5: Retry after a fix.** When a fix lands on the fork, the human
  moves `fleet` to it (Task 8 step 1) and records the new commit, and
  every set in step 2 runs again, in newly started `vrg-vm session`s. Only a
  full pass closes the task.

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
  Rounds log nothing about the fleet checkout: the human records its commit
  on the issue at each move (Task 8 step 1, Task 9 step 5).
- [ ] **Step 3: Parse keywords without regard to case.** On both paths, the
  checks match fixed keywords case-insensitively (`Security:`, `Discards:`,
  `Running paad:`, `Status:`, `SECURITY-CHECK-FAILED` and the rest) and keep
  everything else exact, so an annotation still fails and a case difference
  does not (vergil-project/vergil-claude-plugin#841).
- [ ] **Step 4: Write the cut's test cases first.** Draft results with a
  missing end marker, a duplicated start marker, a `### S<i>` line outside
  the section, a well-formed blocked result, and a well-formed complete
  result. Against today's doctrine none of them is handled (red).
- [ ] **Step 5: Cut the security section out first, and fail closed.** On
  the delegate path, the orchestrator removes the section between `=== paad
  security section ===` and `=== end paad security section ===` before any
  other processing. Mode 1 writes it to the §2.7 scratch copy, at the
  location Task 13a set (`build/.vergil-state/design-review/<N>/<round>/` in
  the docs repo's main clone), then runs the walk-through and
  `vrg-security-record`. Mode 2 writes it to the round's subdirectory of the
  task's private directory (Task 13a, spec 4.11), with its keep and cleanup
  rules. Round comments and findings comments are built only from the result
  with the section
  removed. The cut proceeds only when the result has exactly one start
  marker and one end marker, in that order, matched case-insensitively, and
  no `### S<i>` or `Security discards:` line outside the section. Any other
  shape is a broken round: nothing is posted, the whole result is kept as
  security material in that mode's private location, and the human is asked
  (mode 2 pauses as in §3.8). In mode 2, the cut-out findings then go to
  Task 13's handling: the round pauses and nothing is fixed or recorded
  until the human rules (spec 4.11). Walk the step 4 cases through it (green): the
  three malformed ones break the round with nothing posted, and the two
  well-formed ones proceed. Then tidy the wording (refactor).
- [ ] **Step 6:** Extend Task 13b's mode 2 screening to the delegate path,
  with paad's own definition (spec 4.8). A finding that qualifies moves into
  the cut-out security material, in the task's private directory, before the
  comment is built. From then on it is handled like any security finding:
  held for the human's ruling (spec 4.11), and shown only in the counts.
- [ ] **Step 6a: Carry 13a's guards to the delegate path.** Move the hash
  check around each review, the seal between rounds, and the pre-dispatch
  checks that #829 requires into the doctrine's common core, so they apply
  on both paths (spec 4.11 "Writes are checked", spec 4.8). Write the cases
  first, on the delegate path: a reviewer writes into the task's state
  directory, and each #829 condition from the private advisory. Confirm
  they are not caught on the delegate path today (red), then that they are
  (green).
- [ ] **Step 7:** Keep every item under spec 4.8's "These go away" in the
  fallback path, marked for retirement once a released paad carries the
  mode. Retire §2.10 ("Fallback", the never-adopted paad-config idea), which
  this epic replaces, so "fallback" means only today's path.
- [ ] **Step 8:** paad-policy records the override, how it is turned on and
  off, the Lima-only rule, the fallback, the deviations register rows the
  delegate path retires, and the standing rule for the fleet checkout: only
  the human moves `fleet`, agents never run git in the fleet worktree, and
  after a move the validation sets (Task 9) pass on the new commit, in newly
  started sessions, before vergil relies on it. The same gate applies to
  switching vergil to a released paad: the validation sets pass against it
  first, and if its unattended contract changed (agent name, result syntax,
  status values or input fields), the doctrine is updated before the switch.
  It also records, as an accepted risk, that an uncommitted edit in the
  fleet checkout changes what every later session runs and is not checked
  (spec 6). paad-policy §1 (config files added by a child epic) and §4 (the
  "may add config files as its fallback" bullet, linking §2.10) are updated
  to match §2.10's retirement.
- [ ] **Step 9:** Point the two skills' dispatch steps at the doctrine's
  selection rule. Run `vrg-container-run -- vrg-validate`. Commit and hand
  off with `report-ready`; the handoff notes record
  vergil-project/vergil-claude-plugin#841 as delivered, so the human closes it on merge.

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
    conflicts list together with the running list from the build, the
    spec §4.9 proposal and its severity-scale observation, and two design
    requests from spec 8: a way for a caller to confine where paad writes,
    and a per-run nonce in the security section's delimiters. Anything
    that describes a weakness goes to the author privately, through Task
    15, not in a PR. Record the PR links.
    Upstream's decision is the author's, and nothing in the epic waits on it;
  - **try a different idea:** file a planning issue for the next attempt,
    with what this experiment taught, and link it. The two design requests
    above still go to the author, on their own.
- [ ] **Step 3: File the cleanup as a deferred issue,** with `vrg-intake-route
  … defer --reason … --revisit-when "a paad release carries unattended
  mode"` (or the next attempt replaces it). It covers retiring the fallback
  path's machinery (spec 4.8 "These go away"), removing the `plugin-dirs.toml`
  entry, updating paad-policy, and revisiting the state-directory guards
  (the hash checks, the seal, the sweep exclusion and the hook exception)
  once paad lets a caller confine its writes (spec 8). This is how spec 2's
  goal that the
  doctrine shrinks is met. Link it on the issue.
- [ ] **Step 4:** Close the issue. The retrospective's §5 records the
  decision and the deferred cleanup.

### Task 13: Mode 2 finding handling (13a, 13b, 13c)

Implements spec 4.11 on today's path. It does not depend on the delegate. It
is too big for one PR, so it lands as three tasks, each with its own PR, its
own slice of the cases in step 1, and its own walk in step 7. Each bullet
below is tagged with the task that owns it.

- **13a: the state directory and its guards.** Issue:
  vergil-project/vergil-claude-plugin#829. It covers:
  - `build/.vergil-state/` and its levels, created mode 700;
  - the mode check, the hash check around each review, and the seal
    between rounds;
  - the hook exception, with path normalisation;
  - the pre-dispatch checks that #829 requires;
  - mode 1's scratch-copy move;
  - the two fixes to mode 1's checks (#839, #840; spec 4.8);
  - the sweep exclusion;
  - the upstream-report directory.

  Cases 18–20, 22–25, 28, 38. Blocked by 16, and by a vergil-tooling
  release carrying Task 16, installed and synced to the consumer repos, so
  no repo's `CLAUDE.md` still calls the main worktree strictly read-only.
- **13b: the mode 2 loop.** Issue: vergil-project/vergil-claude-plugin#849,
  also delivering #848 and #838. It covers:
  - sorting, the whole-round pause, acting and passing;
  - records and pauses;
  - the new-session confirmation;
  - one security definition (paad's) for every path, and screening with it;
  - the security-fix hold-back.

  Cases 1–14, 16, 17, 21, 26, 29–32, 39. Blocked by 13a.
- **13c: creating an issue needs approval.** It covers the default and its
  exemptions across `issue-implement`, `epic-implement`, `issue-deploy`,
  `issue-validate` (with its lifecycle reference) and `epic-retrospective`,
  the retrospective sweep's reading of the new records, and the operational
  directory those skills use for security items. Its issue is filed with
  the epic's other new tasks. Cases 15, 27, 33–37. Blocked by 13b,
  because both edit the doctrine, `issue-implement` and `epic-implement`.

**Files:** Modify `docs/paad-review-doctrine.md` (§2.4 and §2.7 for mode 1's
state check and scratch copy, and §3.1, §3.3, §3.4, §3.5, §3.6, §3.7 and
§3.8), `skills/issue-implement/SKILL.md`, `skills/epic-implement/SKILL.md`,
`skills/issue-deploy/SKILL.md`, `skills/issue-validate/SKILL.md`,
`skills/issue-validate/references/operational-task-lifecycle.md`,
`skills/epic-retrospective/SKILL.md` and
`hooks/scripts/block-worktree-bypass-write.sh` (with its test) in
vergil-claude-plugin.

- [ ] **Step 1: Write the cases first.** In the issue, write invented rounds,
  each with its expected outcome under spec 4.11:
  1. three in-scope Important findings and one security finding: the round
     pauses, nothing is committed, and the terminal shows the security
     finding while the issue shows only its count;
  2. two in-scope findings the agent can fix, such as a missing bound and a
     name that doesn't match the code around it: the agent fixes both, and
     round 2 runs;
  3. a single in-scope Suggestion the agent would decline: the round pauses;
  4. an out-of-scope bug and an out-of-scope addition: the round pauses, and
     no issue or task is filed before the ruling;
  5. a fix that would change behavior the plan describes: the round pauses;
  6. round 3 returns one in-scope Suggestion the agent can fix: the run
     pauses, and nothing is committed after the reviewed HEAD;
  7. a round with no findings: a pass, and the handoff lists no unreviewed
     commit;
  8. a failed output check: the response comment is posted with the
     `Paused:` line and its timestamp;
  9. the human approves a security finding to be fixed on the branch: it is
     fixed, round 2 runs, and it is never recorded. The human rejects one
     instead: no fix, no record, and no reason in any comment;
  10. a run under `epic-implement` pauses on a security finding, and a new
      run resumes it with the human's ruling. The new run reads the finding
      from the private directory, the main session's conversation is the only
      place the details appeared, and the round count continues;
  11. at round 3 the human approves one fix and accepts the rest: round 4
      reviews the fix, and every other item from round 3 is ruled on before
      the summary;
  12. a finding the human ruled `fix` in this conversation comes back in the
      next round: the agent fixes it again without asking, and the branch
      does not pass on that round. In a new session, the same recorded
      ruling is shown to the human for confirmation first, and the
      confirmation goes into the requirements file;
  13. a fix breaks `vrg-validate`: it is repaired before the next review is
      dispatched;
  14. the human approves a security fix on the branch. The next round's
      requirements file does not mention it, and a later session reads the
      ruling from the task's private directory and does not ask again;
  15. under `issue-validate`, a failed check finds a bug: the fix task is
      filed (the failure exemption) and listed at the next gate as filed
      without a ruling. A different follow-up the agent notices during the
      run is listed under "Awaiting a ruling", not filed. During
      `issue-implement`, an operational need the agent discovers is also
      listed, not filed;
  16. an out-of-scope bug the human ruled `file` comes back every round:
      the branch still passes, because `file` ended it;
  17. after an approved security fix, Spec Compliance reports the fixed
      code as an unplanned addition: the finding matches the private record
      by file and symbol, and is held back from the findings comment;
  18. two tasks run in parallel: task A's reviewer writes into task B's
      state directory between B's rounds, and B's seal check at the start
      of its next round breaks that round. Where a main clone is under
      `$HOME`, neither round fails the stray-write sweep because of the
      other's legitimate writes;
  19. each condition the pre-dispatch checks for #829 refuse (listed in its
      private advisory, which the implementer reads) is refused before
      dispatch, and the run pauses;
  20. the reviewer writes a file into the task's state directory: the
      before-and-after hash differs and the round is broken;
  21. an ordinary finding qualifies as security under paad's definition in
      screening: it is held back from the findings comment and sorted under
      reason 1;
  22. the orchestrator writes the round's result to
      `<main clone>/build/.vergil-state/agentic-review/<N>/<round>/` with
      Write: the hook allows it. A Write elsewhere in the main worktree is
      still denied;
  23. `build/.vergil-state/` already exists with mode 755: the run tightens
      it to 700 or pauses, and writes nothing until the mode is 700;
  24. the hook's path-normalisation cases recorded in the private record
      for this task are denied;
  25. a fresh worktree has no `paad/security/`: the orchestrator creates it
      as a real directory with its `.gitignore` holding `*`, the checks
      pass, and round 1 runs;
  26. a round on today's path pauses, the session ends, and a new session
      resumes it: the findings come from the round's private directory, and
      the worktree's `paad/` was emptied cleanly;
  27. `issue-validate` meets a security item: its details go to
      `build/.vergil-state/operational/<N>/`, mode 700, and are deleted once
      ruled and recorded;
  28. a mode 1 round's scratch copy lands in the docs repo's main clone
      under `build/.vergil-state/design-review/<N>/<round>/`, and §2.7's
      location check passes;
  29. the mode 1 reviewer prompt (§2.3) and every screening step carry only
      paad's definition from §1, with no widened self-test;
  30. a finding ended by a ruling read from the issue is listed in the
      summary comment and the `report-ready` notes;
  31. the review's completion notice carries no token figures: the findings
      comment says `not reported`;
  32. the result's security section holds a non-security finding moved there
      because it shows a security finding's code: it is sorted under reason
      1 and counted as security, not as ordinary;
  33. `issue-implement` discovers an operational need before its first
      review round: it posts an "Awaiting a ruling" comment on its task
      issue at once, lists the need there and in its final report, and does
      not mint the task;
  34. `issue-deploy` fails and needs a code change: the fix task is filed
      and listed at the next gate as filed without a ruling;
  35. the documentation-review sweep spawns a per-repo doc task: it is filed
      under the exemption;
  36. `epic-retrospective` finds an untracked problem and a security-class
      one: both are listed in the draft, the first filed on approval, the
      second presented in the terminal and recorded only on an approving
      ruling;
  37. `epic-retrospective`'s sweep meets `decline`, `dismiss` and `reject`
      rulings, a "Not fixed" entry the human dropped, an "Awaiting a ruling"
      list and the "filed without a ruling" list: none is reported as
      untracked;
  38. a mode 1 reviewer puts `SECURITY-CHECK-FAILED` in backticks or after
      a list marker: the round fails, as mode 2's check already does
      (#839);
  39. a new session touches a task whose private directory remains from
      abandoned work, with no open round: it tells the human the directory
      exists, before it does anything else.

  Walk each through today's text and confirm it comes out wrong (red). Cases
  1, 2, 7 and 16 are the exceptions: 2, 7 and 16 come out right today
  (today an out-of-scope bug never blocks), and 1 is partly right.
- [ ] **Step 1a (13a): The hook's test first.** Add the test rows in
  `hooks/tests/` for writes under `build/.vergil-state/` (allowed), writes
  elsewhere in the main worktree (denied), and the normalisation cases (24).
  Run `vrg-container-run -- vrg-validate` and confirm that the "allowed" row
  and the normalisation rows fail (red).
- [ ] **Step 2: Rewrite §3.1 steps 1 and 4–7** as spec 4.11's procedure:
  - [13b] step 1 keeps the requirements file in a `mktemp -d` directory, as
    today. It is rebuilt each round, needs no persistence, and its path is
    given to the reviewer, so it must not point into the private directory;
  - [13b] step 4 moves paad's report and every other paad output into the
    round's private directory before it empties the worktree's `paad/`, so
    a paused round keeps its material (case 26);
  - [13b] sort every finding, with reasons 1–5;
  - [13b] pause the whole round if any finding needs the human, and act on
    nothing before the ruling;
  - [13b] act on the whole round;
  - [13b] always run another round after acting;
  - [13b] pass only on a round with nothing left to act on, or on the human's
    ruling. A known finding counts only when its ruling needs no further
    change on the branch (`decline`, `dismiss`, `file`, `record`,
    `reject`);
  - [13b] run `vrg-container-run -- vrg-validate` after acting, before the
    next dispatch;
  - [13a] keep each task's material in its fixed private directory,
    `<main clone>/build/.vergil-state/agentic-review/<N>/<round>/`, in place
    of §3.6's per-file `mktemp -d`. Create every level (`.vergil-state`, the
    kind, `<N>`, `<round>`) with `install -d -m 700` before any Write, check
    the mode with `stat` before each write, and pause if it is wider than
    700 (case 23). State the read-only exception for ignored scratch under
    the main clone's `build/.vergil-state/`;
  - [13b] record each security ruling in the task's directory, with the file
    and symbol of each approved fix;
  - [13a] delete a round's subdirectory once it is acted on and its rulings
    are recorded, and the task's directory once the summary comment is
    posted;
  - [13b] in §3.3, never restate a security ruling into the requirements
    file. In a new session, put the human's confirmations of recorded `fix`
    rulings into it;
  - [13b] before the findings comment is posted, hold back any ordinary
    finding whose file and symbol match an approved security fix on the
    branch;
  - [13a] in §3.4, exclude the other tasks' directories under
    `build/.vergil-state/` from the stray-write sweep, each by its path;
  - [13a] in §2.7, move mode 1's scratch copy to the docs repo's main clone,
    `build/.vergil-state/design-review/<N>/<round>/`, so it survives a VM
    rebuild. Replace §2.7's "outside every repo" check with one that the
    resolved path is under the main clone's `build/.vergil-state/`. Nothing
    checks that `build/` is ignored: the vergil architecture guarantees it;
  - [13a] in §2.4 and §3.4, take one hash over the task's state directory
    before dispatch, keep it in the orchestrator's context, and compare
    after the review. A difference is a broken round. Also seal the
    directory between rounds: keep the hash taken at the end of each round,
    and check it at the start of the next. State that on the VMs the
    stray-write sweep does not reach the main clones, so these hashes are
    what cover them;
  - [13a] in §3.4, before dispatch, first create `paad/security/` as a real
    directory, with its `.gitignore` holding `*`, when they are absent (as
    mode 1 does for #840, case 25). Then run the pre-dispatch checks that
    #829 requires. Their exact form is in #829's private advisory, which the
    implementer reads, and it is never written into public text. On any
    failure, pause;
  - [13b] write paad's shared security definition once in the doctrine's
    common core (§1). Remove vergil's widened self-test sentence from §2.3's
    reviewer prompt template, and point §2.7's screening at §1 (case 29);
  - [13b] add mode 2 screening on this path (spec 4.8), with that
    definition: before the findings comment is posted, every ordinary
    finding and discard is screened, and one that qualifies moves to the
    security material under reason 1. A non-security finding that the
    result's security section carries is sorted and counted the same way
    (case 32). Task 10 later extends screening to the delegate path;
  - [13c] define `build/.vergil-state/operational/<N>/` for security items
    in `issue-validate` and `issue-deploy`, in those skills and the
    lifecycle reference, under the same deletion rules (case 27);
  - [13a] define `build/.vergil-state/upstream-report/<N>/` in the `.github`
    main clone for private reports to the paad author;
  - [13a] fix mode 1's two checks (spec 4.8): `SECURITY-CHECK-FAILED` fails
    a round on any line that contains it, and the §2.3 prompt asks for the
    bare line on failure and no mention otherwise (#839, case 38); §2.4's
    pre-dispatch steps create `paad/security/.gitignore`, holding `*`, when
    it is absent (#840). 13a's handoff notes record #839 and #840 as
    delivered;
  - [13a] in `block-worktree-bypass-write.sh`, normalise every target path
    before it is matched, for both the existing `.worktrees/` arm and the
    new one, as this task's private record describes. Then add the arm that
    allows Write and Edit under `<main_root>/build/.vergil-state/`. Step 1a's
    rows now pass (green); then tidy;
  - [13b] pause at round 3, or the last round the human allowed.

  Waiting on the human does not use up a round. Keep the repair pause and
  the classification (known, repair, new).
- [ ] **Step 3 (13b): §3.6 and §3.7.**
  - Security findings, intake issues, removed-addition tasks and security
    records are created only on the human's approving ruling, in the form
    they approved.
  - Remove "a late Suggestion fix" and every other post-review commit from
    §3.7's handoff. Nothing is committed after the last reviewed HEAD.
  - §3.7's step-6 filing becomes "file what the human approved".
- [ ] **Step 4 (13b): §3.5 records.**
  - The findings comment gives each finding's sorting and reason, the
    dispatch and return timestamps, and the review's `totalTokens`,
    `toolUses` and `durationMs` from its completion notice, or
    `not reported`.
  - The response comment is posted every round, including after a failed
    output check (#838), and ends with the agent's finish timestamp.
  - `Paused:` and `Human ruling:` lines carry their timestamps.
  - Rulings are recorded per finding ID and reason, with security rulings
    as counts.
  - The summary adds rounds, pauses, agent time with the human's waits left
    out, and review tokens.
  - Update the §3.5 examples to match.
- [ ] **Step 5 (13b): §3.8 pauses.** Add the new pause reasons and spec 4.11's
  "Pausing, resuming and closing" rules:
  - the round's material is kept privately until it is ruled on;
  - under `epic-implement`, the conversation counts as the terminal;
  - an approved fix at the cap grants a round;
  - recorded rulings count as known, and a new session confirms recorded
    `fix` rulings before its next round (§3.8's "restate" becomes
    "confirm");
  - nothing is left unruled at the end;
  - a new session that touches a task whose private directory remains from
    abandoned work, with no open round, tells the human it exists before
    anything else (case 39);
  - the summary and the `report-ready` notes list every finding ended by a
    ruling read from the issue.

  The terminal presentation for each item gives:
  - the finding;
  - the reason (1–5);
  - the agent's recommendation and the alternatives, best first.

  Security items appear in the terminal only.
- [ ] **Step 6: The skills.**
  - [13b] `issue-implement`'s gate steps follow the new §3.1 and add no rule of
    their own.
  - [13b] `epic-implement` lists each paused task with its reason and the number
    of items waiting on the human. Its "needs scheduling" list holds only
    tasks the human approved. Its main session only presents a paused run's
    items, and files or posts none of them.
  - [13c] In `issue-implement`, `epic-implement`, `issue-deploy`, `issue-validate`
    and the lifecycle reference, "file follow-ups at once" becomes "list it
    under 'Awaiting a ruling' in the comment you already post, and in your
    final report; file it only as approved". An item that arises before any
    such comment exists, such as an operational need found before the first
    review round, gets its own "Awaiting a ruling" comment on the task issue
    at once (case 33). In the posted comment,
    security items are counts only, with details in the task's private
    directory. The final message to `epic-implement`'s main session carries
    the details and is never posted (spec 4.11). `epic-implement`
    presents every sub-agent's awaiting items at the gate.
  - [13c] State spec 4.11's default ("creating an issue needs the human's
    approval") once in the doctrine, with its three exemptions, and point
    each skill at it. `issue-implement`'s "Discovering an operational need"
    lists the need instead of minting the task. `epic-retrospective` step 4
    lists untracked problems in its draft and files those the human
    approves. The FAILURE branches of `issue-validate`, `issue-deploy` and
    the lifecycle reference keep filing their fix task, and add it to the
    gate's "filed without a ruling" list.
  - [13c] `epic-retrospective` step 4 presents a security-class problem in
    the terminal and records it only on an approving ruling. Its sweep
    (steps 1–2) learns the new records: the closed ruling words, "Not
    fixed" entries the human dropped, "Awaiting a ruling" lists and the
    "filed without a ruling" list (cases 36, 37).
  - [13b] The `Human ruling:` line takes spec 4.11's fixed form and closed word
    set.
- [ ] **Step 7: Walk each sub-task's cases through its new text (green).**
  Record the walk on that sub-task's issue, then tidy the wording.
- [ ] **Step 8:** For each sub-task, run `vrg-container-run -- vrg-validate`,
  commit and hand off with `report-ready`. 13b's handoff notes record #848
  and #838 as delivered, so the human closes them on merge.

### Task 14: `make` on the agent VM

Issue: vergil-project/vergil-vm#376.

**Files:** Modify `templates/provision/10-base.sh` and `tests/test_tools.sh`
in vergil-vm, then regenerate `opentofu/modules/*/vm/cloud-init.yaml` with
`scripts/build-cloud-init.sh` and commit the result. The generated files are
never edited by hand.

- [ ] **Step 1:** Add `check_command make` to `tests/test_tools.sh`. Run the
  test on a VM provisioned from today's templates, and confirm that it fails
  (red).
- [ ] **Step 2:** Add `make` to the base `apt-get install` list in
  `10-base.sh`, and regenerate the cloud-init files.
- [ ] **Step 3: Green before the handoff.** Show `tests/test_tools.sh`
  passing against a VM provisioned from the branch, or, if that cannot run
  in the agent's environment, the validator's provisioning check passing.
  Record which one on the issue.
- [ ] **Step 4:** Tidy the change, run `vrg-container-run -- vrg-validate`,
  commit and hand off with `report-ready`.
- [ ] **Step 5 (human):** After merge, cut the vergil-vm release and
  reprovision the agent VM. In the new VM, confirm that `make --version` runs
  and that `make export && make test` passes in
  `/Users/pmoore/dev/projects/wphillipmoore/paad`. Record both outputs on
  the issue.

### Task 15: Report paad-side findings to the paad author privately (human)

Issue: vergil-project/.github#768. Not blocked: each report goes out as soon
as the finding is known, because a bug in paad is worth reporting whatever
Task 12 decides, and other paad users stay exposed until then.

- [ ] **Step 1:** Draft each report privately in
  `build/.vergil-state/upstream-report/<N>/` in the `.github` main clone,
  creating each level with `install -d -m 700` and the file mode 600, never
  in public text. Until 13a's hook exception lands, write the draft in the
  session's scratch directory first, then put it in place with Bash
  `install` (the hook guards only Write and Edit), and delete the scratch
  copy. The first is the one behind vergil-project/vergil-claude-plugin#829.
- [ ] **Step 2 (human):** Send each one to the paad author through a private
  channel: GitHub private vulnerability reporting on `Ovid/paad` if it is
  enabled, otherwise direct contact. Never in a public issue, PR or the
  public conflicts list. Then delete the draft.
- [ ] **Step 3:** Record on the issue the GHSA ID, channel and date of each
  report, with no details. Any paad-side finding recorded later in the epic
  goes through steps 1–3 the same way. Close the issue at Task 12.

### Task 16: The `build/` exception in the worktree convention

Issue: filed in vergil-tooling when the spec and plan merge.

**Files:** Modify `docs/specs/worktree-convention.md` and
`src/vergil_tooling/data/claude_md_consumer.md` in vergil-tooling.

- [ ] **Step 1:** Write the convention's new rule first, as the test the
  template must meet. The main worktree stays read-only for source.
  Ignored scratch in the main clone's `build/` is allowed, and
  `build/.vergil-state/` is the namespace for agent state that must
  survive a session or a VM rebuild. Show that today's convention and
  today's consumer template both contradict it (red).
- [ ] **Step 2:** Add the rule to `worktree-convention.md`, beside the
  read-only rule it qualifies. Cite the MQ lab's shared live-lab state as an
  existing use of `build/`.
- [ ] **Step 3:** Add the shortest clause that carries the exception to the
  consumer `CLAUDE.md` template's read-only line. Stay within the template's
  line budget (vergil-project/vergil-tooling#3797). The regenerated
  `CLAUDE.md` then agrees with the convention (green).
- [ ] **Step 4:** Tidy, run `vrg-container-run -- vrg-validate`, commit and
  hand off with `report-ready`.
