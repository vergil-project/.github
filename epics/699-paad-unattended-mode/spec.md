# paad unattended mode

- **Epic:** vergil-project/.github#699
- **Planning issue:** vergil-project/.github#698 (review history is in its
  comments)
- **Status:** approved (amended by vergil-project/.github#764)
- **Repos:** the paad fork `wphillipmoore/paad` (the paad change),
  vergil-claude-plugin (the doctrine, the policy and the skills that dispatch
  paad), vergil-tooling (the `--plugin-dir` override in `vrg-vm session`,
  and the worktree convention's `build/` exception),
  vergil-vm (`make` on the agent VM), `.github` (this spec and plan)
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
sweeps `/tmp`, `/var/tmp` and `$HOME` (mode 2). By length, about 40% of
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
paad's own conventions, and, if it proves out, offers it to the paad author.
The vergil workflow then uses it, and the doctrine sheds the interface work.

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
- It is backward compatible: interactive use of every skill is unchanged.
- The patch follows paad's development conventions, so it can be offered
  upstream.
- The vergil doctrine shrinks to match: the stray-write machinery and most
  prompt directives go. They stay as the fallback until a released paad
  carries the mode (4.8), so the epic gains the delegate path and files the
  retirement as a tracked, deferred follow-up that revisits on that release.
- In mode 2, the implementing agent acts alone only on findings about the
  branch's own code that need no human judgment. Security findings,
  expansions of work, behavior changes outside the plan and findings it would
  not act on wait for the human. Every change is reviewed, and every round is
  recorded (4.11).

**Packaging.** The paad change is built and demonstrated as one change. How it
is split into pull requests for the paad author is settled with the author
later (4.9).

**Non-goals:**

- Changing how any paad skill analyses anything (its phases, lenses, verifier
  or severity scales). Under the delegate, pushback and alignment classify
  security findings for the result's security section (4.2); a matching
  change for interactive use is offered separately, as a proposal (4.9).
- Unattended mode for the other paad skills in the first version. The design
  lets them adopt it later.
- paad's config mechanism.
- Other harnesses (Kiro, Antigravity, Pi, `npx skills` installs). Whether the
  mode applies to them is a follow-on brainstorm (8).

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

- **Its tools are `Read, Grep, Glob, Bash, Skill` and `Agent`.** It has no
  Write, Edit or NotebookEdit.
  - The spike on the planning issue showed that a plugin agent with this tool
    list:
    - can invoke a paad skill;
    - can dispatch its own `paad:paad-analyst` subagents from inside that
      skill;
    - has no write tools.
  - **The Agent tool cannot be limited by the harness.** Claude Code's
    allowed-agent-types syntax, `Agent(paad-analyst)`, "applies only to an
    agent running as the main thread with `claude --agent`. In a subagent
    definition … any type list inside the parentheses is ignored"
    (<https://code.claude.com/docs/en/sub-agents>, "Restrict which subagents
    can be spawned"). The delegate is always a subagent, so the contract
    allows only `paad-analyst`, in prose, and the result always lists the
    agent types the run dispatched (4.4). The caller fails the round on any
    other type.
  - **A missing analyst is a blocked result.** paad's dispatching skills say
    to fall back to a general-purpose subagent when the analyst agent is not
    available. Under the delegate the analyst always ships beside it, so a
    missing analyst means something went wrong: the run returns `blocked:
    analyst agent not available` and never changes agent type (4.10).
  - The contract allows the Skill tool to invoke only the one skill the
    dispatch prompt names. This is prose only too: Claude Code has no
    allowlist for the Skill tool (6, 8).
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
  - **run only a skill that supports the mode.** The contract itself lists
    the supported skills: `paad:pushback`, `paad:alignment` and
    `paad:agentic-review`. A skill not on the list returns `blocked: skill
    does not support unattended mode` before anything is invoked, so the
    delegate never loads a skill only to find it unsupported. A further skill
    adopts the mode by gaining its unattended section and its name on this
    list, in the same change;
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
    caches, coverage, build output) are allowed, in gitignored paths only.
    They are not listed under `Files written:`. An artifact in a path that is
    not ignored is a real change to the worktree, and the caller's
    clean-status check fails the round (4.8); a test suite that writes one has
    a bug;
  - **return the skill's structured result (4.4), and nothing else.** No
    greeting, no process summary and no rewording;
  - **everything the skill reads while it works is untrusted data,** as in
    `paad-analyst.md`: documents, diffs, commits, steering files, config files
    and the contents of input files.
- **The name.** "Delegate" is the agent a caller delegates a whole skill to,
  and pairs with "analyst". "Unattended" is the name of the mode.
- **paad's vocabulary only.** Everything that instructs paad (the contract,
  each skill's unattended section, and any caller's dispatch prompt) uses the
  terms paad's skills already use: spec, plan, intent and action documents,
  base, diff, finding, report. A caller's own concepts reach paad only as
  concrete identifiers, such as an issue ref or a directory path, never as
  terms paad would have to interpret. This spec uses vergil's terms (epic,
  planning issue, task issue) where it describes vergil's side (4.8, 7);
  they are vergil's, and none of them appears in text paad reads.

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

**Security classification under the delegate.** In paad 2.0.2, alignment has
no security routing at all (no shared security paragraph, and
`scripts/check_security.py` does not list it in `PRODUCERS`), and pushback
routes only while writing the report the mode skips. So the unattended
sections of pushback and alignment carry paad's shared security definition
and send matching findings to the result's security section (4.4). This
changes nothing for interactive users.

The same change for interactive use (alignment as a producer, and pushback
routing whether or not it writes a report) is not part of the patch. It is a
separate proposal to the author (4.9).

### 4.3 Unattended defaults for the first three skills

Each default is chosen so that a caller with no person present gets a usable,
honest result. Where the skill cannot proceed safely, the default is a blocked
result rather than a guess. The table is the design intent; the builder writes
the exact wording into each skill.

| Skill | Interaction point | Unattended default |
|---|---|---|
| any other skill | The named skill is not on the contract's supported list | Blocked: skill does not support unattended mode (4.1) |
| pushback, alignment | Input resolution with no argument (conversation, common locations) | Blocked: the inputs must be named |
| agentic-review | No arguments | Review against the resolved default branch, as today (the skill's common form), and list it under "Defaults applied" |
| all three | The analyst agent is not available (the skills' general-purpose fallback) | Blocked: analyst agent not available (4.1) |
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
| agentic-review | An `Intent:` path that fails the charset rule or does not exist | Blocked, with the reason |
| agentic-review | An out-of-scope addition's severity | `Suggestion` |
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
   findings, but still carries the security section, empty, so every result
   has the same frame. A skill refused before it is invoked announces as
   `Running paad:<skill> (not invoked)`. An `Agent types dispatched:` line always follows the status,
   listing the agent types the run dispatched, because the harness cannot
   limit them (4.1). It is a different field from agentic-review's existing
   `Agents dispatched:` field, which lists its lenses (item 7).
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
   security definition is paad's existing shared one. For agentic-review, any
   other finding whose symbol, excerpt or fix shows the code a security
   finding is about also moves into this section, keeping its own bug class,
   so nothing outside the section points at that code. The caller sorts such
   a finding as a security finding and counts it in the security section.
9. **Files written:** `none`. Incidental artifacts in gitignored paths are
   not listed (4.1). If the line ever reads otherwise, the contract was
   broken.

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
- **`make export && make test` passes before each commit.** It checks both
  trees, `preview/paad` and the shipped `plugins/paad`, which changes only at
  a release. So the patch leaves paad's shared configuration and security
  paragraphs, `check-config` and `check_security.py` exactly as they are: a
  changed shared paragraph could never pass on the shipped tree. The
  delegate's overrides live in each supported skill's own unattended
  paragraph instead (4.10). The delegate refuses every other skill, so only
  the supported skills ever need them.
- **One new check,** `check-unattended`, in the style of
  `check_security.py`: every skill that carries the unattended paragraph also
  has its unattended section, and each unattended paragraph contains the
  override sentence verbatim. A skill without the paragraph is not checked,
  so the shipped tree passes.
- **No new text names `paad/security/`** in a skill outside
  `check_security.py`'s allowlist (alignment): it says "writes no security
  file" instead.
- **The export pipeline stays correct.** paad also ships outside Claude Code:
  - `npx skills add` installs skill folders but never `agents/`;
  - the exporter converts only `paad-analyst.md` among the agents, and Kiro
    and Antigravity fail at run time on an agent they do not know;
  - the Pi export's tool map has no Skill or Agent tool.

  The mode is Claude Code only (2), so the patch keeps every export correct.
  The skill paragraph is worded so it still reads correctly where no
  delegate exists, and it follows paad's "one phrasing" rule: prose says
  "the analyst" and "the delegate", and literal agent names appear only in
  the delegate's contract and in `subagent_type` dispatch lines. No Pi, Kiro
  or Antigravity delegate is added. `check-references` and the
  `check-export-*` checks pass.
- **`paad-help` documents the mode** and says it is Claude Code only. A
  changelog entry of 1–3 lines goes under `[Unreleased]`. There is no version
  bump: the release is the author's.
- **Digraphs stay accurate.** Each skill's control-flow digraph shows the
  unattended branch where it changes the flow.

### 4.7 The fork, the clone, and how the fleet uses it

- **The fork** is `wphillipmoore/paad` on GitHub, under the human's personal
  account. It is a personal contribution to paad, outside the vergil project.
  It holds the change branches and is where PRs to `Ovid/paad` come from.
- **The local clone** is a clone of the fork at
  `/Users/pmoore/dev/projects/wphillipmoore/paad`, a path that is the same on
  the Mac and in the Lima VMs. Its `origin` is the fork, and `Ovid/paad`
  is added as the `upstream` remote, used only to pull. Nothing is ever pushed
  to `upstream`. The fork and the clone are the standing workspace for
  contributions to paad, during this epic and after it.
- **Branches.** The fork's `main` tracks upstream's `main`; keeping in step is
  a pull from `upstream`. Each change is a branch from `main`, made as
  upstream expects, so it can become a PR to `Ovid/paad` unchanged.
- **Nothing is installed from the fork.** The installed paad stays `Ovid/paad`,
  from its marketplace, as today. There are no fork releases, no version
  scheme and no marketplace change. A marketplace install reads `plugins/paad/`,
  which paad changes only at a release, so installing unreleased work would
  mean running paad's whole release process for every change.
- **The fleet runs unreleased work the way paad's author does: from the
  preview tree.** paad's README and `CLAUDE.md` drive changes with
  `claude --plugin-dir ./preview/paad`, and `/reload-plugins` picks up edits
  without a restart. The spike showed that `--plugin-dir` replaces the
  installed paad for that session alone, and that the delegate and its
  `paad-analyst` subagents run from it.
- **The fleet runs a dedicated checkout, never the development tree.** The
  clone is where development happens, so the fleet loads a separate
  `git worktree` of it, `/Users/pmoore/dev/projects/wphillipmoore/paad-fleet`
  on branch `fleet`. That checkout moves only on purpose, by the human,
  after the gate below passes; moving it is the one deliberate step that puts
  a change in front of the fleet. Agents never run git in it. Development in
  the clone never disturbs it.
- **A standing override in vergil.** Running the development tree across the
  fleet is a deliberate, relatively permanent setup, so it must be easy:
  - a host config file under `~/.config/vergil/` lists the plugin directories
    every session loads by default (the fleet checkout's `preview/paad`). It
    is personal, outside every repo, and off when absent;
  - `vrg-vm session` adds `--plugin-dir <path>` for each entry to every
    `claude` command it launches on a Lima VM (`_session_inner` in
    `vrg_vm.py`, beside the `--model` it already adds). Removing the entry
    returns the fleet to the installed paad;
  - the path is checked inside the guest, and a missing directory fails the
    launch, loudly. Lima VMs already see the Mac's
    `/Users/pmoore/dev/projects` at the same path;
  - **Lima only.** The paad development environment is supported only on
    macOS with Lima VMs. A cloud box has no host mount (`_cloud_session` in
    `vrg_vm.py`), so on a cloud target `vrg-vm session` does not apply the
    override and says plainly that the paad development tree is Lima only.
    Cloud sessions launch normally and use the fallback path (4.8). Cloud
    support is not planned;
  - the plan settles the file's name and format.
- **Some volatility is accepted.** The human records the fleet checkout's
  commit on the issue each time they move it; rounds log nothing about it. A
  session reads the plugin at launch and on `/reload-plugins`, so a tester
  who wants a set of results to stand for one commit does not move the
  checkout while the test runs, which is the human's call.
- **After upstream releases the change,** the installed paad carries it, the
  override is removed, and `--plugin-dir` is needed again only for the next
  unreleased change.
- **Every change the fleet depends on is gated.** Before vergil's doctrine
  relies on a paad change, whether a new commit for the fleet checkout or an
  upstream release, the validation sets (5) pass against that same tree. If
  an upstream release changes the unattended contract (agent name, result
  syntax, status values or input fields), the doctrine is updated to match
  first. The first move of the fleet checkout is itself the deployment under
  test: validation runs with the override on, since the new text applies
  only under the delegate and leaves the skills unchanged otherwise. The gate
  applies to every later move.

### 4.8 The vergil side

Once the fleet runs a paad with unattended mode:

- **The current path stays as a fallback until a released paad has the mode.**
  The doctrine gains the new path and uses it when `paad:paad-delegate` is
  loaded. When it is not (a session started without the override, a cloud
  box, or anyone else using vergil-claude-plugin), it runs today's prompt and
  checks, and the round records the deviation. The old machinery listed
  under "These go away" below is retired only once a released paad carries
  the mode, because the author may take it in a different shape, later, or
  not at all.

- **The two modes differ in one essential variable: who decides at the end of
  each round.** In mode 1, one unattended reviewer runs per round and the
  human decides each round. Mode 2 has two unattended layers: the implementing
  agent runs the review loop itself, and brings in the human only when a
  round cannot be resolved. Everything else is the same in both modes: the
  paad contract, the output checks, screening, security handling and the
  no-anchoring rule. Only the iteration rules and the metadata differ (round
  comments on the planning issue in mode 1, task-issue records in mode 2).
  The doctrine rewrite follows that shape: the shared procedure moves into
  the doctrine's common core (§1), and each mode's section keeps only what
  differs.
- **Both modes dispatch `paad:paad-delegate`.** The reviewer prompt shrinks to:
  - the skill and its arguments;
  - the document roles (mode 1) or the `Intent:` input (mode 2);
  - the working directory;
  - vergil's own rules: the review-the-whole-artifact sentence and the
    no-anchoring rule. In mode 1 the reviewer does not read the planning
    issue or the history of the epic's directory. In mode 2 it does not read
    the task issue, its comments or any pull request.

  **One security definition: paad's own.** What counts as a security
  finding is paad's shared definition, unchanged: "reading it would help an
  attacker", any OWASP Top 10:2025 category, payment, tenant-isolation or
  secret-handling paths, and "on the edge, treat it as security". It is
  written once in the doctrine's common core (§1) and used everywhere: by
  the reviewer, and by the orchestrator's screening in both modes. vergil's
  widening goes: the self-test that extended the definition to design gaps
  about access and triggering. The proposed exclusion of findings about how
  security is handled (vergil-project/vergil-claude-plugin#846), never
  adopted, is declined. Both existed to cut noise or catch mislabelled gaps
  when security findings were recorded with no person involved. Under 4.11 every security finding waits
  for the human's ruling, and in mode 1 the human walks every finding
  before anything is posted, so paad may err toward reporting more. The
  human's rejections, recorded in the counts and the private ruling record,
  show where to tune.

  Directives that restate the contract go. The prompt names vergil's things
  only by concrete identifier, such as an issue ref or a path (4.1), and is
  reviewed for vergil vocabulary before the validation runs.
- **The output checks become a parse of the result.** The workflow checks:
  - the announce line. Under the override it reads `v<X.Y.Z>-preview`, which
    is expected, not a version warning;
  - no `Config:` line saying a config file was followed, in either mode;
  - the status line. A `blocked` status pauses for the human, with its reason,
    and is never re-run automatically;
  - the presence of the fields;
  - severities from the skill's scale;
  - for agentic-review, the values the doctrine checks today (§3.4):
    - all six lenses dispatched, none `NOT DISPATCHED`;
    - Spec Compliance did not bail out, and its intent sources include the
      `Intent:` files;
    - `Verifier warnings: none`;
  - the `Agent types dispatched:` line (4.4): `paad-analyst` only;
  - `Files written: none`;
  - that the worktree's branch, HEAD SHA and clean status are unchanged.
- **These go away** once a released paad carries the mode (the fallback
  above keeps them until then):
  - in both modes, the stray-write check, the round's `TMPDIR`, the
    `paad/security/` checks before and after each round, and the listing of
    the starting directory's `paad/`;
  - in mode 1, `SECURITY-CHECK-FAILED` and the step that moves files out of
    `paad/security/`;
  - in mode 2, reading the report back from `paad/code-reviews/` and emptying
    `paad/code-reviews/` and `paad/security/` after each round, and the `find`
    sweep with its exclusions.

  §2.10 ("Fallback", the never-adopted paad-config idea) is retired by this
  epic now, not at the release, because this epic replaces it. "Fallback"
  then means only today's path.
- **Security findings** go from the result's security section to the existing
  handling:
  - **mode 1:** the orchestrator writes the security section to the scratch
    copy (§2.7), now under the docs repo's main clone,
    `build/.vergil-state/design-review/` (4.11), mode 700. That copy is
    vergil's own bookkeeping, and it keeps unrecorded findings safe if a
    session is interrupted. The keep-until-recorded and cleanup rules stay.
    Then come the walk-through in the terminal and `vrg-security-record`;
  - **mode 2:** the orchestrator writes the section to the round's
    subdirectory of the task's private directory (4.11: in the main clone's
    `build/.vergil-state/agentic-review/`, mode 700, with its keep and
    cleanup rules). Then the round pauses, and nothing is fixed or recorded
    until the human has ruled on each finding (4.11).

  In both modes the security section is cut out before anything else, and
  only from a well-formed result: exactly one start and one end marker, in
  order. Any other shape breaks the round, nothing is posted, and the whole
  result is kept in that mode's private location.

  Screening of ordinary findings and discards (§2.7) stays, and **applies in
  both modes**: it is cheap, routing by the model is still not guaranteed,
  and it is the backstop for forged delimiters (8). Mode 2 has no screening
  today, and it posts its ordinary findings to the task issue, so it gains the
  step: before the findings comment is posted, every ordinary finding and
  discard is screened. Any that qualify move into the cut-out security
  material and are handled as security findings from then on: in mode 1
  walked and recorded, in mode 2 held for the human's ruling (4.11).
- **Keywords match without regard to case.** On both paths, the output checks
  match fixed keywords (`Security:`, `Discards:`, `Running paad:`, `Status:`
  and the rest) case-insensitively and keep everything else exact, so an
  annotation still fails a check and a case difference does not
  (vergil-project/vergil-claude-plugin#841).
- **Two fixes to today's mode 1 checks**, which stay as the fallback until a
  released paad carries the mode:
  - `SECURITY-CHECK-FAILED` fails a round on any line that contains it, as
    mode 2's check does. The prompt asks for the bare line when a check
    fails, and for no mention of the token at all otherwise
    (vergil-project/vergil-claude-plugin#839);
  - the orchestrator creates `paad/security/.gitignore`, holding `*`, in its
    pre-dispatch steps, so the reviewer never finds it missing and never
    reports a check failure for creating it
    (vergil-project/vergil-claude-plugin#840). The repo's managed root
    `.gitignore` already ignores the directory. The file only satisfies
    paad's own guard, which is raised with the author (4.9).
- **One security finding in the gate's own rules** is tracked privately
  (vergil-project/vergil-claude-plugin#829). It concerns today's path in
  mode 2, so the work on that path (4.11) fixes it, with checks before
  dispatch. paad itself shares the weakness, so it is also reported to the
  author through a private channel, never the public conflicts list.
- **What stays unchanged:**
  - one fresh reviewer per round;
  - classification, using trusted authors only;
  - the severity mapping;
  - mode 1's stop and continue signals;
  - mode 1's round comments;
  - the human gates. Mode 2 gains more of them: its loop, its task-issue
    records and its pauses change as 4.11 sets out, on both paths.
- **`paad-policy`** records:
  - that the fleet may run unreleased paad work from the fork's preview tree
    through the `--plugin-dir` override, and how that is turned on and off
    (4.7);
  - the deviations register rows that unattended mode retires;
  - that the skills run under `paad-delegate` when it is loaded, and on the
    fallback path otherwise.

The new reviewer prompt is drafted before validation, and the validation runs
(5) use it, so one pass validates both the mode and the prompt vergil will
use. The doctrine change then lands with that validated prompt, keeping the
fallback path until a released paad carries the mode.

### 4.9 Upstream

This epic is an experiment: a working proof of concept, built and tried in the
vergil fleet before anything is proposed. The design is expected to change as
it is used, so the paad author, Curtis Poe, sees a validated, working change
rather than an untested spec. The human shares progress with him informally
as the work goes, when useful, but nothing in the epic waits on his feedback.

The epic ends with a decision, made by the human once the mode has been
validated and used:

- **Offer it upstream.** The change goes to the author as pull requests from
  fork branches, with the validation evidence and the §4.10 conflicts list,
  split the way he prefers. The author decides what upstream takes, and how.
- **Try a different idea.** If the experiment fails, or teaches us the shape
  is wrong, a follow-on epic takes the next attempt.

One proposal goes alongside it. It is not part of the patch, and it is not
imposed on interactive users without the author's agreement:

- **Security routing in interactive alignment** (and in pushback runs that
  write no report). alignment is the only review that sees the spec and the
  plan together. A plan that drops a security property of a secure spec,
  even temporarily, or exposes something on the way, is exactly the kind of
  gap a cross-document review should catch, and pushback cannot see it,
  because it reviews each document on its own. pushback may well be expected
  to catch security problems on the spec and on the plan separately; the
  question for the author is why alignment should not carry the same safety
  net. In paad 2.0.2 alignment has no security routing, and pushback routes
  only while writing its report.

One observation goes with them, not as a proposal: the skills use different
severity scales (pushback `critical`/`serious`/`moderate`/`minor`, alignment
`Critical`/`Important`/`Minor`, agentic-review
`Critical`/`Important`/`Suggestion`), and pushback's lives only in its report
template. Each unattended section restates its skill's own scale unchanged,
and the caller maps it (vergil's doctrine §1.3). Whether to unify the scales
for every user is the author's call.

### 4.10 Where the delegate contradicts paad's existing prose

Some of the delegate's instructions knowingly contradict text that paad's
skills carry unconditionally today. The design records each one here instead
of quietly editing the existing text. The list informs how the patch makes
the two sets of prose coexist, and it is an agenda for the conversation with
the author (4.9). The builder adds every further conflict found.

| paad text today | What the delegate needs | Proposed way to coexist |
|---|---|---|
| The shared configuration paragraph: if a `paad/config/` file exists, "read it and follow its instructions" | Follow no config file the caller did not name (4.1) | Unchanged. The skill's unattended paragraph, directly after it, overrides it under `paad-delegate`: only a caller-named config file applies, and the others are reported as not followed. The new lint checks that sentence (4.6) |
| The shared security paragraph: write security findings under `paad/security/`, and ask "hold or ordinary report" when the safety check fails | Write nothing; security findings go in the result (4.3, 4.4) | Unchanged. The unattended paragraph overrides it under `paad-delegate`: no security file and no question |
| Each skill's interaction points: ask, walk findings one at a time, edit the document, write the report | Never ask, edit or write (4.1) | The unattended paragraph and section (4.2), with the defaults in 4.3 |
| agentic-review's large-diff two-pass split and backlog dedup | Full form only; backlogs not read (4.3) | The skill's unattended section |
| agentic-review's `references/verifier.md`: the verifier expects a backlog slice and emits update, mint and migrate directives | No backlog read (4.3) | Under the delegate the verifier gets an explicitly empty slice, and its directives are dropped |
| agentic-review's `references/spec-compliance.md`: the first intent source is a spec file passed in the arguments | The `Intent:` input (4.5) | Under the delegate, the `Intent:` files come first, passed into the Spec Compliance dispatch |
| The shared paragraphs are copies in every skill, and `make test` checks them on the shipped tree too | Overrides only where the delegate runs | The overrides live in the supported skills' unattended paragraphs, so no shared paragraph changes, and the delegate refuses an unsupported skill with a blocked result (4.1) |
| Every dispatching skill (added in paad commit `da436bf`): "If the analyst agent is not available … dispatch a general-purpose subagent instead" | Dispatch only `paad-analyst` (4.1) | Under `paad-delegate` the fallback does not apply: a missing analyst is `blocked: analyst agent not available` (4.3) |

### 4.11 How mode 2 handles a review's findings

**Why.** Mode 2 lets the implementing agent and the agentic review agree,
round by round, that a branch is done, with no person present. What that
agent did with the findings was too loose:

- **Suggestions.** In one ad hoc batch of ten branches, four ended in a
  commit no review saw. The agent made it on the strength of a Suggestion.
  Two of those commits changed behavior and design, not polish
  (vergil-project/vergil-claude-plugin#849).
- **Security findings.** On the abandoned epic 683, the gate fixed security
  findings and recorded advisories with no person involved. Most rested on a
  premise only the human could see was wrong. The noise they left still
  needs cleaning up (vergil-project/vergil-claude-plugin#848).

Mode 1 works because the human steers each round and stops the review from
digging forever. In mode 2 the work is already fixed by mode 1's planning. The
review's job is to confirm that the branch does what was planned, and to
improve it within that, not to reopen the design. The review may report more
than is wanted. The fault was in how the agent handled what it reported.

**Goals:**

- No change is made that the human has not approved, beyond the agent's own
  narrow judgment defined here.
- Nothing the review reports is lost.
- Every run leaves a record good enough to analyse later: what the review
  found, what was done with it, and what it cost in time and tokens.

The bar for involving the human starts very low on purpose. Nobody yet knows
how low it can safely go. The recorded rulings are the evidence for lowering it
later. Until then, the agent errs toward asking.

**This applies on both paths:** the delegate path and today's fallback path
(4.8). It changes the doctrine's mode 2 (§3), five skills
(`issue-implement`, `epic-implement`, `issue-deploy`, `issue-validate` and
`epic-retrospective`) and one hook. Mode 1 changes in three ways only: where
its scratch copy lives, the hash check around each review, and the single
security definition (4.8).

**Creating an issue needs the human's approval by default.** Four skills
today say "file follow-ups at once … do not ask whether to", covering anything
a sub-agent reports, any gap the agent hits, and any "we should also…". The
rule becomes: present the item to the human, and file it only on an approving
ruling, in the form approved. The same applies to the other places an agent
decides on its own that new work exists: an operational need `issue-implement`
discovers (a new deployment or validation task changes the plan), and the
untracked problems `epic-retrospective` finds (listed in its draft, which the
human already reviews; a security-class problem is presented in the terminal
and recorded only on an approving ruling). The retrospective's sweep also
learns the new records: the closed ruling words, "Not fixed" entries the
human dropped, "Awaiting a ruling" lists and the "filed without a ruling"
list, so a finding the human closed is not reported as untracked. The exemptions are only these:

- issues the human asked for or approved directly: through planning
  (`epic-create`), capture (`intake-capture`), queue reviews
  (`intake-review`, `deferred-review`), or a ruling, including a mode 1
  design-review ruling and the neutral issue `vrg-security-record` pairs with
  an advisory;
- the per-repo doc tasks that the documentation-review sweep spawns, which
  the bookend convention prescribes;
- fix tasks that `issue-validate` and `issue-deploy` file on a recorded
  `Outcome: FAILURE`, part of the operational lifecycle. Each one is listed
  at the next gate as filed without a ruling, so the human sees it.

This touches `epic-retrospective` as well as the four skills above. The main session of `epic-implement` only presents a paused run's
items. It files nothing from them, and never files or posts a security item.
Its "needs scheduling" list holds only additions the human approved as tasks.

**Items awaiting a ruling are written down at once, best effort.** An item
held only in a session's context is lost when the session compacts or ends,
which is why the old rule filed at once. Instead:

- Each item awaiting a ruling goes under an "Awaiting a ruling" heading in a
  comment the skill already posts on its issue. That is the round's response
  comment in mode 2, and the result comment in `issue-deploy` and
  `issue-validate`. Nothing new is filed, so no issues are opened only to be
  closed.
- A security item appears there only as a count. Its details go to the
  task's private directory (below).
- Every sub-agent's final report lists its items awaiting a ruling, whether
  the run paused or not. `epic-implement` presents them at the gate, beside
  the PRs ready to submit.

This is best effort, not a guarantee. A review finding comes from the code,
so a finding whose record is lost is likely to be found again by a later
review.

#### Each finding needs the human or not

The agent sorts each finding of a round, security findings included, into one
of two groups.

**It needs the human** if any of these apply:

1. **Security.** It is a security finding: in the result's security section,
   or moved there by screening (4.8). The bar is zero: even a one-line fix to
   the branch's own code waits. The human seeing every security finding is
   also how they learn where these come from. A design gap from planning is
   structural; a mistake in the agent's code is mechanical.
2. **Expansion of work.** Acting on it would change code the task does not
   cover, or create anything new: an intake issue, a task, a follow-up or a
   security record. This covers:
   - out-of-scope bugs;
   - out-of-scope additions (whether to remove each one, and whether to file
     it as a task);
   - a fix to neighbouring code, however small, such as a typo in the next
     function.
3. **A behavior change outside the plan.** Acting on it would make the branch
   behave differently from what the task and the plan describe, even though
   the finding is about the branch's own code. Fixing the branch so it does
   what the plan says is not this.
4. **Not acted on.** The agent would decline it, dismiss it as wrong or defer
   it, whatever its severity. Declining a Suggestion drops an idea, and the
   human wants to see each one while the gray area is being mapped.
5. **Unsure.** The agent cannot tell whether one of 1–4 applies.

**Otherwise it is the agent's:** a finding about the code this branch adds or
changes, whose fix makes the branch do what the plan says, or improves it
without changing behavior. A clearer name, a name matching the surrounding
code, or a missing check the task implies are examples. The agent acts on these
without asking.

A **known** finding that the human ruled on earlier is not asked about again.
What happens to it depends on the ruling (step 5 below).

#### A round is handled as a whole

1. The review returns. The agent sorts every finding first, and acts on none
   until all are sorted.
2. **If any finding needs the human, nothing in the round is acted on yet.**
   The ruling on one item can change what the others need, so even the
   agent's own items wait. The round pauses (§3.8), and the agent presents
   every item that needs the human, in the terminal:
   - what the finding is and where;
   - which of 1–5 sent it to the human;
   - the agent's recommendation and the alternatives, best first.

   Security findings are presented in the terminal only. The human rules on
   each item. Then the agent acts on the whole round, the agent's own items
   and the rulings together.
3. **If none needs the human,** the agent acts on the whole round.
4. **Every change goes back to review.** After acting on a round,
   `vrg-container-run -- vrg-validate` must pass. Then the agent runs another
   round. Nothing is committed after the last reviewed HEAD, ever. That
   includes the late Suggestion fix that doctrine §3.7 allows today, which
   goes away.
5. **Done means the review says so.** The branch passes only when a round
   returns nothing left to act on. That means no findings, or only known
   findings whose ruling ended them. A ruling ends a finding when it needs no
   further change on the branch: `decline`, `dismiss`, `file`, `record` and
   `reject` do; `fix` does not. A known finding ruled `fix` that comes back
   means the fix failed. If that ruling was given in the current
   conversation, the agent acts on it again without asking. If it was read
   from the issue, the agent asks the human to confirm it first (below). The
   human can also rule the branch done. The agent never decides on its own
   that it is done.
6. **Three rounds without agreement, and the human decides.** If round 3, or
   the last round the human allowed, still returns something to act on, the
   run pauses. The human rules: allow another round, accept the branch as it
   stands, or drop it. A pause waiting on the human does not use up a round.
   Only review passes count.

**What the human's rulings do.**

- **A security finding:**
  - **approved:** the human also says whether it is fixed on the branch or
    recorded with `vrg-security-record`. A fix on the branch is not
    recorded, as today (§3.6);
  - **rejected:** dropped, with no fix and no record. No public record gives
    a reason. The round's counts show it as rejected, and the private ruling
    record keeps it from being asked about again (below).

  A ruling on a security finding is never restated into the requirements
  file (§3.3). It is kept only in the per-task directory and the
  conversation, because Spec Compliance reads the requirements as intent and
  could quote them in an ordinary, public finding. The known-finding rules
  carry it: a working fix stops the finding coming back, and a failed one
  brings it back as in step 5.

  Because the requirements do not mention it, Spec Compliance may report an
  approved security fix as an unplanned addition or a behavior change. So,
  before the findings comment is posted, every ordinary finding is compared
  with the task's private record of security fixes on the branch, by file and
  symbol. A match is held back from the comment and handled as security
  material, whatever screening (4.8) concluded. On a branch with no approved
  security fix, the step does nothing.
- **Anything that would create an issue or a task:** it is filed only if the
  human approves it, and only in the form they approved.
- **A declined or dismissed finding:** declined only if the human agrees.

The repair pause (most of a round's findings are repairs) and the other
pauses in §3.8 stay.

#### Pausing, resuming and closing

- **Each task has one fixed private directory,** in the `build/` of the
  task repo's main clone:
  `<main clone>/build/.vergil-state/agentic-review/<N>/`, with one
  subdirectory per round, each mode 700. The path follows from the task and
  the round, so a resumed run, and any later session on the task, finds it.
  Mode 2's security handling (§3.6, 4.8) uses it too, in place of today's
  per-file `mktemp -d`.
  - **Why there.** Every repo with a `vergil.toml` ignores `build/` through
    the managed, synced `.gitignore`. That is a permanent architectural
    decision, made by the human, so nothing there is ever committed and
    nothing checks it. The main clones sit on the
    persistent projects volume: the Mac's disk, mounted into the local VMs,
    or the cloud VMs' persistent disk. So the state survives a VM rebuild,
    as the worktrees do. A VM's home directory, where the XDG convention
    would put it, does not.
  - **Not in the reviewer's working tree.** The reviewer works inside the
    task's worktree, under `<main clone>/.worktrees/`. The main clone's
    `build/` is outside that tree, so the reviewer does not come across
    earlier rounds, rulings or security details in the course of its work.
    It could still read them, as it could any path the user can read.
  - **Writes are checked.** Before each review, the orchestrator takes one
    hash over every file in the task's state directory and keeps it in its
    own context, not on disk. After the review it hashes again. Any
    difference is a broken round, like any failed output check. The
    orchestrator also seals the directory between rounds: it keeps the hash
    taken at the end of each round, and checks it at the start of the next,
    so a write by another task's reviewer in the meantime is caught. Across
    a session boundary the kept hash is gone, so a change made while no
    session was running is not detected, which is accepted. This applies in
    both modes and on both paths.
  - **The read-only exception.** The main worktree stays read-only for
    source. Ignored scratch under its `build/.vergil-state/` is not a source
    edit. The doctrine says so, and the worktree-write hook
    (`block-worktree-bypass-write.sh`) allows writes under
    `<main clone>/build/.vergil-state/` and nowhere else in the main
    worktree. The worktree convention in vergil-tooling and the consumer
    `CLAUDE.md` template state the same exception, so no steering file
    contradicts it: the main clone's `build/` is a valid place for ignored
    scratch, as the MQ lab's shared live-lab state already uses it. That
    change is released and synced to the consumer repos before the doctrine
    and the hook rely on it.
  - **Operational tasks** (`issue-validate`, `issue-deploy`) keep any
    security item's details in `build/.vergil-state/operational/<N>/` in
    the main clone of the repo the task is filed in. The same deletion rules
    apply.
  - **Private reports to the paad author** are drafted in
    `build/.vergil-state/upstream-report/<N>/` in the `.github` main clone,
    and deleted once sent.
  - **The fallback path's stray-write sweep** (§3.4) searches `/tmp`,
    `/var/tmp` and `$HOME`. On the VMs the main clones are not under
    `$HOME`, so the sweep does not reach `build/.vergil-state/` there; the
    hash checks above are what cover it. Where a main clone is under
    `$HOME`, the sweep excludes the other tasks' directories under
    `build/.vergil-state/`, each by its path, so a parallel task's writes do
    not break this round, and this task's own directory stays inside it.
  - **Mode 1's scratch copy** (§2.7) moves to the same root, in the docs
    repo's main clone: `build/.vergil-state/design-review/<N>/<round>/`.
- **The paused round's material survives the pause.** The round's full
  result is kept in the round's subdirectory until the round has been acted
  on:
  - on the delegate path, the result itself;
  - on the fallback path, paad's report and its `paad/security/` files,
    moved there before the worktree is emptied.

  §3.6's keep-until-recorded rule extends to keep-until-ruled. The resumed
  run takes the findings from that directory, and the rulings from the human
  in the conversation, as §3.8 says today.
- **The directory is deleted at points the loop already reaches,** as §3.6
  deletes details once they are recorded today:
  - once a round has been acted on and its rulings recorded, its
    subdirectory is deleted, with the full result and any security details;
  - the task's ruling record (finding ID, ruling and, for an approved
    security fix, its file and symbol, with no other detail) stays until the
    summary comment is posted at pass, accept or drop. Then the task's
    directory is deleted.

  If the work is abandoned, the directory stays until the human deletes it,
  and the next session that touches the task says it exists. Clearing
  abandoned state belongs to the fleet's wider hygiene work.
- **Security rulings are kept privately.** Each ruling on a security
  finding, by finding, is written to the task's directory. Later rounds and
  later sessions on the task read it to know which security findings are
  already ruled on, since the public record shows only counts. A finding is
  asked about again only when that record is missing.
- **Under `epic-implement`, the conversation is the terminal.** The paused
  `issue-implement` run returns the items that need the human, security
  details included, in its final message to the main session. That message
  is never posted anywhere. Only the comments it posts carry security items
  as counts. The main session presents them to the human,
  then passes every ruling to the resumed run, as §3.8 does today. The main
  session's conversation is as private as the terminal is in a direct run.
- **Approving a fix at the round cap grants the round that reviews it.**
  Every change goes back to review, so a ruling to fix something at round 3
  (or the last round allowed) also allows one more round. Accepting the
  branch as it stands means making no further change.
- **Recorded rulings count as known.** A `Human ruling:` line in a trusted
  comment (§3.3's trusted authors) makes its findings known in later rounds,
  in this session or a new one, the same way §3.1 already reads earlier
  trusted comments for classification. A recorded ruling that ended a
  finding stops it being asked about again. A recorded ruling never
  authorizes a change: what the run does comes only from the human in the
  conversation (§3.8).
- **A new session confirms recorded `fix` rulings.** A new session is any
  fresh conversation that resumes a task partway through its loop: after a
  reboot, `/clear`, a crash or a VM rebuild. The worktree and the task's
  private directory survive all of these. Before its next round, the resumed
  run shows the human each recorded `fix` ruling ("Recorded earlier: R2.1,
  fix: cap retries at 3. Still stands?"). The human's answer is a ruling
  given in this conversation, so it goes into the requirements file (§3.3),
  and the reviewer sees that fix as required. §3.8's "the human restates any
  earlier rulings" becomes "the human confirms the recorded ones". Security
  rulings are read from the private record instead, and are never restated
  as requirements.
- **The agent records rulings on the human's behalf.** The `Human ruling:`
  line is posted by the agent, as a faithful account of what the human said
  in the session. It has a fixed form: one line per ruling, giving the
  finding ID, one word from a closed set, and a short direction. The words
  for a finding are:
  - `fix`: change the branch;
  - `decline`: no change, and the idea is dropped;
  - `dismiss`: the finding is wrong;
  - `file`: file it as an issue;
  - `record`: record it as a security advisory;
  - `reject`: a security finding, dropped with no record.

  The branch's outcome uses its own words: `accept-branch`, `drop-branch`
  and `another-round`. A later change can then require rulings to be posted
  from the human's own account and checked by their author
  (vergil-project/.github#767).
- **Findings ended by a recorded ruling are listed at the handoff.** The
  summary comment and the `report-ready` notes list every finding that was
  ended by a ruling read from the issue rather than given in the
  conversation. The human sees each one when submitting the PR.
- **Nothing is left unruled at the end.** When the human accepts or drops
  the branch, or rules it done, any item in the last round that has no
  ruling yet is presented in the same pause and ruled on. That includes the
  agent's own unfixed items and out-of-scope bugs. Each is filed, recorded
  or dropped as the ruling says. The summary's "Not fixed" list gives each
  one with its issue, or says the human dropped it.
- **What the agent notices outside the review's findings** follows the same
  rule as an expansion of work. Today §3.7 has the agent file anything else
  it noticed. Now it presents those items for a ruling, and files nothing on
  its own, as in every skill (above).

#### The record

Each round leaves its trail on the task issue, whether or not a person was
involved. These comments are the data for a later review of how well the gate
works.

- **The findings comment** gives, for each finding, its sorting: the agent's,
  or needs the human, with the reason (1–5). Security findings are counts
  only, as today.
- **The response comment is posted in every round.** That includes a round
  whose output check failed: today that round has no response comment for its
  pause line to go on (vergil-project/vergil-claude-plugin#838). It says what
  was done with each finding, and what was paused for the human.
- **Timestamps** (UTC, ISO 8601) mark each handoff, so that the human's
  waits, which can last minutes or weeks, can be left out of the agents' time:
  - when the review was dispatched, and when its result came back;
  - when the round was handed to the human (`Paused:`), and when the human
    answered (`Human ruling:`);
  - when the agent finished acting on the round.
- **The review's cost:** the review subagent's completion notice reports its
  tokens, tool uses and duration (seen in this epic's own sessions as
  `totalTokens`, `toolUses` and `durationMs`). The findings comment records
  them, or says `not reported` when they are missing. The implementing
  session's own token use is not visible to it and is not recorded.
- **Each ruling** is recorded per finding, by its ID and its reason (1–5), in
  clear prose. That is the evidence for tuning the bar. A ruling on a security
  finding appears only in the counts.
- **The summary comment** adds the totals: rounds, pauses, agent time with the
  human's waits left out, and the reviews' tokens.

#### Out of scope here

- Lowering the bar. That is a later decision, made from the recorded rulings.
- How agentic-review itself works. Reporting more than is wanted is fine.
- Mode 1, which already gates every round on the human. Only its scratch
  copy's location changes (above).

## 5. Validation

The mode is ready for the fleet when both seeded scenario sets pass cleanly
against the fork's preview tree, loaded with `--plugin-dir`: the same tree the
fleet then runs (4.7). The runs use the drafted vergil reviewer prompt (4.8).

- **Mode 1, as in vergil-project/.github#625.**
  - Plain runs: spec pushback, plan pushback and alignment, each twice, on a
    published spec and plan.
  - Seeded runs: pushback on a seeded spec, and alignment on a seeded spec and
    plan, each carrying a planted security finding.
- **Mode 2, as in vergil-project/.github#590.** agentic-review on two recorded
  target commits, each twice.
- **Mode 2 seeded.** agentic-review on a branch carrying a planted security
  bug in the diff and one out of scope. It passes when each appears in the
  security section and nowhere else, and the findings comment the run would
  post carries neither.
- **A planted dispatch instruction.** One mode 2 run on a branch whose content
  asks the reviewer to dispatch a general-purpose helper. It tests the
  delegate's prose-only Agent rule (4.1), and passes when no agent type other
  than `paad-analyst` is dispatched, judged from the harness's own session
  transcripts, not from the result's self-reported line.
- **A seeded access-control gap.** One mode 1 run on a seeded spec with an
  access-control gap framed as an omission. It passes when the finding comes
  back in the security section.
- **An unsupported skill.** One dispatch of the delegate on a skill with no
  unattended section. It passes when the result is `blocked: skill does not
  support unattended mode` and the skill did not run.
- **A planted config file.** One mode 2 run on a branch carrying
  `paad/config/agentic-review.md`. It passes when the config file is reported
  as not followed and the review is unchanged by it.
- **Interactive runs.** One interactive run of each changed skill, from the
  fork's preview tree. Each passes when:
  - the skill behaves as in 2.0.2: it asks, walks its findings, offers the
    update or the report, and routes security findings as it does today;
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
are fixed on a fork branch, and the fleet picks them up when the fleet
checkout moves (4.7).

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

**Scope: risks this design creates or worsens.** The baseline is today's
reviewer: a general-purpose agent with Write, Edit, Bash, and unrestricted
Agent and Skill tools. The design only takes capabilities away. A risk that
exists whether or not this spec ships is inherited and out of scope. It may be
worth improving for the system as a whole, but not in this epic. In
particular, the design adds no network capability: outbound requests from
the delegate or its analysts are an existing risk, out of scope here.

**Running the development tree is a deliberate choice.** Loading unreleased
paad from the clone across the fleet (4.7) means whatever is in that tree runs
in every review. That is the point: the fleet is the development environment
for the change, and testing it there is how it is proven before it goes
upstream. The risk is accepted. The fleet runs a dedicated checkout that
moves only on purpose (4.7). The fleet checkout sits on the folder every Lima
VM mounts read-write, so an uncommitted edit there changes what every later
session runs, and nothing checks for it; that is accepted too.

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
- **Interactive users.** `paad/security/` stays their channel, unchanged.
  Extending it to interactive alignment is a proposal to the author (4.9).
- **A branch under review cannot steer its own review.** Under `paad-delegate`
  a `paad/config/` file in the working directory is not followed unless the
  caller names it (4.1), and the caller still fails a round whose result says a
  config file was followed.
- **Handing work to a write-capable agent is discouraged, not prevented.**
  The harness enforces neither limit: Claude Code ignores an Agent type list
  in a subagent definition (4.1), and has no Skill allowlist. Both limits are
  prose, the `Agent types dispatched:` line is self-reported, and Bash can
  start a headless `claude -p` session. Today's reviewer can do all of this
  and more, so these routes are inherited (8).

## 7. Tracking the fork work

The paad change lands in `wphillipmoore/paad` and, upstream, in `Ovid/paad`.
Neither is in the vergil-project org. The placement law says a task lives in
the repo where its PR lands, and cross-org links are out of scope. So the tasks
for the fork work cannot be filed in those repos under this epic.

The proposed answer, which design review should test:

- File the fork tasks in `vergil-project/.github` under this epic.
- Close each on a comment recording the fork branch, its commit and its
  `make test` result, not by a merged PR.
- The validation task (5) and a deployment task (turning on the `--plugin-dir`
  override across the fleet, 4.7) follow the usual operational rules. The
  override itself is a vergil-tooling change, filed and closed there as
  usual.

**Working in the fork follows paad's workflow, not vergil's.** For the first
time, vergil tooling is used on code that vergil does not own, so vergil's
conventions and configuration cannot be imposed on it:

- Changes go through paad's own process: `preview/paad/`, `make export && make
  test`, and its changelog rules. Releases are the author's.
- Nothing vergil-specific goes into the paad repo: no vergil configuration, no
  vergil names, and no changes to paad's tooling for vergil's sake.
- Any helper vergil needs to drive that workflow (a wrapper script, a sync
  step) lives outside the paad repo.

**Git writes in the fork are the human's.** The fork clone is not a
vergil-managed repo: the vergil hook guard blocks raw `git`, `vrg-git` places
worktrees only under a repo's `.worktrees/`, and `vrg-commit` writes vergil's
commit style rather than paad's. So the human clones, creates the fleet
worktree, commits and pushes, with `!`. Agents edit files, run paad's checks
and prepare commit messages in paad's style, and use `vrg-git` only for
read-only commands shown to work there.

**paad's checks need `make`,** and neither the agent VM nor the `prod-base`
container has it, so the human ran every `make test` of Tasks 1–6 on the
Mac (vergil-project/vergil-vm#376). The agent VM gains `make` in its
provisioned packages, and that change is released, so agents can run paad's
checks themselves from then on.

## 8. Risks and open points

- **The author may want a different shape.** The upstream conversation (4.9)
  comes at the end, after the proof of concept, so some rework is possible and
  accepted. Building only the first skill end to end before the other two
  limits it.
- **Bash can still write.** This is the one remaining way to write, and the
  design accepts it as a residual risk:
  - there are no write tools;
  - the contract allows only `paad-analyst` subagents, in prose;
  - nothing is read from paad's own directories;
  - the contract forbids writing, in prose.

  A misbehaving Bash command could still write a file that no round check
  sees, outside the worktree or under an ignored path such as the worktree's
  `paad/` or the starting directory's `paad/`. HEAD and the clean status catch
  only writes to tracked paths. Checking every round for the rest would be the
  routine write-catching this epic removes (2), so the validation sets (5)
  check for it instead. Any exposure would be no worse than today's: a local
  file on the same machine.
- **Inherited, out of scope (6).** Bash writes anywhere else, including files
  that run code later, the delegate reaching other agents through the Skill
  tool or `claude -p`, and outbound network requests. Today's reviewer can do
  all of these, and the design does not make any of them worse.
- **The state-directory guards are workarounds** (4.11). The hash checks,
  the seal between rounds, the sweep and the hook exception exist because a
  caller cannot yet tell paad where it may write: the delegate removes
  paad's own writes, but a reviewer can still write through Bash. A way for
  a caller to confine paad's writes is raised with the author (Task 12).
  Once paad offers one, these guards are revisited and, where they can, they
  go.
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
    the caller's screening (4.8) is the backstop, in both modes, the caller
    refuses any result whose markers are missing, repeated or out of order,
    and the delimiter design is raised with the author.
  - **A read-only mount of the fleet checkout.** If Lima can mount one folder
    read-only for the VMs, the fleet checkout could not be edited from a VM
    at all, which removes the accepted risk in 6.
- **The override's mechanics** (4.7) are unverified until the plan's
  deployment task: the config file, the launch failing on a missing
  directory, and the path resolving inside the Lima VMs.
- **More pauses** (4.11). Mode 2 will stop for the human far more often
  than it does today, which costs autonomy on purpose. A run that ends at
  the round 3 pause may well be the normal outcome while the rules are
  learned. Each pause needs a judgment call from the human, and that is how
  the gate is learned. The recorded rulings show which kinds of pause the
  human always approves. Those are the candidates for the agent's own
  judgment in a later change.
- **Rulings are relayed, not authenticated** (4.11). The agent records the
  human's rulings on the human's behalf, and later sessions trust those
  records, so the record is only as reliable as the agent that wrote it. A
  recorded ruling never authorizes a change on its own: a recorded `fix` is
  confirmed by the human in a new session, and every finding ended by a
  recorded ruling is listed at the handoff. Rulings posted from the human's
  own account, checked by author, are planned as a separate change
  (vergil-project/.github#767).
- **The fallback path lives on** until a released paad carries the mode
  (4.8), so the doctrine carries both paths for a while.
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
- The name and format of the `--plugin-dir` override's config file (4.7).
