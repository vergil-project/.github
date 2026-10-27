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

This epic builds that step into `issue-implement` as mode 2 of the paad review
doctrine: an autonomous loop in which the implementer and a fresh reviewer
settle the review between them, every round is recorded on the task issue,
every problem that is not fixed ends up in intake, and the handoff says
honestly what was not fixed. The human is pulled in only where the rules say
so. `epic-retrospective` gains the sweep that confirms no epic closes with an
untracked problem.

## 2. Inherited rules (binding, not reopened here)

Umbrella §2, §3.1 and §3.3 are binding on this epic as written (umbrella §4,
"How binding section 3 is"). In short:

- **Where and when:** after `vrg-container-run -- vrg-validate` passes and
  before `report-ready`, with all work committed; it **always runs**, including
  on docs-only and trivial changes.
- **One round:** a fresh subagent runs `paad:agentic-review` against the
  issue's requirements with the base passed explicitly; its findings are posted
  as an issue comment; the implementer fixes each finding or records why not,
  under `issue-implement`'s normal validate-and-commit discipline; its response
  is posted as a second issue comment; a new round follows if any blocking
  finding was fixed.
- **The implementer classifies** each finding as new, known or repair; the
  reviewer never does. The reviewer's severity is mapped, never re-graded.
- **Blocking:** in-scope Critical and Important. **Discretionary:** in-scope
  Suggestion, fixed or answered, never causing a round on its own.
- **Out-of-scope additions** are removed from the branch and filed as their own
  task under the same epic (or the repo's ad-hoc epic); they never pause the
  run and never go to intake. **Out-of-scope pre-existing bugs** go to intake.
- **Pass:** a round with no blocking findings.
- **Pause** (stop before `report-ready`, branch unfrozen, reason reported)
  when round 3 had any blocking finding; when the implementer wants to dismiss
  a blocking finding as wrong; when most of a round's findings repair the
  previous round's fixes; when an output check fails. A resumed run continues
  the round count from the issue comments; the human may allow more rounds.
- **Honest handoff:** the `report-ready` notes and the summary list every
  finding not fixed, with its reason, plus the round count and outcome.
- **Intake at handoff:** every finding not addressed, and anything else noticed,
  is filed with `vrg-intake-create`, linked to its task and round; filing too
  much is preferred; the rule applies to every task, epic or ad hoc.
- **Retrospective sweep:** `epic-retrospective` confirms an intake issue exists
  for every unaddressed finding recorded during the epic, creates one, or has
  the human drop it with a reason. It covers all recorded feedback, including
  mode 1 checkpoints and human review comments.
- **Results through the issue:** results reach the implementer through the
  subagent's answer and are recorded in issue comments; no paad output is left
  untracked in the worktree; paad's out-of-scope de-duplication is preserved or
  deliberately replaced, and the choice is in the deviations register.
- **Output checks** by structure: a `verifier-warning` or a missing structural
  element is a failed round, never a pass; every lens must be dispatched.
- **Security details** appear in public records only as counts, and go through
  child A's mechanism.

Umbrella §4.D also lists the problems this design must solve: the Spec
Compliance input, the recorded form of a human decision, the
finding-to-intake mapping, the `origin/<base>` rule, the missing stop cases,
and commits made after the last round. Each is answered below.

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
- **What it showed for the design:**
  - a requirements file named in the prompt works as Spec Compliance's intent
    source;
  - paad wrote its report and `backlog.md` to `paad/code-reviews/` in the
    worktree, its default. The design adapts to that default instead of
    redirecting it (4.4);
  - the reviewer silently corrected paad's output (it recomputed the backlog
    IDs), so the prompt must forbid editing paad's output;
  - the spike did not test the full prompt contract (4.5) or a plan-driven
    task; the plan's first task does (4.19);
  - raw `git` is denied by the vergil hook in a managed repo. With the prompt
    naming `vrg-git`, the explicit base and a `cd <worktree> &&` prefix,
    nothing was denied;
  - the review found two Important bugs in already merged code (filed as
    intake vergil-project/.github#563–#567 with three other findings).

## 4. Design

### 4.1 Where it lives

- **The doctrine.** Child C creates
  `vergil-claude-plugin/docs/paad-review-doctrine.md` with §1 (common core) and
  §2 (mode 1), and a stub §3 that this epic writes. §3 holds the mode 2 rules
  (section 2 above, as operating instructions) and every mechanism in this
  section: where paad's output goes and how it is collected, the requirements
  file, the reviewer prompt contract, the output checks,
  the comment protocol, loop control, security handling, intake filing and the
  handoff. §1's canonical severity table keeps one source; this epic confirms
  the agentic-review column (4.7).
- **`issue-implement`** gains a section, "The agentic-review gate", between
  "Implement" and "Record the PR metadata and hand off". It keeps only the
  procedure and links to doctrine §3 for the rules, as C does for
  `epic-create`.
- **`epic-implement`** and **`epic-retrospective`** change as 4.14 and 4.15
  describe.

### 4.2 Trusted authors

The gate acts only on issue content written by a **trusted author**: the
human's own GitHub account, or one of the human's agent App identities. Content
from any other author is never input. This applies to everything the gate
reads from an issue: the body and the "Requirement changes" comments that go
into `requirements.md` (4.4), every `agentic-review` header comment it uses to
rebuild state, and every decision comment (4.9).

- **An explicit allowlist of logins,** not GitHub's author association. The
  agents' own comments carry `authorAssociation: NONE`, so an association
  check cannot recognise them. Logins are compared after removing the `app/`
  prefix that GitHub puts on an App-authored issue (`app/<name>` as issue
  author, `<name>` as comment author).
- **Read with authors:** `vrg-gh issue view <N> --repo <owner>/<repo> --json
  author,body,comments`, which gives each comment's `author.login`.
- **Untrusted content is skipped and noted:** a comment from any other author
  is ignored, even one that carries an `agentic-review` header, and the next
  comment the gate posts lists it by URL as ignored. An issue whose body is
  not from a trusted author pauses the run.
- **Round, response, pause and summary comments** are trusted only from an
  agent identity. **Decision comments** are trusted from the human's account,
  or from an agent identity when the comment quotes the human's instruction
  (4.9).
- **Where the list lives:** the system-wide rule for every tool and skill that
  reads issues is planned separately (vergil-project/.github#579), and will own
  the canonical list. Until it ships, doctrine §3 names the trusted logins, and
  the gate reads them from there.

### 4.3 The gate in `issue-implement`

The gate is entered when the implementation is committed and
`vrg-container-run -- vrg-validate` is green. Its steps:

1. **Resume check.** Read the task issue's comments with their authors
   (4.2) and rebuild the state from the fixed header lines of trusted
   comments (4.9): the last round number, whether it ended in a pause, and any
   `agentic-review decision:` comment after the pause. With no earlier rounds,
   start at round 1.
2. **Round** (4.4 to 4.8), then the response comment.
3. **Decide** (4.10): a new round, a pass, or a pause.
4. **On a pass,** including one reached by an `accept` decision: file the
   intake issues (4.12), post the summary comment (4.9), confirm `vrg-validate` is green at the final
   commit, and go to `report-ready` with the handoff notes (4.13).
5. **On a pause:** post the pause comment and stop, reporting the reason. The
   branch is not reported ready, so it is not frozen.
6. **On a `stop:` decision:** file the intake issues exactly as on a pass,
   post the summary comment with outcome `stopped`, and end without
   `report-ready`. Every end state files what was found, so nothing depends on
   a pass (umbrella §2.4).

**Entering the gate on a resumed run.** `issue-implement` gains a resume entry
before its preflight step 5 (worktree creation): when the worktree for this
task's branch already exists and the issue carries trusted `agentic-review`
comments, it skips worktree creation and implementation and goes straight to
the gate's resume check. Without this, preflight would try to create a
worktree that already exists.

**A paused run is unfinished work, not a checkpoint.** The branch is not
pushed at a pause: `issue-implement` pushes only at `report-ready`, which is the
checkpoint. The commits a pause comment cites therefore exist only in that
machine's worktree, and the human inspects them there or asks the agent to
show them. The pause comment says so, and names the machine (the
`vrg-whoami --platform` value) and the worktree path. The risk is accepted: a
pause asks for the human's prompt attention, and paused work that is not
resumed promptly is treated as abandoned. If the worktree is lost or goes
stale, the human records `stop: <reason>` and the task is restarted from
`develop`, not salvaged. Long-lived unfinished branches are a hygiene problem
the workflow avoids.

### 4.4 Where paad's output goes, and the requirements file

**Adapt to paad; do not redirect it.** paad's skills are prose a model
follows, and the parts of them we would override are marked binding in paad's
own text (for example, its report template calls a skipped backlog write "a
review defect"). Instructions that contradict a skill are the least reliable
part of any prompt. The caller is code we control, so the design lets paad do
what it does by default and does the routing in the caller.

- **paad writes where it always does,** relative to its working directory,
  which is the task's own worktree: the report and `backlog.md` under
  `paad/code-reviews/`, and security findings under `paad/security/`.
- **Both directories are ignored.** `paad/code-reviews/` joins `paad/security/`
  in the managed `.gitignore` block that `vrg-gitignore-sync` keeps in every
  repo, so paad's files never show in `vrg-git status --porcelain`. The
  worktree stays clean for the next round and for `vrg-finalize-pr`, and
  nothing paad writes can be committed by accident.
- **Every task's output stays in its own worktree.** Concurrent tasks in the
  same repo each review in their own worktree, so they never share or
  overwrite each other's paad files, and nothing is written to the main
  checkout. The output checks (4.6) assert this every round.
- **The implementer collects after each round** (4.6 first, then the rest):
  it reads the report at the path paad lists under "Files written", posts the
  findings comment from it, records each security finding through child A's
  mechanism (4.11), and then **empties `paad/code-reviews/` and
  `paad/security/`** in the worktree. The next round's reviewer therefore
  starts with no paad backlog and no earlier report, so nothing from earlier
  rounds reaches it through paad's files.

**The requirements file.** Each round also uses a fresh **temporary
directory** outside the worktree, created by the implementer with
`mktemp -d`. The reviewer runs on the same machine, so it reads the same path,
and the random name tells it nothing about which round it is in. The
implementer deletes the directory once the round's comments are posted;
anything a crashed run leaves behind is ordinary temp-directory litter that
the OS cleans up. It holds one file:

- `requirements.md`, written by the implementer before dispatch, with three
  parts:
  - the task issue's title and body (`vrg-gh issue view <N> --json
    title,body`);
  - when the issue belongs to a finite epic with a plan, the text of the plan
    task the issue implements, verbatim, and the spec sections that task
    cites, verbatim. Both are read from the epic home's `origin/develop`
    (`epics/<N>-<slug>/plan.md` and `spec.md`). An epic task's body is often
    only a pointer to its plan task, so without this Spec Compliance would
    have almost no intent to check against;
  - quoted under a "Requirement changes" heading, any comment from the
    human's own account (4.2) that changes what the task must do.

  It never contains `agentic-review` comments, so the reviewer's fresh context
  holds.

This answers the Spec Compliance problem in §4.D. agentic-review's argument
grammar accepts only a base and a path, so a spec file cannot be passed as an
argument: the skill would read it as a path filter and find an empty diff. The
prompt instead names the file and tells the orchestrator to give it to Spec
Compliance as its explicit spec file. That is Spec Compliance's own
first-listed intent source, so this supplies an input paad expects rather
than changing its behaviour, and the spike showed it works.

### 4.5 The reviewer prompt contract

The doctrine gives a prompt template. The implementer starts a fresh
`general-purpose` subagent with it, in the background, and waits for it. Every
reviewer prompt must:

- **Invoke** `paad:agentic-review` through the Skill tool with exactly one
  argument: `origin/<base>`, after the implementer has run `vrg-git fetch
  origin`. `<base>` is the PR's base branch (the repo's default branch).
  A single remote-tracking ref never matches a path in the worktree, so the
  "both a ref and a path" question cannot arise, and the base is never a stale
  local branch.
- **Set the working directory:** the worktree's absolute path. Every Bash
  command starts `cd <worktree> && …`, because the shell resets after each
  call, and the reviewer passes the same rule to every subagent it dispatches.
- **Name `vrg-git`.** Raw `git` is denied by the hook in managed repos; the
  reviewer and every specialist use `vrg-git` with the same subcommands. Until
  vergil-tooling#3722 lands, the prompt also says `vrg-git` has no `-C` and no
  `symbolic-ref`; no default-branch resolution is needed because the base is
  explicit.
- **Give the requirements file:** its path, to be passed to Spec Compliance as
  the explicit spec file and to the verifier. The reviewer must not read the
  task issue, its comments, or any PR for the branch.
- **Pass no information about earlier rounds:** no round number, no earlier
  findings, no decisions. Review the whole change against the base, including
  code that looks like recent fixes.
- **Leave paad's file handling alone.** The prompt says nothing about where
  paad writes; paad writes its default paths in the worktree (4.4).
- **Forbid edits:** change no file in the repository other than what paad
  itself writes under `paad/`, and never edit paad's output after it is
  produced. A discrepancy the reviewer notices (as in the spike) is reported
  as a note, not corrected.
- **Give the standing answers,** so the run never stops to ask:
  - *conversation history:* none exists (fresh subagent);
  - *uncommitted changes:* stop and return `precondition failed: uncommitted
    changes`. The implementer always commits first, so this is a bug, and the
    round fails (4.6);
  - *ref or path:* cannot arise with `origin/<base>`; if asked, it is the base;
  - *empty diff:* return paad's "No changes to review" answer; the round
    fails (4.6);
  - *security hold or ordinary report:* always **hold**. paad asks this only
    when `paad/security/` is unsafe, and a hold means paad writes no security
    file, so the round **fails** (4.6): the `Questions:` line says only
    `security hold`, and the human fixes `paad/security/` before resuming;
  - *anything else:* do not ask; take the most conservative option, continue,
    and list the question and the choice in the answer.
- **Give the return format:**
  1. first line `Running paad:agentic-review v<version>`;
  2. a `Config: <path>` line if a paad config file was followed;
  3. paad's Post-Review answer as paad produced it, including its
     `Files written or updated:` list with **absolute** paths and its counts
     line (`Critical: N (in-scope) / X (out-of-scope), Important: …,
     Suggestion: …`). The full report is read from its file by the
     implementer, not retyped by the reviewer;
  4. `Files written:` every path written, absolute;
  5. `Questions:` each question met and the standing answer used, or `none`;
  6. `Notes:` discrepancies noticed, or `none`;
  7. `Security: <k> finding(s) in paad/security/<file>`, or `Security: 0`.

### 4.6 Output checks and failed rounds

After every review the implementer checks structure, not prose:

- the version line, the counts line, and the `Files written:`, `Questions:`,
  `Notes:` and `Security:` lines are present; a `Config:` line is present when
  a config file was followed;
- the report has its Review Metadata block;
- **every lens dispatched:** `Agents dispatched:` names all six lenses, none
  marked `NOT DISPATCHED`. On a large diff paad partitions each lens into two
  instances; each lens must still appear;
- **Spec Compliance ran:** it is not `bailed`, and `Intent sources consulted`
  names the requirements file. A Spec Compliance bail is a failed round (§4.D).
  Other lenses may bail with their own reason token; that is a lens doing its
  job, not a failure;
- `Verifier warnings: none`;
- **no security hold:** the `Questions:` line does not record a security hold;
- **no security detail outside `paad/security/`:** the `Questions:` and
  `Notes:` lines, which are copied into the public findings comment, carry no
  security detail;
- **the output is in this worktree:** every path under `Files written:`
  resolves to a file under this task's worktree, inside `paad/code-reviews/` or
  `paad/security/`, and the report file exists there;
- **nothing escaped it:** the main checkout's `paad/code-reviews/` and
  `paad/security/` are unchanged from before the round (the implementer lists
  them before dispatch and compares), and no path outside the worktree was
  written;
- `vrg-git status --porcelain` in the worktree is empty. Both `paad/`
  directories are ignored, so paad's files do not show; anything that does show
  is a stray edit.

A failed check is a **failed round**: it is recorded in the findings comment
(4.9) and **pauses** the run (§3.3). It is never a pass, and the findings of a
failed round are not acted on. Before pausing, the implementer restores the
worktree to its committed state: it reverts any tracked file the reviewer
changed and deletes any untracked file it created. paad's files from a failed
round are kept until the pause comment is posted, so the human can inspect
them, and security details among them are handled under 4.11.

### 4.7 Classification and severity

**Severity.** agentic-review's scale is Critical, Important and Suggestion
(umbrella §3.1). This epic confirms the starting agentic-review column of the
canonical table in doctrine §1 (child C's spec, §3.6): Critical → `critical`,
Important → `high`, Suggestion → `medium`. The mapped value is used for
counts and for `vrg-security-record --severity`; blocking is decided on paad's
own scale.

**Scope bucket.** paad's verifier assigns each finding to in-scope,
out-of-scope (bug) or out-of-scope addition. The implementer takes the bucket
as given, as it takes severity.

**Classification** (the implementer's job, after the round):

- **known:** an earlier round on this task raised the same problem (the same
  claim about the same code, matched by file, symbol and excerpt), as recorded
  in its findings comments;
- **repair:** the code the finding is anchored on is code an earlier round's
  fix added or changed, matched by content against the `fixed:` lines of
  earlier response comments (file, symbol and excerpt), not by SHA or line, so
  the match survives a rebase;
- **new:** anything else. Round 1 has only new findings.

Where a finding is both known and repair, it is known. A known blocking finding
is blocking again: an earlier fix did not hold.

### 4.8 Acting on findings

| Finding | Implementer's action |
|---|---|
| In-scope Critical or Important | Fix it. If the implementer believes it is wrong, it does not fix it and the run pauses (4.10). |
| In-scope Suggestion | Fix it, or record why not. A deferred one goes to intake at handoff. |
| Out-of-scope addition | Remove it from the branch and file it as a task (below). If the implementer judges the code is required by `requirements.md` and was misclassified, it keeps it instead, and records `kept: <quote from requirements.md>` in the response. |
| Out-of-scope bug | Leave it; at handoff, file it in intake (4.12). |

- **Fix commits** are ordinary `vrg-commit` commits whose messages describe the
  change, never the review: no round number, finding ID or "per review". Commit
  messages are one of Spec Compliance's intent sources, so a message naming the
  review would carry earlier rounds into the next reviewer's context. The
  response comment maps each finding to its fix by content (4.9), except
  security findings, whose fixes are never mapped in public (4.11).
- **Validation** follows `issue-implement`'s normal discipline; there is no
  review-specific rule. Each round starts from a committed state, and the
  branch must be green at its final commit before `report-ready`.
- **Removal tasks** are filed **at removal**, not at handoff, so removed code
  is never lost when a run ends early: `vrg-issue-create --epic <the task's
  epic, or adhoc> --repo <owner>/<repo> --blocked-by <task ref> --title …
  --body-file …`. `--blocked-by` keeps `epic-implement` from starting it
  before the code it builds on has merged. The body describes the removed
  code (enough to rebuild it), and names the task and the round finding it
  came from. Removal never pauses the run.
- **Anything else noticed** along the way (a problem the review did not raise)
  is filed in intake at handoff, like an out-of-scope bug (umbrella §2.4).

### 4.9 The issue-comment protocol

All comments go on the task issue, posted with `vrg-gh issue comment <N>
--repo <owner>/<repo> --body-file <tmp>`. Each opens with a fixed header line,
which is how a resumed run rebuilds its state:

| Header line | Posted | Contents |
|---|---|---|
| `agentic-review round <n>: findings` | after each review | the reviewed base and head SHAs; the paad version; the check result (`checks: passed`, or `checks: failed (<which>)`); counts by paad severity, mapped severity, bucket and classification; one entry per finding; discards; security count; the `Questions:` and `Notes:` lines |
| `agentic-review round <n>: response` | after acting | one line per finding: `fixed: <file> :: <symbol>: <what changed, with a short excerpt of the new code> (commit <sha> at the time)`, `not fixed: <reason>`, `removed -> <task ref>`, `kept: <requirement quote>`, `deferred` (filed at handoff), or `dismissal requested: <reason>`; then the decision: `next: round <n+1>`, `next: pass` or `next: pause` |
| `agentic-review paused: <reason>` | on a pause | the reason, the decision needed from the human, and how to resume |
| `agentic-review decision: <decision>` | after a pause | the human's ruling (below) |
| `agentic-review summary: <outcome> after <n> round(s)` | before `report-ready` | the handoff record and mapping block (below) |

**References are by content, never by SHA or line alone.** `vrg-submit-pr`
rebases every branch onto its base at submit, and a mid-gate rebase is allowed
when a conflict forces one, so commit SHAs and line numbers change. Every
record therefore identifies code by its **file and symbol plus a short excerpt
of the code itself**, and any SHA or line number it carries is marked as "at
the time" and is informational only. Matching (classification in 4.7, the
retrospective sweep in 4.15) uses the content, never the SHA.

**Finding entries.** Each finding has an ID `R<n>.<i>` (round, then index),
its paad and mapped severity, bucket, classification, its location (file,
symbol, and the anchored code excerpt, with paad's `file:line` at the time),
the claim, and paad's suggested fix. The full report follows in a collapsed `<details>`
block. A comment over GitHub's 65,536-character limit is split into parts, each
first line followed by `(part <i>/<m>)`, as in C.

**Human decisions** (§4.D). At a pause, the human decides in conversation.
The decision is recorded as an `agentic-review decision:` comment, posted by
the human or, on the human's explicit instruction, by the agent, which states
that it is posting the human's decision. An agent never posts a decision on
its own judgment.

**The human has the final say.** The agents follow deterministic rules, and
wherever the rules leave a judgment call, the run pauses and the human makes
it. A decision is never second-guessed: the agent records it, with the human's
reason, and acts on it. The human may disagree with the review, or agree with
it and accept the work anyway, or accept it and have the finding filed for
later; each is a legitimate ruling. The forms are:

- `accept: <reason>`: the run passes as it stands. Any finding still open is
  recorded as accepted by the human with that reason, and the last round's
  fixes are listed as `unreviewed:`;
- `accept, file R<n>.<i>: <reason>`: the same, and the named findings are
  filed in intake at handoff, mapped as `unaddressed:` with the reason;
- `allow <k> more round(s)`;
- `R<n>.<i> dismissed: <reason>` (the finding stops blocking; it is recorded as
  dismissed and filed in intake at handoff, per §3.3);
- `R<n>.<i> must be fixed`;
- `stop: <reason>` (the task is abandoned or re-planned; the run does not
  continue).

**The summary mapping block** (§4.D). The summary comment ends with a fenced
block of one line per unaddressed item, so the retrospective sweep can check
them mechanically:

```text
unaddressed: R1.3 -> vergil-project/.github#600 (suggestion, deferred)
unaddressed: R2.1 -> vergil-project/.github#601 (out-of-scope bug)
unaddressed: R2.4 -> vergil-project/.github#602 (dismissed by decision)
accepted: R3.2 (accepted by the human: "<reason>")
noticed: vergil-project/.github#603 (found while implementing)
removed: R1.5 -> vergil-project/vergil-tooling#3800
kept: R1.6 (required by: "<requirement quote>")
unreviewed: <commit subject> (files: <paths>; 1a2b3c4 at the time)
security: S1.2 -> fixed
security: S2.1 -> recorded
```

A finding matched to an existing issue (4.12) maps to that issue.

### 4.10 Loop control

- **New round** when the round's checks passed, it had a blocking finding,
  and every blocking finding was fixed.
- **Pass** when the checks passed and there was no blocking finding (after any
  decision comment is applied).
- **Pause** when:
  - the checks failed (4.6);
  - the round was the last allowed one (3, plus any rounds a decision allowed)
    and it had any blocking finding, fixed or not;
  - the implementer requests dismissal of a blocking finding;
  - a fresh reviewer flags again, as an out-of-scope addition, code the
    implementer kept as required in an earlier round (`kept:`). The second
    flag is the signal that the call needs the human;
  - from round 2 on, more than half of the round's findings are repairs.

  Pause conditions are checked after the response, so a pause comment always
  follows a complete response.
- **Resume.** The run reads the comments, applies the decision comment, and
  continues: `allow <k> more round(s)` raises the cap and starts round
  `<n+1>`; a dismissal or a "must be fixed" ruling is applied to the open
  finding and the decision in 4.10 is re-taken for that round. A pause with no
  decision comment does not resume; the run reports that it is still waiting.
- **A half-finished round.** A round with a findings comment and no response
  comment (the run crashed or lost its context mid-fix) is rerun as a fresh
  review of the branch as it stands. The new findings comment keeps the same
  round number with `(rerun)` added to its header, and the earlier findings
  comment of that round is not acted on.
- **Commits after the last round** (§4.D). Suggestion fixes and removals in the
  passing round produce commits the reviewer never saw. They do not trigger
  another round, since a Suggestion never causes a round on its own. Each is
  listed as `unreviewed:` in the summary and in the handoff, and the final
  commit must pass `vrg-validate`.

### 4.11 Security findings

- The reviewer **holds** every security finding, so paad writes it under
  `paad/security/` in the worktree (ignored by paad's own `.gitignore` and by
  the managed `.gitignore`).
- The implementer handles each security finding under the same rules as any
  other: blocking or not, fixed or not, in scope or not.
- **Recorded when the round is collected,** not at handoff, because the
  implementer empties `paad/security/` before the next round (4.4) and a
  worktree's ignored files do not survive finalize or a move to another
  machine.
- **Through the one-to-many records model of child A′**
  (vergil-project/.github#573), which this part of the design depends on:
  - at collection, every new security finding gets its **own** record: a
    private advisory and a dedicated neutral record issue that names the task
    as the work that found it (`Found in: <task ref>`). One task can produce
    several, which A′'s model holds;
  - a finding that repeats one already recorded for this task (matched by the
    implementer against the task's records, since paad's security backlog is
    emptied each round) is not recorded twice;
  - **fixed on the branch:** the advisory is handed to the task under A′'s
    attach rule ("this issue's own work fixes it"), so it closes when the task
    does;
  - **not fixed** (deferred, dismissed or out of scope): the record stays
    with its own record issue, so the problem stays open after the task
    closes.
- **Public records carry no security detail,** only neutral IDs and counts:
  - security findings get IDs `S<n>.<i>` (round, index) that carry no detail.
    In findings and response comments each appears only as its ID and
    `security: see advisory`; the response gives counts
    (`security: <k> fixed, <m> recorded`);
  - a commit that fixes a security finding gets a neutral message that says
    nothing about the weakness, and its SHA is never mapped to the finding in
    any public place; the advisory's private details record the fix by
    content (file, symbol and excerpt), with the SHA at the time;
  - the summary's mapping block carries one line per security finding with no
    detail (`security: S1.2 -> recorded` or `security: S1.2 -> fixed`), which
    the retrospective sweep checks against `vrg-security-record list` (IDs and
    states only, 4.15);
  - the counts-only rule covers task-issue comments, commit messages, the PR
    title and body, the pause comment and the `report-ready` notes. No public
    record carries a description, file, symbol or GHSA ID. A blocking security
    finding appears in a pause comment only as "a security finding blocks; see
    its advisory".

### 4.12 Intake filing and de-duplication

paad's committed code-review backlog is **retired in mode 2** and replaced by
intake, the system of record for follow-on work (umbrella §2.4). This is the
de-duplication choice §3.3 asks for, and goes in the deviations register.

- **What is filed at handoff:** every deferred Suggestion, every dismissed
  finding, every out-of-scope bug, and everything noticed along the way, with
  `vrg-intake-create --title … --body-file …`.
- **Body:** a source line (`Source: agentic-review of <task ref>, round <n>,
  finding R<n>.<i>`, or `Source: noticed while implementing <task ref>`), then
  a match line `Finding: <file> :: <symbol> :: <bug class>`, then the claim,
  the impact and paad's suggested fix.
- **De-duplication:** before filing, the implementer searches open issues in
  the intake queue's home and in the task's repo for the same file and symbol
  (`vrg-gh issue list --search`). On a match it comments on the existing issue
  (`Seen again: <task ref>, round <n>, R<n>.<i>`) and maps the finding to it,
  instead of filing a duplicate. The `Finding:` line exists to make this match
  mechanical over time.
- Filing too much is preferred (§3.3): when unsure whether two are the same,
  file a new one and name the possible duplicate.

### 4.13 The handoff

The `report-ready --notes` gain an `agentic-review` section: the outcome and
round count; every finding not fixed, with its reason and intake ref; every
removal with its task; every flagged addition kept as required, with the
requirement it rests on; the unreviewed commits; the security count; and the
paad version (with a drift note, 4.16). The same text is the summary shown to
the human, or returned to `epic-implement`. The notes are public through the
relay ref, which is why security appears only as a count.

### 4.14 `epic-implement`

- **Step 4 (the gate):** tasks paused at the agentic-review gate are listed
  among the items waiting on the human, each with its pause reason and the
  decision needed.
- **Step 2 (runnable frontier):** a task whose latest trusted gate comment is
  an `agentic-review paused:` comment with no later decision comment is
  **waiting on the human**, not runnable, so re-entry does not re-dispatch it.
- **Re-entry:** a paused task is not frozen and not finished. Once the human
  has decided, the decision comment is posted (4.9), and the task is run
  through `issue-implement` again, which resumes from its comments.
- **No other change.** `epic-implement` keeps no state of its own; the task
  issue holds it. Parallelism is unchanged; each running task now runs its own
  reviews.

### 4.15 The retrospective sweep

`epic-retrospective` gains a step after its preflight and before drafting:
**the unaddressed-findings sweep**.

1. For every closed task of the epic, read the task's comments:
   - each `agentic-review summary` mapping line: confirm the named issue
     exists; for `removed:`, confirm it is a task (§3.3);
   - each `agentic-review round <n>: response` line marked `not fixed`,
     `deferred` or dismissed that the summary does not map: an untracked
     problem;
   - other comments that record a problem (human review comments, notes made
     along the way) with no issue ref.
2. Read the planning issue's mode 1 round comments (child C's format) for
   dropped findings without a reason or an intake ref.
3. Read the human review comments on each task's PR for problems raised and
   not resolved in the PR.
4. For every untracked problem: find an existing intake issue, or create one
   (`vrg-intake-create`, linking the task and the comment), or have the human
   drop it with a reason. Security items are counted only, and handled through
   `vrg-security-record`.
5. Record the result in retrospective §4 ("New problems & opportunities"): the
   count of items checked, and each item that needed action with where it went.

Tasks implemented before this epic's plugin release have no agentic-review
comments; the sweep checks only steps 2–4 for them and says so.

### 4.16 paad version drift and the config fallback

- **Drift.** Doctrine §3 names the paad version the prompt contract was last
  verified against. Each findings comment records the version the reviewer
  announced. A difference is **not** a pause: the structural checks (4.6) catch
  real breakage. It is noted in the handoff (`paad <x> ran; contract verified
  on <y>`) so drift is visible, and a re-verification task is filed in intake
  the first time a new version is seen.
- **No config file.** `paad/config/agentic-review.md` is **not adopted**.
  paad reads config only relative to the working directory, so there is no
  global location: the file would have to exist in every managed repo,
  distributed by vergil-tooling's managed-file sync. And it is prose the skill
  follows, the same kind of instruction as the reviewer prompt, so it would
  not be more reliable. The design instead avoids overriding paad at all
  (4.4): the prompt supplies only environment facts, the requirements file and
  standing answers. If the re-spike (4.19) shows that one of those still needs
  a persistent home, adopting a managed config file is the next step, with its
  paad version named (paad-policy §1, §4).

### 4.17 `vrg-git` additions

vergil-tooling#3722 adds the read form of `symbolic-ref` and the global `-C`
option to `vrg-git`, and checks every other read-only command agentic-review
uses. The read/write policy is unchanged: read-only inspection is allowed, and
every command that changes things stays controlled. The gate does **not**
depend on it: the prompt's `cd` prefix and explicit base work today. Once it
ships, the prompt template drops the "no `-C`, no `symbolic-ref`" sentence.

### 4.18 Deviations register additions

Replace the existing register row "agentic-review results are not left as
report files in the worktree; D routes them through the subagent's answer into
task-issue comments" with the first row below, and add the rest to
`vergil-claude-plugin/docs/paad-policy.md` §7. Update paad-policy §2 ("Report
files") to match the first row.

| Deviation from paad | Reason |
|---|---|
| agentic-review writes its report and backlog where it always does, but `paad/code-reviews/` is ignored (managed `.gitignore`) rather than committed, and the implementer collects the files into task-issue comments and empties the directory after each round | The task issue is the record of each round (umbrella §3.3). Ignoring the directory keeps the worktree clean for the next round and for finalize without changing paad's behaviour; emptying it keeps earlier rounds out of the next reviewer's context. |
| paad's code-review backlog does not persist in mode 2; out-of-scope bugs go to intake and are de-duplicated against open issues by a `Finding:` line | Intake is the system of record for follow-on work (umbrella §2.4). A second backlog would drift from it. |
| Out-of-scope additions are removed and filed as their own task by the implementer, not left to a per-PR keep/split/revert decision by the user | An agent implements only its issue (umbrella §3.3). The removal is reversible: the task holds enough to rebuild it. |
| Out-of-scope bugs are filed in intake by the implementer, not presented to the user batched by tier | The run is unattended; intake is where the human triages them. |
| Commits after the final round (Suggestion fixes, removals) are not re-reviewed; they are listed as unreviewed in the handoff | A Suggestion never causes a round on its own (umbrella §3.3). Listing them keeps the handoff honest. |
| The Spec Compliance intent source is a requirements file named in the reviewer prompt, not a spec-file argument | The skill's argument grammar takes only a base and a path; a file argument is read as a path filter. Raise with the paad authors. |

### 4.19 Validation

- **Feasibility spike:** done at planning time (section 3).
- **Re-spike, first:** a `validation` task runs the full gate procedure by
  hand, with the exact prompt template from doctrine §3, on a real
  **plan-driven** task branch, twice. Success: both runs pass every output
  check (4.6), Spec Compliance names the requirements file (including the plan
  task text) as its intent source, paad's files land only in that worktree's
  ignored `paad/` directories, and collecting and emptying them leaves the
  worktree clean. It runs before the skill change ships. A failure goes back to
  the human before the skill rewrite continues.
- **Dogfood, last:** once the plugin release containing the gate is installed
  (a human-gated release, then a deployment task), run `issue-implement` on a
  real task, ideally one that needs a fix round, and check: the comments follow
  4.9; the checks ran; intake issues and removal tasks were filed and mapped;
  the handoff lists everything not fixed; the worktree is clean at finalize.
  Pause and resume are checked if the run pauses; otherwise the validation
  records that they were not exercised.
- **The sweep** is exercised by this epic's own retrospective only for its
  pre-release tasks (4.15); its first full run is on the next epic implemented
  after the release, which the dogfood task names.

## 5. Goals, non-goals, anti-goals

**Goals.**

- Every `issue-implement` run ends with an agentic-review pass or an explained
  pause, and its rounds are fully recorded on the task issue.
- No problem found during implementation is lost: each is fixed, filed, or
  dropped by the human with a reason.
- The handoff shows the human exactly what was not fixed and why.

**Non-goals.**

- An exemption rule by change size or risk (umbrella §3.3: later, from real
  cost data).
- Tooling to parse comments, classify findings or run the sweep. The implementer
  and the retrospective agent do it from the fixed formats; tooling can come
  later if the records show a need.
- The automated merge-gate review agent (.github#90).
- Changing paad itself.

**Anti-goals** (inherited from umbrella §5): a review loop that runs forever or
passes without a well-formed report; findings silently dropped, or a handoff
that hides them; security details in any public record; an implementer that
treats the gate as optional.

## 6. Deliverables by repo

| Repo | Deliverable |
|---|---|
| `vergil-claude-plugin` | doctrine §3 (in C's `docs/paad-review-doctrine.md`); the `issue-implement` gate; the `epic-implement` and `epic-retrospective` changes; `docs/paad-policy.md` register rows; the skills index page |
| `vergil-tooling` | `paad/code-reviews/` in the managed `.gitignore` block, and a fleet sync; the `vrg-git` additions (#3722); `docs/site/docs/standards/github-issues.md` (via the documentation review) |
| `docs` | `management/workflow.md` and `management/tools-and-skills.md` (via the documentation review) |
| `vergil-project/.github` | this spec and plan |

## 7. Constraints

- Public repos: task-issue comments, commit messages, PR bodies and the relay
  ref (`report-ready` notes) are public.
- Agents use `vrg-git` and `vrg-gh` only; agents never run `vrg-submit-pr`.
- `issue-implement` must still finish without a human in the normal case
  (umbrella §7).
- The doctrine §3 work depends on child C creating
  `docs/paad-review-doctrine.md`.
- The security handling (4.11) depends on child A′
  (vergil-project/.github#573): today's tooling reads only one advisory per
  issue.
- Cost: about 6.5 minutes and 7 subagents per round on a 250-line diff (spike);
  up to three rounds per task before a pause.
- paad is unpinned and its output format can change in any release; the
  structural checks are the guard (paad-policy §1).
