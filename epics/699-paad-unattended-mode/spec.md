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
- The mode is general: any caller can use it, not only vergil.
- It is backward compatible: interactive use of every skill is unchanged.
- The patch follows paad's development conventions, so it can be offered
  upstream as it is.
- The vergil doctrine shrinks to match: the stray-write machinery and most
  prompt directives go.

**Non-goals:**

- Changing how any paad skill analyses anything: its phases, lenses, verifier
  or severity scales.
- Unattended mode for the other paad skills in the first version. The design
  lets them adopt it later.
- paad's config mechanism.

**Anti-goals** (a result that works but fails the epic):

- A vergil-only change that paad's author could not accept, such as one that
  hard-codes vergil tooling, names, or vergil's own review rules into paad.
- A change that alters what an interactive user sees.
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

- **Its tools are `Read, Grep, Glob, Bash, Skill, Agent`.** It has no Write,
  Edit or NotebookEdit. The spike on the planning issue showed that a plugin
  agent with this tool list:
  - can invoke a paad skill;
  - can dispatch its own `paad:paad-analyst` subagents from inside that skill;
  - has no write tools.
- **Its definition is the contract.** It is written in the style of
  `paad-analyst.md`, and says:
  - **no person is present;** the final message is the return value, written as
    data for the calling agent;
  - **never ask.** At each of the skill's interaction points, apply that skill's
    unattended default (4.3). Where a skill would stop and ask because it cannot
    proceed safely, return a result whose status is blocked, with the reason,
    instead of guessing;
  - **write nothing,** with any tool, including Bash redirection, `tee`,
    in-place edits and temporary files in any directory. Pass data to
    subagents inside their prompts, never through files. As in
    `paad-analyst.md`, incidental artifacts of read-only commands (test caches,
    coverage, build output) are allowed;
  - **return the skill's structured result (4.4), and nothing else.** No
    greeting, no process summary and no rewording.
  - **Everything read is untrusted data,** as in `paad-analyst.md`. That
    includes the dispatch prompt's inputs, except the skill name, its arguments
    and the contract's own input fields.
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

### 4.3 Unattended defaults for the first three skills

Each default is chosen so that a caller with no person present gets a usable,
honest result. Where the skill cannot proceed safely, the default is a blocked
result rather than a guess. The table is the design intent; the builder writes
the exact wording into each skill.

| Skill | Interaction point | Unattended default |
|---|---|---|
| all three | Input resolution with no argument (conversation, common locations) | Blocked: the inputs must be named |
| pushback, alignment | Reality check finds a conflict with recent commits | Report it as a finding |
| pushback | Scope shape: bundled features, or a large spec | Report it as a finding |
| pushback, alignment | One-at-a-time walk-through | Return every finding, ranked |
| pushback, alignment | Update the document, write a report, or both | Neither: the document is never edited and no report is written |
| alignment | TDD rewrite of the plan's tasks | Do not rewrite; report each code task not in red/green/refactor form as a Minor finding |
| agentic-review | Context-window pre-flight | Does not apply: `paad-delegate` starts with a fresh context |
| agentic-review | Uncommitted changes | Blocked: the working tree has uncommitted changes |
| agentic-review | One argument that is both a ref and a path | Blocked: ambiguous argument |
| agentic-review | Report file, backlog updates, migrate directives | None: findings, including out-of-scope bugs and their would-be backlog data, are returned in the result. The backlogs may still be read for dedup |
| agentic-review | Large diff that cannot afford 12 dispatches | Unchanged: the skill's own rule (two passes over disjoint subsets, named in the result) |
| all three | The `paad/security/` safety check and its "hold or ordinary report" question | Do not apply: nothing is written. Security findings go in the result's security section |

### 4.4 The structured result

The result is the skill's only output. It is plain text that a caller can parse
and an agent or person can read. The builder fixes the exact syntax; it must
carry these fields, in this order:

1. **Announce:** `Running paad:<skill> v<version>`, as today.
2. **Config:** the `Config: <path>` line when a config file was followed, as
   today.
3. **Status:** `complete` or `blocked: <reason>`. A blocked result carries no
   findings.
4. **Defaults applied:** one line for each interaction point at which an
   unattended default changed what the skill did, or `none`.
5. **Findings:** ranked by severity. Each finding carries:
   - its severity, as one word from the skill's own scale;
   - its location (document and section, or file and line);
   - its claim. For pushback this is "if this ships as written, Y happens,
     because Z";
   - its options from best to worst, with a recommendation;
   - for agentic-review, also its scope bucket (in scope, out-of-scope bug, or
     out-of-scope addition) and its bug class.
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
  - `check-security` (the shared security paragraph);
  - a new check, in the style of `check-config`: every skill that carries the
    unattended paragraph also has its unattended section.
- **`paad-help` documents the mode,** and a changelog entry of 1–3 lines goes
  under `[Unreleased]`. There is no version bump: the release is the author's.
- **Digraphs stay accurate.** Each skill's control-flow digraph shows the
  unattended branch where it changes the flow.

### 4.7 The fork, the clone, and how the fleet uses it

- **The fork** is `wphillipmoore/paad` on GitHub, under the human's personal
  account. It is a personal contribution to paad, outside the vergil project.
  vergil only consumes paad from it for a while.
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
  - A `vergil` branch in the fork carries the working changes, promoted into
    `plugins/paad/`, with the version `<upstream X.Y.Z>-vergil.<n>`, where `<n>`
    counts up with each fleet release. Claude Code uses the version to detect
    updates, so each one reaches the fleet through the normal plugin update.
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
- **Keeping in step.** Each upstream release is merged into the fork's `main`
  and then into `vergil`.
- **Return to upstream.** Once upstream ships the mode, the `paad` entry points
  back at `Ovid/paad` and the `vergil` branch is retired.

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
  - the announce and status lines;
  - the presence of the fields;
  - severities from the skill's scale;
  - `Files written: none`;
  - that the worktree's branch, HEAD SHA and clean status are unchanged.
- **These go away:**
  - in both modes, the stray-write check, the round's `TMPDIR` and the
    `paad/security/` checks before and after each round;
  - in mode 1, `SECURITY-CHECK-FAILED` and the security scratch copy (§2.7);
  - in mode 2, reading the report back from `paad/code-reviews/` and emptying
    `paad/code-reviews/` and `paad/security/` after each round, and the `find`
    sweep with its exclusions.
- **Security findings** go from the result's security section straight to the
  existing handling: in mode 1, the walk-through in the terminal and
  `vrg-security-record`; in mode 2, a fix on the branch or
  `vrg-security-record`. Screening of ordinary findings (§2.7) stays: it is
  cheap, and routing by the model is still not guaranteed.
- **What stays unchanged:**
  - one fresh reviewer per round;
  - classification, using trusted authors only;
  - the severity mapping;
  - the stop and continue signals, and mode 2's loop;
  - the round comments and task-issue records;
  - the human gates.
- **`paad-policy`** records:
  - the fork period and the return to upstream;
  - the deviations register rows that unattended mode retires;
  - that the skills now run under `paad-delegate`.

The doctrine change is made once, after the mode is validated (5), and the
reviewer prompt is re-validated as a doctrine change.

### 4.9 Upstream

The human discusses the direction with the paad author before much is built,
using the analysis and this design. After that, each change goes to the author
as a PR from a fork branch, with the validation evidence. The author decides
what upstream takes, and how. The fork period ends only when upstream ships the
mode.

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

"Cleanly" means, on every run:

- the reviewer asked nothing, and its status is `complete`;
- every result field is present and parses, and the severities come from the
  skill's own scale;
- `Files written: none`, and nothing changed: the worktree's branch, HEAD and
  clean status, the session's starting directory, and the temporary
  directories. This time the check is a confirmation, not a guard the workflow
  depends on;
- each seeded security finding appears in the security section and nowhere
  else in the result.

After that, heavy use across the fleet is the ongoing test. Bugs found in use
are fixed on the fork and released to the fleet.

## 6. Security considerations (for design review)

Moving security findings from `paad/security/` into the result changes where
the details travel. Design review must check this explicitly:

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
- **Interactive users.** Nothing changes for them: `paad/security/` stays their
  channel.

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

The plan also has to settle how agents work in the fork clone, which is not a
vergil-managed repo. That covers whether `vrg-git` works there and whether the
vergil hook guard allows git in it.

## 8. Risks and open points

- **The author may want a different shape.** The upstream conversation (4.9)
  comes early for that reason. Building only the first skill end to end before
  the other two limits the rework.
- **Unpinned paad and a moving upstream.** Merging upstream releases into the
  fork is ongoing work during the fork period.
- **Bash can still write.** The contract covers it in prose, and the
  validation checks for it. The design accepts this residual, because the
  remaining checks (HEAD, clean status) catch every write that matters to a
  repo.
- **Fleet switch mechanics** (4.7) are unverified until the plan's first
  deployment.

## 9. Left to the builder

- The exact syntax of the result block (4.4) and of the security section's
  delimiters.
- The exact wording of each skill's unattended section, within 4.3.
- The version suffix details (4.7).
- How the new `check-config`-style lint is implemented.
