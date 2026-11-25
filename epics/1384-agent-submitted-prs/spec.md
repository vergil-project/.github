# Agents submit and finalize their own PRs into develop — design

- **Epic:** vergil-project/.github#1384
- **Planning issue:** vergil-project/.github#1383 (review history is in its
  comments)
- **Status:** approved
- **Repos:** vergil-tooling, vergil-claude-plugin, vergil-actions, .github,
  and every member repo (the audit job's callers)
- **Related:** epic vergil-project/.github#889 (authenticated human rulings),
  epic vergil-project/.github#1218 (parallel PR merging: auto-merge and the
  provenance check), epic vergil-project/.github#1467 (trusted content only:
  the trust layer every comment read goes through), epic
  vergil-project/.github#1494 (the operations queue and its security-signal
  jobs), epic vergil-project/.github#1440 (release records), planning items vergil-project/.github#1088
  (fixture-scoped agent rights), vergil-project/vergil-actions#1185
  (environment-scoped automation credentials, absorbed by §4.2),
  vergil-project/.github#467 (release holds)
  and vergil-project/.github#892 (authenticate every ruling); the
  permission-model spec,
  `vergil-tooling/docs/specs/2026-05-29-agent-permission-model-design.md`

> **Amended 2026-11-24.** By the human's ruling N0.12 on the epic, the
> protected core and its protected-change approval are removed: there is
> no protected-path set, no approval and no registry. Merge review on every
> PR and the human-approved merge-review exemption stay. By the same
> ruling's addition, the human's two submit-time skips, `--human-only` and
> the merge-review override (`--override-merge-review`), are removed too:
> merge-review coverage is a record or an approved exemption only. §4.2's
> account of `secrets: inherit` is also corrected (N0.6, N0.7, N0.16). The
> sections below are amended in place; the plan's
> [Evolution during execution](plan.md#evolution-during-execution) records
> the ruling and the tasks it changes.

## 1. Intent

An agent must be able to take a task from implementation to merged code in
`develop` with no human step.

Vergil's premise is that the up-front design (spec and plan), together with
mechanical gates (CI/CD, `vrg-validate`, merge review), establishes that a
change is correct, without a human reading the code. The approval is in the
design at the start and in the gates at the end. Today one more step sits
between them: the agent stops at `vrg-pr-workflow report-ready`, and the
human runs `vrg-submit-pr` and `vrg-finalize-pr`. In practice the human now
approves every submission without changes. The handoff adds delay without
adding safety, and it is the main thing holding work back: an epic cannot
move to its next task until the human is at the keyboard.

This epic moves the boundary. The working agent (the user identity) submits
its own PRs, with auto-merge on, and finalizes them. The human is pulled in
only for:

- judgment calls and merge-review rulings, as today;
- a merge-review exemption, when the human wants one (§4.6);
- failure triage and recovery decisions.

Releases and promotion stay human-only. Coordinating releases across epics
running in parallel (holding a release until related changes land, without
two epics deadlocking each other) needs its own design, which is left to
later work (vergil-project/.github#467, and vergil-project/.github#1088
rethought on top of this epic).

This is one step toward the long-term goal of agents taking an epic from
idea to production, with the human handling only exceptions.

### 1.1 Why there is no protected core

Every right the agent gains here follows the same argument: the human was
approving without looking, so the approval protected nothing. Changes to the
CI wiring (workflow files and `vergil.toml`) are no exception. On a
`pull_request` event GitHub runs the workflow file as the PR itself has it,
so a PR that edits a workflow is checked by its own edited version, and an
honest agent under pressure may weaken a check instead of fixing what it
caught. Agents in this fleet have done that with coverage and test failures.

The approved design answered this with a protected core: a human approval on
every change to those files. Execution showed that this answer fails the
same argument. Such changes are worked out in the epic's spec and plan, and
merge review judges each one, so a human approval on every one would be the
rubber stamp this epic removes elsewhere. The protected core is therefore
dropped (§4.4). The trust bar for wiring changes is the same as for any
change: the up-front design, merge review on every PR, the records the tools
check at submit, in the watch and in the daily audit, and the human-gated
release.

## 2. Goals

1. The user agent runs `vrg-submit-pr` (non-release paths) and
   `vrg-finalize-pr` (non-release paths) itself, in its own environment.
2. `issue-implement` and `epic-implement` carry a task through report-ready,
   submit, `pr-watch` and finalize to a merge, and `epic-implement` moves on
   to the next runnable task without waiting for the human.
3. A PR merges only if merge review completed for the head it merges at,
   enforced by `vrg-submit-pr`.
4. A daily audit reports any agent-involved merged PR without a merge-review
   record or exemption covering it, and any merge by an unexpected account.
5. Agents can push workflow changes themselves (`workflows: write`), so a
   workflow-touching PR no longer needs the human's push.
6. The rollout ships in an order where no check precedes what passes it and
   no instruction precedes its permission, then proves the first live cycle
   on the fixture repos.

## 3. Non-goals and anti-goals

### Non-goals

- **A separate automation identity that submits on the agent's behalf.** It
  would keep the merge right away from the coding agent and run the
  gates outside its reach. It is set aside because the GitHub
  interaction layer is to be rethought from scratch on a self-hosted Forgejo
  next year. It is not built on GitHub.
- **Merging `vrg-pr-workflow` into `vrg-submit-pr`.** That is the expected
  end state, but the two tools stay separate in this epic.
- **Agent-run releases or promotion**, in any repo, fixtures included.
  `vrg-release`, `vrg-promote`, and the `--release` paths of `vrg-submit-pr`
  and `vrg-finalize-pr` keep their human-only gates.
- **Cloud-side submission.** Cloud sessions keep the relay handoff of the
  cloud PR handoff design (`vergil-tooling/docs/specs/2026-06-24-cloud-pr-handoff-design.md`);
  this epic covers the user agent in its VM.
- **Lifecycle and dependency-update PRs** (`vrg-update-deps`). They are moving
  to the automation identity under separate work
  (vergil-project/.github#1128, #1248).
- **The exception registry** (§7).

### Anti-goals

- **Defending against a deliberately hostile agent.** An agent inside its own
  VM can edit its copy of the tooling. This design aims for about 95% coverage
  of honest mistakes, backed by layered auditing (§4.8), not for adversarial
  resistance. That seam is named, not hidden. It includes records the agent
  login itself posts: a deliberately falsified merge-review closing comment
  would pass the checks in §4.6 and §4.8. The human's exemptions are not
  in this seam, since an agent cannot post as the human. The seam also
  includes a deliberate weakening of the gate logic itself, in a workflow
  file, `vergil.toml`, vergil-tooling or the plugin, since no path is held
  back for the human (§4.4). An honest agent's change to the gate logic,
  including a weakening made by mistake under pressure (§1.1), is not part
  of the seam: it is judged by merge review on every PR and held back from
  the fleet by the human-gated release, with no approval on it.
- **Bringing human review back by another route.** If the design makes the
  human approve ordinary PRs, it has failed even if it works. No class of
  change needs the human's approval to merge; only rulings, exemptions and
  failures reach the human.
- **A record that rebases void.** A merge-review record or exemption tied to
  a commit would be voided by every rebase and CI fix, and the same change
  would be reviewed or exempted repeatedly. It is tied to the change
  fingerprint instead (§4.6).
- **A silent pass.** A failure to read the merge-review record or the
  exemption refuses the submit. It never lets the PR
  through.

## 4. Design

### 4.1 Where the boundary is encoded today

Paths are in vergil-tooling's `src/vergil_tooling/` unless stated. Each
location is named by its symbol, since line numbers drift.

- **`bin/vrg_submit_pr.py`, `main`'s identity gate.** Today `is_agent()`
  refuses the whole
  command. Change: refuse AUDIT; refuse agents on the `--release` path, in
  the CLI emergency mode and on the relay path, so agents stay on the path
  report-ready drives (the relay serves cloud sessions, a non-goal); allow
  USER otherwise.
- **`bin/vrg_finalize_pr.py`.** No identity check today; `--release` chains
  `vrg-release`, which refuses agents. Its `--release` gate runs before both
  `--cleanup-only` and the watch. Change: at that gate, refuse `--release`
  for agents up front, before any PR is watched or cleaned up.
- **`bin/vrg_release.py`, `bin/vrg_promote.py`, `lib/promote.py`.**
  Refuse agents, the release's `--resume` and `--supersede` paths included.
  Unchanged.
- **`lib/pr_provenance.py`, `_FORBIDDEN`.** USER is forbidden `created`, `merged`,
  `auto_merge_enabled`, `edited` and more. Roles come from `classify`, which
  judges each identity through the trust layer of epic
  vergil-project/.github#1467 (`lib/trust.py`), with an `UNTRUSTED` role for
  everyone else. Change: §4.3.
- **`bin/vrg_gh.py`, `_DENIED_AGENT`.** Denies agents `pr create`,
  `pr edit` and `pr merge`. Change: keep the denials, point the message at
  `vrg-submit-pr` (§4.9).
- **Plugin hooks** `block-agent-merge.sh`, `block-raw-gh-pr-create.sh`,
  `block-github-contents-api.sh`. Block raw merges, approvals and PR creation;
  the text says the human submits. Change: keep the blocks, change the text
  (§4.9).
- **Plugin skills** `issue-implement`, `pr-watch`, `merge-review`,
  `epic-implement`. Stop at report-ready and tell the human to rerun
  `vrg-finalize-pr`. Change: §4.10.
- **The "PR handoff" sections of `.github/CLAUDE.md` and
  `data/claude_md_consumer.md`.**
  "Agents never run `vrg-submit-pr`." Change: §4.10.
- **`lib/github_config.py`, above the "Branch protection" ruleset.** A
  comment says agents cannot turn
  auto-merge on. Change: update the comment.
- **The user GitHub App.** `pull_requests: read`, `workflows: none`. Change:
  `pull_requests: write`, `workflows: write` (§4.2).

The "Branch protection" ruleset covers both `main` and `develop`, requires no
approving reviews and has no bypass actors (`lib/github_config.py`).
For `develop` that is not changed.

**`main` is closed to agents server-side.** A GitHub App's permissions apply
to a whole repository, never to one branch, so once the user App holds
`pull_requests: write` its rights reach `main` as well as `develop`. The
tooling refuses any base other than `develop`, but that is a client-side
gate. So the desired GitHub config (`lib/github_config.py`, kept converged by
the daily config job) gains a ruleset for `main` that restricts updates to
listed bypass actors: the human (the admin role), and the org automation App
only if the release flow merges as it. A merge into `main` by any other
identity, the user agent included, is then refused by GitHub. This is in place
before the user App gains `pull_requests: write` (§5).

### 4.2 The user GitHub App's permissions

The user App gains `pull_requests: write` (to create PRs and turn auto-merge
on) and `workflows: write` (to push changes under `.github/workflows/`). It
keeps `contents: write`. These are App-level settings changed by the human in
the App's configuration, so the change is a deployment step in the plan, not
code.

With `pull_requests: write`, the hard gate that stopped an agent from creating
or merging a PR server-side is gone. What remains is the tooling gate (submit
is the sanctioned path), the soft gates (§4.9) and detection (§4.3, §4.8).
That is the accepted seam named in §3.

**Secrets only on protected branches, before `workflows: write`.** Every
credential a workflow reads as a secret that can write or act (at least the
org automation App's `APP_CLIENT_ID` and `APP_PRIVATE_KEY`) lives only in a
GitHub environment whose deployment branch policy allows only `develop` and
`main` (releases run on `main` in every repo), never at org or repo level.
Every job that needs one declares the environment. A workflow running on any
other branch therefore
runs without them. A read-only token that PR-time CI needs, such as a
third-party vulnerability database key, may stay outside the rule. Each one is
named in the plan with its reason. This absorbs
vergil-project/vergil-actions#1185 in full: its scope (every in-scope org
(§4.11),
the reusable workflows and any caller that mints the token, the repo-config
desired state keeping the environment converged, and the human's moving of
the secrets) and its acceptance criteria become this epic's.
`workflows: write` is granted only after this is in place and verified (§5).
The plan gives this work its own, explicitly sequenced phase with #1185's
acceptance criteria. It gates only the `workflows: write` grant, so the rest
of the epic does not wait on it.

**`secrets: inherit`, on purpose and only here.** A job that calls a
reusable workflow cannot declare an environment, so the caller must still
pass the secrets the called workflow reads. Two routes carry the App
keys, environment secrets included, to the called workflow's jobs that
declare the environment:

- **`secrets: inherit`, within one org.** It works within vergil-project
  (vergil-project/vergil-fixture-python-lib run 45306218449), and is used
  only to the allow-listed vergil-actions workflows. Across orgs `inherit`
  passes nothing.
- **The two App keys by name**, `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.
  Keys passed by name reach a called workflow's environment-declaring jobs
  across orgs and within a repo
  (logical-minds-foundry/mq-gateway-replay-lab run 45390389192, green with
  the org keys hidden). A caller in logical-minds-foundry or
  mnemosys-project passes them by name to the vergil-actions workflows, and
  a call job passes them by name to its repo's own local reusable workflow,
  whose receiving jobs declare `environment: automation` (ruling N0.16).

So the vergil-project callers of the reusable workflows that read these credentials (`cd-release.yml`,
`ops-github-config.yml`, the epic sweep and rollup, the audit workflow, and
`cd-docs.yml`, `cd-docs-refresh.yml` and `publish-index.yml`, which gained
the App keys for the ops reporting of epic vergil-project/.github#1494) pass
`secrets: inherit`. A job that mints the App token itself, such as each ops
reporting step, declares the environment. This deliberately narrows the
least-privilege rule of epic vergil-project/.github#189, and supersedes the
explicit App-key maps added just before this epic and those epic #1494
added to its callers. The over-sharing it permits is small once no
org- or repo-level secret that can write or act remains, and every change
to the called workflows gets merge review. Every other caller keeps an explicit map, and
a `vrg-validate` check refuses `secrets: inherit` on any workflow outside
that allow-list.

With `workflows: write`, the "workflow-touching branch makes GitHub refuse the
agent push" special case disappears from report-ready, `issue-implement` and
CLAUDE.md. The agent pushes and submits workflow changes itself, with merge
review like any change (§4.4).

### 4.3 The provenance check

The check (`lib/pr_provenance.py`) keeps its two outcomes, allowed or
violation, and its two windows (at submit, and in finalize's watch). Its USER
rules change from "never" to "only on its own PR":

- **Its own PR:** a PR whose author is the user agent's login. On its own PR
  the user agent may `create` it, turn auto-merge on (`auto_merge_enabled`
  under every event name, as today), and `merge` it (the `merged_now` path of
  `lib/auto_merge.py`, taken when GitHub refuses auto-merge on a PR that can
  already merge). Whether `edited` is allowed depends on whether submit
  renames a PR. The builder settles that from the code and states it.
- **Anyone else's PR** (the human's, automation's, another identity's): every
  action that is forbidden today stays a violation for the user agent.
- **Approving** stays a violation for the user agent on every PR, its own
  included. No approval is required, so an agent approval can only be a
  mistake.

The AUDIT rules are unchanged.

### 4.4 No protected core

The approved design held back a **protected core** for the human: workflow
files and `vergil.toml` in every repo, vergil-actions `actions/**`, the
hoshin logbook and a per-repo protected-path registry, each change to them
needing a protected-change approval. It is removed (§1.1). There is no
protected-path set, no protected-change approval and no registry. Every
path is judged the same way.

A change to the CI wiring, like a change to the tooling's own gate logic
(in vergil-tooling the identity, provenance and submit checks, the `vrg-gh`
tables, the validators and the desired GitHub config, and in
vergil-claude-plugin the hooks and the merge-review skill), is covered by:

- the up-front design: such changes are worked out in the epic's spec and
  plan;
- merge review on every PR (§4.6), its record checked at submit, in the
  watch and by the daily audit (§4.8);
- the human-gated release: consumer repos, the agent VMs and the audit job
  run released vergil-tooling, plugin and vergil-actions versions, so a
  weakened check merged into one of those repos' `develop` affects nothing
  else until the human releases it;
- the exception registry (§7), aimed at the failure actually seen in the
  fleet, exceptions added in place of fixes.

The builder confirms that consumers and the audit run released versions,
and whether vergil-tooling's own CI runs the PR's own code. If it does,
that repo's PRs check themselves, which is part of the accepted seam (§3).

### 4.5 The authenticated exchange for the human's exemption

The human's merge-review exemption (§4.6) is an approving comment from the
human, nothing cryptographic. The only proof involved is GitHub's record
of who posted each comment.

**Built on the authenticated ruling exchange** of epic
vergil-project/.github#889 (`lib/rulings/`). The agent posts a request
comment. The human replies from their own shell with `vrg-ruling-reply`, which
refuses to post unless the shell's credentials are the one human login. The
verifier accepts the exchange only if the request is by the agent login, the
reply is by the human login, the reply is the first trusted comment after the
request, and neither comment was edited. GitHub attributes each comment to its
author, and an agent VM holds no human credentials, so an agent cannot post
the reply.

**Trusted means what the trust layer says.** Since epic
vergil-project/.github#1467, every comment read goes through the trust layer
(`lib/trust.py`), and the ruling exchange reads its comment stream
(`trust.read_comment_stream`). A comment is trusted when its author, and
every editor, holds a role the reader accepts: the human or the agents, with
the account type checked. A comment by the automation App, a deleted
account, an outsider, or one edited by anyone outside those roles is not in
the stream. The human and agent logins come from the trusted-authors
configuration (`load_trusted_authors()`), never from a second definition.
Every reader this epic adds, of exemptions and merge-review records, is built on that stream, narrowed to the role it accepts.

**Verifying a past exemption.** The exchange verifier in `lib/rulings/`
checks a live exchange, and treats any trusted comment after the response
as fatal. A task issue always gains later comments (merge review's closing
comment, other rulings, report-ready), so submit (§4.6) and the audit
(§4.8) need a separate verifier for an exemption already given. It accepts
one only if the request is by the agent login and the reply by the human
login, neither was edited or deleted, the reply is the first trusted
comment after the request, and the reply approves that request. Comments
after the reply are ignored. Submit and the audit share this one verifier.

**What the reply does.** The reply carries a structured decision: a fixed
word, `approve` or `refuse`, on a defined line. `vrg-ruling-reply` refuses to
post a reply to the exemption request kind without exactly one valid
decision line, and the verifier parses it strictly: anything other than a
well-formed `approve` counts as not approved. A `refuse` stops the PR for
the human, like any ruling.

### 4.6 What `vrg-submit-pr` enforces

Submit runs one new check on each PR before it pushes anything or opens the
PR, where it already checks the ready state and the freeze. A failure
refuses that PR's submit, names the cause and the fix, and leaves no PR open,
so once the cause is fixed (for example, merge review has run) a re-run
submits normally.

1. **Merge-review eligibility.** Merge review's verdict is trusted by design:
   the human is involved only when the working agent and the merge reviewer
   agree that a ruling is needed, and that path already exists. The check does
   not second-guess the verdict. It only confirms that merge review completed
   for the head being submitted, so a skipped or stale review cannot reach
   `develop`.

   **The record is merge review's closing comment.** Merge review already
   posts a closing comment on the task issue from both of its entry points:
   the review before the first submit, and the review of a fix to an open PR
   (which runs no report-ready). That comment, by the agent login, is the one
   merge-review record. It gains a machine-readable line naming the head it
   judged, the change fingerprint (below), and an outcome of `eligible`
   or `not-eligible (<reason>)`. Submit, the watch and the audit (§4.8) all
   read it from GitHub, with one shared reader built on the trust layer's
   comment stream (§4.5), accepting the agent role only. There is no separate field
   in the PR workflow state, and the eligibility line in report-ready's
   `--notes` stays prose for readers.

   **Merge review runs on every PR.** No class of PR is exempt by its kind,
   labels or path. Docs and logbook PRs already get the lighter review:
   merge review's change class, decided mechanically from the changed files
   at the merge base (`vrg-change-class`), makes a `docs-only` change get
   only the merge-quality question. The skills that skip merge review today
   (`hoshin`'s logbook PR, `epic-create`'s spec-and-plan PR,
   `epic-retrospective`'s PR) start running it (§4.10).

   **The only skip is the human's exemption.** A merge-review exemption is a
   new request kind in the authenticated ruling exchange (§4.5). The agent
   posts the request on the task
   issue with the reason and the change fingerprint it covers, and hands
   the human the exact `vrg-ruling-reply` command to run. A verified,
   approving reply stands in for merge review's closing comment as the
   record. It covers the change by fingerprint, like the record it replaces: a
   clean rebase keeps it, and any change to the diff needs a new exemption or
   a merge review. It is the path for the human's "skip merge review on this one", and it
   leaves the human's own decision on the record each time the bar is
   lowered.

   Both outcomes pass: `not-eligible` is the normal result of a run that
   stopped for a ruling the human has answered, and the human's ruling is the
   decision. Submit refuses only a missing record, a record that does not
   cover the head (see "What a merge-review record covers" below), or a run
   that did not complete (stopped, waiting, fatal or in error). There is no
   other skip: the human has no override, and no flag marks a PR as the
   human's own. Coverage is a merge-review record or an approved exemption.

Submit runs the check on every submission, whoever submits it and whoever
recorded the ready state, the human included, since agents write every PR.
The watch and the audit (§4.8) apply it to every agent-involved PR.

**Re-checked while the PR is open.** Auto-merge stays on when a new commit is
pushed, so a check made only at submit could be outrun by a later push (a
`pr-watch` CI fix merge review never saw, a rebase that changed the
branch's own lines). So `vrg-finalize-pr`'s watch runs the check wherever it
already runs the provenance check: when the watch starts, and at every new
head. A failure turns auto-merge off, confirmed by reading it back, as the
watch already does for a provenance violation, and the PR is reported
failed with the cause. Any path that turns auto-merge on, or leaves it on
across a new head, runs the check first. The builder confirms from the code how auto-merge behaves across
a push today and how a fixed PR is re-armed, and puts the checks on that path.

**What a merge-review record covers.** So that a clean rebase does not void
it, the merge-review record names the change it reviewed by its **change
fingerprint**, not only by the head commit. The fingerprint is a hash of the
branch's diff against its merge base, taken with zero context lines, so only
the branch's own added and removed lines count. Line numbers and the
whole-file `index` hashes are dropped, since both change when `develop` moves
underneath. Whitespace is kept exactly, and a binary file counts by its
content. `git patch-id` is not used: it hashes context lines (in its plain
form), ignores whitespace, and sees binaries only as "differ". A head whose
change fingerprint matches is
covered: a clean rebase keeps it, and any change to the branch's own lines,
including a conflict resolution, needs a new merge review. The merge-review
exemption uses the same fingerprint.

**Batch PRs.** A batch PR carries one commit per issue, and each issue's merge
review judges that issue's own commit. So for a batch, each issue's record must
cover that issue's own commit in the batch, matched by the commit's change
fingerprint (its own diff against its parent, with zero context lines). A fix
pushed to an open batch PR is reviewed as a whole PR, as today, and that
review's record covers the change from the base to the new head. Submit, the
watch and the audit apply the same batch rule.

**The enforcement is in the tools, not the skills.** The skills (§4.10) make
the normal path the easy one, but an agent may depart from a skill, for
example on the human's instruction. The checks in submit and in the watch run
whatever the skill did, so a departure cannot merge unchecked.

### 4.7 Finalize in the agent's environment

`vrg-finalize-pr` has no host-specific code. It needs the main clone, its
`.worktrees/`, and `vrg-container-run`. Agent VMs already have all three,
since they validate there. The agent runs finalize in its own VM clone, after
`vrg-submit-pr`, which always finalizes. The VM's develop is then
synced and validated, and the VM's merged worktrees are cleaned up.

The human's Mac clone is not touched. It is synced when the human next
releases, as `vrg-release` already does through `vrg-finalize-pr
--cleanup-only`.

The checkout lock is per clone (`lib/checkout_lock.py`). The builder confirms
that no agent VM shares a clone with the host over a mount. If one does, the
lock does not serialise across them, and the plan addresses it.

### 4.8 The daily audit

This is a new job, separate from the daily GitHub config job in `ops.yml`
(which converges repo settings to their desired state).

The audit covers every merge since the last one it checked, from a stored
marker, not a fixed look-back window, so a run that is skipped or fails is
caught up by the next. A finding, or any read failure, fails the run. It
never passes quietly.

**A security-signal job.** A finding means a merge may have bypassed the
gates, which is a security finding. So the audit is a security-signal job
in the sense of epic vergil-project/.github#1494: it files no public issue
and has no ops reporting step. The `hoshin` daily review reads its runs from
the Actions history, beside the `github-config` and `dependabot-audit` jobs,
and records each failure as a security record through
`vrg-security-record`, naming it in public only by that record's neutral
ref (§4.10).

It also reports every PR merged into `develop` where the merge, or turning
auto-merge on, was done by an account outside the expected set: the human,
the user agent through `vrg-submit-pr`, and the org automation App for its
own fenced PRs, each judged by the trust layer's roles (§4.5). Any other
actor, an `UNTRUSTED` one included, is reported, whatever paths the PR
changed.
It covers merges into `main` too: any merge into `main` whose merge, or
turning auto-merge on, was done by anyone but the human (or the automation
App, if the release flow merges as it) is reported.

It also checks that every agent-involved PR merged into `develop` has a
merge-review record covering its merged change, or a verified, approved
merge-review exemption (§4.6), read with the same shared reader submit and
the watch use. A PR is **agent-involved** when an agent login opened it, or
GitHub attributes any action on it to an agent login. This is decided from
the identities GitHub records, never from branch names. A PR done entirely
by the automation identity or entirely by the human (release and post-release
PRs, update-deps PRs, mechanized PRs) is exempt from this check: the
human's own PR carries the human's own decision,
and an automation PR is confined by its fence (the upstream-refresh fence,
the managed-file sync). The actor check still applies to every PR.

**Where the audit starts.** The audit covers only merges after its marker is
first set, when it goes live before the permission grant (§5). It never
audits earlier merges, which predate the mechanism.

**Who sets the markers.** Repository init (`vrg-github-repo-init`), already
a human-run step, creates both markers at the tips of `develop` and `main`,
for a new repo and when it adopts an existing one. The repos that exist at
go-live get theirs once, as part of the rollout. No recurring human step
remains. From then on every repo has its markers, so a missing marker is a
finding, reported like any other (§4.8, security signal). The audit never
re-creates a marker, since re-creating a deleted one at the current tip
would silently skip every merge since the last run.

What this detects, and what it does not. GitHub records the login that
merged, not the tool, so the audit cannot tell a merge by the user agent
through `vrg-submit-pr` from one by the same login through the raw API or an
edited local tool. It does not try to. It re-checks what the tools enforce on
every merged PR: a merge-review record or exemption covering the change. A
merge that bypassed the tools and skipped it is reported, as is any merge by an unexpected account (a
workflow's own token, for example). A bypass that would have passed every
check anyway goes unreported, and does no harm. The audit does not prevent a
merge; it makes a harmful one visible within a day.

The builder decides where it runs and as which identity, most naturally beside
the daily GitHub config job in `.github`'s `ops.yml`, as the automation App.
It needs the trusted-authors configuration to verify the exchange, and reads
it the way every content-reading workflow already does since epic #1467:
the org variable `VRG_TRUSTED_AUTHORS`, which the daily config audit keeps
free of drift. The audit's workflow, its callers and its code (in
vergil-tooling) are covered, like the rest of the gate logic (§4.4), by
merge review and the human-gated release.

### 4.9 Soft gates: `vrg-gh` and the hooks

The soft gates now say "use the tool" instead of "ask the human":

- **`vrg-gh`**: keep `_DENIED_AGENT`'s denial of `pr create`, `pr edit` and
  `pr merge`, and the user agent's `api` denial. Submit and the auto-merge
  helper call `gh` directly, not through `vrg-gh`, so these denials are what
  keep agent PRs on the sanctioned path. Change the messages to point at
  `vrg-pr-workflow report-ready` and `vrg-submit-pr`.
- **`block-agent-merge.sh`**: keep blocking raw `gh pr merge`, approvals, and
  the merge and review API calls. Change the text: agents merge only through
  `vrg-submit-pr`'s auto-merge.
- **`block-raw-gh-pr-create.sh`**: keep the block. Its text already points at
  `vrg-submit-pr`.
- **`block-github-contents-api.sh`**: change "the human submits the PR" to the
  agent's own flow.

### 4.10 Skills, steering and the permission-model spec

- **`issue-implement`**: the implementing agent now owns the PR through to the
  merge. After report-ready it runs `vrg-submit-pr`, and it carries out
  everything `pr-watch` does today until the PR merges or needs the human. The
  two main cases are a failed merge (a conflict to rebase) and a CI/CD failure
  to fix, each fix merge-reviewed as today and followed by a rerun of
  `vrg-finalize-pr`. Drop "Only the human opens the PR" and the workflow-push
  special case. Keep the cloud relay section for cloud sessions (§3).
- **`pr-watch`**: made fully consistent with the new `issue-implement`. Either
  `issue-implement` takes over its procedure and `pr-watch` stays only as the
  standalone entry for re-entering an open PR, or both share one procedure.
  The builder chooses, but there is one procedure, not two that can drift. It
  reruns `vrg-finalize-pr` itself after pushing a fix, instead of telling the
  human to. Its rule "you never run `vrg-submit-pr` or `vrg-finalize-pr`"
  becomes "never with `--release`". Its rule for a fix whose merge review
  ends `not-eligible` ("do not push; the human decides") changes to match
  §4.6: it pushes once every ruling on the fix is answered. Its skip of
  merge review for a fix to a `Logbook:` PR goes: merge review runs on
  every fix (§4.6), or the watch's re-check would turn auto-merge off on the
  fixed logbook PR.
- **`merge-review`**: request the merge-review exemption only on the
  human's instruction (§4.6). Add the machine-readable record line to its closing comment, from both entry
  points (§4.6).
- **`epic-implement`**: after a task merges, re-read the epic and continue with
  the next runnable task. Stop for the human only on a ruling, a failure, or a task blocked on a release or a human-gated deployment
  precondition. Drop "Your responsibility ends here", and the line saying it
  never submits or merges the terminal retrospective PR.
- **`epic-create`** and **`epic-retrospective`**: the agent submits the docs
  PR. The human's review of the spec, plan and retrospective already happens
  in the session. Both PRs get merge review (§4.6).
- **`fixture-run`**: the agent submits the fixture change PRs. The fixture
  releases stay human-gated.
- **`hoshin`**: the logbook PR gets merge review (§4.6). Its daily review
  also reads the agent audit's runs as a security-signal job (§4.8). That
  change needs no new permission and ships before the audit goes live. The
  protected-change approval that was to replace the human's submit as the
  review's sign-off is removed (§4.4), so this epic does not otherwise
  change the logbook PR's handoff.
- **Other wording on the old boundary:** `issue-deploy`'s line that agents
  hand off `vrg-submit-pr`, and the vergil-tooling PR-workflow messages that
  tell the agent the human submits (the freeze refusal among them), move to
  the new boundary with the skills. The site docs and guides are the
  documentation review's sweep.
- **CLAUDE.md** (`.github` and the consumer template in vergil-tooling): replace
  "Agents never run `vrg-submit-pr`" with the new boundary. Agents submit and
  finalize through the tools and never with `--release`. Releases and
  promotion are human actions. The frozen-branch rule stays (the freeze now lasts from report-ready
  to the agent's own submit). The workflow-push paragraph becomes one line
  until `workflows: write` is granted: a refused workflow push is handed off
  as before, since report-ready has recorded the state. After the grant it
  goes.
- **The permission-model spec** in vergil-tooling: amend "the human triggers;
  the agent doesn't" and the user App permission table. State the new principle:
  agents trigger the PR path, held by its gates (merge review on every PR,
  its record checked at submit, in the watch and by the daily audit), and
  humans trigger releases.

### 4.11 The orgs in scope

> **Amended 2026-11-25.** The user App's `workflows: write` and
> `pull_requests: write` were accepted on the three out-of-scope
> installations too. The human accepted that risk until those orgs are
> renovated (rulings N0.13 and N0.21; see the plan's Evolution during
> execution). The last sentence below is superseded.

The epic covers three orgs: vergil-project, logical-minds-foundry and
mnemosys-project, the ones under active work. Archived repos in them are
left alone. diogenes-project, mq-rest-admin-project and vergils-nemesis
are out of scope: they are not maintained now, and some of their repos
cannot merge a PR until they are renovated. The user App's new permissions
are not accepted on any installation outside the three orgs, so the
accepted seam (§3) never reaches a repo the audit does not watch.

## 5. Rollout

The rollout is ordinary fleet-wide releases in the right order. There is no
per-repo switch: a release switches the fleet at once, as every vergil-tooling
and plugin release does. The order exists only so that no check ships before
anything can pass it, and no instruction ships before its permission.

0. **Standalone hardening first.** The `main` ruleset (§4.1) and the
   environment-scoped secrets (§4.2) are the plan's first, separately
   mergeable tasks. Each is valuable hardening on its own, and both land even
   if the rest of the epic stalls.
1. **Records first.** A plugin release in which merge review writes the
   machine-readable record line (§4.6) through the new tooling command. It
   changes nothing else, and it ships before any check that reads the record.
2. **Gates.** The vergil-tooling release with the submit and watch checks,
   the provenance change, the exemption kind, and the audit.
   The human's own submits pass these checks from the start, since merge
   review already writes records.
3. **App permissions.** Two preconditions, both fleet-wide: the `main`
   ruleset (§4.1) is converged in every repo, and the daily audit (§4.8), all
   of its checks, runs green in every repo with its marker initialised. Only
   then does the human grant the user App `pull_requests: write`. From this
   point the accepted seam exists fleet-wide, and the audit is already
   watching it. `workflows: write` follows only once the environment-scoped
   secrets (§4.2) are in place and verified; until then workflow-touching
   branches keep the human's push.
4. **Instructions.** The plugin release whose skills tell the agent to submit
   and finalize, and the steering text, ship only after the grant, so the
   instructions never run ahead of the permission.
5. **First live cycle on the fixtures.** The fixture runs are the first agent
   submissions, followed by a validation that an agent PR with the human's
   merge-review exemption merges, and that the daily audit reports one
   deliberate merge, made by the human, of an agent-opened PR with no
   merge-review record.

## 6. Verification

- Unit tests for the provenance rules (own PR allowed, others' PRs and approvals
  forbidden), exemption verification (an edited comment or a reply from the
  wrong login rejects it) and the eligibility check. The change fingerprint
  is tested by rebasing a branch over a `develop` edit next to its hunks: the
  record still covers it. A conflict resolution that changes the branch's own
  lines does not.
- The first live cycle of §5 step 5, recorded in a validation task.
- A failure to read any of the inputs refuses, and that is tested.

## 7. Follow-on

- **The exception registry.** The authenticated ruling exchange (§4.5),
  applied to a new,
  unregistered coverage exclusion, lint suppression or skipped test: merge
  review raises it, the human attests to it, and it is entered in a registry.
  Plus a retroactive audit of the exceptions agents added in the past instead
  of fixing the underlying problem.
- **Releases and promotion by agents**, once cross-epic release coordination is
  designed (vergil-project/.github#467), and vergil-project/.github#1088
  rethought on top of this epic.
- **Merging `vrg-pr-workflow` into `vrg-submit-pr`.**
- **Cloud-side submission.**

## 8. Left to the builder

- The exemption request kind's exact shape (§4.5).
- Whether `edited` is allowed on the agent's own PR (§4.3).
- The exact form of the closing comment's record line (§4.6). The human
  gets no override (ruling N0.12).
- Where the daily audit runs and where it stores its marker (§4.8).
- Confirming no VM shares a clone with the host (§4.7).
- Confirming that the restrict-updates rule is available on the org's GitHub
  plan and refuses a PR merge into `main` by a non-bypass identity, and which
  identity merges release PRs today (§4.1). If the rule cannot do this, the
  audit's `main` check remains and the gap is stated in §3 as an accepted
  seam.
- Whether the permissions a workflow may request for its own `GITHUB_TOKEN`
  can be capped below write at the org level, and if so, capping them (§4.2,
  §4.8).
- Which GitHub records reliably name the identity that pushed a commit to a
  PR's branch, since commit author fields can be set freely. If none does,
  "agent-involved" means an agent login opened the PR or acted on it (§4.8).
