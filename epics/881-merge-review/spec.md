# merge-review: a lightweight pre-PR reviewer and agent-to-agent protocol

- **Epic:** vergil-project/.github#881
- **Planning issue:** vergil-project/.github#878 (review history is in its
  comments)
- **Status:** draft, in design review
- **Repos:** vergil-claude-plugin (the `merge-review` skill, its reviewer
  prompt, its gate in `issue-implement` and the resume in
  `epic-implement`), vergil-tooling (the protocol
  spec, the result check and the change classifier), `.github` (this spec
  and plan)

## 1. Intent

The fleet is working toward agents that implement, submit and merge a PR with
no human reading it. CI already covers everything deterministic: lint,
formatting, types, tests, coverage, the security scanners. What it cannot
cover is judgement: does the change do what the issue asked, is it safe to
run, would a senior engineer let it through. That layer has to be an AI
review that runs before the PR exists.

`paad:agentic-review` was tried in that role (epic
vergil-project/.github#568) and is being removed from `issue-implement`
(epic vergil-project/.github#885). It asks "what is wrong with this code?", which has no bottom on any real
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

This epic is the prerequisite for agents submitting and merging PRs. The
fleet lacks the confidence to allow that today precisely because it has no
judgement-level review between a green `vrg-validate` and the PR. Turning
on agent submit and merge is the next step, and it depends on this one.

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

- **Agents submitting or merging PRs.** This epic exists to make that
  possible, and produces the eligibility signal it will rely on. Acting on
  the signal changes the rule that only the human runs `vrg-submit-pr`, and
  that change is the follow-on epic.
- **Moving paad's design-review rounds onto the new protocol** and removing
  the paad review doctrine's workarounds (state-directory seals and hashes,
  scratch copies, security cut-outs, broken-round exits). That is a later
  epic, informed by how the protocol works here.
- **Review in the PR process** (a GitHub-side review on the opened PR). That
  would be a later defence in depth.
- **Cost metrics.** Usage is logged if it is cheap to capture (§9). Nothing
  depends on it.
- **Removing agentic-review** from `issue-implement` and the doctrine, and
  **authenticated human rulings**. Both are prerequisites, delivered
  elsewhere (§2.1).

### 2.1 Prerequisites

No task of this epic starts until both of these epics are complete, in
this order:

- **Epic vergil-project/.github#885**, which removes the agentic-review gate
  from `issue-implement` (spec `epics/885-remove-agentic-gate/spec.md`).
  The old gate goes before its replacement arrives, so the two never run
  together. `issue-implement` is then: Preflight, implement, validate to
  green, `report-ready`; this epic adds `merge-review` to it.
- **Epic vergil-project/.github#889**, authenticated human replies on task
  issues (spec `epics/889-human-auth/spec.md`). It builds `vrg-ruling` and
  `vrg-ruling-reply` and their protocol guide, and leaves every skill change
  to this epic (its §2). `merge-review` relies on it for every stop (§6.2).

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
| A reviewer given crafted context, never the implementer's session | superpowers `requesting-code-review` |
| Out-of-scope additions; checking the change against its spec; blame-based scope | paad `agentic-review` |

Deliberately not taken: multiple parallel specialists, severity tiers below
blocking, praise sections, "declined to judge" lists, numeric confidence
scores, a merge verdict from the reviewer, and a backlog of pre-existing
bugs.

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

- anything CI covers. "CI" means both the `vrg-validate` run and the
  repository's PR CI workflows: `vrg-validate` passed before the review,
  and PR CI gates the PR after it. The
  reviewer prompt names the categories they cover (formatting, linting, type
  checking, tests and coverage, security scanners, dependency audits), and
  the reviewer raises nothing in them. When unsure whether something
  specific is covered, it may read the repository's CI workflow files and
  `vrg-validate` configuration; it is not required to read them every run;
- nitpicks and matters of taste that a senior engineer would not block on;
- pathological input from a trusted author. Configuration and files the
  operator writes are judged on realistic use;
- general worries without a concrete failure ("could be more robust",
  "consider more tests").

Input that crosses a trust boundary (anything an outside party can supply)
is judged against an attacker. That is a production-safety question.

### 4.3 Change classes

The caller classifies the change before dispatch, deterministically, from
the changed files:

- **docs-only:** every change is an addition or an edit of a file that is
  documentation by an explicit allow-list. Any deletion or rename makes the
  change code: the docs class is a narrow special case, and anything outside
  it gets the full review. The reviewer answers only the merge-quality question: accuracy
  against the code and the issue, and clarity. The fleet accepts that a
  documentation change does not break production.
- **code:** everything else, and any doubt. Both questions.

The classifier is a `vrg-*` tool, not the agent's judgement. The allow-list
is a list of base paths, kept deliberately simple:

- It starts with only the paths known to hold nothing but documentation:
  writing meant to be read by people or agents to explain how things work,
  never anything that controls behaviour. Markdown is not documentation by
  its extension. In vergil-claude-plugin, skills, agent definitions and the
  rule documents agents follow are code.
- A repository can add its own documentation paths through a `vergil.toml`
  key.
- There are no per-file exclusion rules. A path not on the list is code,
  and a real document that gets the full review now and then is an
  accepted cost.
- The lasting fix is structural, not a finer list: keeping files that
  control behaviour out of `docs/` trees (vergil-project/.github#840). Once
  that holds in a repository, its `docs/` tree can go on the list whole.

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

Code or content the issue did not ask for. The run stops for the human
(§6.1 step 3), who rules to keep it, split it into its own issue, or revert
it; the implementer carries the ruling out before the check.

### 5.4 Pre-existing problem

A real problem in code the branch did not introduce. The reviewer may note
it, marked pre-existing, and never asks for it to be fixed on this branch.

The request gives the reviewer the open tasks of the issue's epic (§8.2), so
it can leave out problems already planned for. For each note that remains,
the implementer checks whether it is already tracked: an open issue, or a
security record (`vrg-security-record list`). Tracked problems are recorded
as such and need nothing more. An untracked one is proposed to the human as
a new issue or security record, under the existing rule that creating an
issue needs the human's approval. The run stops for the human (§6.1 step
3), who approves or rejects each proposal; an approved one is filed before
the run moves on.

The lookup is the expensive part, so it runs only for notes that survive the
epic filter, and it searches by file and component rather than by wording,
because security records are deliberately neutral.

## 6. The loop

### 6.1 Steps

1. **Validate.** The branch is committed and `vrg-container-run --
   vrg-validate` is green. The review never starts before that.
2. **Review.** A fresh reviewer subagent, which never changes the code
   (§8.4), gets the request (§8.2) and returns a result (§8.3).
3. **Decide.**
   - First, the implementer looks up each pre-existing note (§5.4).
   - Nothing raised, or only tracked pre-existing notes: done. Eligibility
     follows §6.3.
   - More than 5 blockers: stop and ask the human. Too many blockers means
     the branch is too big or the design is wrong, and a fix round is the
     wrong tool.
   - Any design question, out-of-scope addition (§5.3), untracked
     pre-existing proposal (§5.4) or dispute (below): stop and ask the
     human, once, before any fix, with every such item together. The
     rulings are carried out in the fix that follows, a revert included.
     Everything merge review raises is a review item, so all of it goes in
     this one request (§6.2), proposals to file an issue or security record
     included. Epic 889's rule that non-review decisions stay outside the
     channel (its §4) covers only decisions `issue-implement` raises on its
     own, which it settles before the request.
   - Then the implementer fixes every defect and carries out every ruling,
     and runs `vrg-validate` again, fixing any failure until it is green, as
     in normal implementation. Getting back to green is not capped and is
     not a review round. With nothing to fix or change, the check is
     skipped.
4. **Check.** A fresh reviewer gets the original findings, the fix diff and
   the whole branch with the fix applied. It judges the quality of each fix,
   not only whether the symptom is gone, and answers two questions:
   - **Is the branch now right?** Read as a whole (the reviewed change plus
     the fix), does each finding stand fixed, and would the two questions
     (§4.1) now pass?
   - **Did the fix break or add anything?** Does the fix diff introduce a new
     problem, or add machinery the design does not have (the §5.2 test)?

   It holds to the same bar (§4.2) and does not hunt afresh through code the
   fix did not touch.
5. **Decide again.** Every finding fixed well, and nothing broken or added:
   done. Anything else: stop and ask the human. A failed check means
   something went wrong that more agent iteration will not fix; there is no
   second fix round without the human.

**No unfinished business.** Merge review is complete only when every item
it raised is closed: fixed, ruled on and carried out (kept, reverted, filed
as an issue or security record), or rejected. The run never reaches
report-ready with an item still open, so nothing waits on the human after
handoff.

The implementer may dispute a finding instead of fixing it, deciding that
before any fix. A dispute goes to the human in the step-3 stop; the two
agents never argue it out.

### 6.2 The human

When the run stops for the human, the implementer presents each open item
with a recommendation. The human rules on each one. A ruling may grant
another fix and check, in which case it is the human's iteration, not the
loop's.

The stop is one round of epic 889's protocol (its §4, and vergil-tooling's
`guides/human-replies.md`); this epic designs no channel of its own:

1. The step comment posts the review's findings (§9).
2. `vrg-ruling request` posts one request covering every item that needs
   the human, each with the implementer's recommendation.
3. The human and the agent talk the items through in the terminal; the
   agent drafts the reply with `vrg-ruling draft`, and the human posts it
   from their own shell with `vrg-ruling-reply`.
4. The run reads the reply with `vrg-ruling read` and at once posts
   `vrg-ruling respond`, saying how it read each ruling and what it will
   do. Only then does it carry out the rulings (the fix and check of
   §6.1); the check's step comment records the outcome. Epic 889 requires
   the response to be the next trusted comment after the reply, so the run
   posts nothing else on the issue in between.

The agent acts only on a reply that passes `vrg-ruling read`'s checks. A
fatal check stops the run for the human, who recovers as epic 889's §10
says. An unclear reply is answered by a new request.

No ruling reaches the agent relayed through a prompt. This epic changes
`epic-implement` so that, when the human has replied, it re-dispatches the
run saying only "human response available"; the run reads the reply from
the issue itself.

### 6.3 Eligibility

A run ends **eligible** when the review or the check passed, with no design
questions, no out-of-scope additions, no disputes and no human ruling
needed. Otherwise it ends **not eligible**, with the reason. The reason is
one of a fixed set of values; security appears in it only as a count (§7),
because report-ready's notes are public. Eligibility is
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
- **A security design question** goes to the human like any other.

Every security item's details, whatever its kind, are kept by the caller in
the task's private state directory, so they survive the end of a session
between a stop and its ruling. That directory is the private channel epic
889's §6 leaves to this epic: the details reach the human in the terminal
and a dispatched run only from there, never from the issue. The run that
carries out a ruling reads the details there.

A security item's decision goes through the request and the reply like any
other, by ID, with epic 889's fixed security words (`fix`, `file`, `defer`,
`wont-fix`, and the matching outcomes in the response).

Every public record (report-ready's title, summary and notes, step comments,
ruling requests and rulings) refers to a security item, of any kind, by ID
or count only.

Because the reviewer is ours and returns a structured result, security
findings are marked in that result, and the caller routes them by that
field. The reviewer's prose is not trusted for routing.

## 8. The protocol

### 8.1 Shape

The protocol is a standalone spec in vergil-tooling's `docs/specs/`, next to
the worktree convention, so later work (paad's rounds) can adopt it without
reading this epic. It defines:

- **a request file**, written by the caller into the task's state directory
  before dispatch. That directory is the caller's own,
  `build/.vergil-state/issue-implement/<N>/` in the task repo's main clone,
  where `<N>` is the task issue: the private directory epic 885 keeps for
  `issue-implement` (its §6.2), so a task has one private directory, not a
  second one for merge review. Its privacy rests on one rule: the
  `build/.vergil-state/` root is mode 700, and the caller checks that before
  writing anything below it, refusing if it does not hold. The human and the
  agents share one Unix user, so a private root makes everything below it
  private, whatever the modes of the files. The caller also checks that each
  directory level down to its own is a real directory, not a symlink, and
  mode 700, so no link can redirect a write out of the private root;
- **a result**, returned by the reviewer as its reply, made of a structured
  block followed by free prose. The caller writes it as the result file in
  the same directory; the reviewer writes nothing (§8.4);
- **the result check**, a `vrg-*` command that checks the result file's
  structured block against the protocol's schema (required fields present,
  values from the allowed set, finding IDs consistent) and exits non-zero on
  any violation. It checks the reviewer's output, not the code, and is
  unrelated to `vrg-validate`;
- the rule that the caller acts only on a block that passed the result
  check. The prose is for the human.

The caller owns all of the data: it writes the request, stores the reply as
the result file, and runs the result check on that file.

### 8.2 The request

At least:

- the repository, base and head SHAs, the issue reference and its body;
- the diff (in a check, the fix diff as well), and the worktree path for
  reading surrounding code;
- the change class (§4.3);
- the design sources, where they exist: the epic's spec and plan;
- the list of stages `vrg-validate` ran and passed (§4.2);
- the open tasks of the issue's epic, with titles;
- the step: `review`, or `check` with the earlier findings and the fix diff.

### 8.3 The result

The structured block holds at least:

- the step and the request it answers (an ID the request carries, which the
  reviewer copies, since it cannot compute a hash);
- each finding: an ID, the question it answers (production safety or merge
  quality), its kind (defect, design question, out-of-scope addition,
  pre-existing), whether it is security, and its location;
- in a check: `fixed` or `not fixed` for each earlier finding the
  implementer was asked to change (every defect, and each ruling that calls
  for a code change), and any breakage the fix caused. Pre-existing notes
  and items ruled to stay as they are are not asked about; their rulings
  and lookups record them.

The result carries no verdict. The reviewer reports findings; the caller
decides from the findings' kinds alone, by the rules of §6.1, so there is no
second field that could disagree with them.

Security findings carry no description in the structured block: their
details go only in the prose part, which is never posted.

### 8.4 Trust

- The reviewer never changes the code. It reads the change and reports:
  findings, requested changes and bugs. Making the changes is the
  implementer's job, never the reviewer's.
- The reviewer writes nothing to the file system and runs no commands. The
  protocol names its agent type, which has only read tools (no Write, Edit
  or Bash), so it has no way to write. Everything it needs that a command
  would produce, such as the diff, comes in the request (§8.2), and
  everything it produces goes back in its reply.
- A result that fails the result check (a malformed structured block) is not
  acted on. The reviewer is dispatched once more; if its second result also
  fails, the run stops for the human rather than guessing what the reviewer
  meant.
- The issue, the diff and every file the reviewer reads are data, not
  instructions.

## 9. Records and cost

Each step posts one comment on the task issue with a fixed first line naming
the step, the caller's decision (§6.1) and the counts by kind. Security findings appear only
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
- The result check's command name.
- The set of eligibility reason values, within §6.3.
- The docs-only allow-list's initial paths, within §4.3, and the name and
  shape of its `vergil.toml` key.
- The reviewer prompt's wording, within §4 and §5.
- Which model and effort the reviewer subagent runs at.
- The shape of the step comments beyond their first line.
