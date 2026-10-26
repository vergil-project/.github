# Umbrella epic: paad-first AI-assisted engineering

- **Epic:** vergil-project/.github#500
- **Planning issue:** vergil-project/.github#499
- **Status:** approved design (2026-10-26), in design review
- **Review history:** the comments on the planning issue,
  vergil-project/.github#499
- **Brainstorm source:** superpowers brainstorming session, 2026-10-26
- **Absorbs:** vergil-project/.github#469 (iterating pushback/alignment) and
  vergil-project/vergil-claude-plugin#677 (agentic-review before report-ready).
- **Related:** vergil-project/.github#88 (inter-issue / epic dependency
  metadata). This umbrella tracks its children with text links and gate
  tasks, not nested epics. Native nesting is left to #88.
  vergil-project/.github#90 (automated PR-review agent) stays deferred
  and waits on child D below.

## 1. Intent

paad (https://github.com/Ovid/paad, installed version 2.0.2) is the most
important AI-engineering tool in our workflow. Its authors run it at team scale
in a company and it works. We adopted it about eight months ago, wired two of
its skills into `epic-create` once each, and have not revisited it since. paad
2.0 has shipped since then.

The long-term goal of the vergil workflow is that heavy front-loaded design
lets `epic-implement` run an epic to completion with as few human touch points
as possible. Getting more out of paad, both in design and in implementation, is
the main lever for that.

This umbrella epic:

1. Sets a **shared doctrine** for running paad reviews repeatedly: when to run
   another round, when to stop, what gets recorded, and who decides.
2. Brings our use of paad **in line with the authors' recommendations and paad
   2.0**.
3. Splits the work into child epics and tasks, ordered by explicit
   dependencies.

## 2. Governing principles

These apply to every child item.

### 2.1 paad-first; deviations need a stated reason

Follow paad's recommended workflow and conventions by default. Deviate only for
a stated reason. Every deviation goes in the **deviations register** (section
6) with that reason, so it stays visible and can be revisited. "We have always
done it this way" is not a reason.

The authors' per-feature loop, from the paad README and tutorial Module 10
(https://curtispoe.org/paad/):

1. Write the spec.
2. Pushback on the spec, revising and repeating as needed.
3. Write the task list (our plan).
4. Pushback on the plan.
5. Alignment of plan against spec.
6. Run each task in a fresh session.
7. agentic-review, fix, and repeat.
8. A human approves and merges.

Today we run steps 1, 2 (once), 3, 5 (once) and 8. Step 6 is roughly covered
because `epic-implement` hands each task to its own subagent. We skip steps 4
and 7.

### 2.2 Use subagents where the result matters more than watching the work

When the human cares about the result rather than watching the work in the
foreground, our skills must say to run the work in a subagent. For repeated
reviews this is also a correctness requirement: each round needs a context that
hasn't seen earlier rounds, so it doesn't defer to decisions made in them
(section 3.1).

### 2.3 Record everything

Nothing a review produces is thrown away. Every finding and the response to it
is recorded in issue comments: on the planning issue (mode 1) or on the
task (mode 2). Retrospectives can then judge whether the reviews earned their cost.
Security findings are recorded too, but only where the security-handling policy
(child A) allows.

### 2.4 No nits, no cracks

There is no such thing as a nit. Any problem found during work, whether by a
review or by an agent noticing something along the way, is recorded, even if
it is not this task's job to fix it. The plumber doesn't fix the crack in the
wall, but does report it.

Recording a problem is not enough on its own: every problem that was found
and not addressed must end up either **in the intake queue** or **explicitly
dropped by the human with a reason**. The intake queue is our system of record
for follow-on work. A finding that stays only in a comment, a report, or a
backlog file has slipped through a crack.

## 3. The shared iteration doctrine

One written doctrine covers every repeated paad review: pushback on a spec,
pushback on a plan, alignment, and agentic-review on code. It has two modes,
chosen by **whether a human is in the loop**, and both share a common core.

### 3.1 Common core

- **One round:** a fresh subagent runs the paad skill on the current artifact,
  in its current state, and returns its full findings. Each round starts with
  no memory of earlier rounds. It is told to review the whole artifact,
  **including mechanisms that earlier rounds added**, and removing an earlier
  addition is a legitimate finding.
  - *Why:* in #469's four-round experiment on epic #443, each round treated
    earlier decisions as settled and only added on top. Complexity compounded,
    later rounds mostly repaired earlier rounds' additions, and the epic was
    abandoned (see also .github#456).
- **Classify every finding** as one of:
  - **new:** a problem with the original design or implementation;
  - **known:** already raised or deliberately accepted in an earlier round;
  - **repair:** the problem is in something a previous round added.

  **The reviewer never classifies; the orchestrator does,** after the round.
  In mode 1 that is the main session, and in mode 2 the implementer. It
  matches each finding against the planning-issue or task-issue comment
  history and the diffs of earlier rounds. Keeping this step out of the
  reviewer preserves the reviewer's fresh context.
- **Precedence:**
  - continue signals override stop signals;
  - in mode 2, the rules in 3.3 override those in 3.1. For example, a round
    that is mostly repairs is a *stop* signal in mode 1 but a *pause* in
    mode 2.
- **One severity scale for design reviews**.
  - **The reviewer assigns severity.** The review subagent returns a severity
    with every finding: alignment uses Critical, Important and Minor, and
    pushback's report scale runs from critical to minor.
  - **The orchestrator maps it** onto *high*, *medium* or *low* so that
    checkpoint counts mean the same thing from round to round. It does not
    re-grade.
  - agentic-review keeps its own Critical, Important and Suggestion levels.
- **Reviewer discards are not findings**.
  - Candidates the reviewer discards are recorded only as a count plus one
    line each in the round's record, and 2.4 does not apply to them.
  - "Dropped" means only a finding the human decided not to act on. That
    always carries a reason or goes to intake.
- **Stop signals:**
  - a round raises nothing new (the authors' rule: "stop when a pass returns
    nothing you did not already know");
  - only one or two minor findings remain;
  - most findings are repairs;
  - the artifact has grown in each of several consecutive rounds.
- **Continue signals:**
  - any high-severity or structural finding;
  - the previous round changed the design or code materially.
- **Treat every round as a sample,** not a complete list (the authors'
  wording). A clean round is evidence, not proof.
- **Security findings follow the same rules** as every other finding. The only
  difference is where their details are recorded (child A). Until A lands,
  details stay in the worktree's `paad/security/` (which paad 2.0 creates as
  gitignored), and every public record shows only a count and where the details
  live.

### 3.2 Mode 1: human-gated (design review in `epic-create`)

Used in the interactive design session, where judgment calls belong to the
human.

- **Stages.** The authors' full design loop: pushback rounds on the spec, then
  the plan is written, then pushback rounds on the plan, then alignment rounds.
  Every stage uses this mode.
- **Running the review without the human.** pushback and alignment normally
  present one finding at a time and edit the artifact as they go. In mode 1
  the subagent prompt tells the skill to review only:
  - collect every finding;
  - edit nothing and write nothing (except paad's `paad/security/` file);
  - ask nothing;
  - return the full ranked list with its discards.

  If prompting alone proves unreliable, the fallback is
  `paad/config/pushback.md` and `paad/config/alignment.md` (experimental; B
  decides adoption and version pinning), and after that a request to the paad
  authors for a report-only mode.
- **Each round:**
  1. A subagent runs the skill and returns findings (non-interactive).
  2. The main session walks the human through the findings one at a time, as
     pushback does today, and applies what the human decides.
  3. The change is committed.
  4. The main session writes the checkpoint report.
- **Checkpoint report** (short prose plus counts):
  - findings raised, accepted, rejected, and dropped;
  - the severity mix;
  - counts of new, known and repair findings;
  - whether the artifact grew or shrank (rough line or section delta);
  - a **recommendation** to continue or stop, with the signal (3.1) behind it.
  From round 4 on, the report says explicitly that the loop is past the
  typical number of rounds.
- **The human decides** whether to run another round, every time. The
  recommendation is advice only. Stopping early on simple work is a normal,
  expected outcome; this pipeline is sometimes used for small things.
- **No review history inside the reviewed document.** The artifact holds
  no per-round log and no inline tags saying which round or decision
  produced which text. The next reviewer reads the artifact, and that
  provenance would bring back the deference to earlier rounds that 3.1
  rules out. The artifact carries one pointer line to the planning issue,
  and commit messages say what each round changed. This replaces today's
  one-line `Review: paad pushback, N issues…` header and the ad hoc
  "Alignment (paad, …)" bullet.
  - git history leaks provenance too: pushback's own history check read the
    previous round's commit diff. C decides how to handle that, for example
    by telling the reviewer to use git history only for conflicts with
    reality.
- **The full record goes in planning-issue comments**.
  - **What gets posted:** after each round, the main session posts a comment
    on the epic's planning issue with every finding, the human's decision on
    each, and the checkpoint report.
  - **Security findings** appear only as counts, with their details kept
    where child A's mechanism keeps them.
  - **Dropped findings** carry the human's reason, or go to intake (2.4).
  - **Every epic must have a planning issue.**
    - If `epic-create` starts from one (usually an intake item promoted to
      planning), that issue is where the planning record goes.
    - If it doesn't, step 0 creates one.
    - Its comments become the permanent log of how the epic was planned, from
      the original idea through every design-review round. It is expected to
      grow large.
  - **Afterwards it stays where it is.** The planning issue closes in the
    planning epic and drains into its archive like every other planning item.
    It is never moved to the epic it produced.
    - The two are linked in both directions: the epic body and the spec header
      carry a `Planning:` / review-history line, and the planning issue's
      closing comment points to the epic.
    - Re-parenting stays an occasional manual action and is not part of the
      tooling.
  - **Why issues and not paad's report files:** the spec, the plan, the
    retrospective and the issues are our durable record. paad's report files
    are not used for mode 1.

### 3.3 Mode 2: autonomous (agentic-review in `issue-implement`)

Used inside `issue-implement`, which is meant to run without a human from start
until the branch is ready. The two agents (implementer and reviewer) settle the
review between them and pause for a human only when they can't.

- **Where:** after `vrg-container-run -- vrg-validate` passes and before
  `vrg-pr-workflow report-ready`, with all work committed.
- **Always runs**, on every `issue-implement` run, including docs-only and
  trivial changes (decided at the 2026-10-24 intake review of plugin#677). An
  exemption rule may come later, after we see real cost and value.
- **Issue-comment format.** Each comment opens with a fixed header line
  (`agentic-review round <n>: findings` or `agentic-review round <n>: response`),
  so that a resumed session can rebuild the round count and state from the
  comments.
- **A pause** stops `issue-implement` before report-ready, with the branch
  unfrozen. The run reports the reason, as any other human-intervention stop
  does. `epic-implement` lists it among the items waiting on the human. When
  the run resumes, it reads the issue comments and continues the round count;
  it does not start from round 1.
- **Each round:**
  1. A fresh subagent runs `paad:agentic-review` against the issue's
     requirements, passing the base branch explicitly.
  2. The reviewer's findings are posted as an **issue comment**.
  3. The implementer acts on each finding: it fixes it, or records a reason
     for not fixing it. Fixes are ordinary code changes and follow
     `issue-implement`'s normal validate-and-commit discipline. There is no
     review-specific validation rule.
  4. The implementer posts its response to every finding as a second **issue
     comment**.
  5. If any blocking finding was fixed, a new round starts. Otherwise the loop
     ends.

  As always, the branch must pass `vrg-validate` at its final commit before
  report-ready.
- **Severity rules** (starting settings; tune after a few weeks of real use):
  - **Blocking:** in-scope *Critical* and *Important* findings. The implementer
    must fix them, and a new round follows.
  - **Discretionary:** in-scope *Suggestion* findings. The implementer fixes
    each one or records why not, using its wider context. They never cause a
    new round on their own.
  - **Out-of-scope additions** (code this branch changed that the issue did
    not ask for) are **scope violations, not findings**. An agent implements
    only its issue; any other problem it notices is *reported* (2.4), not
    fixed on the branch.
    - When agentic-review flags an out-of-scope addition, the implementer
      removes it from the branch and files it as its own task: under the same
      epic, or under the repo's ad-hoc epic when there is no finite epic. That
      task gets its own PR.
    - It does not go to intake, because it is understood work, not an
      uncurated problem.
    - The handoff lists every removal with the task it became.
    - It never pauses the run.
  - **Out-of-scope pre-existing bugs:** feed our **intake queue** (2.4), which
    is the system of record. paad also writes them to its own backlog file;
    D decides whether we keep that file only as agentic-review's
    de-duplication aid or retire it, and records the choice in the deviations
    register.
- **Pass:** a round with no blocking findings. The run continues to
  report-ready.
- **Pause for the human** (the run stops and says why) when:
  - round 3 had any blocking finding, fixed or not.
    When the human resumes the run, they may allow more rounds; without that,
    the cap is 3 reviewed rounds;
  - the implementer wants to dismiss a blocking finding as wrong;
  - most of a round's findings are repairs of the previous round's fixes.
- **Honest handoff.** The report-ready notes, and the summary shown to the
  human, list **every finding that was not fixed**, each with the implementer's
  reason. They also give the number of rounds and the outcome. This is a new
  quality gate, and the human needs to see how it is performing. (Report-ready
  notes are public through the relay ref, so security findings appear there
  only as counts.)
- **The implementer files intake issues at handoff.** Before report-ready,
  the implementer runs `vrg-intake-create` for every finding that was not
  addressed:
  - Suggestion findings it deferred;
  - findings it dismissed;
  - out-of-scope (pre-existing) bugs;
  - anything else it noticed along the way.

  Each intake issue links back to the task and the review round it came from.
  The report-ready notes list the intake issues created. Security findings go
  through A's mechanism instead, never into a public intake issue.
  - Filing too much is preferred. Some noise in intake, filtered later,
    beats a small problem that is skipped and becomes a large one.
  - The rule applies to **every** task, epic or ad hoc. Ad-hoc tasks under
    standing epics never get a retrospective, so a uniform rule leaves no
    gap.
- **Unaddressed-findings sweep in the retrospective.** `epic-retrospective`
  takes on a new duty and acts as the catch-all that verifies the handoff
  filing: no epic closes with an untracked problem. It collects
  every unaddressed finding recorded on the epic's tasks:
  - deferred Suggestion findings;
  - dismissed findings;
  - out-of-scope (pre-existing) bugs;
  - out-of-scope additions that were removed, checking that each has its
    own task;
  - any other problems noted in issue comments during the epic.

  For each one, it confirms that an intake issue exists, or creates one, or
  the human explicitly drops it with a reason. The sweep covers all recorded
  feedback, not only agentic-review's, so mode 1 checkpoints and human review
  comments are included too.
- **Running unattended.** agentic-review 2.0.2 already runs to completion and
  ends with a report. It stops on its own in only four cases, and each is
  avoided by construction:
  1. *The conversation has history:* a fresh subagent has none.
  2. *Uncommitted changes:* the implementer commits first. paad's own output
     files also count as uncommitted changes, and are handled by the
     output-redirection requirement below.
  3. *`paad/security/` is unsafe:* paad 2.0 creates
     `paad/security/.gitignore` (containing `*`) itself when it is missing.
     It stops only when that file holds anything else, or when git already
     tracks files under `paad/security/`. Neither happens if nothing there
     is ever committed. B's root-level ignore line is defence in depth.
  4. *Ambiguous arguments:* the base is passed explicitly.

  If prompting proves insufficient, add `paad/config/agentic-review.md` saying
  "unattended run: never ask, finish with the report". This is paad's
  experimental config mechanism (CONFIG.md). paad stays unpinned (see B), so
  every config file names the paad version it was written for, and D detects
  drift. If neither works, raise it with the paad authors.
- **Results come back through the issue, not the file system.**
  - **What paad does by default:** agentic-review returns its results as files
    (`paad/code-reviews/<report>.md`, `backlog.md`). Left untracked, those
    files make round 2's clean-state check stop and ask, and they make
    `vrg-finalize-pr` treat the worktree as dirty (`worktrees.py:292-293`).
  - **Why we can change it:** paad's file layout is its chosen mechanism, not
    a requirement. We already replaced it for specs and plans with our own
    `epics/` layout. paad's skills are instructions to an agent, not a library
    API, so our own instructions (the subagent prompt and
    `paad/config/agentic-review.md`) can redirect how results reach the
    caller, as long as they stay compatible with paad's.
  - **D designs this,** with three requirements:
    - the results reach the implementer through the subagent's answer and are
      recorded in issue comments;
    - no paad output is left untracked in the worktree between rounds or at
      finalize;
    - paad's de-duplication of out-of-scope findings is either preserved or
      deliberately replaced (for example by matching against intake), and the
      choice goes in the deviations register.
  - **If we can't redirect it ourselves,** take it to the paad authors and
    ask for an option to configure where output goes.
- **Checking the output every time.** Because this step is not deterministic,
  each run checks two outputs, by their structure rather than their prose:
  - **the report:** its Review Metadata has no `verifier-warning` line;
  - **the orchestrator's final answer:** it has the counts line, and a
    `Config:` line when a config file was used.

  A `verifier-warning` (for example `verifier ref-token-missing`) or a missing
  structural element counts as a **failed round** and pauses the run. It is
  never treated as a pass. D adapts these checks to whatever output routing
  it designs.

## 4. Child items and dependencies

Ad hoc dependency handling until .github#88 provides a real mechanism:
- children are linked to the umbrella **by text only**. Each child's body
  names the umbrella (`Umbrella:`) and its prerequisites (`Depends on:`). A
  child epic stays top-level, because `vrg-epic-create` makes top-level epics
  and `vrg-epic-link` refuses to nest an epic (vergil-tooling #3591);
- **gate tasks** (below) are what hold the umbrella open;
- this table is the authoritative map.

**Order.**
The order follows the dependencies only; anything without a dependency runs
whenever it is ready.
- **A and B** have no dependencies and can run in parallel. A is the first
  priority: the security-finding infrastructure and its tooling must exist
  before any child that produces findings (C and D) runs.
- **C and D** need A and B.
- **E** needs B.
- **F** comes last.

**Lifecycle of a child.**
- Until it is promoted, a child that starts as a planning item sits under the
  planning queue epic (.github#439), as every planning item does.
- When `epic-create` promotes it, the new epic is created top-level with an
  `Umbrella:` line in its body, and the planning item is closed with a
  `Planned as <epic>` comment.
- **Gate tasks hold the umbrella open.** Text links do not count toward
  rollup, so each of A, C, D, E and F gets a **gate task** filed under the
  umbrella when the umbrella is set up, titled `Child <X> complete: <name>`.
  - It is a `validation`-kind task.
  - It names its child's planning item.
  - Its precondition is that the child's work is finished: the planning item
    is closed, and every issue its closing comment names (`Planned as <epic>`
    or `Filed as task <ref>`) is closed too.
  - It closes on `Outcome: SUCCESS` through `issue-validate`, which is
    existing machinery.
  - Gate tasks carry no `Blocked-by:` links. With those links, a gate would
    become "runnable" as soon as its child *may start*, and then fail its own
    precondition over and over. The dependency order lives only in the table
    below and in each child's `Depends on:` line. The plan schedules every
    gate task to run last, just before the retrospective.
  - The umbrella's retrospective cannot run until every gate task is closed.

**How the umbrella itself runs.**
- It goes through the normal `epic-create` pipeline: spec (this document),
  pushback, plan, alignment, docs PR, and the bookend tasks.
- Its `plan.md` covers B, filing the child planning items, the gate tasks,
  the dependency order, and the bookends.
- `epic-implement` on the umbrella runs B's tasks. Child epics are not its
  sub-issues, so it never meets them; each child epic gets its own
  `epic-implement` run.
- The retrospective runs last, after every gate task has closed.

**How binding section 3 is**.
Section 3 holds two kinds of content.
- **Rules (the what) are binding on C and D.** These include the round caps,
  what blocks, when to pause, record everything, no nits and no cracks,
  results through the issue, and security details only as counts. C and D
  build them as written and do not reopen them during their own design
  review.
  - They are the **initial operating settings**. They are expected to be
    tuned after a few months of real use, through a deliberate pass based on
    the data the records provide, and not during this development.
- **Mechanisms (the how) belong to C and D.** These are the exact comment
  formats, the report fields to check, how the retrospective sweep matches
  findings, and the tooling. Where section 3 gives a mechanism, it is an
  example the child may improve, as long as the rules still hold.
- **Changes to a rule** after this point are a human decision, recorded in
  the deviations register if they move us further from paad.

| ID | Item | Shape | Depends on |
|---|---|---|---|
| A | Security-finding handling | planning item, then a small epic | none |
| B | paad 2.0 compatibility and hygiene | four tasks under the umbrella (plan Tasks 1–4) | none |
| C | Iterating design review in `epic-create` (mode 1) | planning item, then an epic (absorbs .github#469) | A; B's Tasks 1 and 3 |
| D | Agentic review in `issue-implement` (mode 2) | planning item, then an epic (absorbs plugin#677) | A; B's Tasks 1, 3 and 4 |
| E | Steering-file and Claude settings review | planning item | B's Tasks 1 and 3 |
| F | Wider paad adoption | planning item | C, D |

**A. Security-finding handling.**
- **Problem:** paad 2.0 keeps security findings out of git (`paad/security/`,
  gitignored). Worktrees are deleted at finalize, so the details are lost. Yet
  they must never land in public issues, comments, or relay-ref notes, and we
  have been careless about this in past issues.
- **Starting direction:** GitHub **draft repository security advisories**. They
  are private to admins and invited collaborators even on a public repo, can
  have a temporary private fork for the fix, and can be created through the
  REST API (`POST /repos/{owner}/{repo}/security-advisories`); see
  https://docs.github.com/en/code-security/security-advisories/working-with-repository-security-advisories/about-repository-security-advisories.
- **Expected deliverable:** a small tool to create, list and reference these
  records, so that issue comments can point to them without disclosing
  anything.
- **To verify:** that advisories exist only for public repos (if so, private
  repos use issues); and the `vrg-gh api` permission that agent identities
  would need.
- **Also in scope:** a policy for past issues that may have disclosed security
  problems.

**B. paad 2.0 compatibility and hygiene.**
- Add `paad/security/` to the managed `.gitignore` block in every repo,
  following paad 2.0's breaking change.
- Decide and write down where paad report files go (`paad/*-reviews/`):
  - Since the epic-create workflow moved into `.github`, reports are not
    committed anywhere.
  - The authors recommend committing them and recording the policy in the
    steering file.
  - Under 3.2, design-review records live in planning-issue comments, so
    report files are not needed for them; record whichever is
    chosen in the deviations register.
- Settle the overlap between `paad:handoff` and `vergil:handoff`.
- **paad version:** unpinned, following the latest release.
  - Agent VMs install plugins from the user-level `~/.claude/settings.json`
    copied from the host, and `vrg-vm update` moves every enabled plugin to
    its latest version, so a pin would not hold where agents run.
  - Every `paad/config/` file that C or D adopts names the paad version it was
    written for, and D detects version drift.
- Check that paad is installed on the agent VMs, through the user-level
  settings. D needs paad wherever `issue-implement` runs.

**C. Iterating design review in `epic-create`.**
- Implement mode 1 (3.2) across the three review stages: spec pushback, plan
  pushback, and alignment. Plan writing stays a single automated pass.
- Write the shared doctrine as a reference document in the vergil plugin that
  C and D both point to. C owns writing it; D extends it.
- Add the checkpoint report and the per-round comments on the planning issue,
  and remove review history from specs and plans (3.2).
- Make the planning-issue requirement explicit in `epic-create`, with the
  links in both directions described in 3.2.
- Rewrite the pipeline description in the docs listed under "Current state".

**D. Agentic review in `issue-implement`.**
- **First task: a spike**.
  - **What it tests:** the real chain, three levels below the main session:
    main session → `epic-implement` → `issue-implement` subagent → the fresh
    reviewer subagent → agentic-review's 6–7 specialists and verifier.
  - **What success means:** every lens is actually dispatched. A well-formed
    report from a single context, with any lens `NOT DISPATCHED`, is a
    failure. This "every lens dispatched" check also becomes one of mode 2's
    permanent output checks.
  - **If the spike fails,** D's design changes, and the spike's result goes
    back to the human before D is designed.
- Then implement mode 2 (3.3), including the issue-comment protocol, the
  honest handoff, and the retrospective roll-up in `epic-retrospective`.
- **Required inputs.** These are mechanism problems D
  must solve under section 3's rules:
  - agentic-review's Spec Compliance lens has no way to receive the
    issue's requirements. It accepts only a spec-file argument, a PR
    description, `docs/plans/`, or commit messages. A Spec Compliance bail
    must count as a failed round.
  - a human's decision at a pause (more rounds, or a ruling on a
    dismissed finding) needs a recorded form that a resumed run can read.
    `epic-implement` keeps no state of its own.
  - the handoff must leave a structured mapping from each unfixed
    finding to its intake issue, so the retrospective sweep can check it
    mechanically.
  - pass `origin/<default>` as the base after a fetch, never a
    possibly stale local branch.
  - the list of agentic-review's stop cases is incomplete. It misses
    the "hold or ordinary report" security question, the empty diff, and
    "both a ref and a path". `symbolic-ref` is also not on `vrg-git`'s
    allowlist. Every ask must have a standing answer, and the security
    question's answer is always *hold*.
  - a commit made after the last review round (for example a
    Suggestion fix) either triggers another round or is listed as unreviewed
    in the handoff.

**E. Steering-file review.**
- **Problem:** the repo CLAUDE.md and AGENTS.md files and the global
  `~/.claude/CLAUDE.md` have grown by small additions for months without ever
  being reviewed. Known problems:
  - this repo's CLAUDE.md is far longer than the authors' rule of thumb of
    about 200 lines;
  - AGENTS.md still tells agents to use raw `git checkout -b`.
- **The authors' guidance:**
  - validate the steering files first, because everything else inherits them;
  - keep them short;
  - keep one canonical file and have the others import it;
  - `/paad:pushback` can review steering files directly (since 1.30).
- **Scope:** a one-time cleanup plus a recurring review, possibly by extending
  `memory-audit`, which today covers only memory files.
- **The Claude Code settings files are also in scope.** That means the
  user-level `~/.claude/settings.json`, each repo's checked-in
  `.claude/settings.json`, and the managed template
  (`vergil-tooling/src/vergil_tooling/data/claude_settings.json`).
  - **The problem:** no one has written down which file is responsible for
    what. The repo files carry redundant and possibly conflicting entries.
  - **Starting facts,** from the current Claude Code docs, fetched 2026-10-26
    (https://code.claude.com/docs/en/settings.md,
    https://code.claude.com/docs/en/permissions.md,
    https://code.claude.com/docs/en/plugins/loading.md,
    https://code.claude.com/docs/en/hooks-guide.md):
    - precedence is managed, then CLI, then project-local, then project, then
      user;
    - permission lists combine, and a deny anywhere wins;
    - hooks add up across all sources;
    - `enabledPlugins` merges per plugin, and a plugin enabled only at project
      level is **not** installed: the fetch needs a user, local, flag or
      managed `true`;
    - a project's `permissions.allow` and `extraKnownMarketplaces` apply only
      after folder trust, and hooks run regardless.
  - **Known problems to resolve:**
    - the repo's `enabledPlugins` for `vergil` is redundant with user
      settings;
    - the repo pins `vergil-marketplace` to `ref: "main"`, which the user
      file and the managed template don't, and the docs don't say which
      wins;
    - it is unconfirmed whether empty-string `attribution` suppresses
      attribution;
    - the "v2.1.195 stopped auto-install" comment in `vm_guest.py` is
      unverified.
  - **The deliverable** is a written "which file owns what" model,
    re-verified against the docs at the time, plus a cleanup of all three.
    Any change to the managed template must keep the config audit passing
    across the fleet. Plugin enablement stays at user scope, which agent VMs
    copy from the host.

**F. Wider paad adoption.** Evaluate, one by one:
- `rethink` at decision points during brainstorming;
- `agentic-architecture` and `fix-architecture`, `test-roadmap`,
  `agentic-owasp` and `agentic-dedup` as periodic codebase-health runs;
- `backlog` (`clean` and `fix`) as tools, now that the intake queue is the
  system of record for out-of-scope findings (2.4).

Each one is either adopted, with a place in the workflow, or recorded as a
deviation with a reason.

## 5. Goals, non-goals, anti-goals

**Goals**
- Every repeated paad review follows one written doctrine.
- Design review runs the authors' full loop, with the human deciding how many
  rounds.
- Implementation gains an autonomous review gate whose outcome is fully
  recorded and reported honestly.
- Every difference from paad's recommendations is written down with its
  reason.

**Non-goals**
- A general epic-dependency mechanism. That is .github#88; this umbrella only
  tests the idea by hand.
- The automated merge-gate review agent (.github#90).
- Changing paad itself, unless an authors-side change turns out to be
  necessary (for example, an unattended mode). The user knows the authors
  personally.

**Anti-goals** (failures even if everything technically works)
- Rounds that only add to the design, so it gets more complex each round. This
  is the #469 failure.
- A review loop that runs forever, or that the agents treat as passed without
  a well-formed report.
- Review findings that are silently dropped, or a handoff that hides findings
  the implementer chose not to fix.
- Security details appearing in public issues, comments, PR bodies, or the
  relay ref.
- A design-time ceremony so heavy that the human can't stop it after one round
  on simple work.

## 6. Deviations register (initial seed)

This table is the seed. Once plan Task 3 lands, the living register is
`vergil-claude-plugin/docs/paad-policy.md`, and changes go there.

| Deviation from paad | Reason |
|---|---|
| Mode 2 caps autonomous rounds at 3, then pauses for the human | The authors say "repeat" with a human present. Unattended, an unbounded loop is unsafe. |
| paad is unpinned and follows the latest release, although D and C may rely on the experimental `paad/config/` | A pin would not hold on agent VMs: `vrg-vm update` moves every enabled plugin to latest. Mitigation: every config file names the version it was written for, and D detects drift. |
| Design-review report files (pushback, alignment) are not written; the planning-issue comments are the record (§3.2) | The planning issue is the epic's permanent planning log, and review history must stay out of the reviewed document so the next round's reviewer has a fresh context. |

## 7. Constraints

- Public repos: issue comments, PR bodies, report-ready notes and the relay ref
  are public.
- Agents use `vrg-git` and `vrg-gh` only. Agents never run `vrg-submit-pr`.
- paad's per-project config and backlog are **experimental** and can change in
  any release.
- Child items must keep `issue-implement` able to finish without a human in the
  normal case.

## 8. Current state (for reference)

- **Where paad is wired in:** only in `vergil-claude-plugin/skills/epic-create/SKILL.md`:
  - step 4 is `paad:pushback`, once;
  - step 7 is `paad:alignment`, once.
- **Docs that restate the pipeline:**
  - `vergil-tooling/docs/site/docs/standards/github-issues.md:302-305`
  - `docs/docs/site/docs/management/workflow.md:38-61`
  - `docs/docs/site/docs/management/tools-and-skills.md:27,111`
- **Implementation-side review:** none in `issue-implement` or
  `epic-implement`. The only gate before report-ready is `vrg-validate`.
- **Last committed paad reports:** `paad/pushback-reviews/` and
  `paad/alignment-reviews/` in vergil-tooling, vergil-actions,
  vergil-claude-plugin and vergil-containers, dated 2026-04-22 to 2026-05-20.
