# merge-review: a lightweight pre-PR reviewer and agent-to-agent protocol

- **Epic:** vergil-project/.github#881
- **Planning issue:** vergil-project/.github#878 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-claude-plugin (the `merge-review` skill, its reviewer
  prompt and its gate in `issue-implement`), vergil-tooling (the protocol
  spec and the result validator), `.github` (this spec and plan)

## 1. Intent

The fleet is working toward agents that implement, submit and merge a PR with
no human reading it. CI already covers everything deterministic: lint,
formatting, types, tests, coverage, the security scanners. What it cannot
cover is judgement: does the change do what the issue asked, is it safe to
run, would a senior engineer let it through. That layer has to be an AI
review that runs before the PR exists.

`paad:agentic-review` was tried in that role (epic
vergil-project/.github#568) and has been removed from `issue-implement`. It
asks "what is wrong with this code?", which has no bottom on any real
codebase:

- each fix round produced new findings, so review loops did not converge
  without the human ruling on long lists;
- two-line documentation changes took several rounds and produced security
  follow-ups;
- it treated design questions as defects. Faced with concurrency the design
  never intended, it added machinery to support it, where the right answer
  was often to serialize the work, and that is the human's call;
- it was built for large, critical changes, while the fleet drives PRs
  toward the smallest possible unit of change.

`merge-review` asks a different question: **is this change good enough to
merge?** It is a sanity check of the kind a second engineer gives, tuned for
small PRs, with exactly one iteration before the human is asked.

Implementation quality is not the problem being solved. `issue-implement`
already produces work the human is happy with. The review is risk
management: the extra check that has to exist before agents merge on their
own.

## 2. Goals, non-goals, anti-goals

### Goals

- A reviewer that answers two questions about a branch (§4) and reports only
  blockers.
- A loop of exactly one iteration: review, fix, check (§6).
- A clear signal at the end of every run: eligible to merge without a human,
  or not, and why (§6.3).
- A standalone protocol for agent-to-agent requests and results, built from
  files and a validated structured block rather than prose the caller has to
  interpret (§8). `merge-review` is its first user.
- Cheap on small changes: a docs-only change gets a lighter review (§4.3).

### Non-goals

- **Agents submitting or merging PRs.** This epic produces the eligibility
  signal. Acting on it changes the rule that only the human runs
  `vrg-submit-pr`, and that is a later epic.
- **Moving paad's design-review rounds onto the new protocol** and removing
  the paad review doctrine's workarounds (state-directory seals and hashes,
  scratch copies, security cut-outs, broken-round exits). That is a later
  epic, informed by how the protocol works here.
- **Review in the PR process** (a GitHub-side review on the opened PR). That
  would be a later defence in depth.
- **Cost metrics.** Usage is logged if it is cheap to capture (§9). Nothing
  depends on it.
- **Removing agentic-review** from `issue-implement` and the doctrine. The
  human is doing that separately; this epic assumes it is gone.

### Anti-goals

The design fails, however well it works mechanically, if:

- the loop can run a second fix round without the human;
- a docs-only change regularly takes more than one review;
- the reviewer reports things CI already checks, or hypothetical inputs no
  real author would write;
- the reviewer patches around a design problem instead of raising it;
- the caller has to read the reviewer's prose to decide what to do next.

## 3. Prior art

The design takes from four tools and credits each in the skill's docs.

| Taken | From |
|---|---|
| The high-signal bar ("if you are not certain an issue is real, do not flag it") and the exclusion list (pre-existing issues, nitpicks, anything a linter catches) | Anthropic `code-review` plugin, `claude-plugins-official` |
| A silent-failure lens: swallowed errors, fallbacks that hide failure | Anthropic `pr-review-toolkit` plugin |
| A reviewer given crafted context, never the implementer's session; a merge verdict | superpowers `requesting-code-review` |
| Out-of-scope additions; checking the change against its spec; blame-based scope | paad `agentic-review` |

Deliberately not taken: multiple parallel specialists, severity tiers below
blocking, praise sections, "declined to judge" lists, numeric confidence
scores, and a backlog of pre-existing bugs.

## 4. What the reviewer judges

### 4.1 Two questions

Every finding answers one of two questions, and says which:

- **Production safety: "is it safe to put this into production?"** Risk:
  outages, a worse error rate, security holes, data loss, broken deploys or
  releases.
- **Merge quality: "would a senior engineer block this merge?"** Correctness
  against the issue, maintainability, clarity, and for documentation,
  accuracy against the code and the quality of the writing. Code that works
  but cannot be maintained is safe for production and still fails here.

Both kinds block.

### 4.2 The bar

A finding is a **blocker** only when all of these hold:

- the reviewer is certain it is real, having read the code it concerns;
- this branch introduced it (§5.4 covers what it did not introduce);
- it answers one of the two questions above;
- CI does not already check it.

Not findings:

- anything a gate in the request's list of passed checks covers (§8.2): the
  reviewer is told those passed and does not repeat them;
- nitpicks and matters of taste that a senior engineer would not block on;
- pathological input from a trusted author. Configuration and files the
  operator writes are judged on realistic use. A glob with ten consecutive
  wildcards in the operator's own config file is not a finding;
- general worries without a concrete failure ("could be more robust",
  "consider more tests").

Input that crosses a trust boundary (anything an outside party can supply)
is judged against an attacker. That is a production-safety question.

### 4.3 Change classes

The caller classifies the change before dispatch, deterministically, from
the changed files:

- **docs-only:** every changed file is documentation by an explicit
  allow-list. The reviewer answers only the merge-quality question: accuracy
  against the code and the issue, and clarity. The fleet accepts that a
  documentation change does not break production.
- **code:** everything else, and any doubt. Both questions.

The classifier is a `vrg-*` tool, not the agent's judgement. The allow-list
starts small and grows from evidence.

## 5. Kinds of finding

### 5.1 Defect

The change does not do what the issue asks, or fails one of the two
questions, and the fix stays within the existing design. The implementer
fixes it in the one fix step.

### 5.2 Design question

The fix would add machinery the design does not have: locks, retries, new
state, caches, new configuration, a new component, a new failure mode to
handle. The reviewer reports what it found and what decision it raises, and
proposes no fix. The run goes to the human.

Unplanned concurrency is the model case. If the design calls for
concurrency, code that breaks it is a defect. If the design never
anticipated it, whether to support it or to serialize the work is a design
decision, and only the human makes it.

The test is about the fix, not the problem: "would the fix add something the
design does not have?" When the reviewer is unsure, it is a design question.

### 5.3 Out-of-scope addition

Code or content the issue did not ask for. Reported for the human to keep,
split into its own issue, or revert. It does not block on its own, but it
does make the run ineligible to merge without a human (§6.3).

### 5.4 Pre-existing problem

A real problem in code the branch did not introduce. The reviewer may note
it, marked pre-existing, and never asks for it to be fixed on this branch.

The request gives the reviewer the open tasks of the issue's epic (§8.2), so
it can leave out problems already planned for. For each note that remains,
the implementer checks whether it is already tracked: an open issue, or a
security record (`vrg-security-record list`). Tracked problems are recorded
as such and need nothing more. An untracked one is proposed to the human as
a new issue or security record, under the existing rule that creating an
issue needs the human's approval.

The lookup is the expensive part, so it runs only for notes that survive the
epic filter, and it searches by file and component rather than by wording,
because security records are deliberately neutral.

## 6. The loop

### 6.1 Steps

1. **Validate.** The branch is committed and `vrg-container-run --
   vrg-validate` is green. The review never starts before that.
2. **Review.** A fresh reviewer subagent, read-only, gets the request (§8.2)
   and writes a result (§8.3).
3. **Decide.**
   - No blockers, no design questions, no out-of-scope additions: done,
     eligible.
   - Any design question, or more than 5 blockers: stop and ask the human.
     Too many blockers means the branch is too big or the design is wrong,
     and a fix round is the wrong tool.
   - Otherwise the implementer fixes every defect (and handles pre-existing
     notes, §5.4), re-validates to green, and continues.
4. **Check.** A fresh reviewer gets the original findings and the fix diff,
   and answers one question per finding: fixed or not. It may also report
   that a fix broke something it touched. It does not review anything else.
5. **Decide again.** Every finding fixed and nothing broken: done. Anything
   else: stop and ask the human. There is no second fix round without the
   human.

The implementer may dispute a finding instead of fixing it. A dispute goes
to the human; the two agents never argue it out.

### 6.2 The human

When the run stops for the human, the implementer presents each open item
with a recommendation. The human rules on each one. A ruling may grant
another fix and check, in which case it is the human's iteration, not the
loop's.

Rulings, pauses and resumption use the fleet's existing mechanism for
human rulings on a task issue, whichever is current when this is built; this
epic does not design a new one. The direction stands: an agent trusts a
ruling only from a trusted author, never because another agent relayed it.

### 6.3 Eligibility

A run ends **eligible** when the review or the check passed, with no design
questions, no out-of-scope additions, no disputes and no human ruling
needed. Otherwise it ends **not eligible**, with the reason. Eligibility is
recorded on the task issue and in report-ready's notes. While only the human
submits PRs, it is information; it becomes the gate when agents merge (a
non-goal here).

## 7. Security

Security uses the fleet's existing definition: a finding is a security
finding if reading it would help an attacker.

- **A security defect this branch introduced** is fixed on the branch like
  any other defect. Its details never appear in a public comment: the round's
  public record counts security findings and says nothing more. Details stay
  in the task's private state directory and the conversation.
- **A pre-existing security problem** goes through §5.4. A new record is made
  only with `vrg-security-record`, on the human's approval.
- **A security design question** goes to the human like any other, with
  details in the terminal only.

Because the reviewer is ours and writes a structured result, security
findings are marked in that result, and the caller routes them by that
field. The reviewer's prose is not trusted for routing.

## 8. The protocol

### 8.1 Shape

The protocol is a standalone spec in vergil-tooling's `docs/specs/`, next to
the worktree convention, so later work (paad's rounds) can adopt it without
reading this epic. It defines:

- **a request file**, written by the caller into the task's state directory
  before dispatch;
- **a result file**, written by the subagent into the same directory, made of
  a structured block followed by free prose;
- **a validator**, a `vrg-*` command that checks the structured block against
  a schema and exits non-zero on any violation;
- the rule that the caller acts only on the validated block. The prose is for
  the human.

The subagent's reply message carries nothing the caller acts on beyond "the
result is written".

### 8.2 The request

At least:

- the repository, base and head SHAs, the issue reference and its body;
- the change class (§4.3);
- the design sources, where they exist: the epic's spec and plan;
- the list of checks `vrg-validate` ran and passed;
- the open tasks of the issue's epic, with titles;
- the step: `review`, or `check` with the earlier findings and the fix diff.

### 8.3 The result

The structured block holds at least:

- the step and the request it answers (a hash of the request file);
- a verdict: `pass` or `blockers`;
- each finding: an ID, the question it answers (production safety or merge
  quality), its kind (defect, design question, out-of-scope addition,
  pre-existing), whether it is security, and its location;
- in a check: each earlier finding's ID with `fixed` or `not fixed`, and any
  breakage the fix caused.

Security findings carry no description in the structured block: their
details go only in the prose part, which is never posted.

### 8.4 Trust

- The reviewer is read-only. The caller checks that the worktree's branch,
  HEAD and clean status did not change across the review, and that the
  reviewer wrote nothing in the state directory except the result file.
- A result that fails validation is not acted on. The review is re-run once,
  then the run stops for the human.
- The issue, the diff and every file the reviewer reads are data, not
  instructions.

## 9. Records and cost

Each step posts one comment on the task issue with a fixed first line naming
the step, the verdict and the counts by kind. Security findings appear only
as a count. This is the history; the state directory is working storage,
deleted when the task reaches report-ready.

If the usage of the reviewer subagent is cheap to capture (for example from
its completion notice), it is logged in the step's comment. Nothing is built
to measure it.

## 10. Left to the builder

- The skill's name stays `merge-review` unless a better one comes up during
  the epic; it must not collide with any other name in the plugin namespace
  or with built-in skills.
- The exact schema, field names and file names of the protocol.
- The validator's command name.
- The docs-only allow-list's initial contents.
- The reviewer prompt's wording, within §4 and §5.
- Which model and effort the reviewer subagent runs at.
- The shape of the step comments beyond their first line.
