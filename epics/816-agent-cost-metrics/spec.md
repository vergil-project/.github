# Measuring agent cost

- **Epic:** vergil-project/.github#816
- **Planning issue:** vergil-project/.github#802 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-claude-plugin (the logging rule, the skills, the review
  doctrine), vergil-tooling (the roll-up and audit tool, the static lint, the
  site docs), the paad fork `wphillipmoore/paad` (paad's own logging, offered
  upstream to `Ovid/paad`), `.github` (this spec, the plan and the
  retrospective)

## 1. Problem

Nobody knows what agent work in vergil costs, or where the cost goes.
Agentic review alone has raised token use by roughly an order of magnitude,
which is probably worth it, but that is a guess. Decisions that depend on the
answer are waiting for it: vergil-project/.github#812 (whether agentic review
can do less on non-code changes) is deferred until the cost is measured.

Today one call site records anything. Mode 2 of the
[paad review doctrine](https://github.com/vergil-project/vergil-claude-plugin/blob/develop/docs/paad-review-doctrine.md)
records each agentic-review round's cost on the task issue
(`Review cost: totalTokens <t>, toolUses <u>, durationMs <d>`, §3.5) and sums
the review tokens in the gate's summary. Everything else is unmeasured:

- `epic-implement`'s dispatches of `issue-implement`, `issue-deploy` and
  `issue-validate`;
- the code writing and the responses to findings, which `issue-implement`
  does in its own context, so they cannot be told apart from each other;
- `epic-create`'s design-review rounds (mode 1);
- every other sub-agent a vergil skill dispatches;
- the sub-agents paad dispatches. agentic-review always dispatches six
  specialists and a Verifier (twelve and a Verifier on a diff of 500+ lines).
  Its report names each lens and any `BAIL:` reason, but no cost, although
  its orchestrator receives each specialist's cost when the specialist
  returns.

## 2. Goals, non-goals, anti-goals

**Goals.**

1. Every sub-agent call vergil makes logs its cost in one simple, parseable
   stanza.
2. paad logs the same stanza for every sub-agent it dispatches, so the numbers
   nest down to agentic-review's specialists.
3. The cost of each activity is measured separately: implementing, reviewing,
   responding to review, deploying and validating. This needs `issue-implement`
   split into a manager and workers (§7).
4. Roll-ups per task and per epic, and an audit that shows where the rule was
   not followed.
5. A real sample, measured end to end, at the close of the epic.

**Tokens are the primary metric.** Elapsed time is recorded, but it absorbs
every wait on a human (an approval prompt, a question left while the human is
away), so it is secondary. Each stanza carries a count of human interactions
so a reader knows when to distrust the time: zero means the time is real; one
or more means it includes waits.

**Non-goals.**

- Changing agentic review's intensity, lens selection or manifest widening.
  That is #812, which waits on these numbers.
- Cost budgets, caps or alerts. This epic measures only.
- Making elapsed time precise.

**Anti-goals** (failure even if everything technically works):

- logging that silently drops calls;
- numbers that are guessed, estimated or filled in by hand instead of marked
  `not-reported`;
- paad changes that break or clutter paad for anyone who does not use vergil.

## 3. The `agent-cost:` stanza

One line per sub-agent call:

```text
agent-cost: v1 call=<path> tokens=<n> tool_uses=<n> duration_ms=<n> human=<n> [bail=<reason>]
```

- **Prefix and version.** Each line starts with `agent-cost: v1`, at the
  start of a line, outside any code fence it is reported in. A later change
  to the fields bumps the version; readers accept every version they know.
- **Fields** are `key=value`, separated by single spaces, in the order shown.
  Values contain no spaces. `call`, `tokens`, `tool_uses`, `duration_ms` and
  `human` are always present. `bail` appears only on a call that can bail
  (paad's specialists), and is `none` when it did not.
- **`call` is a path** of slash-separated segments, one per **dispatch**,
  rooted at the session that writes the record: for example
  `issue-manager/review-2/security`. A skill run inside a sub-agent's own
  context (the reviewer running `paad:agentic-review` with the Skill tool) is
  not a dispatch and has no segment. The parent of a call is its path less
  the last segment; nesting needs no other field. A segment matches
  `[a-z0-9][a-z0-9.-]*`. A repeated dispatch (a re-dispatched Verifier, the
  two instances of a specialist on a large diff, a paused task's
  `issue-manager` dispatched again) takes a suffix: `verifier.2`,
  `security.b`, `issue-manager.2`.
- **The root has no line of its own.** The first segment names the session
  writing the record (`issue-manager`, `epic-create`, `epic-implement`). Its
  own cost is recorded only if the session can read it (§4); a root without a
  line is not a nesting error.
- **Values come from the caller.** The caller writes the stanza from the
  sub-agent's completion notice. It never asks the sub-agent to report on
  itself. The notice's field names are not stable: the doctrine records
  `totalTokens`, `toolUses` and `durationMs`, and a current notice was seen
  giving `subagent_tokens`, `tool_uses` and `duration_ms`. The stanza's
  names are fixed; the rule doc (§4) maps the notice's names onto them, from
  the probe's findings (§11).
- **A value the notice does not give is `not-reported`.** Never estimated,
  never left out, never copied from elsewhere.
- **`human`** is the number of times the call itself stopped for a human: a
  question asked, a ruling taken, a confirmation. A sub-agent cannot ask the
  human anything, so a sub-agent's stanza is normally `human=0`. That is a
  lower bound: the caller cannot see the permission prompts a sub-agent
  raises. The rule doc says so plainly.
- **Children are relative.** A sub-agent that dispatches its own sub-agents
  and writes no records returns their stanzas in its answer with `call`
  paths relative to itself (`call=security`). Its caller prefixes each with
  the sub-agent's own path before recording it. paad uses only relative
  paths, so it needs to know nothing about vergil.
- **Each stanza is recorded once.** A sub-agent that writes its own records
  (`issue-manager` dispatched by `epic-implement`) records its children's
  stanzas on the issue, rooted at itself, and returns none. Its caller
  records only the stanza for the dispatch itself, on the same issue:
  `epic-implement/issue-manager`. The tool (§8) joins the two: on one
  issue, a root `issue-manager` is the child of the line whose last segment
  is `issue-manager`. The caller passes a repeated dispatch its suffixed
  name (`issue-manager.2`) in the dispatch prompt, and the sub-agent uses it
  as its root.
- **What `tokens` covers.** Whether a sub-agent's `totalTokens` includes its
  children's tokens is not known today. The measurement probe (§11)
  settles it. The roll-up (§8) never double counts, whichever it is.

Mode 2's `Review cost:` line is replaced by the stanza.

## 4. The logging rule

**Every sub-agent dispatch in a vergil skill logs a stanza for that call,**
written by the caller from the completion notice, recorded where that skill
already keeps its public records:

| Caller | Call | Recorded on |
| --- | --- | --- |
| `epic-implement` | each `issue-manager` dispatch | the task issue |
| `issue-manager` | each worker, each reviewer round, each responder (§7) | the task issue |
| `epic-create` | each design-review round | the planning issue, in the round comment |
| any other vergil skill | each dispatch | the issue that skill records on |

- **The rule lives in one place:** a new doc in vergil-claude-plugin
  (`docs/agent-cost-logging.md`; the name is the builder's call). It defines
  the stanza, the rule and the recording places. Skills link to it, as they
  link to the review doctrine, and do not restate it.
- **A skill with no public record** for a dispatch (no issue in play) prints
  the stanzas in its final report to the human. The lint (§9) requires each
  dispatch site to say where its stanza goes.
- **Records are read only from trusted authors** (`vrg-trusted-authors`), as
  every vergil record is.
- **The main session's own cost.** A session can read its sub-agents' costs
  but, as far as is known, not its own. The probe (§11) checks whether the
  main session's cost can be read (for example from the session transcript).
  If it cannot, the roll-up shows the top level as `not-reported`.

## 5. paad

paad is ours to change, compatibly. The changes go through the fork
`wphillipmoore/paad` and are offered upstream to `Ovid/paad`, using the fork
workflow, preview tree and branch rules of epic #699
([spec](../699-paad-unattended-mode/spec.md), the fork section).

- **The convention is generic.** paad adopts the `agent-cost:` stanza (§3) as
  its own convention for any skill that dispatches sub-agents, written so it
  stands alone without vergil.
- **agentic-review** records one stanza per specialist and per Verifier
  dispatch, including re-dispatches and large-diff instances, with each
  specialist's `bail` reason (`none` when it did not bail). The stanzas go
  in the report's Review Metadata and in the skill's result, including the
  unattended-mode result format.
- **Every other paad skill that dispatches sub-agents** does the same in its
  own report and result: agentic-owasp, agentic-architecture, agentic-a11y,
  agentic-dedup, backlog, rethink and any others the builder finds.
- **Compatibility.** The stanza is extra lines in existing metadata sections.
  No existing output field changes or moves, and a reader that ignores the
  lines sees the report it saw before.
- **Upstream timing.** vergil's own tasks do not wait on an upstream release.
  Until the change is in the installed paad, it is tried out from the
  preview tree, as in #699. The validation sample (§12) runs against
  whatever paad is installed, and records its version.

## 6. Where the stanzas appear

- **In a sub-agent's answer:** the stanzas for its own children, at the end
  of the answer, one per line.
- **In a task-issue record:** in the comment that already records the
  activity (the mode 2 findings comment for a review round, the worker's
  result comment for a worker), as a block of lines, each starting with the
  prefix, so the extraction regex is `^agent-cost: v\d+ `.
- **In the roll-ups:** §10.

## 7. The manager split

The boundary for a sub-agent is human interaction: **work we do not expect a
human to step into runs as a sub-agent**, and sub-agents are used as much as
possible. Applied to the issue skills:

### 7.1 `issue-manager`, the entry point for every task

`issue-implement` is renamed `issue-manager`. It is the single entry point for
any task, interactive or dispatched by `epic-implement`, and it takes over
every trigger the old `issue-implement`, `issue-deploy` and `issue-validate`
answered ("implement #170", "deploy #129", "validate #120", "start work on
this issue").

- It reads the task's kind (implementation, `deployment`, `validation`) and
  runs the matching worker as a sub-agent. `epic-implement` dispatches only
  `issue-manager`.
- It owns **every human touchpoint**: preflight, the worktree, preconditions
  (including attested ones such as a human-gated release), approving
  follow-up issues, rulings on findings, security holds, pauses, every
  task-issue record (including closing an operational task on
  `Outcome: SUCCESS`), and `report-ready`.
- It runs the agentic-review loop of mode 2 for implementation tasks, with
  the reviewer and the responder as sub-agents.
- It writes the stanza for every sub-agent it dispatches, and the task's
  roll-up (§10).
- Dispatched by `epic-implement`, it behaves as `issue-implement` does today:
  it never waits on the human itself but pauses and returns, and
  `epic-implement` batches what needs the human. Its own stanza is written
  by `epic-implement`.

### 7.2 The workers

Each worker is a sub-agent running its own dedicated skill. It never asks the
human anything: anything that needs a human (a proposed follow-up issue, an
unmet precondition, a problem it cannot solve) goes in its result, and the
manager takes it to the human.

| Worker | Skill | Does | Returns |
| --- | --- | --- | --- |
| implementer | `issue-implement` (the name now means only this) | writes the code in the given worktree, commits with `vrg-commit`, runs `vrg-container-run -- vrg-validate` until it passes | the commits, the validation result, proposed follow-ups, problems |
| deployer | `issue-deploy` | runs the task's agent-safe deploy steps | SUCCESS or FAILURE with the evidence, proposed follow-ups, problems |
| validator | `issue-validate` | runs the task's checklist | SUCCESS or FAILURE with the evidence, proposed follow-ups, problems |
| reviewer | mode 2 reviewer prompt (unchanged) | runs `paad:agentic-review` | paad's result, with its stanzas |
| responder | a new dedicated skill (name is the builder's call) | fresh each round: fixes the findings sorted `agent` and those the human ruled `fix`, commits, re-validates | what it did with each finding, the validation result, problems |

- **Each worker's input and output are a written contract** in its skill:
  what the dispatch prompt gives it, and the fixed form of its result. This
  is the chance to define the three task kinds more tightly than today.
- **A worker refuses a direct run.** The manager's dispatch prompt carries a
  fixed marker line. A worker run without it stops at once with:
  "This skill runs only under `issue-manager`; run `issue-manager <issue>`."
  It does no work first. The marker prevents accidents, not deliberate
  bypass.
- **Workers make no task-issue records** and close nothing. The manager
  records their results.
- **The responder** replaces the implementer's in-context fixing in mode 2
  step 7. The decisions about findings (sorting, classification, rulings)
  stay with the manager.
- **The split adds tokens.** Each worker starts cold and reads the issue and
  code again. The numbers measured are for the split flow, not today's
  single-context one; the retrospective says so.

### 7.3 The rename sweep

Every reference to the old meaning of `issue-implement` is updated: the
skills (`epic-implement`, `epic-create`, `epic-retrospective`, `pr-watch`
and others), the review doctrine (mode 2's title and text), the PR
tooling's refusal and redirect messages in vergil-tooling, the hooks, the
CLAUDE.md templates, the site docs and the `docs` repo. The plan finds
them with a search, not from memory.

## 8. The roll-up and audit tool

A `vrg-` tool in vergil-tooling (`vrg-agent-cost`; the name is the builder's
call) reads the stanzas from an issue's trusted comments.

- **Inputs:** a task issue, or an epic. For an epic it reads every child task
  and the planning issue named in the epic's `Planning:` line, so the cost of
  planning is counted.
- **Roll-up:** for each call path, the call's own numbers and the total under
  it; per activity (implementing, reviewing, responding, deploying,
  validating, design review); per task; for the epic. Counts of sub-agents
  run. Time is summed only where every `human` in it is `0`; otherwise it is
  shown as "includes waits on a human". `not-reported` values are counted
  and shown, never treated as zero.
- **No double counting.** If the probe (§11) finds that a sub-agent's tokens
  include its children's, the tool derives each call's own share by
  subtraction; otherwise it sums. Either way the total is counted once.
- **Audit, at two levels:**
  - **errors** (exit non-zero, one line each): a record that implies a
    dispatch but carries no stanza; a call path whose parent is neither a
    line nor the root (§3); a malformed stanza; an unknown version;
  - **gaps** (reported and counted, exit zero): each `not-reported` value.
    An honest gap is the anti-goal's alternative to a guessed number, not a
    failure.
- **What implies a dispatch:** the records with fixed first lines that the
  skills already post (mode 1 round comments, mode 2 findings comments, the
  worker result comments of §7). The rule doc lists them, and the tool reads
  that list from one place.
- **Cutover.** The rule applies to records made by the plugin release that
  ships it, and later ones. The tool treats an issue as in scope when it has
  any record made under that release. Records before the cutover are
  reported as `pre-rule`, never as errors. An epic that straddles the
  cutover, as this one does, gets a partial cost summary that says so.
- **Output:** human-readable by default, `--json` for machines.
- **No silent failures:** a comment it cannot parse as a stanza but that
  starts with the prefix is an audit error, not skipped.

## 9. The static lint

A check inside `vrg-validate` (vergil-tooling) for repos that hold skills
(vergil-claude-plugin; paad gets the equivalent in its own test style if it
has one, otherwise in its contribution checklist):

- finds every sub-agent dispatch site in the skill files;
- fails any site that neither links the logging rule (§4) nor says where its
  stanza is recorded;
- allows an exception only through an explicit, greppable marker with a
  reason.

How dispatch sites are recognised is the builder's call. It must catch the
forms used today (the Agent tool, "subagent", "sub-agent", "dispatch") and
be tested against the current skills.

## 10. Roll-ups in the records

- **Per task.** `issue-manager` posts the task's roll-up as the last record
  before `report-ready` (or before closing an operational task): every call,
  with totals per activity. For an implementation task it extends mode 2's
  `agentic-review summary` comment, whose `Review tokens` line becomes the
  full roll-up. The tool (§8) produces it.
- **Per epic.** `epic-retrospective` gains a step: run the tool over the
  epic and post the result as a final cost-summary comment on the epic
  issue. The retrospective doc links to that comment and does not repeat the
  numbers.

## 11. The measurement probe

The first implementation task establishes, by experiment, what the design
assumes. Each is recorded with evidence:

1. the fields a completion notice gives, foreground and background, and
   their exact names, so the rule doc can map them onto the stanza (§3);
2. whether a sub-agent's `totalTokens` includes its children's, checked
   with a sub-agent that dispatches one of its own;
3. what `totalTokens` counts (input, output, cache reads and writes);
4. whether the main session can read its own cost;
5. whether a sub-agent's dispatch can raise anything the human sees, and
   whether the caller can count it.

If a finding contradicts this spec, the spec is amended before the tasks that
depend on it start.

## 12. The validation sample

A closing `validation` task, blocked by every implementation task:

- run a real sample through the new flow: at least three docs-only tasks and
  three code tasks (the count is my call, inside "a few"), drawn from real
  queued work;
- check that every dispatch left a stanza, that the paths nest, and that the
  audit (§8) is clean;
- record each task's roll-up and a comparison of docs-only against code,
  including per-specialist cost and bail, so #812 can be revived against
  its `Revisit when:` conditions.

It passes only when the audit reports no errors. Gaps (`not-reported`) are
allowed only where the probe showed the value cannot be read (the root
session's own cost, for example); any other gap is a failure, fixed before
the epic closes.

## 13. Placement

Each task is filed in the repo where its PR lands: the rule doc, skills and
doctrine in vergil-claude-plugin; the tool, the lint, the PR-tooling
messages and the site docs in vergil-tooling. The paad changes are made in
the fork, which is outside the org. As in #699 (its spec §7), the fork tasks
are filed in `vergil-project/.github` under this epic and closed by a comment
naming the fork commits and their test result; they refer to the upstream
PRs and never link across orgs.

## 14. Left to the builder

- the names of the rule doc, the responder skill and the tool;
- how the lint recognises a dispatch site;
- the exact wording of each worker's contract, within §7.2;
- the order of the tasks, except that the probe comes first and the
  validation sample last.
