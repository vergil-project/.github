# model-tiering: per-skill model selection — design

- **Epic:** vergil-project/.github#1823
- **Planning issue:** vergil-project/.github#1822 (review history is in its
  comments)
- **Status:** approved

## 1. Intent

The fleet now works fast enough to use up the weekly token allowance of the
Claude subscription: it ran out on the evening of Wednesday 2026-11-25. Every
skill and agent runs on Opus, the most capable and most expensive model,
whatever the work. Nobody chose that on purpose. Two agents pin a model
(`vergil:bootstrap` on `haiku`, `vergil:merge-reviewer` on `opus`), and
everything else inherits whatever model the session happens to use.

The workflow puts its judgment up front: `epic-create` produces detailed
specs and plans, and implementations then usually pass on the first try. So
the expensive model is needed where the thinking happens, and much less where
a well-specified plan is carried out. This epic sets the model **by design**
for every piece of the fleet that can carry one. Opus stays where the big
thinking is (`epic-create` and its design reviews). Sonnet drives most of
the work. Haiku writes the changes, code and prose alike.

The constraint is a good one: a fixed budget forces efficient use, which is
worth having for its own sake. This is a first pass, judged by judgment
rather than metrics. The human runs with it for a few weeks, watches the
weekly usage, and then decides whether to iterate.

## 2. Goals, non-goals and anti-goals

### 2.1 Goals

1. **Every subagent the fleet dispatches runs on a model chosen by design**,
   recorded in its agent definition or its dispatch, never inherited from
   the session by accident.
2. **The `issue-implement` split.** A Haiku-pinned writer agent writes the
   code. A Sonnet-pinned driver runs the rest of the procedure (§5).
3. **A written model map** in the plugin's site docs: which model each skill
   and agent runs on, why, and how to change it (§6).
4. **A wave-2 roadmap**: the skills worth splitting next, each proposed as
   a follow-on planning item and filed as the human approves (§7).
5. **Guards** that keep the pins from drifting: plugin CI fails on an
   agent definition with no `model:`, and a hook refuses a task run
   dispatched as any agent type but `vergil:task-runner` (§5.5).

### 2.2 Non-goals

- Token-usage metrics. vergil-project/.github#1599 (token logging at
  epic-create checkpoints) stays separate. The success signal for this pass
  is the human's weekly usage.
- Splitting any skill other than `issue-implement` (wave 2, §7).
- Removing skills that are no longer needed. That is a separate concern.
- Changing which model the human's own session uses. Opus stays the default
  when the human talks to the fleet.

### 2.3 Anti-goals

- A noticeable drop in first-pass success under `epic-implement`. If one
  shows up, each pin is a one-line revert (§6).
- A writer model that silently produces weaker code. The writer validates
  to green and the merge review still gates every change (§5.3).
- Pins that live only in prose a model may forget. Each pin sits in an agent
  definition wherever one can hold it.

## 3. How a model can be pinned

These are facts from the Claude Code documentation, checked on 2026-11-27:
[skills](https://code.claude.com/docs/en/skills) and
[sub-agents](https://code.claude.com/docs/en/sub-agents).

- **A skill's frontmatter `model:` is weak.** It sets the model "for the
  turn when you run this skill with `/name`". The docs also say: "When
  Claude runs the skill, the session keeps its model." A skill invoked by
  another skill, or inside a subagent, ignores it.
- **A skill with `context: fork`** runs in a new subagent, and its `model:`
  becomes that subagent's model. The subagent has no conversation history
  and runs in the background by default, so it cannot hold a conversation
  with the human.
- **An agent definition's `model:`** (an alias such as `sonnet`, `opus` or
  `haiku`, a full model ID, or `inherit`) pins the subagent reliably.
- **Precedence**, first match wins: the Agent call's per-invocation `model`
  parameter, then the agent definition's `model:`, then the
  `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, then the parent's
  model.
- **Nesting.** A subagent may spawn subagents up to three levels below the
  main conversation, so main session → task driver → writer or reviewer
  fits.

What this means for the design: a model can be pinned only on a piece that
runs as a subagent. Interactive skills run in the human's session and keep
its model. So "pin everything" means "every piece worth pinning runs as a
pinned agent", and that is also the mechanism behind every split.

## 4. Wave 1: the largest model each skill needs

The rule: each skill runs on the smallest model that can do its hardest
part. These assignments are judgment, not measurement.

| Skill or agent | How it runs | Its hardest part | Model |
| --- | --- | --- | --- |
| `epic-create` | human session, interactive | design judgment | Opus (session) |
| `paad:paad-delegate` (design-review rounds) | subagent, no pin today | adversarial design review | **Opus**, passed by the dispatch |
| `steering-audit` | human session, interactive | judging steering files | Opus (session) |
| `epic-implement` | human session, long-running | dispatch, gates, batching | session model |
| `issue-implement` driver | subagent, no pin today | the long procedure | **Sonnet** (new `vergil:task-runner`) |
| code writer | new | writing code | **Haiku** (new `vergil:implementer`) |
| `merge-review` | inside the driver | rulings, the fix loop | Sonnet (driver) |
| `vergil:merge-reviewer` | subagent, pinned `opus` | is this good enough to merge | **Sonnet** |
| `pr-watch` | inside the driver | diagnosing CI failures | Sonnet (driver) |
| `issue-deploy`, `issue-validate` | subagent, no pin today | preconditions, judging an outcome | **Sonnet** (`vergil:task-runner`) |
| `fixture-run` | agent-invoked, in its caller's session | rule-following task filing | caller's model |
| `epic-retrospective`, `hoshin`, `epic-review`, `summarize`, `deprecation-triage`, `intake-capture` | human session | various | session model |
| `vergil:bootstrap` | subagent, pinned `haiku` | checks | Haiku (unchanged) |

Notes:

- **Skills the human runs stay on the session's model**, and the model map
  says so explicitly. The analysis judged several of them (`epic-review`,
  `hoshin`, `summarize`, `epic-retrospective`) to need no more than Sonnet,
  and `intake-capture` no more than Haiku. Moving them needs a fork or a
  split, so they go to the wave-2 roadmap (§7), not to frontmatter `model:`,
  which lasts one turn (§3).
- **`fixture-run` is not a human-run skill.** The agent runs it, not the
  human: it files the fixture trios that put a change through the whole
  cycle on the fixture repos, the fleet's end-to-end quality check, and
  `epic-implement` then drives each trio. It runs on the model of the
  session that invokes it, and needs no more than Sonnet. With no
  conversation with the human to keep, it is a direct candidate for a
  pinned agent or a forked skill (§7).
- **The merge reviewer moves from Opus to Sonnet.** That is the one change
  in this pass that lowers a model on purpose. It stays a tier above the
  Haiku writer whose work it reviews. It runs on Sonnet for every PR, the
  docs PRs of `epic-create`, `epic-retrospective` and `hoshin` included,
  with no exception by caller or by change class: a spec and plan reach
  merge review only after their Opus design review and the human's own.
- **`epic-implement`'s own session** mostly waits on the subagents it
  dispatches, so it costs little on any model. It is not pinned. The model
  map notes that the human may run it on Sonnet with `/model`.

## 5. The `issue-implement` split

### 5.1 Approaches considered

- **A. A Sonnet driver with a Haiku writer (chosen).** The task subagent
  stays the driver, pinned to Sonnet. It runs preflight, merge review,
  handoff and watch as it does today, and dispatches a Haiku writer for the
  code. This moves the heaviest token work to Haiku at one clean boundary,
  keeps the long procedure on a model that follows it reliably, and leaves
  resume, batch mode and rulings unchanged.
- **B. A pipeline of pinned phases.** `epic-implement` dispatches each phase
  (preflight, write, merge review, handoff, watch) as its own pinned
  subagent, with state handed on through the worktree and the issue. This
  would save the most, but it redesigns resume, batch mode and the rulings
  across five handoffs, so in practice it rewrites three skills. The human
  prefers B as the end state. **It is held as a follow-on epic** that
  optimizes this design (§7).
- **C. Haiku drives everything** and calls Sonnet only for merge review and
  CI diagnosis. Rejected: it puts the long procedure on the weakest model,
  and following that procedure, not writing code, is the real risk for a
  small model.

### 5.2 The agents

- **`vergil:task-runner`**, a new agent pinned `model: sonnet`.
  `epic-implement` dispatches every `issue-implement`, `issue-validate` and
  `issue-deploy` run as this agent type. Its body says only that it runs the
  routed skill under the Agent prompt contract. The skills carry the
  procedure.
- **`vergil:implementer`**, a new agent pinned `model: haiku`. Its job: in a
  given worktree, carry out a given change (any change to the task's
  tracked files, code, docs and config alike, §5.3), commit it in small
  `vrg-commit` commits, and validate with `vrg-container-run -- vrg-validate`
  until green.
  It never suppresses a gate, never submits, pushes or posts on GitHub, never
  creates issues and never runs merge review. It returns one of two
  results: `green` (with its commits), or `stuck` (what it tried and what
  still fails). It returns `stuck` once five `vrg-validate` runs in one
  dispatch have all exited non-zero, judged by the exit code alone. Both
  carry `notes`: any
  deviation from the issue and why,
  and any concern it noticed (a gap in the issue, a defect outside the
  change), which the driver screens and handles as follow-ups, as it does a
  concern any sub-agent reports today.

### 5.3 The driver's use of the writer

- **Implement.** The driver dispatches the writer with the issue as
  `vrg-issue-read` prints it (the body and the trusted comments, whose scope
  changes are binding), any rulings the run is carrying out, the worktree
  path and the commit conventions (a batch's first commit for each issue, then fixups of it).
  The driver checks the result for itself (the branch's commits, a clean
  tree, a green validate) before merge review. A fabricated `green` must
  not reach merge review.
- **Fixes.** Code fixes asked for by merge review's fix step, and by
  pr-watch's fixes for failing CI and review comments, go through the same
  writer (conflicts and security fixes are below). Only the code goes to the
  writer. The driver still diagnoses the problem and runs every check and
  push.
- **Who has a writer.** A caller has a writer only under an
  `issue-implement` driver: `issue-implement` itself, and `pr-watch` when
  that driver runs it. `pr-watch` run by the docs-PR skills (`epic-create`,
  `epic-retrospective`, `hoshin`), or started by the human outside an
  epic run, has none and makes its fixes in the session, as today.
- **`epic-implement`'s re-entries.** `epic-implement`'s main session never
  runs `pr-watch` on a task's PR, and never submits a task, itself. When
  the human reports that a task's PR did not merge or asks for it to be
  fixed, when a reported-ready task with an open PR has no run watching it,
  and when a reported-ready task's session was lost before submit, it
  dispatches a task run of `issue-implement`, whose Preflight resumes at
  submit and watch. So every fix to a task's PR has the writer.
- **merge-review's fix step.** `merge-review`'s "Fix and check" steps 1 and
  2 (fix every defect and carry out every ruling, then validate to green)
  take the writer as an input from the caller. When the caller has a
  writer, those steps hand only the file edits of step 1, and step 2's
  validation, to `vergil:implementer`, with the reviewer's findings and the
  rulings in its prompt, and the driver then runs the check. Every filing
  step 1 calls for (issues and security records) stays with the driver,
  as does every security-item fix (below). When the caller has no writer,
  the caller's session makes the fix, as today.
- **Security fixes.** The driver makes the fix for a security item itself
  and never passes the item's details to the writer.
- **Rebase conflicts.** The driver resolves every conflict of a stopped
  `vrg-git rebase` itself, a resolution that needs new logic included, and
  continues the rebase. This is an exception to the writer rule, since a
  conflict can only be resolved inside the stopped rebase. Merge review
  gates the resolved conflict as it gates any fix.
- **Escalation.** A writer failure is a `stuck` result, or a `green` that
  fails the driver's check. On the first, the driver dispatches the writer
  once more with a per-invocation `model: sonnet`, passing on what the
  first attempt found or what the check found. The retry is per writer
  dispatch, shared by both kinds of failure: each of a run's dispatches
  (the implementation, a merge-review fix, a CI fix) gets one. A second
  failure of either kind on the same dispatch stops the run for the human,
  as a problem the run cannot resolve does today.
- **Resume.** A dirty task worktree on a re-dispatch is sorted by owner
  before anything is dispatched. A rebase in progress, and uncommitted
  work from the driver's own exceptions (below), stay with the driver,
  which finishes them; the driver marks such work before it starts it, so
  the marker alone decides ownership. Everything else is the writer's
  interrupted work, and the writer is dispatched to finish it.
- **The writer rule.** `issue-implement` alone is changed to say that
  changes to the task's tracked files in its worktree (code, docs and
  config) go only to `vergil:implementer`, never to an ad-hoc helper and
  never in the driver itself, with its two exceptions stated in the same
  place: the driver makes security fixes itself, and resolves rebase
  conflicts itself (both above). Every other file stays with the driver and
  never goes to the writer: files under `build/.vergil-state/` (security
  details and merge review's working files included) and scratch files
  that feed a tool's input. `issue-deploy` and `issue-validate` have no
  worktree and edit no tracked files, so they do not carry the rule.
- **Helpers.** Every helper any skill dispatches carries an explicit
  per-invocation `model`, named in the skill text: `haiku` for a
  read-only search or exploration helper, `sonnet` for any other helper,
  and `opus` for a design-review delegate. The model map records this
  rule (§6). `issue-implement`,
  `issue-deploy` and `issue-validate` invite ad-hoc subagents today
  (research fan-out, parallel sub-tasks, isolated exploration, a fresh
  subagent per acceptance check), as do `epic-implement` ("use sub-agents
  wherever they make this efficient") and `pr-watch` ("sub-agents remain
  fine elsewhere"). The task that makes this change greps `skills/` for
  every other skill that does and changes each the same way. The one
  exception to the explicit `model` is `issue-validate`'s subagent that
  loads a skill to check it: it runs on the validation alias the model map
  gives the skill under test (§6), passed per invocation, so the check sees
  the skill as it runs in production.
- **The human-run path.** When the human runs `/issue-implement` in a
  session, the skill dispatches the writer just the same, so the code is
  always written on Haiku whatever the session's model.

### 5.4 Other pins

- `vergil:merge-reviewer` moves from `model: opus` to `model: sonnet`.
- Design-review rounds dispatch `paad:paad-delegate` with a
  per-invocation `model: opus`. The pin is stated in the paad review
  doctrine (`docs/paad-review-doctrine.md`, §1.9 and §2.2's dispatch step)
  and in `epic-create`'s own dispatch sentence (its round step 1), which
  restates the dispatch. The paad fork is not changed. Its nested
  `paad-analyst` agents inherit Opus from it.
- `vergil:bootstrap` stays `model: haiku`.

### 5.5 The guards

Plugin CI gains a check that fails when any file in `agents/` lacks a
`model:` frontmatter field, and on any agent whose `model:` is `inherit`
unless the model map page names that agent with the reason.

The plugin also gains a PreToolUse hook on the Agent tool, so a task run
can never fall back to an inheriting agent type or a model other than its
pin. A dispatch is a task run when its whole `description` matches one of
the four forms in `epic-implement`'s task-description table (one task, a
batch, and each with no plan task ID), anchored at both ends, with `<id>`
any token without spaces. The hook refuses a task run unless its
`subagent_type` is `vergil:task-runner`, and refuses a `vergil:task-runner`
dispatch that carries a per-invocation `model`, which would outrank the
agent's pin (§3). A description that only names a task among other words,
as the driver's dispatches of the writer or reviewer may, is not a task
run. A description whose first word is `Task` or `Batch`, followed by a
space, but which matches none
of the four forms is refused as a malformed task run, whatever its
`subagent_type`, so a near-miss never runs on an inheriting agent type.
Its tests cover each of the four forms, an ID that is not digits
(`D2`), a writer dispatch whose description names the task, and a
task-runner dispatch with `model` set.

## 6. The model map

The plugin's site docs gain one page that records the model each skill and
agent runs on: the §4 table, the reason for each entry, the pinning facts in
§3, and how to change a pin (edit the agent definition's `model:`, or the
dispatch's `model` parameter) or revert one. It also gives every skill a
**validation alias**, the model alias `issue-validate` checks it on (§5.3):
the smallest model the skill runs on in any production path, or `opus`,
the session's default, for a skill that runs only on the session model. So
`merge-review`, `pr-watch` and `issue-implement`, which run on Sonnet under
the `vergil:task-runner` driver and on the session model elsewhere, take
`sonnet`. For each skill that runs in more than one place, the page lists
every place and the model it runs on there. Every later change to a pin
updates this page in the same PR.

The model map is the single source for models. The existing agents page
(`docs/site/docs/agents/index.md`) drops its per-agent `**Model**:` lines
and links to the model map instead.

Dispatching task runs as `vergil:task-runner` changes the agent type a task
row shows, so three places that say rows show `general-purpose` are updated
with it: the "Agent row labels" section of `docs/site/docs/hooks/index.md`,
the header comment of `hooks/register.tsx`, and
`skills/epic-implement/SKILL.md`.

## 7. Wave 2: the roadmap

The model map ends with the follow-on candidates. When this epic closes,
each is proposed as a follow-on planning item and filed as the human
approves (the review doctrine's §1.7 approval for creating an issue):

- **Approach B** (§5.1): the pipeline of pinned phases for
  `issue-implement`, the optimization the human prefers once A has proved
  itself.
- **`issue-deploy` and `issue-validate`**: split mechanical steps onto Haiku.
- **`pr-watch`**: separate the wait-and-report loop from diagnosis.
- **`fixture-run`**: move its rule-following filing to a pinned agent or a
  forked skill, on Haiku or Sonnet. No conversation with the human holds it
  in the session.
- **Human-run skills judged below Opus** (`epic-review`, `hoshin`,
  `summarize`, `epic-retrospective`, `intake-capture`): which parts can run
  as forked or agent pieces without losing the conversation with the human.
- **`epic-implement`**: whether its orchestration can run in a pinned
  subagent.

## 8. Proof

After the plugin release, once the human attests that one real
implementation task has run through `epic-implement` and one design-review
round has run, a validation task reads those runs' session transcripts to
confirm
that the driver ran on Sonnet, the writer on Haiku and the merge reviewer on
Sonnet. If the attested task's PR had a re-entry (§5.3), it confirms that
the re-entry was dispatched as a task run of `issue-implement`, not run in
the main session. A re-entry is hard to arrange on demand, so when none
occurred the model map records the re-entry path as not verified. It also
reads the transcript of one `epic-create` design-review
round and confirms that its Agent call dispatching `paad:paad-delegate`
carried `model: "opus"`, since an unpinned delegate would also run on the
session's Opus. The escalation's
per-invocation `model: sonnet` is hard to trigger on demand, so the model
map records it as not verified. The weekly usage over the following weeks is the human's own signal,
and no tooling records it in this epic.

## 9. Left to the builder

- The exact wording of the two new agent definitions and their tool lists
  (the writer needs Read, Edit, Write, Glob, Grep and Bash, and no Agent).
- Where the guard lives in the plugin's CI and how it parses frontmatter.
- The model map's page location in the plugin's site docs.
- How the transcript check in §8 reads the model of each subagent's turns.
