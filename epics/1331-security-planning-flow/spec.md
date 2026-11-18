# Harden the security-planning flow — design

- **Epic:** vergil-project/.github#1331
- **Planning issue:** vergil-project/vergil-claude-plugin#828 (review history
  is in its comments)
- **Status:** draft, in design review

## 1. Intent

A `security` planning issue is planned in public. The design of its fix is
published in `.github` like any other epic's, while the finding itself lives
only in the private advisories that track the issue. This epic makes that
flow safe from end to end:

- every public artifact a security planning run produces stays in general
  terms, from its first comment on, with a human gate before any design is
  published;
- an advisory leaves a planning issue only for the tasks that fix it, and
  only from an issue it actually tracks;
- no path that closes a planning issue, by tool or by hand, closes its
  advisory before the fix exists.

Where a check can be mechanical, it is: the tools refuse, and the skill text
says what to do when they do. The skill text is the guide, not the guard.

## 2. Why

The private advisories that track the planning issue hold the findings.
In general terms: the skill text states its public-wording rule late and
narrowly, some of its close paths do not hand the advisory on, and the
hand-off itself relies on the agent getting each step right. Several of the
same weaknesses can be closed in the tools, where a refusal does not depend
on an agent reading the right paragraph.

## 3. Goals, non-goals, anti-goals

### Goals

1. `vrg-security-record retarget` moves an advisory only from an issue it
   currently tracks, and never silently stops tracking an open issue.
2. The scheduled security-records sync never auto-closes an advisory because
   a queue item it tracks was closed, or because a tracked issue was closed as
   not planned.
3. The planning hold fails closed everywhere it is used: `vrg-queue
   planning` and the docs-PR close (vergil-project/vergil-tooling#3996).
4. `epic-create` checks for a security item at step 0, states the
   public-wording rule there for every public artifact, and gates the first
   publication of anything about the design on the human.
5. `epic-create` hands every advisory on, or to the human, on every path that
   closes the planning issue, and its step 8 retarget uses the new tool
   contract.
6. The planning states, wherever `epic-create` and `epic-review` describe
   them, include the advisory hold.

### Non-goals

- A private-design flow, in which the spec and plan live outside public
  `.github`. The human gate (§6.4) can stop a run; designing what happens
  next is left to the human for that item.
- Changing how advisories are created, closed by hand
  (`vrg-security-record close`), or synced from their own state.
- Neutrality checking by machine. Whether a sentence describes a weakness is
  judgment; the gate and the rule cover it.
- Any change to the `epic-implement` or `issue-implement` skills, beyond what
  the tool refusals already make them hit.

### Anti-goals (the epic fails if any of these happen)

- A tool change that breaks an existing route: `vrg-intake-route`'s split,
  transfer and recreate each call retarget and must keep working.
- A guard that holds an advisory open forever with no report: every hold the
  sync makes is reported, with what a human does next.
- Any public artifact of this epic, including this spec, that describes a
  finding rather than a mechanism.

## 4. Current state (facts the design builds on)

### vergil-tooling

- `security_records.retarget(home, ghsa_id, targets, *, renamed=())` marks
  every target, rewrites the advisory's `Tracking issue:` lines to the target
  set, then unmarks every dropped issue. A re-run with the same arguments
  finishes an interrupted run. It takes the advisory ID as given.
- `vrg-security-record retarget <ghsa> --to <ref> [--to …]` is its CLI.
- `vrg-intake-route` calls `retarget` internally for a split, a transfer and a
  recreate, always with the routed item as the source, and prints recovery
  commands (`_retarget_commands`) when one fails.
- `security_records.pending_syncs` sets a paired advisory `closed` when every
  issue it tracks is closed, and `draft` while any is open. It does not look
  at why or where an issue closed.
- `queue_view.tracking_advisories(ref, body)` returns the advisories whose
  `Tracking issue:` lines name `ref`. The planning hold uses the same rule.
  It is built from `list_advisories`; a GHSA the listing does not return is
  treated as not tracking (vergil-project/vergil-tooling#3996, in the deferred
  queue).
- The docs PR is the normal way a planning issue closes: `planning_close.find`
  (behind `vrg-submit-pr` and report-ready's `planning_close` preview) calls
  `tracking_advisories` directly, apart from the planning view's own hold,
  and writes the `Planning issue: Closes <M>` line only when no advisory
  tracks the planning issue.
- `intake_route._security_ghsas` treats an item as a security item only with
  both the label and an advisory line, and warns on a broken pair.

### vergil-claude-plugin

- `skills/epic-create/SKILL.md` names the `security` label first in step 8
  (task filing), with its retarget and public-wording bullets there. Steps
  0–7 publish progress comments, the epic, its bookend tasks, the spec, the
  plan and round comments before that.
- Step 1's two early closes ("Filed as task", "not planned") close the
  planning issue directly.
- The planning states table and the latest-note description do not mention
  the hold; step 10 describes it in its own bullet after the fallback.
- `skills/epic-review/SKILL.md` already says a held item is never
  `ready-to-close` and that a security item's sizing reason stays neutral.

## 5. Approaches considered

**Where the retarget source check lives.**

- **A. In the tool (chosen).** `retarget` takes the source issue and refuses
  unless the advisory tracks it. The skill cannot get it wrong by reading the
  wrong IDs. Costs a vergil-tooling release before the skill can rely on it.
- **B. In the skill.** The skill takes GHSA IDs from the advisories that
  track the issue instead of the body lines. Cheaper, but only as safe as the
  agent's reading of the step.

**How much of a security item's design is public.**

- **A. Public, in general terms.** The spec and plan describe mechanisms;
  specifics go into the advisories.
- **B. Private design.** The spec and plan live only in the advisory. Safer
  for a severe finding, but the design-review rounds and docs PR would need a
  redesign.
- **C. A, with a human gate (chosen).** A by default; before the first design
  is published, the human confirms the item is safe to describe in general
  terms. If not, the run stops (§6.4).

**Early closes.** Skill text alone would cover the two step 1 paths but not a
hand close or an intake-route drop. A guard in the sync (§6.2) covers every
path at once, with the skill text as the first line.

## 6. Design

### 6.1 `retarget` source check (vergil-tooling)

- `vrg-security-record retarget <ghsa> --from <ref> --to <ref> [--to …]`.
  `--from` is **required** and names one issue the advisory currently tracks
  (a `Tracking issue:` line names it). Otherwise retarget refuses before any
  write, naming the issues the advisory does track.
- Retarget refuses, before any write, to drop a tracked issue that is still
  open unless the human passes `--drop <ref>` for it. Re-targeting to a
  growing set of fix tasks therefore names every task still in flight;
  dropping the planning issue (the `--from` of an epic-create hand-off) is
  allowed, since it is the source being handed on.
- A re-run of an interrupted retarget with identical arguments still
  finishes it. Retarget rewrites the tracking lines before it unmarks the
  dropped issues, so the `--from` check accepts an issue when either:
  - a current `Tracking issue:` line names it; or
  - its body still carries a `Security advisory:` line for this advisory
    (it is still marked, as `security_issues` lists it) and every `--to` is
    already a tracking line: the state an interrupted run with the same
    targets leaves.

  The open-issue drop check (above) applies to the same dropped set in
  both cases.
- The library function takes the source as a parameter, and every internal
  caller (`vrg-intake-route` split, transfer, recreate) passes the routed
  item as the source. Each caller's targets are the advisory's current
  tracking set, minus the routed item, plus the new issue or issues, so a
  route never drops another tracked issue and never meets the drop check.
- Every retarget command the tools print carries `--from`: the recovery
  commands (`_retarget_commands`) and the message that tells the human to
  retarget an item naming several advisories by hand.

### 6.2 Sync close guard (vergil-tooling)

- `pending_syncs` does not move an advisory to `closed` while any issue it
  tracks is a queue item, or was closed with the reason `not planned`. It
  stays `draft`. A **queue item** is an issue whose native parent is a live
  intake, planning or deferred epic, or a queue archive epic (labelled
  `archive` plus `intake`, `planning` or `deferred`). The tests cover a
  closed queue item that has already been drained into its archive.
- Only an advisory the guard actually kept from `closed` (every issue it
  tracks is closed, and the guard fired) is reported, on its own line kind
  (`held:`), by GHSA ID and issue ref only, with the next step: retarget it
  to a fix task, or close it with `vrg-security-record close` and a reason.
  Held advisories are counted apart from broken pairs, and a hold does not
  fail the sync: it waits on a human step and is not a fault.
- A failed parent or state lookup is an error for that advisory, reported,
  never a close.

### 6.3 Planning hold fails closed (vergil-tooling)

Delivered by vergil-project/vergil-tooling#3996, routed out of the deferred
queue into this epic, with its scope widened to every user of the hold:

- **One hold rule,** used by the planning view and by the docs-PR close
  (`planning_close.find`). The view derives its hold from
  `tracking_advisories`, or both call one shared function; there is no
  second copy.
- An advisory the planning issue names that cannot be read (the listing
  does not return it) counts as tracking. The planning view keeps the item
  held, with a note saying the advisory cannot be read, and the
  `planning_close` skip reason says the same. Absence is never "not
  tracking".
- The next step the note names: re-run with a token that can read the
  repository's advisories, or have the human check the advisory.

### 6.4 Step 0 security check (vergil-claude-plugin, epic-create)

- At step 0, if the planning issue carries `security` or has a
  `Security advisory:` line, the run is a security run. A label without a
  line, or a line without the label, is a broken pair: stop and ask the human.
- The public-wording rule is stated here and covers every public artifact the
  run produces: progress and round comments, the epic title and body, task
  titles and bodies, the slug and branch names, commit messages, the spec, the
  plan, PR bodies, report-ready notes and the retrospective. They describe
  mechanisms and refer to findings by issue ref and GHSA ID alone. Specifics
  the design needs go into the advisory, with `vrg-security-record note`.
- **Human gate.** At the end of step 1, once the design is approved and
  before anything about it is published (the brainstorm-approved comment
  and the step 2 epic included), the agent asks the human whether the item
  is safe to describe in general terms. On yes, the run continues, and the
  brainstorm-approved comment of a security run says only that the design
  is approved and the epic is next. On no, the agent posts a neutral
  progress comment that the run is paused for a design-placement decision
  and stops; nothing design-shaped is published.
- The round reviewers already keep security findings out of public text
  (doctrine §2.7); the rule adds nothing to the reviewer prompt.

### 6.5 Hand-off on every close path (vergil-claude-plugin, epic-create)

- **Step 1, single task.** File the task, retarget each advisory to it with
  `--from <M>`, then close the planning issue.
- **Step 1, not planned.** Do not close. Stop and hand each advisory to the
  human, who retargets or closes it; the close comment, when it comes, is
  neutral.
- **Step 8.** One retarget per advisory, once, after the last fix task is
  filed, with `--from <M>` and every fix task. A fix task added later is
  added by a retarget `--from` one of the advisory's current tasks with the
  complete set as `--to`. GHSA IDs come from the advisories that track the
  issue (the queue's held note), not from the body lines. On a failure, fix
  the cause and re-run the identical command.

### 6.6 The hold in the planning states (vergil-claude-plugin)

- `epic-create`: the states table, the latest-note description and step 10's
  fallback name the hold and its `held:` note.
- `epic-review`: check its planning mode against the same text, and close any
  gap.

### 6.7 Rollout

The skill work ships in two parts:

- **First, with no tooling dependency:** §6.4, §6.6, and §6.5's
  not-planned branch. They use no new tool behavior, so they land at once,
  in parallel with the tool changes.
- **After the release:** the rest of §6.5 (the single-task hand-off and the
  step 8 retarget), whose commands name `--from`.

The tool changes (§6.1–6.3) land, then a vergil-tooling release as a
**deployment** task, and the second skill part is blocked by that
deployment.

### 6.8 Docs

The documentation-review task (vergil-project/vergil-tooling#4697) sweeps the
security-records guide and the CLI reference in vergil-tooling, the skills
index in vergil-claude-plugin, and the intake page in `docs`.

## 7. Failure model

- Every tool refusal names what it refused and the command that would
  succeed; nothing is written before a refusal.
- A sync that cannot decide an advisory's guard reports it and leaves it as
  it is.

## 8. Constraints

- Public text stays in general terms (§6.4), including this epic's.
- Agents never release; the vergil-tooling release is a human-gated
  precondition of the deployment task.

## 9. Related work

- vergil-project/vergil-tooling#3996 — delivered here (§6.3).
- vergil-project/.github#593 waits on this epic.
- Folded records: vergil-project/vergil-claude-plugin#830 and #937.

## 10. Left to the builder

- The exact refusal and report wording.
- How the sync finds a tracked issue's parent and close reason (batched or
  per issue), within the API limits it already respects.
