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
the work. Haiku writes the code.

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
4. **A wave-2 roadmap**: the skills worth splitting next, each recorded as a
   follow-on planning item (§7).
5. **A guard** that keeps the pins from drifting: plugin CI fails on an
   agent definition with no `model:` (§5.5).

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
| `vergil:merge-reviewer` | subagent, pinned `opus` | is this good enough to merge | **Sonnet**; **Opus** for docs PRs, passed by the dispatch |
| `pr-watch` | inside the driver | diagnosing CI failures | Sonnet (driver) |
| `issue-deploy`, `issue-validate` | subagent, no pin today | preconditions, judging an outcome | **Sonnet** (`vergil:task-runner`) |
| `epic-retrospective`, `hoshin`, `epic-review`, `summarize`, `fixture-run`, `deprecation-triage`, `intake-capture` | human session | various | session model |
| `vergil:bootstrap` | subagent, pinned `haiku` | checks | Haiku (unchanged) |

Notes:

- **Skills the human runs stay on the session's model**, and the model map
  says so explicitly. The analysis judged several of them (`epic-review`,
  `hoshin`, `summarize`, `epic-retrospective`, `fixture-run`) to need no more
  than Sonnet, and `intake-capture` no more than Haiku. Moving them needs a
  fork or a split, so they go to the wave-2 roadmap (§7), not to frontmatter
  `model:`, which lasts one turn (§3).
- **The merge reviewer moves from Opus to Sonnet.** That is the one change
  in this pass that lowers a model on purpose. It stays a tier above the
  Haiku writer whose work it reviews.
- **`merge-review` is shared.** `epic-create`, `epic-retrospective` and
  `hoshin` also run it, on their docs PRs, whose specs, plans,
  retrospectives and logbook entries are written on Opus. Under those three
  callers the skill dispatches `vergil:merge-reviewer` with a per-invocation
  `model: opus`, so docs PRs keep an Opus review. Under `issue-implement`
  and `pr-watch` it relies on the agent's `sonnet` pin.
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
  given worktree, carry out a given change, commit it in small `vrg-commit`
  commits, and validate with `vrg-container-run -- vrg-validate` until green.
  It never suppresses a gate, never submits, pushes or posts on GitHub, never
  creates issues and never runs merge review. It returns one of two
  results: `green` (with its commits), or `stuck` (what it tried and what
  still fails). Both carry `notes`: any deviation from the issue and why,
  and any concern it noticed (a gap in the issue, a defect outside the
  change), which the driver screens and handles as follow-ups, as it does a
  concern any sub-agent reports today.

### 5.3 The driver's use of the writer

- **Implement.** The driver dispatches the writer with the issue's body
  (read through the trust layer), the worktree path and the commit
  conventions (a batch's first commit for each issue, then fixups of it).
  The driver checks the result for itself (the branch's commits, a clean
  tree, a green validate) before merge review. A fabricated `green` must
  not reach merge review.
- **Fixes.** Code fixes asked for by merge review's fix step, and by
  pr-watch's fixes for failing CI, review comments and conflicts, go through
  the same writer. Only the code goes to the writer. The driver still
  diagnoses the problem and runs every check and push. The hand-off to the
  writer lives in `issue-implement` and `pr-watch` only, not in the shared
  `merge-review` skill: under the docs-PR callers (§4), merge review's fix
  step stays in the caller's session, as today.
- **Escalation.** If the writer returns `stuck`, the driver dispatches it
  once more with a per-invocation `model: sonnet`, passing on what the
  first attempt found. A second `stuck` stops the run for the human, as a
  problem the run cannot resolve does today.
- **Other helpers.** `issue-implement`, `issue-deploy` and `issue-validate`
  today invite ad-hoc subagents (research fan-out, parallel sub-tasks,
  isolated exploration, a fresh subagent per acceptance check). Each of the
  three skills is changed to say that code changes go only to
  `vergil:implementer`, never to an ad-hoc helper and never in the driver
  itself, and that every other helper it dispatches carries an explicit
  per-invocation `model`, named in the skill text.
- **The human-run path.** When the human runs `/issue-implement` in a
  session, the skill dispatches the writer just the same, so the code is
  always written on Haiku whatever the session's model.

### 5.4 Other pins

- `vergil:merge-reviewer` moves from `model: opus` to `model: sonnet`, and
  `merge-review` passes a per-invocation `model: opus` when it runs for a
  docs PR (§4).
- `epic-create`'s design-review rounds dispatch `paad:paad-delegate` with a
  per-invocation `model: opus`. The paad fork is not changed. Its nested
  `paad-analyst` agents inherit Opus from it.
- `vergil:bootstrap` stays `model: haiku`.

### 5.5 The guard

Plugin CI gains a check that fails when any file in `agents/` lacks a
`model:` frontmatter field. `inherit` counts as a choice, but only where the
model map records why.

## 6. The model map

The plugin's site docs gain one page that records the model each skill and
agent runs on: the §4 table, the reason for each entry, the pinning facts in
§3, and how to change a pin (edit the agent definition's `model:`, or the
dispatch's `model` parameter) or revert one. Every later change to a pin
updates this page in the same PR.

The model map is the single source for models. The existing agents page
(`docs/site/docs/agents/index.md`) drops its per-agent `**Model**:` lines
and links to the model map instead.

## 7. Wave 2: the roadmap

The model map ends with the follow-on candidates. Each becomes a follow-on
planning item when this epic closes:

- **Approach B** (§5.1): the pipeline of pinned phases for
  `issue-implement`, the optimization the human prefers once A has proved
  itself.
- **`issue-deploy` and `issue-validate`**: split mechanical steps onto Haiku.
- **`pr-watch`**: separate the wait-and-report loop from diagnosis.
- **`fixture-run`**: move its rule-following filing to Haiku, perhaps as a
  forked skill.
- **Human-run skills judged below Opus** (`epic-review`, `hoshin`,
  `summarize`, `epic-retrospective`, `intake-capture`): which parts can run
  as forked or agent pieces without losing the conversation with the human.
- **`epic-implement`**: whether its orchestration can run in a pinned
  subagent.

## 8. Proof

After the plugin release, a validation task runs one real implementation
task through `epic-implement` and reads the session transcripts to confirm
that the driver ran on Sonnet, the writer on Haiku and the merge reviewer on
Sonnet. The weekly usage over the following weeks is the human's own signal,
and no tooling records it in this epic.

## 9. Left to the builder

- The exact wording of the two new agent definitions and their tool lists
  (the writer needs Read, Edit, Write, Glob, Grep and Bash, and no Agent).
- Where the guard lives in the plugin's CI and how it parses frontmatter.
- The model map's page location in the plugin's site docs.
- How the transcript check in §8 reads the model of each subagent's turns.
