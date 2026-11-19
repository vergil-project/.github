# Agents submit and finalize their own PRs into develop — design

- **Epic:** vergil-project/.github#1384
- **Planning issue:** vergil-project/.github#1383 (review history is in its
  comments)
- **Status:** draft
- **Repos:** vergil-tooling, vergil-claude-plugin, vergil-actions, .github,
  and every member repo (registry rollout)
- **Related:** epic vergil-project/.github#889 (authenticated human rulings),
  epic vergil-project/.github#1218 (parallel PR merging: auto-merge and the
  provenance check), planning items vergil-project/.github#1088
  (fixture-scoped agent rights), vergil-project/vergil-actions#1185
  (environment-scoped automation credentials, absorbed by §4.2),
  vergil-project/.github#467 (release holds)
  and vergil-project/.github#892 (authenticate every ruling); the
  permission-model spec,
  `vergil-tooling/docs/specs/2026-05-29-agent-permission-model-design.md`

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
- an approval on a change to the **protected core** (§4.4), the small set of
  files that define the gates themselves;
- failure triage and recovery decisions.

Releases and promotion stay human-only. Coordinating releases across epics
running in parallel (holding a release until related changes land, without
two epics deadlocking each other) needs its own design, which is left to
later work (vergil-project/.github#467, and vergil-project/.github#1088
rethought on top of this epic).

This is one step toward the long-term goal of agents taking an epic from
idea to production, with the human handling only exceptions.

### 1.1 Why the protected core exists

Every right the agent gains here follows the same argument: the human was
approving without looking, so the approval protected nothing. One class of
change is different. On a `pull_request` event GitHub runs the workflow file
as the PR itself has it, so a PR that edits a workflow is checked by its own
edited version. With the `workflows` permission, auto-merge and no required
approvers, a PR could change the checks that judge it. The realistic risk is
not a hostile agent. It is an honest one under pressure: facing a failing
check, the quickest fix is to weaken the check. Agents in this fleet have done
exactly that with coverage and test failures, adding exceptions instead of
fixing the problem. The gates are the approval, so a gate the gated change can
edit is not a gate. The protected core keeps a human approval on exactly
those files, and nowhere else.

## 2. Goals

1. The user agent runs `vrg-submit-pr` (non-release paths) and
   `vrg-finalize-pr` (non-release paths) itself, in its own environment.
2. `issue-implement` and `epic-implement` carry a task through report-ready,
   submit, `pr-watch` and finalize to a merge, and `epic-implement` moves on
   to the next runnable task without waiting for the human.
3. A change to a protected path merges only with a human approval tied to the
   exact contents of each protected file, enforced by `vrg-submit-pr`.
4. A PR merges only if merge review completed for the head it merges at,
   enforced by `vrg-submit-pr`.
5. A daily audit reports any merged PR that touched a protected path without
   a valid approval.
6. Agents can push workflow changes themselves (`workflows: write`), so a
   workflow-touching PR no longer needs the human's push.
7. The rollout proves the whole cycle on the fixture repos before it reaches
   the rest of the fleet.

## 3. Non-goals and anti-goals

### Non-goals

- **A separate automation identity that submits on the agent's behalf.** It
  would keep the merge right away from the coding agent and run the
  protected-path check outside its reach. It is set aside because the GitHub
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
  would pass the checks in §4.6 and §4.8. The human's approvals and
  exemptions are not in this seam, since an agent cannot post as the human.
- **Bringing human review back by another route.** If the design makes the
  human approve ordinary PRs, it has failed even if it works. Only protected
  paths reach the human.
- **An approval that rebases void.** An approval tied to a commit would be voided
  by every rebase and CI fix, and the human would re-approve the same change
  repeatedly. It is tied to file contents instead (§4.5).
- **A silent pass.** A failure to read the registry, the approval or the
  merge-review record refuses the submit. It never lets the PR through.

## 4. Design

### 4.1 Where the boundary is encoded today

Paths are in vergil-tooling's `src/vergil_tooling/` unless stated.

- **`bin/vrg_submit_pr.py:168`.** Today `is_agent()` refuses the whole
  command. Change: refuse only AUDIT, and agents on the `--release` path;
  allow USER otherwise.
- **`bin/vrg_finalize_pr.py`.** No identity check today; `--release` chains
  `vrg-release`, which refuses agents. Change: refuse `--release` for agents
  up front, before any PR is watched.
- **`bin/vrg_release.py:31`, `bin/vrg_promote.py:60`, `lib/promote.py:349`.**
  Refuse agents. Unchanged.
- **`lib/pr_provenance.py:69-76`.** USER is forbidden `created`, `merged`,
  `auto_merge_enabled`, `edited` and more. Change: §4.3.
- **`bin/vrg_gh.py:100-113` (`_DENIED_AGENT`).** Denies agents `pr create`,
  `pr edit` and `pr merge`. Change: keep the denials, point the message at
  `vrg-submit-pr` (§4.9).
- **Plugin hooks** `block-agent-merge.sh`, `block-raw-gh-pr-create.sh`,
  `block-github-contents-api.sh`. Block raw merges, approvals and PR creation;
  the text says the human submits. Change: keep the blocks, change the text
  (§4.9).
- **Plugin skills** `issue-implement`, `pr-watch`, `merge-review`,
  `epic-implement`. Stop at report-ready and tell the human to rerun
  `vrg-finalize-pr`. Change: §4.10.
- **`.github/CLAUDE.md:34-47` and `data/claude_md_consumer.md:29-42`.**
  "Agents never run `vrg-submit-pr`." Change: §4.10.
- **`lib/github_config.py:158-163`.** A comment says agents cannot turn
  auto-merge on. Change: update the comment.
- **The user GitHub App.** `pull_requests: read`, `workflows: none`. Change:
  `pull_requests: write`, `workflows: write` (§4.2).

The "Branch protection" ruleset covers both `main` and `develop`, requires no
approving reviews and has no bypass actors (`lib/github_config.py:226-250`).
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
credential a workflow reads as a secret (at least the org automation App's
`APP_CLIENT_ID` and `APP_PRIVATE_KEY`) lives only in a GitHub environment
whose deployment branch policy allows only `develop` (and `main` where a job
needs it), never at org or repo level. Every job that needs one declares the
environment. A workflow running on any other branch therefore runs without
them. This absorbs vergil-project/vergil-actions#1185 in full: its scope
(every in-scope org, the reusable workflows and any caller that mints the
token, the repo-config desired state keeping the environment converged, and
the human's moving of the secrets) and its acceptance criteria become this
epic's. `workflows: write` is granted only after this is in place and
verified (§5). The plan gives this work its own, explicitly sequenced phase
with #1185's acceptance criteria. It gates only the `workflows: write` grant,
so the rest of the epic does not wait on it.

With `workflows: write`, the "workflow-touching branch makes GitHub refuse the
agent push" special case disappears from report-ready, `issue-implement` and
CLAUDE.md. Workflow files are protected paths (§4.4), so they still need the
human's approval, but the agent pushes and submits them itself.

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

### 4.4 The protected core

A **protected path** is a file whose change needs the human's approval before
it merges. The set is deliberately small: the files that define the gates.

**Two sources, one set.** The set for a repo is the union of:

- **the fleet default**, shipped in vergil-tooling: `.github/workflows/**` in
  every repo, and the protected-path registry file itself;
- **the repo's own registry**, a file in the repo naming further paths. In
  vergil-tooling it names the gate code. In vergil-claude-plugin it names the
  hooks and the merge-review skill. Any other repo names what it needs.

The registry file is itself a protected path, so removing a path from it needs
an approval. The fleet default lives in vergil-tooling code, which is covered by
vergil-tooling's own registry, so changing the default needs an approval too.

**What starts in the core.** The builder fixes the exact list in the plan,
covering at least these categories:

- workflow files (`.github/workflows/**`), in every repo;
- in vergil-tooling: the identity checks (`lib/identity_mode.py`,
  `lib/rulings/`, `lib/trusted_authors.py`), the provenance check, the
  protected-path check and audit (§4.5, §4.8), the `vrg-gh` tables, the
  submit gates in `bin/vrg_submit_pr.py`, and the desired GitHub config
  (`lib/github_config.py`: rulesets, required checks, App and merge settings);
- in vergil-claude-plugin: `hooks/` and the merge-review skill and reviewer;
- in vergil-actions: the reusable workflows the fleet's CI calls.

**What stays out.** Ordinary code, tests and docs, even in the tooling repos.
The core is for the gates, not for anything important. A core that grows to
cover most of a repo has brought human review back (§3).

### 4.5 The protected-change approval

A **protected-change approval** is an approving comment from the human,
nothing cryptographic. The only proof involved is GitHub's record of who
posted each comment. Each protected file's git blob hash, the ID git gives
every version of a file's contents, ties the approval to the exact contents
approved. A protected file is **not approved** when no approval covers its
current blob.

**Built on the authenticated ruling exchange** of epic
vergil-project/.github#889 (`lib/rulings/`). The agent posts a request
comment. The human replies from their own shell with `vrg-ruling-reply`, which
refuses to post unless the shell's credentials are the one human login. The
verifier accepts the exchange only if the request is by the agent login, the
reply is by the human login, the reply is the first trusted comment after the
request, and neither comment was edited. GitHub attributes each comment to its
author, and an agent VM holds no human credentials, so an agent cannot post
the reply.

**Verifying a past approval.** The exchange verifier in `lib/rulings/` checks
a live exchange, and treats any trusted comment after the response as fatal.
A task issue always gains later comments (merge review's closing comment,
other rulings, report-ready), so submit (§4.6) and the audit (§4.8) need a
separate verifier for an approval already given. It accepts an approval only if
the request is by the agent login and the reply by the human login, neither
was edited or deleted, the reply is the first trusted comment after the
request, and the reply approves that request. Comments after the reply are
ignored. Submit and the audit share this one verifier.

**What the request records.** A new request kind for protected changes. Its
text lists every protected path the PR changes, and for each the git blob hash
of the file at the head being submitted (a deleted file is recorded as
deleted). It links the diff for those files, so the human can see what they are
approving.

**What the reply does.** The reply carries a structured decision: a fixed
word, `approve` or `refuse`, on a defined line. `vrg-ruling-reply` refuses to
post a reply to this request kind without exactly one valid decision line, and
the verifier parses it strictly: anything other than a well-formed `approve`
counts as not approved. An `approve` approves exactly the listed blobs. A
`refuse` stops the PR for the human, like any ruling. The merge-review
exemption (§4.6) uses the same structured reply.

**Why blob hashes.** An approval tied to file contents survives everything that
leaves the protected files alone: a rebase, a CI fix elsewhere in the branch, a
`pr-watch` reconcile. Any change to a protected file produces a new blob hash
and voids the approval for that file.

**Where it is raised.** Merge review spots a protected change on its run, names
it in its findings, and asks for the approval through the ruling flow before
report-ready. Merge review is where the human hears about it. It is not where it
is enforced, since merge review is an agent following a skill and can miss a
file.

### 4.6 What `vrg-submit-pr` enforces

Submit runs two new checks on each PR before it pushes anything or opens the
PR, where it already checks the ready state and the freeze. Each failure
refuses that PR's submit, names the cause and the fix, and leaves no PR open,
so once the cause is fixed (for example, the human has approved) a re-run
submits normally.

1. **Protected paths.** Submit lists the files the PR changes against a freshly
   fetched `origin/develop` (the merge base), and matches them against the
   repo's protected set (§4.4) as it stands at the PR's head and on `develop`.
   Both matter: a PR that removes a path from the registry is judged by the
   registry it is removing it from. For each protected file changed, the blob at
   the head must equal a blob recorded in a verified approval on the
   PR's task issue. For a batch PR, an approval may sit on any issue of the batch.
2. **Merge-review eligibility.** Merge review's verdict is trusted by design:
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
   judged, the patch-id of the change (below), and an outcome of `eligible`
   or `not-eligible (<reason>)`. Submit, the watch and the audit (§4.8) all
   read it from GitHub, with one shared reader. There is no separate field
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
   new request kind in the same authenticated ruling exchange as the
   protected-change approval (§4.5). The agent posts the request on the task
   issue with the reason and the patch-id of the change it covers, and hands
   the human the exact `vrg-ruling-reply` command to run. A verified,
   approving reply stands in for merge review's closing comment as the
   record. It covers the change by patch-id, like the record it replaces: a
   clean rebase keeps it, and any change to the diff needs a new exemption or
   a merge review. An exemption never waives the protected-change approval.
   It is the path for the human's "skip merge review on this one", and it
   leaves the human's own decision on the record each time the bar is
   lowered.

   Both outcomes pass: `not-eligible` is the normal result of a run that
   stopped for a ruling the human has answered, and the human's ruling is the
   decision. Submit refuses only a missing record, a record that does not
   cover the head (see "What a merge-review record covers" below), or a run
   that did not complete (stopped, waiting, fatal or in error). The human's
   own `vrg-submit-pr` runs (for example the emergency CLI mode) are subject
   to the same check. The builder decides whether the human gets an
   explicit, logged override.

Both checks run for every identity that submits, the human included.

**Re-checked while the PR is open.** Auto-merge stays on when a new commit is
pushed, so a check made only at submit could be outrun by a later push (a
`pr-watch` CI fix that edits a workflow file, a rebase merge review never
saw). So `vrg-finalize-pr`'s watch runs both checks wherever it already runs
the provenance check: when the watch starts, and at every new head. A failure
turns auto-merge off, confirmed by reading it back, as the watch already does
for a provenance violation, and the PR is reported failed with the cause. Any
path that turns auto-merge on, or leaves it on across a new head, runs both
checks first. The builder confirms from the code how auto-merge behaves across
a push today and how a fixed PR is re-armed, and puts the checks on that path.

**What a merge-review record covers.** So that a clean rebase does not void
it, the merge-review record names the change it reviewed by the git patch-id
of the branch's diff against its base, not only by the head commit. A head
whose diff has the same patch-id is covered: a clean rebase keeps it, and any
change to the diff needs a new merge review. This mirrors the blob-hash rule
for approvals (§4.5).

**The enforcement is in the tools, not the skills.** The skills (§4.10) make
the normal path the easy one, but an agent may depart from a skill, for
example on the human's instruction. The checks in submit and in the watch run
whatever the skill did, so a departure cannot merge unchecked.

### 4.7 Finalize in the agent's environment

`vrg-finalize-pr` has no host-specific code. It needs the main clone, its
`.worktrees/`, and `vrg-container-run`. Agent VMs already have all three,
since they validate there. The agent runs finalize in its own VM clone, after
`vrg-submit-pr` (which chains finalize by default). The VM's develop is then
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
caught up by the next. A daily job reports every PR merged into `develop`
since that marker that changed a protected path, and checks each with the
same check function submit uses: the blob of each protected file at the
PR's merged head commit must be covered by an approval, on the task issue
or, for a batch PR, on any issue of the batch. A file whose blob in the
merge commit differs from its blob at the PR's head (a squash merge onto a
`develop` that also changed it) is reported as its own named condition for
the human to look at, not as a not-approved change. A not-approved file, or
any read failure, is reported loudly as a failing run and an issue for the
human. It never passes quietly.

It also reports every PR merged into `develop` where the merge, or turning
auto-merge on, was done by an account outside the expected set: the human,
the user agent through `vrg-submit-pr`, and the org automation App for its
own fenced PRs. Any other actor is reported, whatever paths the PR changed.
It covers merges into `main` too: any merge into `main` whose merge, or
turning auto-merge on, was done by anyone but the human (or the automation
App, if the release flow merges as it) is reported.

It also checks that every PR merged into `develop` has a merge-review record
covering its merged change, or a verified, approved merge-review exemption
(§4.6), read with the same shared reader submit and the watch use.

What this detects, and what it does not. GitHub records the login that
merged, not the tool, so the audit cannot tell a merge by the user agent
through `vrg-submit-pr` from one by the same login through the raw API or an
edited local tool. It does not try to. It re-checks what the tools enforce on
every merged PR: protected files covered by an approval, and a merge-review
record or exemption covering the change. A merge that bypassed the tools and
skipped either is reported, as is any merge by an unexpected account (a
workflow's own token, for example). A bypass that would have passed every
check anyway goes unreported, and does no harm. The audit does not prevent a
merge; it makes a harmful one visible within a day.

The builder decides where it runs and as which identity, most naturally beside
the daily GitHub config job in `.github`'s `ops.yml`, as the automation App.
That job needs the trusted-authors configuration (the human login) to verify
the exchange, and the plan states how it gets it. The audit's workflow and code
are protected paths.

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
  §4.6: it pushes once every ruling on the fix is answered.
- **`merge-review`**: raise protected changes and request the approval (§4.5).
  Add the machine-readable record line to its closing comment, from both entry
  points (§4.6).
- **`epic-implement`**: after a task merges, re-read the epic and continue with
  the next runnable task. Stop for the human only on a ruling, an approval, a
  failure, or a task blocked on a release or a human-gated deployment
  precondition. Drop "Your responsibility ends here", and the line saying it
  never submits or merges the terminal retrospective PR.
- **`epic-create`** and **`epic-retrospective`**: the agent submits the docs
  PR. The human's review of the spec, plan and retrospective already happens
  in the session. Both PRs get merge review (§4.6).
- **`fixture-run`**: the agent submits the fixture change PRs. The fixture
  releases stay human-gated.
- **`hoshin`**: the agent submits the logbook PR. Today the human's submit is
  the review's sign-off. It becomes an authenticated approval instead: the
  human's in-session approval of the full text is posted as an `approve` reply
  through the same ruling exchange (§4.5), and submit refuses a logbook PR
  without it. The logbook PR gets merge review (§4.6).
- **CLAUDE.md** (`.github` and the consumer template in vergil-tooling): replace
  "Agents never run `vrg-submit-pr`" with the new boundary. Agents submit and
  finalize through the tools and never with `--release`. Releases and
  promotion are human actions. A change to a protected path needs the human's
  approval. The frozen-branch rule stays (the freeze now lasts from report-ready
  to the agent's own submit). The workflow-push paragraph goes.
- **The permission-model spec** in vergil-tooling: amend "the human triggers;
  the agent doesn't" and the user App permission table. State the new principle:
  agents trigger the PR path, humans trigger releases, and the protected core is
  the human-held chokepoint inside the PR path.

## 5. Rollout

1. **Code first, gated off.** The tooling, plugin and steering changes land and
   release with agent submission enabled per repo, off by default. The builder
   chooses the switch (for example a `vergil.toml` key, itself under the
   protected core).
2. **App permissions.** Two preconditions, both fleet-wide: the `main`
   ruleset (§4.1) is converged in every repo, and the daily audit (§4.8), all
   of its checks, runs green in every repo with its marker initialised. Only
   then does the human grant the user App `pull_requests: write`. From this
   point the accepted seam exists fleet-wide, regardless of the switch, and
   the audit is already watching it. `workflows: write` follows only once the environment-scoped
   secrets (§4.2) are in place and verified; until then workflow-touching
   branches keep the human's push.
3. **Fixtures.** Enable agent submission in the fixture repos. Prove one full
   cycle per fixture: an ordinary change submitted and merged by the agent, and
   a workflow change that is refused until approved and then merges. Run the
   daily audit against them, including one deliberate unapproved merge made by the
   human to confirm that the audit reports it.
4. **Fleet.** Enable it repo by repo, with each repo's protected-path registry
   in place first. The builder decides whether the switch stays afterwards (as a
   per-repo opt-out) or is removed.

## 6. Verification

- Unit tests for the provenance rules (own PR allowed, others' PRs and approvals
  forbidden), the protected-path match (including a registry that removes its
  own entry, a deleted file and a batch PR), blob-hash approval verification
  (a rebase keeps it, an edit voids it, an edited comment or a reply from the
  wrong login rejects it) and the eligibility check.
- The fixture cycle of §5 step 3, recorded in a validation task.
- A failure to read any of the inputs refuses, and that is tested.

## 7. Follow-on

- **The exception registry.** The same approval mechanism, applied to a new,
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

- The registry file's name and format, and the exact starting list per repo
  (§4.4).
- The request kind's exact shape and how the diff is linked (§4.5).
- Whether `edited` is allowed on the agent's own PR (§4.3).
- The exact form of the closing comment's record line, and whether the human
  gets an override
  (§4.6).
- The rollout switch and whether it stays (§5).
- Where the daily audit runs, where it stores its marker, and how it gets the
  trusted-authors configuration (§4.8).
- Confirming no VM shares a clone with the host (§4.7).
- Confirming that the restrict-updates rule is available on the org's GitHub
  plan and refuses a PR merge into `main` by a non-bypass identity, and which
  identity merges release PRs today (§4.1). If the rule cannot do this, the
  audit's `main` check remains and the gap is stated in §3 as an accepted
  seam.
- Whether the permissions a workflow may request for its own `GITHUB_TOKEN`
  can be capped below write at the org level, and if so, capping them (§4.2,
  §4.8).
