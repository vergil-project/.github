# Iterating design review in `epic-create` — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `epic-create` runs mode 1 of the paad review doctrine: human-gated
rounds of spec pushback, plan pushback and alignment, each round a fresh
review-only subagent, recorded on the planning issue, with the rules in one
shared doctrine document.

**Architecture:** the change is skill and doctrine text in
vergil-claude-plugin. The main session carries out the loop by following that
text. A validation spike proves the exact reviewer prompt first. A deployment
and a dogfood run on a real `epic-create` close the epic out.

**Tech Stack:** Markdown skills and docs (vergil-claude-plugin); `paad:pushback`
and `paad:alignment` 2.0.2; `vrg-security-record` and `vrg-trusted-authors`
(vergil-tooling); GitHub issues.

**Spec:** `epics/553-iterating-design-review/spec.md`
(vergil-project/.github#553)

**Planning issue:** vergil-project/.github#469 (review history is in its
comments)

**Status:** draft, in design review

**Shape:** an epic-level plan. Each numbered task is **one issue and one PR in
the repo named in its heading**, with paths relative to that repo's root.
`validation` and `deployment` tasks are operational: no code PR, closed on
`Outcome: SUCCESS`. No task changes code, so no task has a red/green test
cycle; each checks its text with `vrg-validate` and named content checks.

## Global Constraints

- **Binding rules** are umbrella spec §3.1 and §3.2
  (`epics/500-paad-first-engineering/spec.md`); they are not reopened (spec
  section 2).
- **One severity table**, in doctrine §1 only; every other document links to
  it (spec 3.6).
- **Only new findings argue for another round** (spec 3.6). The
  recommendation is advice; the human decides every round.
- **paad version recorded in the doctrine:** 2.0.2. A different version is a
  warning in the round comment, not a failure (spec 3.5).
- **Trusted logins are configuration.** The doctrine and the skill name only
  the command `vrg-trusted-authors`, never a login.
- **Public-repo rule:** no security detail in any comment, commit message,
  spec, plan, PR text or `report-ready` notes; counts and GHSA IDs only (spec
  3.8).
- **External prerequisites:**
  - A′'s release and deploy, vergil-project/vergil-tooling#3733: done
    (closed);
  - D's tooling deployment, vergil-project/vergil-tooling#3742 (which ships
    `vrg-trusted-authors`, vergil-project/vergil-tooling#3741), before Task 3
    merges.
- **Tooling:** validation `vrg-container-run -- vrg-validate`; commits
  `vrg-commit`; handoff `vrg-pr-workflow report-ready`. Releases are
  human-gated; the agent never cuts one.

**Order:** Task 1 has no blockers. Task 2 needs Task 1. Task 3 needs Task 2
and vergil-project/vergil-tooling#3742. Task 4 needs Task 3. Task 5 needs Task 4.
Then the bookends: the documentation review
(vergil-project/vergil-tooling#3719), then the retrospective
(vergil-project/.github#555). D's doctrine task
(vergil-project/vergil-claude-plugin#759) needs Task 2: when Task 2 is filed,
add a `Blocked-by:` line for it to #759. At the same time, add
vergil-tooling's `docs/site/docs/guides/security-records.md` "Severity"
table (to be replaced by a link to doctrine §1.3) and its "Recording findings
from a review" recipe (to match spec 3.8's handling of rejected findings) to
the documentation review's (#3719) list of known places.

## Review Focus

1. **A reviewer that changes the worktree** (an edit, a staged change, an
   untracked file, a commit, a moved HEAD). Expected: the output checks catch
   it, the revert restores the recorded branch and SHA with allowed `vrg-git`
   commands only, and the tree is clean before the re-run. Task 1 drills the revert; Task 2 writes the confirmed
   commands into doctrine §2.4.
2. **A security-shaped weakness reviewed by `alignment`,** which has no
   security routing of its own. Expected: the reviewer routes the finding to
   `paad/security/` as its prompt directs; screening catches anything it
   misses; nothing reaches a public record. Task 1's seeded alignment run; Task 5 checks the real records.
3. **A session started in a different repo from the docs worktree.**
   Expected: every file paad writes is under the docs worktree's
   `paad/security/`, and nothing new appears in the starting directory's
   `paad/`. Task 1's seeded runs.
4. **A session resumed mid-stage, or two rounds on the same day.** Expected:
   the stage and round count are rebuilt from the `Design review: <stage>
   round <n>` prefix, and each round's security files are in their own
   scratch copy, found from the planning issue and round ID. Task 3's text;
   Task 5 records whether it was exercised.
5. **A comment on the planning issue from someone untrusted** that claims a
   decision. Expected: classification ignores it. Task 2's text in doctrine
   §1.2; Task 5 checks each round comment's "trusted authors checked" line.

---

### Task 1: Validation spike for the reviewer prompt (.github, validation)

No blockers. Validates spec 3.4, 3.5 and 3.8 before any skill text ships
(spec 3.13). **Precondition (human-attested):** paad 2.0.2 or later is
installed where the runs happen (B Task 4, vergil-project/vergil-tooling#3668,
verified the agent VMs).

**How it runs:** the runs need throwaway worktrees, and `issue-validate`
creates none, so the human directs the runs in a session of their own.
`issue-validate` then checks the recorded results against the acceptance
below and records the outcome. Below, `<T>` is this task's issue number.

**Targets:**

- **Plain runs:** a detached worktree of `vergil-project/.github` at
  `origin/develop`, at `.worktrees/issue-<T>-review-spike/`, reviewing the
  published `epics/517-security-records/spec.md` and `plan.md`. The session
  starts at the `.github` project root.
- **Seeded runs:** a second worktree, `.worktrees/issue-<T>-seeded-spike/`,
  on the local branch `feature/<T>-seeded-spike`, created with
  `vrg-git worktree add -b feature/<T>-seeded-spike <path> origin/develop`
  so that it tracks `origin/develop` and `vrg-git` lets it be deleted later
  without pushing. It holds copies of the 517
  spec and plan, each with one security-shaped weakness added, committed with
  `vrg-commit`. The branch is never pushed, and is deleted with its worktree
  at the end. These runs start from a session at the root of a different
  repo (for example vergil-tooling).

**Procedure:**

- [ ] Draft the reviewer prompt from spec 3.4, as a template with
  placeholders for the skill, the absolute document paths and the docs
  worktree. This task's outcome comment records the final text, which Task 2
  copies into the doctrine verbatim.
- [ ] Before each run, list the starting directory's `paad/`, record the
  worktree's branch and HEAD SHA, confirm `vrg-git status --porcelain` is
  empty and the worktree's `paad/security/` holds only `.gitignore`, and
  fetch the member repos the prompt names (spec 3.4, 3.5).
- [ ] Plain runs, each twice, each with a fresh `general-purpose` subagent:
  spec pushback (the spec), plan pushback (the plan, with the spec as
  context), alignment (both).
- [ ] Seeded runs, once each: pushback on the seeded spec; alignment on the
  seeded spec and plan.
- [ ] After each run, apply every spec 3.5 check and record the result per
  check. Then screen every ordinary finding and every discard line against
  paad's shared security definition (spec 3.8), and record separately how many security findings
  the reviewer routed and how many screening caught.
- [ ] Security files: move each run's files to the scratch copy at
  `${XDG_STATE_HOME:-$HOME/.local/state}/vergil/design-review/vergil-project-.github-<T>/<run-id>/`
  (mode 700). After each move, confirm the resolved path is absolute and
  outside every repo, and that the worktree's `vrg-git status --porcelain`
  is still empty. Record each seeded finding with plain `vrg-security-record
  create` and a neutral `PROBE:` summary, then close its intake issue as "not
  planned". For a security finding from a plain run, tell the human the count
  only; the human decides whether to record it. Delete each scratch copy once
  its findings are recorded or the human has decided.
- [ ] **Revert drill:** in the seeded-run worktree (on a branch, like a real
  docs worktree), record the branch and HEAD SHA, then make one commit, one
  unstaged tracked edit to a file that commit also changed, one staged
  tracked edit, one staged new file and one untracked file. Restore the
  recorded state with spec 3.5's commands only, in its order:
  `vrg-git checkout HEAD -- <file>`, `vrg-git rm --cached <path>` then
  `rm <path>`, `rm <path>`, and last
  `vrg-git switch -C feature/<T>-seeded-spike <sha>`. Then repeat the drill
  in the detached plain-run worktree with a commit and one tracked edit,
  restoring with `vrg-git switch --detach <sha>`. Confirm the HEAD SHA matches the record and
  `vrg-git status --porcelain` is empty. (`vrg-git` on develop allows
  `switch`, `rm` and `checkout`, and denies only `checkout -- .` and
  `checkout -- *`; this drill confirms it in practice.)
- [ ] Remove both worktrees and delete the seeded branch.

**Acceptance:** all eight runs pass every check; every paad file lands in
that run's worktree; in each seeded run, the reviewer itself routed the
seeded finding to `paad/security/` (a finding caught only by screening is
recorded as its own result and fails that run); no security detail reached a
public record; every scratch copy resolved to an absolute path outside
every repo, left the worktree clean, and is deleted; the revert drill restores
the recorded state. The outcome comment carries the final prompt text, the
revert commands as run, and, per run, the time taken and the counts of
findings, discards and security findings. **On failure,** record it and stop:
the result goes to the human, who decides whether the fallback (spec 3.11) is
built, before Task 2 starts.

**Contingent task (filed only if the human chooses the fallback):**
`paad/config/pushback.md` and `paad/config/alignment.md` in
`vergil-project/.github`, per spec 3.11, applying only to a prompt that
carries a review-only marker, and each naming its paad version. Its `Config:`
line becomes an expected value in doctrine §2.4. If it is filed:

- Task 2 gains a `Blocked-by:` line for it;
- after it merges, Task 1 stays open and re-runs every run with the marker
  added to the prompt template, then records its outcome as above;
- Task 2 copies the re-run's prompt, marker included.

---

### Task 2: The review doctrine and the paad policy (vergil-claude-plugin, docs)

Blocked by Task 1.

**Files:**

- Create: `docs/paad-review-doctrine.md`.
- Modify: `docs/paad-policy.md`: a new section 8 and four rows in §7.

**Interfaces:**

- Produces the section numbers that Task 3 and D's #759 link to:
  - §1 Common core: §1.1 Rules, §1.2 Classification, §1.3 Severity, §1.4
    Signals;
  - §2 Mode 1: §2.1 Stages, §2.2 One round, §2.3 The reviewer prompt, §2.4
    Output checks and broken rounds, §2.5 The recommendation, §2.6 The round
    comment, §2.7 Security findings, §2.8 No review history, §2.9 The
    planning issue, §2.10 Fallback;
  - §3 Mode 2: a stub.

- [ ] **Doctrine §1,** written as instructions to an orchestrator:
  - §1.1 from umbrella §3.1;
  - §1.2 from spec 3.6 "Classification": the three sources, the
    `vrg-trusted-authors` rule (only comments whose author it lists count;
    strip a leading `app/` before comparing), and the definitions of known,
    repair and new;
  - §1.3 the severity table from spec 3.6, exactly as there, with the date
    it was checked and the note that paad is unpinned; the definition of
    structural;
  - §1.4 umbrella §3.1's stop and continue signals, with a note that mode 1
    narrows the continue signal and a link to §2.5.
- [ ] **Doctrine §2,** from spec 3.2–3.11 in that order:
  - §2.3 carries **Task 1's prompt template, verbatim**;
  - §2.4 names the revert commands as Task 1's outcome records them, and the
    paad version Task 1's runs reported. If that is not 2.0.2, re-check every
    severity column against that version's skills before writing §1.3;
  - §2.6 gives one example round comment, with an invented planning issue
    and no real finding.
- [ ] **Doctrine §3:** one paragraph saying child D (epic
  vergil-project/.github#568) writes mode 2 here.
- [ ] **paad-policy:** add "## 8. Review doctrine", two or three sentences
  linking `paad-review-doctrine.md` for the rules of repeated review. Add spec
  3.12's four rows to the §7 register, after the existing "Design-review
  report files" row. Do not change §1, §2 or §4: D's #759 rewrites those.
- [ ] **Content checks:** `grep -c` finds the severity table's header row in
  the doctrine once and in no other file of the repo; the doctrine names no
  GitHub login; every link from the doctrine resolves.
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit
  (`vrg-commit --type docs --scope paad`) and run report-ready.

---

### Task 3: Mode 1 in `epic-create` (vergil-claude-plugin, implementation)

Blocked by Task 2 and vergil-project/vergil-tooling#3742. The new skill
reaches every agent VM with the first plugin release after this task merges,
so it must not merge before the commands it uses are deployed: without
`vrg-trusted-authors`, classification and resume cannot run. (A′'s
`--issue` support, vergil-project/vergil-tooling#3733, is already deployed.)
Link both with `Blocked-by:`.

**Files:**

- Modify: `skills/epic-create/SKILL.md`, including the frontmatter
  `description`.
- Modify: `docs/site/docs/skills/index.md`, the `epic-create` table row and
  section, and the execution-doctrine sentence "(brainstorm → pushback →
  alignment)".
- Modify: `CLAUDE.md`, the three-skill lifecycle bullet for `epic-create` and
  the execution-doctrine sentence, only where they still exist when this task
  runs (E's epic #558 Task 5b trims that file).

- [ ] **"The four-stage interaction doctrine"** becomes "The interaction
  doctrine", with spec 3.2's three-row table. Keep the human-judgment
  principle for brainstorming. For design review, say that findings are
  walked one at a time, except that findings mapped to low, and known
  findings with their earlier decision cited, are presented as one batch at
  the end, each with its own decision recorded (spec 3.3 step 4).
- [ ] **New section "Design review rounds"** before "Workflow": the seven
  steps of spec 3.3, one or two lines each, each linking its doctrine §2
  subsection. It says the reviewer runs in the background and the human
  decides every round.
- [ ] **Workflow:** step 3 adds the spec header rule (spec 3.9, linking
  doctrine §2.8), with its reason; step 4 becomes "Stage 1: spec pushback
  rounds"; step 7 becomes "Stage 2: plan pushback rounds, then Stage 3:
  alignment rounds"; steps 3, 5 and 6 each commit their output as an
  ordinary commit, not a round commit (spec 3.2); step 9's advisory handoff
  is unchanged.
- [ ] **Progress comments:** the stage-boundary list names spec pushback
  done, plan pushback done and alignment done; round comments carry no
  marker lines; when the human stops a stage, a progress comment naming the
  next stage follows the round comment.
- [ ] **Resuming** (the entry-points table's planning row): reading only
  comments whose author passes `vrg-trusted-authors` (doctrine §1.2), rebuild
  the stage and round count from the latest `Design review: <stage> round <n>`
  first line, and find any kept scratch copy at the doctrine §2.7 path. If
  the docs worktree is dirty, show the human the diff; the human commits it
  as the interrupted round's commit (with its round comment) or discards it,
  before any reviewer is dispatched (doctrine §2.4).
- [ ] **Planning issue** (spec 3.10): state that every epic has one, that
  links run both ways (the epic's `Planning:` line, the spec header's
  `Planning issue` line, the step 10 closing comment), and that the planning
  issue is never moved to the epic.
- [ ] **Skills index and `CLAUDE.md`:** describe the pipeline as brainstorm →
  spec pushback rounds → plan → plan pushback rounds → alignment rounds.
- [ ] **Content checks:** `SKILL.md` no longer says to run `paad:pushback` or
  `paad:alignment` once in the main session; it does not restate the severity
  table; every doctrine link resolves.
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit
  (`vrg-commit --type feat --scope epic-create`) and run report-ready.

---

### Task 4: Deploy the plugin (vergil-claude-plugin, deployment)

Blocked by Task 3. **Preconditions (human-attested):**

- a vergil-claude-plugin release containing Tasks 2 and 3 is published;
- the human has run `vrg-vm update --all` on the host since that release.

**Procedure:**

- [ ] In the agent VM, confirm the installed `epic-create` skill contains
  "Design review rounds", and the installed plugin's
  `docs/paad-review-doctrine.md` exists.
- [ ] Check every command and flag that doctrine §1.2 and §2.7 name against
  the installed CLIs: `vrg-security-record create --help` lists `--issue`,
  `vrg-security-record show --help` exists, and `vrg-trusted-authors` prints
  a human and an agent login. If any is missing, record it and stop; the
  doctrine is fixed before the skill is used.
- [ ] Record `Outcome: SUCCESS` with the released version.

---

### Task 5: Dogfood mode 1 (.github, validation)

Blocked by Task 4.

- [ ] `issue-validate` cannot run `epic-create`. The human names the next
  real planning issue, and that run's `epic-create` runs mode 1 as written.
  This task then only inspects the records; the outcome names that planning
  issue.
- [ ] Check, on that planning issue: one round comment per round, each in
  doctrine §2.6's form (first line, checkpoint with its "trusted authors
  checked" line, findings, discards, security); a progress comment after each stopped stage; every finding not
  addressed ended accepted, rejected or dropped with a reason or an intake
  ref; no security detail in any public record.
- [ ] Check the spec and plan headers carry the `Planning issue` and `Status`
  fields of doctrine §2.8, and that neither document carries review history.
- [ ] Record whether a broken round, a resumed session, a security finding,
  or a classification against an untrusted comment happened (Review Focus 1,
  2, 4 and 5), and for each that did not, say it was not exercised.
- [ ] Record what the loop actually did: rounds per stage, findings by
  mapped severity and class, each recommendation against the human's
  decision, and anything that surprised.

---

## Self-Review

- **Spec coverage:**
  - 3.1 → Task 2;
  - 3.2, 3.3, 3.9, 3.10 → Task 2 (doctrine) and Task 3 (skill);
  - 3.4, 3.5, 3.8 → Task 1 (proved), Task 2 (doctrine);
  - 3.6, 3.7 → Task 2;
  - 3.11 → the contingent task after Task 1;
  - 3.12 → Task 2;
  - 3.13 → Tasks 1 and 5;
  - section 5 deliverables → Tasks 2 and 3; vergil-tooling's
    `github-issues.md` and `security-records.md` "Severity" link, and the
    `docs` repo's `management/workflow.md` and `management/tools-and-skills.md`,
    through the documentation review (vergil-project/vergil-tooling#3719);
  - section 6 prerequisites → the external prerequisites above.
- **Name consistency:** doctrine section numbers (§1.1–1.4, §2.1–2.10, §3)
  are defined in Task 2's Interfaces and used the same way in Tasks 3, 4
  and 5; "Design review rounds" is used the same way in Tasks 3 and 4.
- **Proportion:** most tasks are text whose content the spec fixes; the plan
  says where each part goes rather than restating it.
