# standalone-adhoc: proxy and hoshin epics keep ad-hoc queues standalone — design

- **Epic:** vergil-project/.github#1417
- **Planning issue:** vergil-project/.github#1146 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

An ad-hoc epic (`Epic (ad hoc): <repo>`) was meant to hold one thing:
standalone, single-issue work that can be picked up, merged and released on
its own, independent of any other work. Two other kinds of task now live in
ad-hoc epics:

1. **Tasks of another org's epic.** GitHub cannot link a sub-issue across
   orgs, so a task an epic in one org needs in another org's repo is filed in
   that repo's ad-hoc epic, and its membership is recorded only as prose.
   vergil-project/vergil-tooling#3188 and #3189, tasks of
   logical-minds-foundry/.github#319, are the evidence: `vrg-queue adhoc`
   reported both `ready`, and only a human reading every body held them back.
2. **Tasks owned by a controlling skill.** `/vergil:hoshin` files one
   `Logbook: <date> <mode>` task per run in the docs repo's ad-hoc epic.
   Working one through `epic-implement adhoc` would write a merge-reviewed PR
   outside hoshin's own security gate and fixed wording. Today a title check
   in `epic-implement`'s adhoc mode skips them, which is a workaround.

Nothing in the tooling tells either kind from standalone work. Ad-hoc work
now runs at scale and is headed for automatic processing, which removes the
human reading that caught the first case.

**The why:** make the ad-hoc epic mean what it was meant to mean, so that
being placed under one is itself the proof a task is safe to work alone. Any
queue processor, human-confirmed or automatic, can then trust the placement
instead of reading every body.

## 2. Goals, non-goals, anti-goals

### 2.1 Goals

- Every task the sanctioned tools place under an ad-hoc epic is standalone
  work. Tasks are filed and moved only through the tooling; a hand-filed task
  is outside the process, and the audit backstop (§4.8) reports one that is
  misplaced.
- A cross-org epic's tasks in another org are native sub-issues of an epic in
  that org, linked machine-readably to the epic that owns the work.
- The owning epic's driver works those tasks in its plan's order, and the
  owning epic cannot finish while any of them is open.
- Hoshin's logbook tasks live under an epic of their own that no queue
  processor works.
- Every existing misplaced task is re-homed.

### 2.2 Non-goals

- A general framework for skill-owned epics. The hoshin epic is specific to
  hoshin until a second skill needs the same thing.
- Proxies within one org: native sub-issues already work there.
- Linking across orgs in GitHub itself.
- Building automatic ad-hoc processing. This epic makes it safe; it does not
  build it.

### 2.3 Anti-goals

A failure even if it technically works:

- A marker that must be remembered on every hand-filed task. A task filed
  without it would be dispatched; that fails open.
- A proxy that can be driven on its own, so the owning epic's frontier and
  plan-evolution record drift.
- An owning epic that rolls up, or a retrospective that runs, while a proxy
  task is still open.

## 3. Approaches

- **A. Mark it.** Owned tasks stay in ad-hoc epics with a stamped line (such
  as `Owned-by: <epic ref | skill>`), and `vrg-queue adhoc` gives them a state
  that is never `ready`. Cheap, but it fails open: an unmarked, hand-filed
  task is dispatched.
- **B. Place it (chosen).** Owned tasks never sit in an ad-hoc epic. A
  cross-org task goes under a same-org **proxy epic** for the owning epic, and
  hoshin's tasks go under a **hoshin epic**. Placement under the ad-hoc epic
  is then the proof of standalone work for every task the tooling files. It
  fails closed on the sanctioned paths and removes the
  overlap between what ad-hoc and other epics hold, at the cost of new epic
  kinds and a migration.
- **C. Both.** Placement as the rule, plus a marker check as a backstop.
  Rejected: once placement is the rule, the marker adds a second thing to keep
  consistent and catches nothing placement does not.

The human chose B.

## 4. Design

### 4.1 Vocabulary

- **Primary epic:** the one development epic that owns a cross-org effort,
  homed in the org the work is mainly for. It holds the spec, the plan and the
  retrospective, exactly as any development epic does today. Which org it
  lives in is a judgment of where the work mainly is; nothing depends on it.
- **Proxy epic:** an epic in another org's epic home that stands in for the
  primary there. It holds that org's tasks of the effort as native sub-issues
  and nothing else: no spec, no plan, no retrospective, no bookends.

### 4.2 The proxy epic

- **Home.** The epic home of the repos its tasks land in, resolved by
  `epics.resolve_epic_home` as today: `<owner>/.github` for a public repo, the
  repo itself for a self-homed private one. One proxy per primary per home.
- **Shape.** Title `Epic (proxy): <primary epic title without its "Epic: "
  prefix>`; labels `epic` + `proxy`; body carrying a `Primary:
  <owner>/<repo>#<N>` line (one, fully qualified), a one-line explanation, and
  nothing a reader needs to act on: the work is described in the primary.
- **The links run both ways.** The primary's body carries one `Proxy:
  <owner>/<repo>#<N>` line per proxy, replacing today's hand-written
  "Cross-org dependencies" prose. Both line kinds are machine-read and alone
  on their line, like `Planning:`.
- **Creation is sanctioned.** `vrg-epic-create --proxy-for <primary ref>
  --repo <owner>/<repo>` creates the proxy in the target's home, derives its
  title, stamps the `Primary:` line and appends the `Proxy:` line to the
  primary's body. It refuses:
  - a primary in the same org as the target (no proxy is needed);
  - a primary that is not an open development epic (a standing epic, a proxy,
    a closed epic);
  - a second proxy for the same primary in the same home (it prints the
    existing one);
  - a primary in a private repo whose proxy home is public (§5).
- **It is not a development epic.** One predicate, `epics.is_proxy`, keyed on
  the `proxy` label, is used by every check that applies only to development
  epics: the Documents-section audit (`epic_docs.is_skipped`) skips a proxy,
  the roadmap does not list it as an epic of its own, and the audit checks
  its `Primary:`/`Proxy:` pair instead (§4.3).
- **Tasks are filed under it** with the existing `vrg-issue-create --epic
  <proxy ref>` and moved with `vrg-epic-move`. Each task body links the
  primary's plan and spec, as #3188 does today, so its implementer and merge
  reviewer judge it against the primary's design.
- **It lives as long as its primary.** A proxy is exempt from the rollup that
  closes a finite epic when its last task closes, so it stays open, and work
  can always be filed under it, until the primary closes. That includes the
  per-repo doc tasks of the primary's documentation review (§4.4). It needs
  no bookends because the primary carries them.
- **Its own org closes it.** The daily epic sweep of the proxy's home
  (`vrg-epic-audit --close`) closes a proxy whose primary is closed and which
  has no open sub-issue. A proxy that still has an open sub-issue after its
  primary closed is reported as drift and left open, never closed silently.
  The primary's own rollup runs in the primary's org, whose token cannot be
  assumed to write to another org, so it does not close proxies.
  *(Builder's call: whether a session that closes the primary with a token
  valid in both orgs also closes its proxies at once, as a fast path; the
  sweep stays the mechanism either way.)*

### 4.3 Driving cross-org work

- **Only the primary's `epic-implement` drives proxy tasks.** The primary's
  plan names each proxy task by its full ref, as it already does for its own
  tasks. `epic-implement` rebuilds its frontier from the primary's sub-issues
  **and** the sub-issues of every proxy its `Proxy:` lines name, reconciled
  against the plan as today, and dispatches them through the same skills and
  gates. Plan evolution for proxy tasks is recorded in the primary's plan.
- **`epic-implement` on a proxy refuses** with
  `<ref> is a proxy for <primary ref>; drive the primary epic instead.`
- **`issue-implement` on a proxy task** works as for any finite-epic task,
  except that its context comes from the primary's spec and plan, which the
  task body links. *(Builder's call: whether `issue-implement` resolves the
  primary from the proxy's `Primary:` line itself or relies on the body's
  links; either is acceptable if it is not silent when neither is present.)*
- **`vrg-queue adhoc` never lists a proxy task**, by construction: it lists
  only children of ad-hoc epics. No change is needed there for this case.
- **`vrg-epic-audit`** on a primary includes each proxy's tasks, and checks
  the links in both directions: every `Proxy:` line names an existing proxy
  whose `Primary:` line names this epic, and the reverse. A disagreement is
  reported as drift, never fixed silently. On a proxy, the audit reports the
  primary it belongs to and checks the same pair.

### 4.4 Closing a cross-org epic

- The primary cannot finish while any proxy task is open.
  `epic-retrospective`'s preflight, which today refuses unless the
  retrospective is the primary's only open child, also refuses while any
  epic named by a `Proxy:` line has an open sub-issue, naming each one. The
  proxies themselves are still open at that point, by design (§4.2).
- Because the primary rolls up only when its retrospective closes, this gates
  rollup too. The primary's rollup is unchanged.
- The documentation-review bookend's sweep covers the repos the proxy tasks
  touched, spawning per-repo doc tasks under the proxy of that org (the
  placement law: same-repo close, same-org parent). The proxy is still open
  then, so no reopen is needed.

### 4.5 The hoshin epic

- **A new standing kind**, `HOSHIN`, beside the ad-hoc, intake, planning and
  deferred kinds in `epics.STANDING_KINDS`: title `Epic (hoshin): logbook`,
  labels `epic` + `hoshin`, one per org epic home, created and reused
  idempotently, its closed children draining into per-quarter
  `Archive (hoshin): logbook` archives like the other standing kinds. Its
  `--epic` sentinel is `hoshin`.
- **It is not a queue kind.** Its tasks are PR-workable: hoshin hands them off
  with `vrg-pr-workflow report-ready` as it does today.
- **The daily sweep drains it.** Closed children drain on the event path
  (rollup) like every standing kind, and the sweep's backstop drain
  (`vrg-epic-audit --close`) covers every standing kind other than ad-hoc
  (which has its own drain), not only the queue kinds, so the hoshin epic
  never accumulates closed children toward GitHub's sub-issue limit.
- **Only `/vergil:hoshin` files and works its tasks.** Hoshin files its
  `Logbook:` task with `vrg-issue-create --epic hoshin --repo
  vergil-project/docs`, and its stale-task check lists the hoshin epic's open
  tasks instead of the docs repo's ad-hoc tasks. The listing comes from a
  `vrg-queue hoshin` view with the same fields, in-flight marks and
  disposition as `vrg-queue adhoc`, sharing its code.
- **Refusals.** `epic-implement` refuses the hoshin epic (as it refuses a
  queue epic), and `issue-implement` refuses a task under it, naming
  `/vergil:hoshin`. *(My call: the `issue-implement` refusal makes the
  ownership a rule rather than a convention; hoshin never calls
  `issue-implement`.)*
- **The workaround is retired.** The `Logbook:` title skip in
  `epic-implement`'s adhoc mode is removed once the hoshin epic is in use and
  no logbook task remains in an ad-hoc epic.

### 4.6 The convention in `epic-create`

`epic-create`'s note "Cross-org is out of scope: never link epics or tasks
across orgs" becomes the proxy convention: a cross-org effort has one primary
epic; when the plan files a task in another org, `epic-create` (step 8)
creates that org's proxy with `vrg-epic-create --proxy-for` and files the task
under it. The plan's delivery table lists proxy tasks with their full refs.
Fixed-epic linking across orgs stays forbidden; the proxy is the sanctioned
form.

The ad-hoc epic's own description (its `StandingKind` body and the docs that
define it) states the rule this epic establishes: an ad-hoc epic holds only
standalone, single-issue work, and a task owned by any epic or skill never
goes there.

### 4.7 Migration

- **The sweep.** List every open task under every ad-hoc and deferred epic in
  each org Vergil manages, and pick out the misplaced ones: any whose body or
  comments tie it to an epic in another org, and any titled `Logbook:`. The
  candidate list is confirmed by the human before anything moves.
- **The known case.** Create the proxy for logical-minds-foundry/.github#319
  in `vergil-project/.github`, move #3188 and #3189 under it (each stays
  blocked as it is; deferral reasons are carried into a comment), and replace
  #319's "Cross-org dependencies" section with `Proxy:` lines.
- **Logbook tasks** still open in the docs repo's ad-hoc epic move under the
  hoshin epic.
- **Validation.** After the moves, a validation task confirms that no ad-hoc
  epic in any swept org holds a cross-org or `Logbook:` task, and that
  `vrg-epic-audit` is clean on every primary and proxy created.

### 4.8 No marker; an audit backstop

No `Owned-by:` or `Remote epic:` marker is added, and no filing-time guard:
a guard must rest on deterministic metadata, never on matching text, and
placement through the tooling is the rule. Tasks are filed and moved only
through the tooling, by the human and agents alike.

The backstop is a report, not a gate. `vrg-epic-audit` reports any open task
under an ad-hoc epic that looks misplaced (a `Logbook:` title, or a body that
names an epic in another org) for a human to rule on; it never refuses,
moves or closes anything on that basis. `epic-implement` adhoc mode's
human-confirmed strategy step keeps reading task bodies, as it already must
to judge overlap.

## 5. Constraints

- **Cross-org access.** Driving a primary in one org dispatches work in
  another. The agent identity that runs `epic-implement` must be able to read
  both orgs' issues and act on the proxy's repos, and `vrg-epic-create
  --proxy-for` writes to the primary's body in its org. The plan verifies the
  identities and tokens this needs and refuses, never degrades, when one is
  missing.
- **Fail loudly.** A `Primary:` or `Proxy:` line that cannot be parsed or
  resolved, a proxy whose primary does not name it, or an unreadable proxy is
  an error in every tool that reads them, never a skipped item.
- **Self-homed private repos.** A proxy for a private repo lives in that repo
  under the existing home rule. A primary epic in a private repo may not have
  a proxy in a public home: `vrg-epic-create --proxy-for` refuses the pair.
  The reverse is allowed: a public primary may have a proxy in a private
  repo, and its `Proxy:` line then names the private repo and issue number,
  and nothing of the private work.
- **The worktree, wrapper and validation conventions** of each repo apply as
  usual.

## 6. Left to the builder

- The exact argument and output shape of `vrg-epic-create --proxy-for` and
  `vrg-queue hoshin`, within §4.
- The two choices marked *Builder's call* in §4.2 and §4.3.
- Which orgs the sweep covers, taken from the fleet's org list, and how the
  candidate list is produced (a one-off script or a `vrg-queue` option).
- Task split and order, in the plan, with one constraint: the hoshin epic
  (§4.5) and its migration are an early task group that depends on no proxy
  task, so it can be released on its own and retires the `Logbook:` title
  skip first.
