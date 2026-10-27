# Agentic review in `issue-implement`

- **Epic:** vergil-project/.github#568
- **Planning issue:** vergil-project/vergil-claude-plugin#677 (review history
  is in its comments)
- **Umbrella:** vergil-project/.github#500 (child D). Child F depends on this
  epic.
- **Status:** draft, in design review
- **Brainstorm source:** none. The umbrella spec (§3.1, §3.3, §4.D) settled
  the rules, the planning-time spike (section 3) settled feasibility, and the
  human skipped the brainstorm questions.

## 1. Intent

Today the only gate between "the implementer thinks it is done" and
`vrg-pr-workflow report-ready` is `vrg-validate`. That checks that the branch
passes; nothing checks that it is right. paad's authors close that gap with
step 7 of their loop: run `paad:agentic-review`, fix, and repeat.

This epic builds that step into `issue-implement`. A fresh reviewer runs
`paad:agentic-review` and tells the implementer what it did wrong; the
implementer fixes it; they repeat for a few rounds. If they cannot settle it,
the run stops and the human makes the call. Every round is recorded on the task
issue, nothing found is lost, and the handoff says honestly what was not fixed.

**Start simple.** This is a real-time loop that produces a result while the
human waits, not a process that runs for weeks. We have little experience of
how the implementer and the reviewer will actually interact, so this design
builds the simplest loop that meets the binding rules, and nothing more.
Mechanisms for problems we have only imagined are listed in section 5 as
deliberately left out, to be added when real use shows they are needed.

## 2. Inherited rules (binding, not reopened here)

Umbrella §2, §3.1 and §3.3 are binding on this epic as written (umbrella §4,
"How binding section 3 is"). They are the initial operating settings, to be
tuned from real use. In short:

- **Where and when:** after `vrg-container-run -- vrg-validate` passes and
  before `report-ready`, with all work committed; it **always runs**, including
  on docs-only and trivial changes.
- **One round:** a fresh subagent runs `paad:agentic-review` against the
  issue's requirements with the base passed explicitly; its findings are posted
  as an issue comment; the implementer fixes each finding or records why not;
  its response is posted as a second issue comment; a new round follows if any
  blocking finding was fixed.
- **Blocking:** in-scope Critical and Important. **Discretionary:** in-scope
  Suggestion, fixed or answered, never causing a round on its own.
- **Out-of-scope additions** are removed from the branch and filed as their own
  task. **Out-of-scope pre-existing bugs** go to intake.
- **Pass:** a round with no blocking findings.
- **Pause** (stop before `report-ready` and report the reason) when round 3
  had any blocking finding; when the implementer wants to dismiss a blocking
  finding as wrong; when most of a round's findings repair the previous round's
  fixes; when an output check fails.
- **Honest handoff** and **intake at handoff** for everything not addressed.
- **Retrospective sweep** in `epic-retrospective`.
- **Output checks** by structure, including that every lens was dispatched.
- **Security details** appear in public records only as counts.

The rest of this spec is mechanism, which umbrella §4 leaves to this epic.

## 3. Planning-time spike (feasibility, already run)

Umbrella §4.D made a spike D's first task, with the rule that a failure goes
back to the human before D is designed. It was run during planning instead,
because its result decides the whole design. Its record is on the planning
issue.

- **Chain tested:** main session (standing in for `epic-implement`) → a
  `general-purpose` subagent standing in for the `issue-implement` subagent →
  a fresh `general-purpose` reviewer → `paad:agentic-review` (2.0.2) → six
  `paad:paad-analyst` specialists and the verifier.
- **Target:** vergil-tooling commit fb85e976ae (+240/−12, two files) in a
  detached throwaway worktree, base passed as the parent commit's full SHA,
  requirements from issue #3695 in a file outside the worktree.
- **Result: passed.** Every level had the Agent tool. All six lenses and the
  verifier were dispatched in their own contexts and returned their
  `[ref-loaded:…]` tokens; no lens bailed; `Verifier warnings: none`. About
  6.5 minutes and 7 subagents.
- **What it showed:**
  - a requirements file named in the prompt works as Spec Compliance's intent
    source;
  - paad wrote its report and `backlog.md` to `paad/code-reviews/` in the
    worktree, its default;
  - the reviewer corrected paad's output on its own initiative, so the prompt
    forbids editing it;
  - raw `git` is denied by the vergil hook in a managed repo; with the prompt
    naming `vrg-git`, nothing was denied;
  - the review found two Important bugs in already merged code (filed with
    three other findings as intake vergil-project/.github#563–#567).
- **What it did not test:** the exact prompt in 4.3, or a plan-driven task. The
  first validation task does (4.11).

## 4. Design

### 4.1 Where it lives

- **`issue-implement`** gains a section, "The agentic-review gate", between
  "Implement" and "Record the PR metadata and hand off".
- **The doctrine.** Child C creates
  `vergil-claude-plugin/docs/paad-review-doctrine.md` with a stub §3 for mode 2.
  This epic writes §3: the loop, the reviewer prompt, the output checks and the
  comment formats below. `issue-implement` links to it rather than repeating
  it. This epic confirms the agentic-review column of doctrine §1's severity
  table: Critical → `critical`, Important → `high`, Suggestion → `medium`.
- **`epic-implement`** and **`epic-retrospective`** each gain a small change
  (4.9, 4.10).

### 4.2 The loop

The gate starts when the implementation is committed and `vrg-validate` is
green. Each round:

1. **Write the requirements file** (4.4) into a fresh `mktemp -d` directory
   outside the worktree.
2. **Dispatch** a fresh reviewer subagent with the prompt in 4.3, and wait.
3. **Check the output** (4.5). A failed check pauses the run.
4. **Collect.** Read paad's report from the worktree, post the findings
   comment (4.6), record any security findings (4.7), then empty the worktree's
   `paad/code-reviews/` and `paad/security/` so the next reviewer starts clean.
   Delete the temp directory.
5. **Act on each finding:**
   - in-scope Critical or Important: fix it. If the implementer thinks it is
     wrong, it marks it "pending human", finishes acting on the round's other
     findings, and pauses after step 6. The round counts;
   - in-scope Suggestion: fix it, or note why not;
   - out-of-scope addition: remove it, and file it as its own task under the
     same epic (or the repo's ad-hoc epic) with `--blocked-by <this task>`;
   - out-of-scope bug: leave it for intake.

   Fixes are ordinary commits under `issue-implement`'s normal discipline.
   Their messages describe the change, never the round or the finding:
   commit messages are one of Spec Compliance's intent sources, so a message
   naming the review would tell the next reviewer about earlier rounds.
6. **Post the response comment** (4.6), then **file this round's intake
   issues** (4.8), so nothing found depends on the run reaching a pass.
7. **Decide:**
   - **pass** if the round had no blocking findings;
   - otherwise **another round**, unless this was round 3 (or the last round
     the human allowed) or most of the round's findings are repairs of the
     previous round's fixes, in which case **pause**.

On a pass, post the summary comment, confirm `vrg-validate` is green at the
final commit, and run `report-ready` with the handoff notes (4.8).

**Classification.** The implementer marks each finding `new`, `known` (an
earlier round raised it, or it was deliberately accepted) or `repair` (it is
about code the previous round's fixes changed), by its own judgement against
the earlier comments on the issue (umbrella §3.1). A known finding that the
human has already ruled on follows that ruling: it does not block, and the
response notes "ruled earlier". The mark is recorded in the findings comment
and drives the repair pause; it needs no other bookkeeping. The reviewer is
never told about earlier rounds.

### 4.3 The reviewer prompt

The doctrine holds the template. It tells the reviewer to:

- invoke `paad:agentic-review` with one argument, `origin/<base>`, after the
  implementer has run `vrg-git fetch origin`;
- work in the worktree: every Bash command starts `cd <worktree> && …`, and
  every subagent is told the same;
- use `vrg-git`, since raw `git` is blocked;
- give the requirements file to Spec Compliance as its explicit spec file, and
  not read the task issue, its comments or any PR;
- never ask: answer *security hold or ordinary report* with **hold** (paad
  asks this only when `paad/security/` is unsafe; 4.7), never split a large
  review into separate passes, and for anything else take the most
  conservative option and say so;
- edit nothing, and never edit paad's output;
- return paad's own final answer, which lists the files paad wrote.

The prompt says nothing about where paad writes. paad writes its default paths
in the worktree, and the caller collects them (4.2 step 4). Adapting to paad's
defaults is more reliable than instructing it to behave differently, and paad
marks much of that behaviour as binding.

### 4.4 The requirements file

`requirements.md` holds:

- the task issue's title and body;
- for a task in an epic with a plan, the plan task's text and the spec sections
  it cites, read from the epic home's `origin/develop`. An epic task's body is
  often only a pointer to its plan task;
- any comment **posted from the human's own account** that changes what the
  task must do;
- the human's rulings from this run's conversation, restated as requirements
  ("the task must do X"), never as review history ("R2.3 was dismissed"). A
  fix the human directed is then something the requirements asked for, not an
  out-of-scope addition.

Comments posted by agent identities, including every gate record (4.6), are
**never** requirements input, and neither is anything from any other author.
The issue body is used when it was written by the human or by one of the
human's agent identities. Logins are matched exactly. The system-wide version
of this rule, for every tool and skill that reads issues, is planned in
vergil-project/.github#579, which will own the list of trusted logins; until
then the doctrine names them.

### 4.5 Output checks

After each review the implementer checks, by structure:

- `Agents dispatched:` lists all six lenses, none `NOT DISPATCHED`;
- Spec Compliance did not bail, and its intent sources include the
  requirements file;
- `Verifier warnings: none`;
- the reviewer did not record a security hold;
- every file paad wrote is under this worktree's `paad/code-reviews/` or
  `paad/security/`, and `vrg-git status --porcelain` is empty.

Any failure pauses the run. A failed round's findings are not acted on.

### 4.6 What goes on the task issue

Three kinds of comment, each with a fixed first line so a reader (or a
fresh session) can see where the gate stands. They are **records**: no run ever
takes instructions from them.

- **`agentic-review round <n>: findings`:** paad's findings, each with an ID
  `R<n>.<i>`, its severity, its scope bucket, whether it is a repair, and its
  location by file and symbol with a short code excerpt. Line numbers and SHAs
  change when `vrg-submit-pr` rebases, so they are given only as "at the time".
  A failed output check is noted here.
- **`agentic-review round <n>: response`:** what the implementer did with each
  finding (fixed, not fixed and why, removed and the task it became), and what
  happens next.
- **`agentic-review summary: <outcome> after <n> round(s)`:** before
  `report-ready`, or when a run ends without a pass. It lists every finding not
  fixed, each with its intake or task issue, so the retrospective sweep can
  check them.

A pause adds a line to the latest response comment saying why the run stopped,
and, once the human has ruled, the human's ruling in their words (4.9).

The reviewer's free-text notes are not copied into these comments, and
security findings appear only as counts (4.7).

### 4.7 Security findings

- paad writes security findings under the worktree's ignored
  `paad/security/`.
- The implementer treats each like any other finding: blocking or not, fixed
  or not.
- When the round is collected, each one is recorded through child A′'s
  one-to-many security-records model (vergil-project/.github#573). Its details
  live only in the private record. This is the **only** way a security-class
  finding is filed: an out-of-scope security bug, a dismissed one, or a
  removed addition that paad classed as security never becomes an ordinary
  intake issue or task. `vrg-security-record` creates the neutral public
  issue.
- **A real security hold** (paad found `paad/security/` unsafe and wrote
  nothing there) is a failed check (4.5). The run pauses, any details stay in
  the conversation with the human, and the issue shows only that the check
  failed.
- Public records carry counts only. That covers the comments, commit messages
  (a security fix gets a neutral message), intake issues and tasks the gate
  files, the PR text and the `report-ready` notes.

### 4.8 Intake and the handoff

- **Intake.** At the end of each round (4.2 step 6), the implementer files
  that round's deferred Suggestions, out-of-scope bugs, dismissed findings and
  anything else it noticed, with `vrg-intake-create` (security-class findings
  go through 4.7 instead). Each issue names the task and the round. It searches for an existing open issue first and comments there
  instead of filing a duplicate; when unsure, it files.
- **The handoff.** The `report-ready` notes, and the summary shown to the human,
  give the outcome and the round count, and list every finding not fixed with
  its reason and issue, every removed addition with its task, and any commit
  made after the last round (a late Suggestion fix), marked unreviewed. Tasks
  split out for removed additions are not in the epic's plan, so the handoff
  asks the human to schedule them.

### 4.9 Pauses and `epic-implement`

A pause is a stop that hands the decision to the human, in real time.

- The run stops before `report-ready`, records why on the issue, and tells the
  human: in the terminal, or, under `epic-implement`, among the items waiting
  on the human.
- **The human decides in conversation,** in their own words: run another
  round, dismiss a finding, fix it a particular way, accept the branch as it
  stands, or drop it. The human has the final say; the agent records the
  ruling on the issue and carries it out. Under `epic-implement` the main
  session passes the ruling to a new `issue-implement` run for that task, which
  continues from the existing worktree and continues the round count shown in
  the issue's findings comments (umbrella §3.3).
- **A pause is not a save point.** The branch is not pushed (pushing happens
  at `report-ready`). If the human does not come back, or the worktree is lost,
  the work is abandoned: the task stays open as unfinished work and is
  implemented again later from `develop`.
- **A pause is resolved only by the human.** A run that reaches the gate and
  finds the task paused, with no ruling from the human in the conversation,
  reports the pause again and stops; it runs no review. `epic-implement` needs
  no special handling: a re-dispatched paused task simply stops again and is
  listed among the items waiting on the human. Once the human has ruled, the
  task is processed normally.

### 4.10 The retrospective sweep

`epic-retrospective` gains a step before drafting: for each task of the epic,
read its agentic-review summary and check that every item not fixed has an
issue. Also look through the task's other comments, its PR review comments and
the planning issue's design-review comments for problems that were noted and
never filed. For each untracked problem, find the existing issue, file one, or
have the human drop it with a reason. The result goes in retrospective §4.
Tasks implemented before the gate shipped have no summaries; the sweep says so.

### 4.11 Prerequisites, tooling and validation

- **`paad/code-reviews/` ignored.** It joins `paad/security/` in the managed
  `.gitignore` block, synced to every repo **before** the gate ships;
  otherwise every round fails the clean-worktree check.
- **`vrg-git` additions** (vergil-tooling#3722) are a **prerequisite**. paad
  resolves the default branch on every run, even with an explicit base, and
  without `symbolic-ref` it can fail in repos with no local `main`.
- **Child C's doctrine document** must exist before §3 is written.
- **Child A′** (#573) is a **prerequisite** of the gate. Without it, emptying
  `paad/security/` after a round would delete security findings that were
  never recorded.
- **Re-spike, first:** a validation task runs the loop by hand with the exact
  prompt on a real plan-driven task, twice, and confirms the output checks
  pass, Spec Compliance uses the requirements file, paad's files land only in
  that worktree, and the worktree is clean after collection.
- **Dogfood, last:** after the plugin release is installed, run
  `issue-implement` on a real task and check the comments, the intake filing
  and the handoff. Record what the loop actually did, as the first data for
  tuning (section 5).

### 4.12 Deviations register additions

Add to `vergil-claude-plugin/docs/paad-policy.md`, replacing the current row
on agentic-review report files, and update §1, §2 and §4 there to match (no
config file is adopted). The first row refines umbrella §3.3's "results reach
the implementer through the subagent's answer": the implementer reads paad's
report from its file in the worktree. That refinement is the human's decision,
made in design review, to adapt to paad's defaults rather than redirect them;
the issue remains the record of every round.

| Deviation from paad | Reason |
|---|---|
| agentic-review writes its usual files, but `paad/code-reviews/` is ignored rather than committed; the implementer posts the report to the task issue and empties the directory after each round | The task issue is the record of each round. Adapting to paad's defaults is more reliable than redirecting them. Emptying the directory keeps earlier rounds out of the next reviewer's context. |
| paad's code-review backlog does not persist; out-of-scope bugs go to intake | Intake is the system of record for follow-on work (umbrella §2.4). |
| Out-of-scope additions are removed and filed as tasks, and out-of-scope bugs filed in intake, by the implementer rather than decided by the user | The run is unattended until it pauses (umbrella §3.3). |
| The Spec Compliance intent source is a requirements file named in the prompt, not a spec-file argument | The skill's arguments take only a base and a path. Raise with the paad authors. |

## 5. Deliberately left out (add when real use shows the need)

These were considered during design review and left out on purpose. Each is a
candidate once the dogfood run and the first weeks of real use show it is
needed:

- a structured grammar for the human's decisions, and resuming a paused run
  from issue comments in a later session;
- forged-comment protection for the gate's own records (they are records, not
  inputs, so there is nothing to forge);
- formal new / known / repair bookkeeping beyond the implementer's judgement;
- a way for the implementer to keep a flagged addition it believes is required;
- an exemption from review by change size or risk;
- paad version-drift detection;
- a `paad/config/agentic-review.md` file;
- tooling to parse the comments or run the sweep.

## 6. Goals, non-goals, anti-goals

**Goals.** Every `issue-implement` run ends with an agentic-review pass or a
pause the human resolves; every round is recorded on the task issue; nothing
found is lost; the handoff shows exactly what was not fixed.

**Non-goals.** Everything in section 5; the automated merge-gate review agent
(.github#90); changing paad.

**Anti-goals** (from umbrella §5): a loop that runs forever or passes without
a well-formed report; findings silently dropped; security details in public;
a design so elaborate that it anticipates problems instead of learning from
them.

## 7. Deliverables by repo

| Repo | Deliverable |
|---|---|
| `vergil-claude-plugin` | doctrine §3; the `issue-implement` gate; the `epic-implement` and `epic-retrospective` changes; `docs/paad-policy.md` updates; the skills index page |
| `vergil-tooling` | `paad/code-reviews/` in the managed `.gitignore` and a fleet sync; the `vrg-git` additions (#3722); site docs via the documentation review |
| `docs` | `management/workflow.md` and `management/tools-and-skills.md` (via the documentation review) |
| `vergil-project/.github` | this spec and plan |

## 8. Constraints

- Public repos: issue comments, commit messages, PR text and the
  `report-ready` notes (relay ref) are public.
- Agents use `vrg-git` and `vrg-gh` only, and never run `vrg-submit-pr`.
- `issue-implement` must still finish without a human in the normal case.
- Cost: about 6.5 minutes and 7 subagents per round on a 250-line diff (spike),
  up to three rounds before a pause.
