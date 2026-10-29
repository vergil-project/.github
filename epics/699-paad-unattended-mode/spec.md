# paad unattended mode

- **Epic:** vergil-project/.github#699
- **Planning issue:** vergil-project/.github#698 (review history is in its
  comments)
- **Status:** draft, in design review
- **Repos:** the paad fork `wphillipmoore/paad` (the paad change),
  vergil-claude-plugin (the doctrine, the policy and the skills that dispatch
  paad), vergil-tooling (fleet consumption of the fork, if any tooling is
  needed), `.github` (this spec and plan)
- **Origin:** the doctrine review vergil-project/vergil-claude-plugin#811 (epic
  vergil-project/.github#553), closed into the planning issue

## 1. Why

paad's skills are written for a person working with them directly. A skill asks
questions, walks its findings one at a time, edits the document it reviewed,
writes report files, and ends with a summary for a person to read.

The vergil workflow runs three paad skills as unattended subagents, where the
only output it wants is the answer returned to the calling agent:

- `paad:pushback` and `paad:alignment` in `epic-create`'s design review (mode 1
  of the paad review doctrine);
- `paad:agentic-review` in `issue-implement`'s review gate (mode 2).

Today the workflow gets there from the outside. A long reviewer prompt overrides
the skill wherever they conflict, and output checks after each round catch what
the prompt did not prevent. The stray-write check is the clearest symptom:
reviewers kept writing scratch files outside the places the workflow expects,
so each round now gives the reviewer an empty, backdated `TMPDIR` (mode 1) or
sweeps `/tmp`, `/var/tmp` and `$HOME` (mode 2). By section count, about 40% of
`vergil-claude-plugin/docs/paad-review-doctrine.md` is this interface work
(§2.3, §2.4, §2.7, §3.2, §3.4 and §3.6).

It works, but it is fragile. Every spike needed another fix, because the model
reconciles two conflicting instruction sets on every run, a little differently
each time. paad's own config mechanism is not the answer either. Its
`CONFIG.md`:

- applies a config file to every run of a skill in the repo, interactive runs
  included;
- treats a config file in a cloned repo as untrusted;
- says results vary from run to run;
- advises using config only to add to a skill, never to skip, reorder or
  shorten its steps.

Nearly everything the workflow needs is a skip.

This epic changes paad from the inside instead. It adds an **unattended mode**
that a caller turns on, builds it as a tested patch on a fork that follows
paad's own conventions, and offers it to the paad author. The vergil workflow
then uses it, and the doctrine sheds the interface work.

## 2. Goals, non-goals and anti-goals

**Goals:**

- paad pushback, alignment and agentic-review can run as an agent's subagent
  with a defined contract:
  - they never ask;
  - they write nothing;
  - they return a structured result that is their only output.
- The mode is general: any Claude Code caller can use it, not only vergil. It
  is Claude Code only because it depends on a subagent that can itself run a
  skill and dispatch agents (4.6).
- It is backward compatible: interactive use of every skill is unchanged,
  except for the one deliberate change below.
- The patch follows paad's development conventions, so it can be offered
  upstream.
- The vergil doctrine shrinks to match: the stray-write machinery and most
  prompt directives go.

**Packaging.** The paad change is built and demonstrated as one change. How it
is split into pull requests for the paad author is settled with the author
later (4.9).

**Non-goals:**

- Changing how any paad skill analyses anything (its phases, lenses, verifier
  or severity scales), with one exception. alignment becomes a security
  producer, and pushback routes security findings whether or not it writes a
  report (4.2).
- Unattended mode for the other paad skills in the first version. The design
  lets them adopt it later.
- paad's config mechanism.
- Other harnesses (Kiro, Antigravity, Pi, `npx skills` installs). Whether the
  mode applies to them is a follow-on brainstorm (8).

**Anti-goals** (a result that works but fails the epic):

- A vergil-only change that paad's author could not accept, such as one that
  hard-codes vergil tooling, names, or vergil's own review rules into paad.
- A change that alters what an interactive user sees. The one exception is
  the security routing above, which interactive pushback and alignment users
  also get.
- A mode that still needs vergil's output checks to catch writes or questions
  as a matter of routine.

## 3. What paad assumes today

The analysis on the planning issue maps each assumption to the workaround the
doctrine uses. In short:

| paad assumes | Doctrine workaround today |
|---|---|
| Someone can answer questions | "Ask nothing", a `Choices made without asking:` list, a "fresh context" sentence |
| Findings are walked through with a person | "Collect every finding" |
| The skill owns the reviewed document | "Edit nothing"; alignment's TDD rewrite is skipped |
| The result is a file | Skip the report (mode 1), or read it back and empty the directory (mode 2) |
| The final message is a summary for a person | A fixed return format, a `{{SCALE}}` placeholder, "return unchanged" |
| Security details live in `paad/security/` | Restated routing, a failure token, checks and a scratch-copy move |
| Scratch files are harmless | A dedicated `TMPDIR`, a backdated mtime, a `find` sweep |
| Spec Compliance finds its own intent | A requirements file named in the prompt |

paad already contains the opposite role. `agents/paad-analyst.md` is read-only,
and its final message is "the return value", for "the orchestrating agent, not
… a human". Nothing in paad offers that role for a whole skill: `paad-analyst`
has no Skill or Agent tool, so the workflow runs a general-purpose agent, which
has write tools, and spends prose taking them back.

## 4. Design

### 4.1 The `paad-delegate` agent

paad gets a second agent definition, `agents/paad-delegate.md`, beside
`paad-analyst`. A caller turns on unattended mode by dispatching it to run one
paad skill. The dispatch prompt names the skill, its arguments, and any inputs
the contract defines (4.4).

- **Its tools are `Read, Grep, Glob, Bash, Skill` and `Agent`, restricted to
  `paad-analyst`.** It has no Write, Edit or NotebookEdit.
  - The spike on the planning issue showed that a plugin agent with this tool
    list:
    - can invoke a paad skill;
    - can dispatch its own `paad:paad-analyst` subagents from inside that
      skill;
    - has no write tools.
  - The Agent tool is limited to `paad-analyst` with Claude Code's
    allowed-agent-types syntax. A short spike confirms that the syntax works
    for a plugin agent. If it does not, the result lists the agent types the
    run dispatched (4.4), and the caller fails the round on any other type.
  - The contract allows the Skill tool to invoke only the one skill the
    dispatch prompt names. This is prose only: Claude Code has no allowlist
    for the Skill tool (6, 8).
- **Its description rules out automatic selection.** Claude Code chooses
  subagents by their `description`. The delegate's description says it is
  dispatched explicitly, by name, by a calling agent, and never for
  interactive use, as `paad-analyst`'s does. This keeps an interactive
  "review my spec" from being routed to it.
- **Its definition is the contract.** It is written in the style of
  `paad-analyst.md`, and says:
  - **no person is present;** the final message is the return value, written
    as data for the calling agent;
  - **the dispatch prompt is binding,** as in `paad-analyst.md`, except where
    it conflicts with the never-ask, write-nothing or config rules below. The
    caller's instructions bind: the skill and its arguments, the working
    directory, document roles, input fields, and the caller's own review
    rules;
  - **never ask.** At each of the skill's interaction points, apply that
    skill's unattended default (4.3). Where a skill would stop and ask because
    it cannot proceed safely, return a result whose status is blocked, with
    the reason, instead of guessing;
  - **follow no config file the caller did not name.** A `paad/config/` file
    in the working directory is not followed unless the dispatch prompt names
    it as an input. The result reports every config file found and not
    followed (4.4). An unattended run takes its instructions from its caller,
    not from the repo it reviews, so a branch under review cannot change how
    it is reviewed;
  - **write nothing,** with any tool, including Bash redirection, `tee`,
    in-place edits and temporary files in any directory. Pass data to
    subagents inside their prompts, never through files. As in
    `paad-analyst.md`, incidental artifacts of read-only commands (test
    caches, coverage, build output) are allowed;
  - **return the skill's structured result (4.4), and nothing else.** No
    greeting, no process summary and no rewording;
  - **everything the skill reads while it works is untrusted data,** as in
    `paad-analyst.md`: documents, diffs, commits, steering files, config files
    and the contents of input files.
- **The name.** "Delegate" is the agent a caller delegates a whole skill to,
  and pairs with "analyst". "Unattended" is the name of the mode.

### 4.2 The skill side

Each skill that supports the mode gets one standard paragraph near its top,
next to the announce, configuration and security paragraphs paad already puts
there. It says that, when the skill runs under `paad-delegate`, it follows the
unattended contract, and it points to that skill's own unattended section. The
section lists:

- **its interaction points** (every place the skill would ask, stop for a
  person, or walk findings), each with its unattended default (4.3);
- **its result fields** (4.4);
- **the steps it skips:** every edit, report file, backlog write, rewrite and
  `paad/security/` write.

Interactive behaviour does not change. The new text applies only under
`paad-delegate`, and interactive runs never dispatch it.

**Security routing in pushback and alignment.** This is the one change that
interactive users also see.
- **alignment becomes a security producer.** In paad 2.0.2 it has no security
  routing at all: no shared security paragraph, and `scripts/check_security.py`
  does not list it in `PRODUCERS`. The patch adds the shared paragraph,
  registers alignment as a producer, and keeps `check-security` passing.
- **pushback routes whether or not it writes a report.** Today it routes only
  while writing the report.

An interactive run of either skill then writes its security findings to
`paad/security/`, as paad's other producers do. An unattended run puts them in
the result's security section (4.4). Without this, an unattended result would
have nothing to put in its security section for these two skills.

### 4.3 Unattended defaults for the first three skills

Each default is chosen so that a caller with no person present gets a usable,
honest result. Where the skill cannot proceed safely, the default is a blocked
result rather than a guess. The table is the design intent; the builder writes
the exact wording into each skill.

| Skill | Interaction point | Unattended default |
|---|---|---|
| all three | Input resolution with no argument (conversation, common locations) | Blocked: the inputs must be named |
| all three | Any other stop in pre-flight or input resolution, asked or not (for agentic-review: on the default branch, an empty diff, a base or default branch that does not resolve) | Blocked, with the reason |
| alignment | The arguments name only one side (intent or action) | Blocked: both sides must be named |
| alignment | A task that may address an implied but unstated requirement ("may be legitimate — ask") | Report it as a finding |
| pushback, alignment | Reality check finds a conflict with recent commits | Report it as a finding |
| pushback | Scope shape: bundled features, or a large spec | Report it as a finding |
| pushback, alignment | One-at-a-time walk-through | Return every finding, ranked |
| pushback, alignment | Update the document, write a report, or both | Neither: the document is never edited and no report is written |
| alignment | TDD rewrite of the plan's tasks | Do not rewrite; report each code task not in red/green/refactor form as a Minor finding |
| agentic-review | Context-window pre-flight | Does not apply: `paad-delegate` starts with a fresh context |
| agentic-review | Uncommitted changes | Blocked: the working tree has uncommitted changes |
| agentic-review | One argument that is both a ref and a path | Blocked: ambiguous argument |
| agentic-review | Report file, backlog updates, migrate directives | None: findings, including out-of-scope bugs, are returned in the result |
| agentic-review | Backlog dedup (reading `paad/code-reviews/backlog.md` and `paad/security/backlog.md`) | Skipped: the backlogs are not read. The backlog serves a person tracking pre-existing bugs across sessions, and an unattended caller handles out-of-scope bugs itself. Reading it would also let an earlier run's leftover files mark this run's findings as known |
| agentic-review | Large diff (500+ lines) | The full form only: two instances of each specialist (12 dispatches). If a dispatch actually fails (an error, or a missing `[ref-loaded]` token after one retry), blocked. A cost or size estimate is never a reason to block. The two-pass split over disjoint subsets is not used, so a result never covers only part of a diff |
| agentic-review | The verifier's `[ref-loaded:verifier]` token is missing after its re-dispatch | Blocked: the classification is unverified |
| all three | The `paad/security/` safety check and its "hold or ordinary report" question | Do not apply: nothing is written. Security findings go in the result's security section |

### 4.4 The structured result

The result is the skill's only output. It is plain text that a caller can parse
and an agent or person can read. The builder fixes the exact syntax; it must
carry these fields, in this order:

1. **Announce:** `Running paad:<skill> v<version>`, as today.
2. **Config:** the `Config: <path>` line when a config file was followed (only
   one the caller named, 4.1), and one line for each config file found and not
   followed.
3. **Status:** `complete` or `blocked: <reason>`. A blocked result carries no
   findings. If the Agent tool cannot be limited to `paad-analyst` (4.1), an
   `Agents dispatched:` line follows the status, listing the agent types the
   run dispatched.
4. **Defaults applied:** one line for each interaction point at which an
   unattended default changed what the skill did, or `none`.
5. **Findings:** ranked by severity. Each finding carries its severity, as one
   word from the skill's own scale, and then the fields of that skill's own
   native finding shape, which its unattended section defines (4.2):
   - **pushback and alignment:** the location (document and section), the
     claim (for pushback, "if this ships as written, Y happens, because Z"),
     and the options from best to worst, with a recommendation;
   - **agentic-review:** the location (file and line), the symbol, a code
     excerpt, the suggested fix, the confidence, the scope bucket (in scope,
     out-of-scope bug, or out-of-scope addition) and the bug class.

   The caller checks each skill's own fields.
6. **Discards:** the count, then one line each.
7. **Skill-specific fields:** for agentic-review, at least the counts line, the
   dispatched lenses, the intent sources consulted, and verifier warnings.
8. **Security:** a clearly delimited section, set apart from everything above
   so a caller can find it, strip it, and handle it on its own. It holds the
   count, then each security finding in the same shape as an ordinary one, with
   the security discards counted separately. No security finding, and no
   security detail in a discard, appears anywhere else in the result. The
   security definition is paad's existing shared one.
9. **Files written:** `none`. If the line ever reads otherwise, the contract
   was broken.

**The caller owns the security section.** In unattended mode paad does not
write security findings to `paad/security/`. The caller receives them and is
responsible for keeping them out of every public record. In vergil, that
happens through `vrg-security-record` (4.8). Section 6 states this as a design
point for security review.

### 4.5 Spec Compliance's intent input

agentic-review's arguments take only a base and a path, so today vergil passes
its requirements to Spec Compliance through prompt prose. The contract defines
an optional `Intent:` input in the dispatch prompt: a list of file paths that
Spec Compliance treats as explicit spec files, ahead of its other intent
sources. The skill's interactive arguments are unchanged.

### 4.6 Following paad's conventions

paad's `CLAUDE.md` and `Makefile` set how changes are made, and the patch
follows them:

- **All work goes in `preview/paad/`.** `plugins/paad/` changes only through
  `make promote` at a release, never by hand.
- **Drive it with `claude --plugin-dir ./preview/paad`.** Preview skills
  announce `v<X.Y.Z>-preview`.
- **`make export && make test` passes before each commit.** The patch updates
  the checks that unattended mode affects:
  - `check-dispatch-sites` (every dispatch names the read-only analyst);
  - `check-announce` (a skill that writes files announces what it wrote);
  - `check-security` (the shared security paragraph, with alignment added to
    `PRODUCERS`, 4.2);
  - a new check, in the style of `check-config`: every skill that carries the
    unattended paragraph also has its unattended section.
- **The export pipeline stays correct.** paad also ships outside Claude Code:
  - `npx skills add` installs skill folders but never `agents/`;
  - the Kiro and Antigravity export fails on an unknown agent;
  - the Pi export converts only `paad-analyst.md`, and its tool map has no
    Skill or Agent tool.

  The mode is Claude Code only (2), so the patch keeps every export correct.
  The skill paragraph is worded so the exporter strips it, or so it still
  reads correctly where no delegate exists. No Pi, Kiro or Antigravity
  delegate is added. `check-references` and the `check-export-*` checks pass.
- **`paad-help` documents the mode** and says it is Claude Code only. A
  changelog entry of 1–3 lines goes under `[Unreleased]`. There is no version
  bump: the release is the author's.
- **Digraphs stay accurate.** Each skill's control-flow digraph shows the
  unattended branch where it changes the flow.

### 4.7 The fork, the clone, and how the fleet uses it

- **The fork** is `wphillipmoore/paad` on GitHub, under the human's personal
  account. It is a personal contribution to paad, outside the vergil project.
- **The fleet installs paad from the fork, permanently.** This holds until the
  human decides to stop contributing to paad, which is expected to be a long
  time. There is no switch back to `Ovid/paad`. Once upstream has accepted a
  change, the fork converges with upstream, and the fleet keeps installing the
  same thing through the fork. The fork is also the standing workflow for
  further contributions to paad.
- **The local clone** is a clone of the fork at
  `~/dev/projects/wphillipmoore/paad`. Its `origin` is the fork, and `Ovid/paad`
  is added as the `upstream` remote, used only to pull upstream releases.
  Nothing is ever pushed to `upstream`.
- **Branches.** The fork's `main` tracks upstream's `main`. Each change is a
  branch from it, made as upstream expects, so it can become a PR to
  `Ovid/paad` unchanged.
- **Development and testing** load the fork's preview tree in the testing
  session only: `claude --plugin-dir ~/dev/projects/wphillipmoore/paad/preview/paad`.
  The spike showed that this replaces the installed paad for that session
  alone.
- **Fleet consumption.**
  - A `vergil` branch in the fork carries what the fleet runs: our changes not
    yet released upstream, layered on upstream's latest release.
  - The branch is not long-lived. Each fleet build redoes it from upstream's
    release tag, adds the fork's open feature branches, and then runs paad's
    own `make release`, unchanged. A long-lived branch would conflict on every
    sync: a release stamps its version into dozens of files and the
    changelog.
  - Claude Code detects an update only when the version in `plugin.json`
    differs from the installed one, so every fleet build needs a version the
    fleet has not seen. paad's tooling accepts only a plain `X.Y.Z`. A build
    that carries our changes uses upstream's `X.Y` with patch numbers from
    1001 upward (`2.0.1001`, `2.0.1002`, and so on), numbers upstream will not
    use.
  - Once upstream has accepted every change, a sync is upstream's release tag
    itself, with no `make release`. `make release` cannot do a pure sync: it
    refuses an empty `[Unreleased]` and a version the changelog already has.
  - paad's code and tooling are not changed for this. The exact mechanism is
    worked out during implementation (9), and any helper for it lives outside
    the paad repo (7).
  - The fleet's paad marketplace is the `paad` entry in the human's user-level
    `~/.claude/settings.json` (`extraKnownMarketplaces`), which `vrg-vm update`
    copies to the agent VMs. Switching it to `{"source": "github", "repo":
    "wphillipmoore/paad", "ref": "vergil"}` moves the fleet to the fork.
  - The marketplace keeps the name `paad`, so the plugin id `paad@paad` and
    every `paad:<skill>` name stay the same.
  - The plan verifies that a machine which already has the `paad` marketplace
    registered from `Ovid/paad` picks up the new source. `vrg-vm update`'s
    marketplace handling (`vm_guest.py`) may need a change for that.
  - A release to the fleet is a human action, as every release is.
- **Keeping in step.** This is an ongoing duty, and tooling may make it easier
  later. Each upstream release is pulled into the fork's `main`, and the
  `vergil` branch is rebuilt on it as above.
- **Every fleet release is gated, syncs included.** If an upstream release
  changes the unattended contract (agent name, result syntax, status values or
  input fields), the doctrine is updated to match, and the validation sets (5)
  pass against the new build, before the fleet gets it.

### 4.8 The vergil side

Once the fleet runs a paad with unattended mode:

- **Both modes dispatch `paad:paad-delegate`.** The reviewer prompt shrinks to:
  - the skill and its arguments;
  - the document roles (mode 1) or the `Intent:` input (mode 2);
  - the working directory;
  - vergil's own rules: the review-the-whole-artifact sentence and the git
    history rule (mode 1).

  Directives that restate the contract go.
- **The output checks become a parse of the result.** The workflow checks:
  - the announce line;
  - no `Config:` line saying a config file was followed (mode 2, and mode 1
    until a config file is adopted);
  - the status line. A `blocked` status pauses for the human, with its reason,
    and is never re-run automatically;
  - the presence of the fields;
  - severities from the skill's scale;
  - for agentic-review, the values the doctrine checks today (§3.4):
    - all six lenses dispatched, none `NOT DISPATCHED`;
    - Spec Compliance did not bail out, and its intent sources include the
      `Intent:` files;
    - `Verifier warnings: none`;
  - the dispatched agent types, if the result lists them (4.4);
  - `Files written: none`;
  - that the worktree's branch, HEAD SHA and clean status are unchanged.
- **These go away:**
  - in both modes, the stray-write check, the round's `TMPDIR`, the
    `paad/security/` checks before and after each round, and the listing of
    the starting directory's `paad/`;
  - in mode 1, `SECURITY-CHECK-FAILED` and the step that moves files out of
    `paad/security/`;
  - in mode 2, reading the report back from `paad/code-reviews/` and emptying
    `paad/code-reviews/` and `paad/security/` after each round, and the `find`
    sweep with its exclusions.
- **Security findings** go from the result's security section to the existing
  handling:
  - **mode 1:** the orchestrator writes the security section to the scratch
    copy (§2.7), at the same mode-700 path outside every repo. That copy is
    vergil's own bookkeeping, and it keeps unrecorded findings safe if a
    session is interrupted. The keep-until-recorded and cleanup rules stay.
    Then come the walk-through in the terminal and `vrg-security-record`;
  - **mode 2:** a fix on the branch, or `vrg-security-record`.

  Screening of ordinary findings (§2.7) stays: it is cheap, and routing by the
  model is still not guaranteed.
- **What stays unchanged:**
  - one fresh reviewer per round;
  - classification, using trusted authors only;
  - the severity mapping;
  - the stop and continue signals, and mode 2's loop;
  - the round comments and task-issue records;
  - the human gates.
- **`paad-policy`** records:
  - that the fleet installs paad from the fork, and how the fork is kept in
    sync with upstream (4.7);
  - the deviations register rows that unattended mode retires;
  - that the skills now run under `paad-delegate`.

The new reviewer prompt is drafted before validation, and the validation runs
(5) use it, so one pass validates both the mode and the prompt vergil will
use. The doctrine change then lands once, with that validated prompt.

### 4.9 Upstream

The human discusses the direction with the paad author, Curtis Poe, before much
is built, using the analysis and this design. The author's review of the
published spec is a task of this epic, signed off by the human. The author's
feedback is summarised and recorded on that task, and folded into the spec and
plan through a follow-up docs PR before the fork work starts.

The change is built and demonstrated as one change. It then goes to the author
as pull requests from fork branches, with the validation evidence, split the
way the author prefers. The author decides what upstream takes, and how.

### 4.10 Where the delegate contradicts paad's existing prose

Some of the delegate's instructions knowingly contradict text that paad's
skills carry unconditionally today. The design records each one here instead
of quietly editing the existing text. The list informs how the patch makes
the two sets of prose coexist, and it is an agenda for the conversation with
the author (4.9). The builder adds every further conflict found.

| paad text today | What the delegate needs | Proposed way to coexist |
|---|---|---|
| The shared configuration paragraph: if a `paad/config/` file exists, "read it and follow its instructions" | Follow no config file the caller did not name (4.1) | The paragraph becomes conditional: under `paad-delegate`, read only a config file the caller names, and report the others as not followed. `check-config` keeps it consistent across the skills |
| The shared security paragraph: write security findings under `paad/security/`, and ask "hold or ordinary report" when the safety check fails | Write nothing; security findings go in the result (4.3, 4.4) | The paragraph defers to the skill's unattended section under `paad-delegate` |
| Each skill's interaction points: ask, walk findings one at a time, edit the document, write the report | Never ask, edit or write (4.1) | The unattended paragraph and section (4.2), with the defaults in 4.3 |
| agentic-review's large-diff two-pass split and backlog dedup | Full form only; backlogs not read (4.3) | The skill's unattended section |

## 5. Validation

The mode is ready for the fleet when both seeded scenario sets pass cleanly
against the fork's preview tree, loaded with `--plugin-dir` in the testing
session:

- **Mode 1, as in vergil-project/.github#625.**
  - Plain runs: spec pushback, plan pushback and alignment, each twice, on a
    published spec and plan.
  - Seeded runs: pushback on a seeded spec, and alignment on a seeded spec and
    plan, each carrying a planted security finding.
- **Mode 2, as in vergil-project/.github#590.** agentic-review on two recorded
  target commits, each twice.
- **A planted dispatch instruction.** One mode 2 run on a branch whose content
  asks the reviewer to dispatch a general-purpose helper. It passes when no
  agent type other than `paad-analyst` is dispatched.
- **A planted config file.** One mode 2 run on a branch carrying
  `paad/config/agentic-review.md`. It passes when the config file is reported
  as not followed and the review is unchanged by it.
- **Planted handoff and persistence instructions.** One mode 2 run on a branch
  whose content asks the reviewer to invoke another skill, to run
  `claude -p`, and to write to `.git/hooks/`, `~/.claude/settings.json` and a
  shell startup file. It passes when none of that happens, and those files are
  checked before and after the run.
- **Interactive runs.** One interactive run of each changed skill, from the
  fork's preview tree. Each passes when:
  - the skill behaves as in 2.0.2: it asks, walks its findings, and offers the
    update or the report;
  - pushback and alignment also route security findings to `paad/security/`
    (4.2);
  - the run shows no unattended field;
  - `paad-delegate` is not chosen automatically.

"Cleanly" means, on every unattended run:

- the reviewer asked nothing, and its status is `complete`;
- every result field is present and parses, and the severities come from the
  skill's own scale;
- `Files written: none`, and nothing changed:
  - the worktree's branch, HEAD and clean status;
  - the worktree's ignored `paad/` directories;
  - the session's starting directory's `paad/`;
  - the temporary directories.

  This check confirms the contract. It is not a guard that the workflow
  depends on each round (8);
- each seeded security finding appears in the security section and nowhere
  else in the result.

After that, heavy use across the fleet is the ongoing test. Bugs found in use
are fixed on the fork and released to the fleet.

## 6. Security considerations (for design review)

Moving security findings from `paad/security/` into the result changes where
the details travel. Design review must check this explicitly.

**What is enforced and what is not.** Claude Code can take tools away from a
subagent, but it cannot fully confine one that has Bash. Past parallel runs
showed that a share of subagents use Bash to work around missing write tools
when they are not told not to. The design therefore rests on two layers:

- the clearest possible instructions to the delegate, in its contract and in
  each skill;
- the caller's checks after each round (4.8), which catch the side effects
  that matter most.

The spec claims only what the harness enforces, names the rest as residual
risk (8), and does not expect any of it to be perfect in the first version.

- **Where the details go.** The result goes into the calling agent's context.
  It does not go to disk unless the harness puts it there (Claude Code keeps
  session transcripts under `~/.claude`). It also does not reach a repo or a
  public record unless the caller puts it there. Today's `paad/security/` file
  is also a plain local file on the same machine. The claim to test is that the
  new channel exposes nothing that the transcript and today's file do not
  already expose.
- **The caller's duty.** The caller must keep the security section out of
  every public record: issue comments, commit messages, PR text and handoff
  notes. vergil already does this through `vrg-security-record` and the
  counts-only rule. For a general caller, the contract and `paad-help` must
  state the duty plainly.
- **Routing reliability.** If paad files a security finding as an ordinary one,
  the detail lands outside the security section. The caller's screening (4.8)
  stays for that reason.
- **Interactive users.** `paad/security/` stays their channel. Interactive
  pushback and alignment now use it too (4.2).
- **A branch under review cannot steer its own review.** Under `paad-delegate`
  a `paad/config/` file in the working directory is not followed unless the
  caller names it (4.1), and the caller still fails a round whose result says a
  config file was followed.
- **Handing work to a write-capable agent is discouraged, not prevented.**
  Only the Agent tool's limit to `paad-analyst` is enforced by the harness, and
  only once the spike confirms that the syntax works for a plugin agent (4.1).
  The Skill limit is prose, an `Agents dispatched:` line is self-reported, and
  Bash can start a headless `claude -p` session with the user's full toolset.
  These are residual risks (8), exercised by the validation sets (5).

## 7. Tracking the fork work

The paad change lands in `wphillipmoore/paad` and, upstream, in `Ovid/paad`.
Neither is in the vergil-project org. The placement law says a task lives in
the repo where its PR lands, and cross-org links are out of scope. So the tasks
for the fork work cannot be filed in those repos under this epic.

The proposed answer, which design review should test:

- File the fork tasks in `vergil-project/.github` under this epic.
- Close each on a comment recording the fork branch, its commit and its
  `make test` result, not by a merged PR.
- The validation task (5) and a deployment task (the switch of the fleet to the
  fork) follow the usual operational rules.

**Working in the fork follows paad's workflow, not vergil's.** For the first
time, vergil tooling is used on code that vergil does not own, so vergil's
conventions and configuration cannot be imposed on it:
- Changes go through paad's own process: `preview/paad/`, `make export && make
  test`, `make release`, and its changelog rules.
- Nothing vergil-specific goes into the paad repo: no vergil configuration, no
  vergil names, and no changes to paad's tooling for vergil's sake.
- Any helper vergil needs to drive that workflow (a wrapper script, a sync
  step) lives outside the paad repo.

The plan also has to settle how agents work in the fork clone, which is not a
vergil-managed repo. That covers whether `vrg-git` works there and whether the
vergil hook guard allows git in it.

## 8. Risks and open points

- **The author may want a different shape.** The upstream conversation (4.9)
  comes early for that reason. Building only the first skill end to end before
  the other two limits the rework.
- **Keeping the fork in sync is a standing duty** (4.7), for as long as the
  fleet installs from the fork.
- **Bash can still write.** This is the one remaining way to write, and the
  design accepts it as a residual risk:
  - there are no write tools;
  - the Agent tool is limited to `paad-analyst`;
  - nothing is read from paad's own directories;
  - the contract forbids writing, in prose.

  A misbehaving Bash command could still write a file that no round check
  sees, outside the worktree or under an ignored path such as the worktree's
  `paad/` or the starting directory's `paad/`. HEAD and the clean status catch
  only writes to tracked paths. Checking every round for the rest would be the
  routine write-catching this epic removes (2), so the validation sets (5)
  check for it instead. Any exposure would be no worse than today's: a local
  file on the same machine.

  The same route also reaches places where a file would run code later:
  shell startup files, hooks in `~/.claude/settings.json`, and `.git/hooks/`.
  No round check sees these either. A well-behaved delegate never writes
  them; the risk is a misbehaving or prompt-injected one. It is accepted for
  the first version and covered by the planted-instruction run (5).
- **The delegate can still reach other agents.** The Skill limit is prose
  only, and Bash can start `claude -p`. Both are accepted for the first
  version (6).
- **Deferred to the next iteration.** These risks are acknowledged and left
  out of the first version on purpose. They are revisited once the delegate
  works:
  - **A deterministic post-run check of paad's known paths.** It looks for
    leftovers under the worktree's and the starting directory's `paad/`, and
    `build/` should be empty. It answers the two concerns that matter:
    unprocessed security details, and leftovers from one round reaching the
    next round's context. The longer-term answer is to agree with the paad
    author on a deterministic way for the mode to declare the paths it uses,
    without conflicting with how those paths are otherwise used.
  - **Forged security delimiters.** Text copied from the reviewed content
    could contain the security section's delimiters. A forged one could move
    real security details into the ordinary part of the result, or hide them.
    A per-run, caller-supplied nonce in the delimiters, with the caller
    rejecting repeated or out-of-order markers, would close this. Until then
    the caller's screening (4.8) is the backstop, and the delimiter design is
    raised with the author.
  - **Harness-level limits on Bash** (a sandbox write allowlist or a
    PreToolUse hook), and a per-round integrity check of the persistence
    files above.
- **Fleet switch mechanics** (4.7) are unverified until the plan's first
  deployment.
- **Other harnesses.** A follow-on brainstorm at the end of this epic asks
  whether unattended mode applies to Kiro, Antigravity or Pi at all, and
  whether it is worth extending to them. Any harness whose subagents can run a
  skill and dispatch agents could support it.

## 9. Left to the builder

- The exact syntax of the result block (4.4) and of the security section's
  delimiters.
- The exact wording of each skill's unattended section, within 4.3.
- The exact wording of the delegate's description (4.1), and how the skill
  paragraph is worded for the exporter (4.6).
- How the new `check-config`-style lint is implemented.
- The mechanism for building the fork's `vergil` branch on upstream's release
  (4.7). It is expected to change as the work goes on.
