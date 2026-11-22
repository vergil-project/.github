# trusted-content-only: automation acts only on trusted content — design

- **Epic:** vergil-project/.github#1467
- **Planning issue:** vergil-project/.github#579 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Vergil is a single person's development stack. Its automation (the
`vrg-*` tools, the GitHub workflows and the agent skills) reads GitHub
content to decide what to do: issue and PR bodies, comments, reviews,
titles, branch names, who closed an issue. The org's repos are public, so
any GitHub account can write some of that content. The rule this epic
enforces everywhere:

> Automation acts only on GitHub content written by a trusted identity:
> the human's own account, one of the human's agent App accounts, or the
> automation App of a Vergil-managed org, as the human's Vergil
> configuration names them.

Much of this is in place (the `trusted_authors` library, trusted comment
filters in the ruling, scope-change, queue sizing, proxy and planning-close
paths, the issue-author check in `issue-implement`, the trusted comment rule
in `epic-create`, `pr-watch` and `epic-retrospective`). Epic
vergil-project/.github#1440 adds more for releases. This epic finds and
closes **every remaining gap**, and makes the rule structural, so that a new
tool cannot quietly reintroduce one.

The project does not accept outside contributions (its contributing guide
says so). Content from any other account is therefore never legitimate
input. It is never acted on, never adopted and never re-filed; it is
stopped on or reported, and the human decides what to do with it.

## 2. Goals, non-goals, anti-goals

### Goals

- Every read of GitHub content that drives behaviour goes through one trust
  layer, and acts only on trusted content.
- Content an action depends on, from an untrusted identity, stops the run
  loudly.
- Every other untrusted content is skipped and reported, never silently.
- The human sees, each morning, any outsider activity since the last review.
- A test and a hook make it hard to add a new untrusted read by accident.
- The layer is the groundwork for vergil-project/.github#892 (below).

### Non-goals

- Which decisions must be posted by the human's own account rather than by
  an agent. That is vergil-project/.github#892, planned straight after this
  epic and implemented after it ships. Here, "trusted" is the human and the
  agents alike.
- Anything epic vergil-project/.github#1440 delivers: the locked,
  human-only release tracking issue and its identity rule, the trusted
  closing references, the `Released:` records and `vrg-release-record
  check`. This epic builds on them and does not redo them.
- A path for outsider issues to become work (adopting them in place or
  re-filing them). An outsider issue needs the human.
- Cleaning up the existing spam comments.

### Anti-goals

Failures even if the code works:

- A filter that drops untrusted content without saying so.
- A fallback that treats untrusted or unreadable content as "not present"
  and carries on as if nothing happened where the content was load-bearing.
- A second, divergent definition of "trusted" anywhere in the fleet.
- Outsider text, or the existence of outsider activity, published by our
  own tools (in issues, the activity log or the docs site).

## 3. Constraints

- **Order.** This epic starts implementation only once epic #1440 is
  closed, and assumes all of it is delivered. #892 is planned right after
  this epic's docs PR and implemented after this epic closes.
- **Security run.** The findings live in private advisories tracking the
  planning issue. The site inventory is in seven of them: GHSA-jj85-qr28-7jwh, GHSA-r4j2-cwcp-xg56,
  GHSA-hrq4-7qg2-7xxf, GHSA-w5hq-5p7v-qgc8, GHSA-777j-j57r-92ff,
  GHSA-q6hx-8c85-9f56 and GHSA-crcw-h2ff-qgr5. Their site inventory
  (written 2026-10-28) is the work list; each task re-checks its sites
  against current code first, since some are fixed, removed or moved, and
  #1440 changes the release ones. Public text (this spec,
  the plan, issues, commits, PRs) describes mechanisms and names findings
  only by GHSA ID.
- **No silent failures** (fleet principle): a failure to read who wrote
  something is itself a stop, never "trusted" or "absent".
- **Repos.** Every repo of the orgs listed in the human's Vergil
  configuration (§5.1a). The code changes land in vergil-tooling,
  vergil-claude-plugin, vergil-actions and vergil-containers.
- **Proportion.** This is tactical, GitHub-specific hardening: the fleet
  plans to move to a self-hosted forge with no outside access, which will
  reshape this security architecture. Each mechanism is the simplest one
  that gives the protection; nothing is built for cases that cannot occur
  today.
- **Content reads live in vergil-tooling.** vergil-tooling is an
  application, not a library, and the fleet's tooling lives in it. Only
  vergil-tooling reads behaviour-driving GitHub content; other repos
  configure it or call its commands. Its Python code calls `gh` and `git`
  directly and manages its own credentials for its context; the `vrg-gh`
  and `vrg-git` wrappers and the trusted read command (§6.4) are for agents
  working interactively, never called from Python automation.

## 4. Approaches

- **A. One trust layer, enforced (chosen).** A single vergil-tooling module
  is the only way to read behaviour-driving GitHub content; every site moves
  onto it; a guard test and a skill-facing read command with a hook
  backstop keep new gaps out. Chosen because it is the only approach where
  "the last gaps are closed" can be checked rather than asserted, and it
  gives #892 one place to tighten.
- **B. Patch site by site.** Fewer moving parts, but nothing stops the next
  raw read, so the gap list regrows. Rejected: it defeats the point of the
  epic.
- **C. Filter inside the `vrg-gh` wrapper.** One choke point, but `vrg-gh`
  passes through many output shapes and cannot know whether a given read
  should stop or warn. Rejected as a fragile second parser.

## 5. The trust rule

### 5.1 Trusted identity

An identity is trusted when it holds one of three roles:

- **`human`** (global): a `human` login of the trusted-authors
  configuration, and a **User** account.
- **`agents`** (global): an `agents` login of the trusted-authors
  configuration, the human's agent App identities, and a **Bot** account.
- **`automation`**, **for the org whose repo holds the content**: that org's
  automation App, which runs fully mechanized workflows with no agent in
  them, and a **Bot** account. Its slug is the `automation_app` of that
  org's stanza in the human's Vergil configuration (§5.1a), and its login
  is `<slug>[bot]`. An automation identity is not trusted in another org's
  repos.

**Matching.** A login is compared after a leading `app/` or a trailing
`[bot]` is stripped, **and** the identity's account type (GraphQL
`__typename`, or REST `type`) must be the one its role requires. A login
that matches a role with the wrong account type is a stop, never trusted.

**Deleted accounts.** An identity GitHub reports as deleted (a `null`
author, editor or actor, or the `ghost` login) is **untrusted**, and follows
§5.3's outcome for its kind: reported for a comment, a stop only where the
content is load-bearing. A stop for an unreadable identity is kept for
real read failures (an API error, a missing field, an incomplete page).

The three roles stay distinct (PR provenance names the role; `vrg-ruling`
never accepts an automation identity as an agent; #892 can treat each
separately), but by default all three are "trusted". One loader is the only
reader of the roles; a missing or invalid configuration is a stop.

### 5.1a Where the configuration lives

These are fixed values, set once and never looked up at run time. The
human's host `~/.config/vergil/identities.toml` is the single source:

```toml
[trusted_authors]
human  = ["<human-login>"]
agents = ["<agent-app-slug>", "<agent-app-slug>"]

# One stanza per org Vergil manages. Adding an org to Vergil adds its
# stanza, as a step of setting the org up. Values are opaque strings,
# never derived from the org's name.
[orgs.<org>]
automation_app = "<automation-app-slug>"
```

- **Vergil orgs are the listed orgs.** An org without a stanza is not
  managed by Vergil: it is never swept, reported on or trusted, whoever
  owns it.
- **Host and agent VMs** read the file: in place on the host, and on VMs
  through `vrg-vm`'s delivery (as `trusted-authors.toml` today), which
  carries the `orgs` stanzas too.
- **GitHub Actions runners** (the sweep, the rollup, the refresh, any other
  unattended job that reads content) get a derived copy. A new `vrg-*`
  command, run by the human at org setup and whenever the file changes,
  writes each listed org's `VRG_AUTOMATION_APP` and a delivered
  `human`/`agents` list as org Actions variables, from the file. Workflows
  pass them into the job environment; the loader under Actions reads them
  there and stops when they are missing or invalid. The existing uses of
  `VRG_AUTOMATION_APP` (`lib/automation_identity.py`) read the same,
  now-derived value.
- **The copies are checked with the human's token.** A human-run check
  compares each listed org's variables with the file and flags any drift,
  and any repository-level variable of either name (which would override
  the org copy in that repo's jobs). `hoshin start` runs it each morning
  with the outsider report (§6.5). An Actions job trusts the values it was
  handed between checks; the account-type rule (§5.1) still applies in the
  job. Nothing automated needs new permissions.
- **Dev containers:** not needed; tests use fixtures, never live reads.

Today the daily sweep skips its trusted-comment checks under `--close`
because runners have no list; with the delivered copy, they run there too.

### 5.2 Trusted content, by kind

- **Issue or PR body:** its author and **every** editor in its edit history
  are trusted. (This is #1440's identity rule, generalised; #1440 requires
  the `human` login for the release tracking issue, which stays stricter.)
- **Comment, review, or inline review comment:** its author and every
  editor are trusted.
- **Close or reopen:** the actor of the event is trusted. An issue closed
  by an untrusted actor is not "closed" for any decision the automation
  makes on it (rollup, release gates, drift).
- **Issue adopted by title or marker:** its body is trusted (5.2, first
  point). A title or marker alone never identifies an issue.
- **PR found by branch name:** the PR is same-repo (not cross-repository,
  and its head repository is the base repository) and its author is
  trusted. A fork PR never matches a branch lookup.
- **Closing reference:** #1440's trusted closing reference.
- **Structural lines in a body** (`Parent:`, `Blocked-by:`, `Ref` /
  `Closes`, `Security advisory:`, release markers, fix-task links): count
  only from a trusted body.
- **Markers in comments** (planning markers, deferral notes, `Outcome:`,
  `Proxy:`, `Standalone-confirmed`, the latest-note summary, dedup checks):
  count only from trusted comments.
- **Check runs** are matched by their app or workflow, not by name alone,
  and no lookup falls back to "the first PR" of a commit.
- **Actor roles** (PR provenance) come from `vrg-trusted-authors`: a login
  on neither list is **untrusted**, never "human".

### 5.3 Two outcomes

- **Stop** only when the thing the action relies on is itself untrusted and
  no trusted version of it exists: the body of an issue or PR being worked,
  implemented, run or deployed; a trusted body an untrusted identity has
  edited; an outsider-opened issue or PR the action targets; an untrusted
  close or reopen the tool would act on. The tool or skill refuses before
  any write on that item, with an error naming the identity, what it did
  and where. It never falls back to other content or defaults. The human
  decides next.
- **Report** everything else. An untrusted comment, even one carrying a
  well-formed marker (`Outcome: SUCCESS`, `Blocked-by:`, a planning marker
  set, a deferral note), is filtered out; the trusted state is intact, so
  the tool carries on and says what it skipped on stderr (a skill, in the
  terminal). So no outsider comment can halt a tool. A fork PR that shares a
  branch name with a same-repo PR is skipped and reported the same way.
- **Batch runs** (the sweep, the audit, the queue views, the rollup): a
  stop applies to its item only. The run skips that item, carries on with
  the rest, and exits non-zero at the end, listing every stopped item.

**Metadata only.** Stop errors and skip reports carry metadata and links
only: repo, number, kind (issue, PR, comment, review, edit, event), login,
time and the field concerned. They never print untrusted text, so none of
it enters an agent's context through a tool's output. Under GitHub Actions,
whose logs are public, a batch run prints counts by cause: stops caused by
untrusted content or identities are counted only, and the human-run
outsider report (§6.5) lists them; stops caused by a read failure (an API
error, a missing field, an incomplete page) involve no outsider and are
listed with their repo and number, so they can be diagnosed.

**Outsider-opened issues and PRs.** One that a run merely encounters
(orphan detection, listings, the sweep) is reported, with a clean exit;
the outsider report shows it to the human. It is a stop only when a tool
is pointed at that item to act on it: intake routing, sizing,
implementing, validating, deploying, publishing. The daily sweep's orphan
attach never attaches one, and the activity log leaves it out.

### 5.4 Groundwork for #892

Every reader takes the roles it accepts as a parameter (§6.1), and reports
for each trusted item which role it came from, so #892 can require the
`human` role at chosen gates by changing a parameter rather than a parser.

## 6. The design

### 6.1 The trust layer (vergil-tooling)

One module (name the builder's call; it extends or absorbs
`lib/trusted_authors.py`) exposes readers for each kind in §5.2, each
returning only trusted items plus the list of skipped untrusted ones, and
raising a single error type for the stop outcome. It fetches the fields it
needs (author, edit history, event actors, cross-repository flags) itself,
so a caller cannot forget them; the layer owns the list of fields a trust
decision needs.

**Readers are parameterised.** Each reader takes the roles it accepts
(`human`, `agents`, `automation`) and the editor rule (author and every
editor, or author and last editor). The default is §5.2's: all three roles,
author and every editor. Callers that need a narrower rule pass it.

It reuses #1440's identity rule, trusted closing references and record
rule; where #1440 implemented them inside the release package, they move
into the layer and the release package calls them **with #1440's own
parameters**, so its behaviour is unchanged: the release tracking issue
accepts the `human` role only, closing references accept `human` and
`agents` only, and `Released:` records accept the `human` role for author
and last editor. The existing `comments.trusted()`,
`trusted_comment_bodies()`, `trusted_comment_lines()` and
`epic_size.trusted_logins()` become thin calls into it, or are replaced.

### 6.2 Every site moves onto it

The site inventory in the advisories, grouped by mechanism, is the work
list. Each site is fixed by moving it onto the layer and applying §5.3:

- **PR lookup by branch name** (GHSA-r4j2-cwcp-xg56): the branch lookups in
  the release, finalize, worktree, CI-evidence and deps-gate paths, and the
  tracking-issue resolver's fallback. The vergil-containers lifecycle
  publisher's GitHub handling (finding, refreshing and closing its
  `lifecycle/*` PRs, and syncing its labelled issues) moves into
  vergil-tooling, and the containers workflow calls that command;
  vergil-containers keeps only its build logic. The same-repo filter in
  `mechanized_pr` is extracted as a shared helper for it; lifecycle PRs keep
  their own behaviour (one PR per change, refreshed in place, merged by the
  human) and are never auto-merged.
- **State parsed from comments** (GHSA-hrq4-7qg2-7xxf): the queue views
  (planning markers, deferral notes, latest notes), intake routing's
  planning-epic guard, the operational-outcome audit and the security-record
  comment dedup.
- **Structural lines in bodies** (GHSA-w5hq-5p7v-qgc8): the epic audit and
  sweep (body parents, nativising, drift closes from merged PRs, epic
  drift), the epic rollup and its fallbacks, intake routing and the
  security-record paths (`show` prints only advisories that track the
  issue), the blocked-by reader, the open-task check and the release
  marker test.
- **Adoption by title and acting on closes** (GHSA-777j-j57r-92ff): the
  dependency-update fix task (adopted only when trusted, acted on only when
  closed by a trusted actor), the release-start gate's view of open
  `release:` issues (an untrusted one is reported, never a blocker), and
  the rollup's check of who closed an issue.
- **Reviews and identities** (GHSA-q6hx-8c85-9f56): the inline
  review-comment reader that `pr-watch` acts on, PR provenance roles, and
  the CI-evidence PR choice.
- **Skills acting on bodies and comments** (GHSA-crcw-h2ff-qgr5):
  `issue-implement` (body editors and resume comments), `issue-validate`
  and `issue-deploy` (the body's commands and steps), `epic-implement`'s
  dispatch, `epic-review`, `hoshin`'s rulings read and
  `deprecation-triage`'s dedup, all through §6.4's command.
- **The remaining items** of the umbrella finding (GHSA-jj85-qr28-7jwh) and
  any site the guard test (§6.3) turns up.

Smaller items in the inventory (the relay-ref payload author, the submit
linkage's body fallback, the CI full-matrix choice on a fork's `release/*`
head) are in scope too. Sites already fixed, or removed since the audit,
are confirmed in the plan, not re-done.

**Outside vergil-tooling.** The first implementation task inventories every
script and workflow step outside vergil-tooling that reads GitHub content
(the `gh` calls in vergil-actions workflow steps, scripts in the other
repos), and confirms each reads nothing behaviour-driving. Any that does
moves into vergil-tooling the same way as the lifecycle publisher.

### 6.3 The guard test (vergil-tooling)

A test in vergil-tooling fails when code outside the trust layer reads
behaviour-driving GitHub content: `gh`/`vrg-gh` calls or API paths that
fetch issue or PR bodies, comments, reviews, edit history or event actors,
or PR lookups by head branch. Reads of non-content fields (state, labels,
number, title for display) stay allowed. An explicit, commented exemption
list covers reads that are display-only; adding to it is a reviewed change.
How the test detects reads (AST scan, call-site registry, or both) is the
builder's call.

### 6.4 A trusted read command for skills, with a hook backstop

- **The command** (name the builder's call, e.g. `vrg-issue-read`) prints an
  issue's or PR's trusted body, comments, reviews and inline review
  comments, as text and JSON. It exits non-zero with the stop error when the
  body is untrusted, and lists every skipped untrusted item on stderr.
  Skills that read issue or PR content use it instead of raw
  `vrg-gh issue view --comments` or body reads.
- **The hook backstop is an allowlist.** For agents, the hook guard allows
  only named display-only `vrg-gh` read forms and fields (such as state,
  labels, number, title, URL, parent and timestamps). Every other `vrg-gh`
  read of issues, PRs, search results or GraphQL is refused, naming the
  trusted command. A new read form is refused until it is added to the
  allowlist, and each addition is a reviewed change with a test.
- No skill shows untrusted text to the human or the agent; the outsider
  report (§6.5) carries metadata and links only.

### 6.5 The outsider report and the morning review

- A read-only report command (placement the builder's call, e.g. a mode of
  `vrg-epic-audit`) lists, across the repos of every listed org (§5.1a),
  newest first: issues and
  PRs opened by untrusted accounts, untrusted comments and reviews,
  untrusted edits to trusted bodies, and untrusted closes or reopens. It
  takes a `--since` time. It writes nothing on GitHub.
- `hoshin start` runs it, per org, with the time of the last daily review
  and shows the human what is new, together with the configuration check of
  §5.1a. Untrusted edits to trusted bodies and untrusted
  closes are called out first, as possible tampering. The report carries
  metadata and links only (repo, number, kind, login, time), never the
  outsider's text, so none of it enters an agent's context as content.
- The public logbook records only that the outsider check ran, with no
  count and no detail. The count and the listing stay in the terminal.

### 6.6 Tests

Every migrated site gets tests built from synthetic fixtures: ones that
copy the shape of the spam comments seen on vergil-project issues
(structure, marker placement, link patterns) with neutralised URLs, never
their text, and deliberately forged input: a forged `Blocked-by:`,
`Parent:`, planning marker set,
`Outcome: SUCCESS`, deferral note and `Closes` line; a trusted body later
edited by an outsider; a fork PR on a `release/*`, `release/post-*`,
`lifecycle/*` and task branch; an outsider closing a tracked issue; an
issue titled like a fix task or a release by an outsider; a User account
whose login matches an App role; a deleted account as author, editor or
actor; a repository variable shadowing a delivered org variable. Each test
checks the stop or the report, not just that the content is ignored, and
that its output carries no untrusted text. Fixtures copy the field shapes
of real recorded GitHub responses (how each role's author, account type,
edit history, events and deleted accounts actually come back), so a test
cannot pass on a shape GitHub never returns.

### 6.7 Validation

A validation task, after the vergil-tooling and vergil-claude-plugin
releases carrying the epic are installed, checks the layer against **real
data** in the listed orgs, with no scratch repo and no test account. Its
purpose is to prove that live GitHub responses match what the fixtures
assume:

- the trusted read command and the outsider report, run on real issues,
  report the untrusted comments already present and never act on them, and
  pass the content of all three roles (the human, the agents, the
  automation App);
- the automation App's own refresh, lifecycle and archive work runs clean;
- a scheduled sweep under Actions reads the delivered copies and runs its
  trusted checks;
- the morning review shows the outsider listing and the configuration
  check.

The forged cases (fork PRs, forged markers, edits and closes by outsiders,
deleted accounts) are covered by the §6.6 fixtures only.

## 7. Left to the builder

- Module, command and report names and placement.
- The guard test's detection technique and the exemption list's format.
- How the layer batches API calls (edit history and event actors cost
  extra queries); correctness first, then cost.
- The task split, within the plan's sizing rule, and the sequencing against
  the vergil-tooling and vergil-claude-plugin releases. The pieces depend
  on each other, so the order is fixed in outline: the layer and the list
  delivery first, then the sites by group, then the guard test and the hook,
  then the outsider report, then validation.

## 8. Follow-on

vergil-project/.github#892 (decisions posted by the human's own account) is
planned immediately after this epic's docs PR and implemented once this epic
closes, on top of §5.4.
