# Measuring agent cost

- **Epic:** vergil-project/.github#816
- **Planning issue:** vergil-project/.github#802 (review history is in its
  comments)
- **Status:** approved
- **Repos:** vergil-claude-plugin (the logging rule, the ruling protocol, the
  skills, the review doctrine), vergil-tooling (the roll-up and audit tool,
  the static lint, the comment tooling, the site docs), the paad fork
  `wphillipmoore/paad` (paad's own logging, offered upstream to `Ovid/paad`),
  `.github` (this spec, the plan and the retrospective)

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

The records themselves are also weaker than the numbers need. Comments are
edited after posting (`--edit-last`, the "unfinished comment" state), and a
human's decision often reaches the agent that acts on it as text relayed
through a prompt, so the issue does not show who decided what. Once issue
comments carry measurements, they are data, and they need the same
integrity as the numbers.

## 2. Goals, non-goals, anti-goals

**Goals.**

1. Every sub-agent call vergil makes logs its cost in one simple, parseable
   stanza.
2. paad logs the same stanza for every sub-agent it dispatches, so the numbers
   nest down to agentic-review's specialists.
3. The cost of each activity is measured separately: implementing, reviewing,
   responding to review, deploying and validating. This needs `issue-implement`
   split into a manager and workers (§8).
4. Roll-ups per task and per epic, and an audit that shows where the rule was
   not followed.
5. Records that can be trusted as data: comments are written once, and every
   human ruling is posted on the issue by the human (§7).
6. A real sample, measured end to end, at the close of the epic.

**Tokens are the primary metric.** Elapsed time is recorded, but it absorbs
every wait on a human, so it is secondary. The task's roll-up counts the
human interactions the root session had (ruling requests and pauses), so a
reader knows how far to trust its time. A sub-agent's time is labelled as
possibly including permission waits the caller cannot see.

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

One line per sub-agent call, or per turn of a continued call:

```text
agent-cost: v1 call=<path> [turn=<n>] tokens=<n> tool_uses=<n> duration_ms=<n> [bail=<code>] [lost]
```

- **Prefix and version.** Each line starts with `agent-cost: v1`, at the
  start of a line. A later change to the fields bumps the version; readers
  accept every version they know.
- **Fields** are `key=value`, separated by single spaces, in the order shown.
  Values contain no spaces. `call`, `tokens`, `tool_uses` and `duration_ms`
  are always present. `turn` appears only on a continued call (below).
  `bail` appears only on a call that can bail (paad's specialists), and is
  `none` when it did not. `lost` is a bare marker, only on a lost call
  (below).
- **`tokens`.** If the probe (§12) finds that a completion notice breaks
  tokens down (input, output, cache reads and writes), the breakdown is
  recorded as extra fields after `tokens`, named in the rule doc. If the
  notice gives only a total, the rule doc and the roll-ups say exactly what
  that total counts.
- **`call` is a path** of slash-separated segments, one per **dispatch**,
  rooted at the session that writes the record: for example
  `issue-manager.1/review-2/security`. A skill run inside a sub-agent's own
  context (the reviewer running `paad:agentic-review` with the Skill tool) is
  not a dispatch and has no segment. The parent of a call is its path less
  the last segment; nesting needs no other field.
- **Segments come from a closed vocabulary**, defined in the rule doc (§4):
  `issue-manager.<k>`, `implement`, `review-<n>`, `respond-<n>`, `deploy`,
  `validate`, `<stage>-<n>` for design-review rounds, and a fixed slug for
  each paad lens and the Verifier. A segment matches `[a-z0-9][a-z0-9.-]*`.
  An unknown segment is an audit error.
- **Numbering.** A fresh run of something known to iterate takes a plain
  digit index: review rounds (`review-1`, `review-2`), responder rounds,
  design-review rounds. A fresh agent taking over from one that can no
  longer be continued takes a `.2` suffix (`implement.2`, `verifier.2`).
  Partitions of one paad lens on a large diff take `.a`/`.b`
  (`logic.a`, `logic.b`), every instance suffixed. A `.<n>` suffix implies
  its predecessor (`implement.2` implies `implement`), so a predecessor
  with no line is an audit error.
- **A lost call still gets a line.** When a call's completion notice never
  arrives (the worker cannot be continued because the session that started
  it ended, say), whoever takes over writes its stanza with every value
  `not-reported` and the marker `lost`, before dispatching the successor.
- **Continued calls keep their path.** A sub-agent that ended its turn and
  was continued with SendMessage is the same call. Each of its completion
  notices gets its own line, with the same path and `turn=<n>` from 2 up.
  Whether a later turn's tokens are cumulative or only the new part is
  settled by the probe (§12); the roll-up never double counts.
- **Every `issue-manager` run has a unique root**, `issue-manager.<k>`, with
  `k` one past the highest root already on the issue, starting at 1. When
  `epic-implement` dispatches a manager it passes `k` in the dispatch
  prompt. A full path recorded twice on one issue (other than as later
  turns) is an audit error.
- **The root has no line of its own.** The first segment names the session
  writing the record (`issue-manager.<k>`, `epic-create`, `epic-implement`).
  Its own cost is recorded only if the session can read it (§4); a root
  without a line is not a nesting error.
- **Values come from the caller.** The caller writes the stanza from the
  sub-agent's completion notice. It never asks the sub-agent to report on
  itself. The notice's field names are not stable: the doctrine records
  `totalTokens`, `toolUses` and `durationMs`, and a current notice was seen
  giving `subagent_tokens`, `tool_uses` and `duration_ms`. The stanza's
  names are fixed; the rule doc maps the notice's names onto them, from the
  probe's findings.
- **A value the notice does not give is `not-reported`.** Never estimated,
  never left out, never copied from elsewhere.
- **`bail` is a code**, from a short vocabulary that follows paad's own
  `BAIL:` conventions, so that prose reasons fit the no-spaces rule.
- **Children are relative.** A sub-agent that dispatches its own sub-agents
  and writes no records returns their stanzas with `call` paths relative to
  itself (`call=security`). Its caller prefixes each with the sub-agent's
  own path before recording it. paad uses only relative paths, so it needs
  to know nothing about vergil.
- **Each stanza is recorded once.** A sub-agent that writes its own records
  (`issue-manager.<k>` dispatched by `epic-implement`) records its children's
  stanzas on the issue, rooted at itself, and returns none. Its caller
  records only the stanza for the dispatch itself, on the same issue:
  `epic-implement/issue-manager.<k>`. The manager's first record on the
  issue says `dispatched as issue-manager.<k> by epic-implement`, and the
  audit (§9) then expects a matching cost record from `epic-implement`.

Mode 2's `Review cost:` line is replaced by the stanza.

## 4. The logging rule

**Every sub-agent dispatch in a vergil skill logs a stanza for that call,**
written by the caller from the completion notice, recorded where that skill
already keeps its public records:

| Caller | Call | Recorded on |
| --- | --- | --- |
| `epic-implement` | each `issue-manager` dispatch | the task issue |
| `issue-manager` | each worker, each reviewer round, each responder (§8) | the task issue |
| `epic-create` | each design-review round | the planning issue |
| any other vergil skill | each dispatch | the issue that skill records on |

- **The rule lives in one place:** a new doc in vergil-claude-plugin
  (`docs/agent-cost-logging.md`; the name is the builder's call). It defines
  the stanza, the segment vocabulary, the bail codes, the mapping from the
  notice's fields, the rule and the recording places. Skills link to it, as
  they link to the review doctrine, and do not restate it.
- **A skill with no public record** for a dispatch (no issue in play) prints
  the stanzas in its final report to the human. The lint (§10) requires each
  dispatch site to say where its stanza goes.
- **Records are read only from trusted authors** (`vrg-trusted-authors`), as
  every vergil record is.
- **The main session's own cost.** A session can read its sub-agents' costs
  but, as far as is known, not its own. The probe (§12) checks whether the
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
  dispatch, including re-dispatches and large-diff partitions, with each
  specialist's `bail` code (`none` when it did not bail). The stanzas go in
  the report's Review Metadata and in the skill's result, including the
  unattended-mode result format.
- **Every other paad skill that dispatches sub-agents** does the same in its
  own report and result: agentic-owasp, agentic-architecture, agentic-a11y,
  agentic-dedup, backlog, rethink and any others the builder finds.
- **Compatibility.** The stanza is extra lines in existing metadata sections.
  No existing output field changes or moves, and a reader that ignores the
  lines sees the report it saw before.
- **The unattended result.** Every review round runs through
  `paad:paad-delegate`, which writes no report, so the caller gets paad's
  stanzas only from the delegate's structured result. Each dispatching
  skill's unattended result gains a skill-specific field carrying them.
- **Deployment.** This environment always runs the fork's paad preview. A
  paad change counts as deployed once the preview carries it and the
  paad-policy §10 gate has passed on it; a deployment task in this epic
  records both. The validation sample (§13) records which
  preview commit it ran against. The upstream PRs are an offer, and no task
  waits on them.

## 6. Where the stanzas appear

- **In a sub-agent's answer:** the stanzas for its own children, at the end
  of the answer, one per line.
- **In a cost record on an issue:** a standalone, write-once comment, posted
  after the step it measures. Its first line names the step
  (`Agent cost: review round 2`, `Agent cost: implement`); the rest is the
  stanza lines, each starting with the prefix, so the extraction regex is
  `^agent-cost: v\d+` followed by a space. A cost record never shares a
  comment with another record. Stanzas are read only from cost records: a
  stanza-shaped line in any other comment (a code excerpt in a findings
  comment, say) is ignored.
- **In the roll-ups:** §11.

## 7. Write-once records and the ruling protocol

### 7.1 Write-once records

An issue's comments are a running log of how it evolved. Every comment a
vergil skill or tool posts is written once: never edited, never deleted.
`--edit-last`, `--delete-last` and the "unfinished comment" state they
serve are removed from the skills.

- **A correction is a new comment.** A superseding record links to the
  comment it replaces and says that it supersedes it, so the change reads
  as a delta in the log rather than in an edit history. The tool (§9)
  reads a superseded record as replaced, never as a second one.
- **Edits are detected.** GitHub records each comment's edits. An edited
  agent record (a cost record, a ruling request, any record of §9) is an
  audit error. Humans may edit their own comments, to fix a typo say, but
  an edited ruling is not acted on (§7.2).

### 7.2 The ruling protocol

Every decision `issue-manager` takes from the human (rulings on findings,
approving follow-up issues, attested preconditions, security holds, what to
do after a pause) goes through the issue:

- **The request.** The manager posts a ruling-request comment with a fixed
  first line, one ID per item (`R2.3`) and the default answer for each.
- **The reply.** The human replies on the issue, from their own account:
  `approve` takes every default; `deny` comes with their answer; or they
  answer per item (`R2.3: file`).
- **Reading it.** The agent reads a ruling only from a comment by the human's
  account, posted after the request. A reply that is ambiguous, partial,
  out of order or edited is never guessed at: the agent posts a fresh
  request, saying why, and the human answers in a new comment.
- **Rulings are never relayed through prompts.** When work resumes, the
  agent that acts is told only that the human has responded on the issue,
  and reads the ruling there itself.
- **Security items** are ruled on by ID or count only (`R2.S1: fix`); the
  ruling never carries their substance.
- **`epic-implement` relays nothing.** It batches what needs the human,
  points the human at each issue's ruling request, and continues each
  manager once the human has replied there.
- **The agent never posts as the human, checked every session.** Before it
  posts a ruling request or reads a ruling, the session confirms that it
  posts as one of the agent logins `vrg-trusted-authors` lists, never the
  human login. If it does not, the protocol refuses: the run pauses and
  names the fix, and no ruling is requested or read.

A human interaction, for the roll-up's count (§2), is one ruling request or
one pause.

## 8. The manager split

The boundary for a sub-agent is human interaction: **work we do not expect a
human to step into runs as a sub-agent**, and sub-agents are used as much as
possible. Applied to the issue skills:

### 8.1 `issue-manager`, the entry point for every task

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
  follow-up issues, rulings on findings, security holds and the judgement of
  what to hold back, pauses, every task-issue record (including closing an
  operational task on `Outcome: SUCCESS`), and `report-ready`. Each decision
  it takes from the human goes through the ruling protocol (§7).
- It runs the agentic-review loop of mode 2 for implementation tasks, with
  the reviewer and the responder as sub-agents.
- It writes the cost record for every sub-agent it dispatches, and the
  task's roll-up (§11).
- Dispatched by `epic-implement`, it behaves as `issue-implement` does today:
  it never waits on the human itself but pauses and returns, and
  `epic-implement` batches what needs the human. Its own stanza is written
  by `epic-implement`.

### 8.2 The workers

Each worker is a sub-agent running its own dedicated skill. Anything that
needs a human (a proposed follow-up issue, an unmet precondition, a problem
it cannot solve) makes the worker end its turn and return the question. The
manager posts the ruling request (§7).

| Worker | Skill | Does | Returns |
| --- | --- | --- | --- |
| implementer | `issue-implement` (the name now means only this) | writes the code in the given worktree, commits with `vrg-commit`, runs `vrg-container-run -- vrg-validate` until it passes | the commits, the validation result, proposed follow-ups, problems |
| deployer | `issue-deploy` | runs the task's agent-safe deploy steps | SUCCESS or FAILURE with the evidence, proposed follow-ups, problems |
| validator | `issue-validate` | runs the task's checklist | SUCCESS or FAILURE with the evidence, proposed follow-ups, problems |
| reviewer | mode 2 reviewer prompt (unchanged, verbatim) | runs `paad:agentic-review`, fresh every round | paad's result |
| responder | a new dedicated skill (name is the builder's call) | fresh each round: fixes the findings sorted `agent` and those the human ruled `fix`, commits, re-validates | what it did with each finding, the validation result, problems |

- **Resuming after a ruling.** Once the human has replied, the manager
  continues the same worker with SendMessage, telling it only that the human
  has responded on the issue; the worker reads the ruling there (§7). A
  fresh worker (`implement.2`) takes over only when the original can no
  longer be continued, for example because the session that started it has
  ended. It too reads the ruling from the issue. Reviewers are never
  continued: the doctrine requires a fresh one every round.
- **paad's stanzas come from the delegate's result.** The reviewer prompt
  stays verbatim. The manager reads paad's stanzas from the new
  skill-specific field of the delegate's result (§5), matching them with the
  strict stanza regex only, and cross-checks them against paad's `Agents
  dispatched:` list. A malformed line, or a lens with no stanza, is a failed
  output check, handled like any other.
- **Security fixes.** Findings held back as security reach the responder
  only as paths to their details files in the state directory, each with a
  hash the responder checks before reading. The responder reports on them
  by ID only. The manager records security items as counts only, and never
  copies a worker's result into a record verbatim.
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
- **The split adds tokens.** Each fresh worker starts cold and reads the
  issue and code again. The baseline (§12) measures today's flow, so the
  retrospective can compare the two.

### 8.3 The doctrine's mode 2 rewrite

Section 3 of the review doctrine has the implementer do everything: fix,
re-validate, pause on repairs that need the human (step 7), judge what to
hold back (§3.6), and pause in the cases of §3.8. After the split those jobs
belong to the manager, the responder and the ruling protocol. A dedicated
task rewrites, for the new roles and records:

- mode 2's steps 1–7 and §3.6–3.8;
- §3.5 and §1.7: the delete-and-repost and unfinished-comment paths become
  superseding records (§7.1);
- §1.9 and §3.4: the new stanza field's place in the result, the strict
  stanza regex and the cross-check against `Agents dispatched:`.

It is separate from the rename sweep.

### 8.4 The rename sweep

Every other reference to the old meaning of `issue-implement` is updated:
the skills (`epic-implement`, `epic-create`, `epic-retrospective`,
`pr-watch` and others), the doctrine's mode 2 title and remaining text, the
PR tooling's refusal and redirect messages in vergil-tooling, the hooks, the
CLAUDE.md templates, the site docs and the `docs` repo. The plan finds them
with a search, not from memory.

## 9. The roll-up and audit tool

A `vrg-` tool in vergil-tooling (`vrg-agent-cost`; the name is the builder's
call) reads the cost records (§6) from an issue's trusted comments, and
no other comment.

- **Inputs:** a task issue, or an epic. For an epic it reads every child task
  and the planning issue named in the epic's `Planning:` line, so the cost of
  planning is counted.
- **Roll-up:** for each call path, the call's own numbers and the total under
  it, with continued turns combined; per activity (implementing, reviewing,
  responding, deploying, validating, design review); per task; for the epic.
  Counts of sub-agents run, and of human interactions (§7). Sub-agent time
  is labelled as possibly including unseen permission waits. `not-reported`
  values are counted and shown, never treated as zero.
- **No double counting.** If the probe (§12) finds that a sub-agent's tokens
  include its children's, or that a later turn's include earlier turns', the
  tool derives each share by subtraction; otherwise it sums. Either way the
  total is counted once.
- **Audit, at two levels:**
  - **errors** (exit non-zero, one line each): a record that implies a
    dispatch with no matching cost record; a call path whose parent is
    neither a line nor the root (§3); a suffixed path whose predecessor has
    no line; an unknown segment; a full path recorded twice; a dispatched
    manager with no matching `epic-implement` line; a malformed stanza; an
    unknown version; an edited agent record (§7.1);
  - **gaps** (reported and counted, exit zero): each `not-reported` value,
    and each `lost` call. An honest gap is the anti-goal's alternative to a
    guessed number, not a failure.
- **What implies a dispatch:** the records with fixed first lines that the
  skills already post (mode 1 round comments, mode 2 findings comments, the
  worker result comments of §8). The rule doc lists them, and the tool reads
  that list from one place. A superseded record (§7.1) implies nothing; its
  superseding record does.
- **Cutover.** An issue with no cost records is `pre-rule`: reported, never
  an error. An issue with any cost record is in scope. An epic that
  straddles the cutover, as this one does, gets a partial cost summary that
  says so. The residual gap is that a run which logs nothing at all looks
  pre-rule; the lint (§10) and the validation sample (§13) are what catch
  it.
- **Output:** human-readable by default, `--json` for machines. The tool's
  output is the authoritative total.
- **No silent failures:** a line in a cost record that starts with
  `agent-cost: v` but does not parse as a stanza is an audit error, not
  skipped.

## 10. The static lint

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

## 11. Roll-ups in the records

- **Per task.** `issue-manager` posts the task's roll-up as the last record
  before `report-ready` (or before closing an operational task): every call,
  with totals per activity and the count of human interactions. For an
  implementation task it extends mode 2's `agentic-review summary` comment,
  whose `Review tokens` line becomes the full roll-up. The tool (§9)
  produces it. A dispatched manager's own cost is recorded by
  `epic-implement` after this roll-up, and the roll-up says so.
- **Per epic.** `epic-retrospective` gains a step: run the tool over the
  epic and post the result as a final cost-summary comment on the epic
  issue. The summary says it excludes the retrospective task itself, which
  is still running when it is posted. The retrospective doc links to that
  comment and does not repeat the numbers.

## 12. The measurement probe and the baseline

The first implementation task establishes, by experiment, what the design
assumes. Each is recorded with evidence:

1. the fields a completion notice gives, foreground and background, and
   their exact names, so the rule doc can map them onto the stanza (§3);
2. whether a sub-agent's tokens include its children's, checked with a
   sub-agent that dispatches one of its own;
3. what the token count covers (input, output, cache reads and writes), and
   whether the notice breaks it down;
4. whether a call continued with SendMessage produces its own completion
   notice, and whether its tokens are cumulative or only the new turn's;
5. whether the main session can read its own cost;
6. whether a sub-agent's dispatch can raise anything the human sees, and
   whether the caller can count it;
7. nesting at the split's real depth: main session, manager, delegate,
   analyst;
8. whether a continued sub-agent can continue its own sub-agents;
9. whether `paad:paad-delegate` can be dispatched from a sub-agent.

If a finding contradicts this spec, the spec is amended before the tasks that
depend on it start; a negative result on items 7–9 amends §8 before the
manager split or the rename sweep starts.

**The baseline.** Once the rule doc and the stanza logging have landed on
today's skills, and before the manager split, a small baseline sample of
real tasks is run on today's flow and its roll-ups recorded. The validation
sample (§13) is compared against it, so the epic can show what the split
cost or saved. The baseline measures per-specialist cost and bail, so #812
may be revived from it without waiting for the validation sample.

## 13. The validation sample

A closing `validation` task, blocked by every implementation task:

- run a real sample through the new flow: at least three docs-only tasks and
  three code tasks, drawn from real queued work;
- check that every dispatch left a stanza, that the paths nest, that the
  rulings went through the protocol (§7), and that the audit (§9) is clean;
- record each task's roll-up, the paad preview commit it ran against, a
  comparison of docs-only against code, including per-specialist cost and
  bail, and a comparison against the baseline (§12), so #812 can be revived
  against its `Revisit when:` conditions.

It passes only when the audit reports no errors. Gaps are allowed only for
a `lost` call, or where the probe showed the value cannot be read (the root
session's own cost, for example); any other gap is a failure, fixed before
the epic closes.

## 14. Placement

Each task is filed in the repo where its PR lands: the rule doc, the ruling
protocol, skills and doctrine in vergil-claude-plugin; the tool, the lint,
the comment tooling, the PR-tooling messages and the site docs in
vergil-tooling. The paad changes are made in the fork, which is outside the
org. As in #699 (its spec §7), the fork tasks are filed in
`vergil-project/.github` under this epic and closed by a comment naming the
fork commits and their test result; they refer to the upstream PRs and never
link across orgs.

## 15. Left to the builder

- the names of the rule doc, the responder skill and the tool;
- the slugs, the bail codes and the ruling-request wording, within §3 and
  §7;
- how the lint recognises a dispatch site;
- the exact wording of each worker's contract, within §8.2;
- the order of the tasks, except that the probe comes first, the logging and
  the baseline come before the manager split, and the validation sample
  comes last.
