# Sub-issue capacity: chained epics past GitHub's 100-child limit

- **Epic:** [vergil-project/.github#1609](https://github.com/vergil-project/.github/issues/1609)
- **Planning issue:** [vergil-project/vergil-tooling#3840](https://github.com/vergil-project/vergil-tooling/issues/3840)
  (review history is in its comments)
- **Status:** draft

## 1. Intent

GitHub caps a parent issue at **100 sub-issues**, and it allows sub-issues
to be nested at most **eight levels** deep
([GitHub Docs: Adding sub-issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues)).
The epic model links every task, queue item and archived item as a native
sub-issue, and the tooling treats the cap as an unexpected failure. Two cases
show the cost:

- **A queue epic.** The `.github` intake epic, vergil-project/.github#438,
  reached the cap. New intake issues were created but left unlinked, outside
  the queue, with GitHub's raw error naming no limit; vergil-project/.github#707
  was stranded that way until an intake review drained the queue.
- **A development epic.** vergil-project/.github#1384 reached 97 of 100 (58
  closed, 39 open) while its plan still needed 14 tasks. They were filed under
  each repo's ad-hoc epic instead, where `vrg-epic-audit --epic`, the epic
  rollup and `epic-retrospective` do not see them, so the epic driver tracks
  them by hand.

This epic makes an epic a **chain of containers** that grows past the cap on
its own, behind one abstraction, so no caller ever meets the cap and no item
lands outside its epic.

**A chain is a red flag, not a feature.** An epic that needs a second
container has probably grown too big and should have been split. The design
tolerates a chain (it never breaks a filing session halfway through) and keeps
it visible (§4.6), but does not optimise for long ones. An epic past about 200
items is expected to be rare; one in the hundreds is an engineering failure,
not a case to design for.

## 2. Goals, non-goals and anti-goals

### Goals

1. Linking an item under any epic (development, proxy, ad-hoc, every standing
   queue) never fails because a container is full: the epic rolls over into a
   continuation container.
2. Every reader of an epic's children sees one flat list across the whole
   chain, with the continuation containers themselves hidden.
3. A task's epic is its chain's **head**, the epic known by number, wherever in
   the chain the task is linked.
4. Concurrent writers cannot break a chain: a lost race heals itself.
5. A development epic's rollover is loud and stays visible until the epic
   closes. A queue's is reported as the existing nudge to review the queue.
6. A slot freed anywhere in a chain is reused before the chain grows, so an
   epic has only as many containers as its peak number of items held at once
   needs.
7. The 14 tasks of vergil-project/.github#1384 filed under ad-hoc epics are
   re-homed under #1384's chain, if #1384 is still open (§4.8).
8. Closed is closed: no tooling path reopens a closed epic, and linking or
   moving an item into one is refused by name (§4.2).

### Non-goals

- **Archives.** Quarterly archives already roll over into numbered segments
  (`ensure_writable_standing_archive`, segment cap 95). They are append-only
  buckets that nothing reads as one list, and their scheme works; this epic
  leaves it as it is.
- **Unbounded size.** The chain is linear, so the depth limit bounds it at
  roughly seven or eight containers (about 700 items). Past that the tooling
  refuses (§4.5); nothing lifts the bound.
- **Shrinking a chain.** A continuation that drains empty stays linked and is
  reused by later links (§4.3). Empty continuations are harmless; collecting
  them is left for later, if it is ever needed.
- **Cross-org overflow.** Proxy epics stay the only cross-org mechanism. A
  proxy can itself have a chain, like any other epic.
- **Changing the queue-health signal.** How full a queue is allowed to get
  before review is a separate question from the GitHub cap.

### Anti-goals

Failures even if the mechanism works:

- An item created but left outside its epic, as #707 was.
- A caller that reads a raw container and sees half an epic: an audit, rollup
  or retrospective that misses tasks in a continuation.
- A rollup that closes an epic while a task in a continuation is still open.
- A silent rollover of a development epic.
- A refusal that reports GitHub's raw error instead of naming the limit and
  the way out.

## 3. Approaches

Three ways of building the chain were weighed:

1. **Native nested chain behind one chain-aware layer (chosen).** The
   continuation is a real sub-issue, in the last slot of the container before
   it, carrying a `continuation` label and a `Continues:` line naming the head.
   One module owns walking and growing the chain; every caller goes through it.
   It keeps the hierarchy native, so GitHub's own UI and progress bars show it,
   and a task's native parent leads to its head. It costs the depth limit,
   accepted above, and it touches many callers.
2. **Comment-linked chain, as proxies are linked.** Continuations are tied to
   the head by trusted comments rather than as sub-issues, so every container
   keeps all 100 slots and there is no depth limit, and the audit's proxy walk
   is a template. But GitHub's UI shows no link between containers, and every
   walk up to the head depends on parsing comments. Rejected: the native link
   is the simpler, visible model, and the depth limit does not bind.
3. **Same-org proxies.** Lift `--proxy-for`'s same-org refusal and treat a
   continuation as a proxy. Least new code, since `epic_children` already
   merges proxies in, but a proxy is never rolled up, closes only after its
   primary, does not exist for queues and switches org tokens. Rejected:
   bending it would blur both concepts.

## 4. Design

### 4.1 The chain

- An epic's **chain** is its **head** (the epic as created, the ref everything
  names) followed by zero or more **continuations**, in order.
- Each continuation is linked as a native sub-issue of the container before it
  and occupies that container's **last slot**. A container therefore holds at
  most **99 items** and, when it is not the tail, exactly one continuation.
- A continuation is an issue in the head's repo, labelled `epic` and
  `continuation`, titled `<head title> (continuation <k>)` with `k` from 2 (the
  head is container 1). Its body carries, each alone on its line:

  ```text
  Continues: <owner>/<repo>#<head>
  Container: <k>
  ```

  and a sentence saying it holds the overflow of the head and is managed by the
  tooling. The `Container:` line and the title's number are **for display
  only**: a container's position is the native chain's, and no reader checks
  the number, so a stale one is never drift.
- A continuation carries **none** of the head's kind labels (`intake`,
  `planning`, `deferred`, `ad-hoc`, `proxy`, `hoshin`, `operations`,
  `epic-development`, …). Every lookup that finds a canonical epic by label or
  title (`find_standing_epic`, `ensure_standing_epic`, `queue_kind`,
  `standing_kind_of`, `roadmap.gather`, the planning queue view) must
  therefore never find a continuation, and a continuation's kind is always its
  head's.
- **Every consumer of `epic`-labelled listings skips continuations.** One
  predicate, `is_continuation(labels)`, recognises one, and every check that
  skips standing or proxy epics today (by `is_standing` or `is_proxy`) skips
  continuations too. These include the consumers of `list_home_epics`:
  `epic_docs.is_skipped` and the audit's `epic_documents`,
  `closed_epic_open_child`, `proxy_link_drift` and
  `misplaced_adhoc`. A continuation is never reported as a development epic
  missing its Documents section, a closed epic with an open child, or a
  primary epic.
- **A continuation's ref means its head.** GitHub's UI shows a continuation as
  a task's parent, so its ref will be copied into commands. Every command that
  accepts an epic ref (`--epic`, `--parent` and the like) resolves it through
  the shared ref resolution (`resolve_epic_ref`, `qualify_epic_ref`), which
  maps a continuation to its head with `head_of` and prints a line saying so.
  Every kind check then runs against the head.
- **Trust.** A container is accepted as a continuation only when it carries
  the label, its body passes the trust layer, its `Continues:` line names the
  head being walked, and it is a native child of the container before it. Any mismatch is **chain drift**: a
  reader raises it, naming the container and the rule it broke, and the audit
  reports it (§4.7). A reader never guesses past a broken link.

### 4.2 The chain layer

A new module, `lib/epic_chain.py`, owns the chain. Its public surface (names
are the builder's to refine):

- `chain_of(head) -> list[IssueRef]`: the head then each continuation.
- `head_of(container) -> IssueRef`: the head of any container (itself, for a
  head). It walks native parents while the container is a continuation, and
  checks the `Continues:` line agrees. An **orphaned continuation** (§4.3),
  which has no native parent, resolves through its trusted `Continues:` line
  instead, and the call reports it as drift. `parent_of` and the rollup adopt
  such an orphan (find-or-create, §4.3) before they answer or judge.
- `chain_child_states(head)`: the flattened children across the chain, in
  container order, continuations excluded.
- `link_into(head, task)`: link *task* into the first container with a free
  item slot, rolling over first when none has one (§4.3, §4.4). It refuses a
  closed head, naming it, and never reopens one (closed is closed, below).
- `move_into(head, task)`: atomically move *task* from wherever it is into the
  first container with a free item slot (`replaceParent`, as `reparent_child`
  does today), with the same rollover. Like `link_into`, it refuses a closed
  head, naming it, and never reopens one.
- `close_chain(head)`: close the head first, then every continuation (§4.6).
- **A closed continuation in an open chain** (closed by hand, say) is not
  drift and never blocks the chain. Readers still walk it and flatten its
  items, and writers skip it for first fit and never link an item into it.
  When it has no successor and every open container is full, rollover links
  the next continuation into the closed continuation itself: the one link a
  closed container takes, which leaves it closed. If GitHub refuses a link
  under a closed parent, the writer instead refuses by name, naming the
  closed continuation and saying a person may reopen or unlink it, and
  create-then-link commands refuse before creating. The audit reports it as
  a note, naming it, and nothing reopens it (closed is closed, below).
- `unlink_from(head, task)`: unlink *task* from whichever container of the
  chain it is linked in.
- `chain_contains(head, task)`: whether *task* is linked anywhere in the chain.
- **Settling after a transfer.** GitHub's parent field and its sub-issue
  lists can disagree for a while after an issue is transferred (the cases
  `intake_route` recovers from today: vergil-project/vergil-tooling#3839,
  #4004 and #4291). The chain layer owns that recovery as container-aware
  operations: observe a task's parent field against the listings of the
  chain's containers, wait for them to settle, and clear a stale listing of
  the task from the container that holds it. `intake_route`'s
  `_observe`, `_settle_parent` and `_clear_duplicate` call these, and do no
  raw read or write of their own.

**The chain layer owns writes as well as reads.** The raw writers
(`add_child`, `remove_child`, `reparent_child`) become private to `epics` and
`epic_chain`; every caller outside them links, moves and unlinks only through
`link_into`, `move_into` and `unlink_from`. A raw writer paired with a
chain-aware reader would act on the head for an item that sits in a
continuation, so no public raw writer remains.

**Closed is closed.** No tooling path reopens a closed epic. A development
epic closes when its work is finished and is never written to again; a
standing epic stays open for the life of its repository. A closed target is
therefore an error, never a state to repair by reopening:

- `link_into` and `move_into` refuse a closed head with a message naming the
  epic and saying it is closed. Every flow built on them refuses the same way:
  `vrg-issue-create`, `vrg-intake-create`, `vrg-epic-link`,
  `vrg-security-record`, `vrg-epic-move` and `intake_route`'s links and
  re-parents.
- The reopen-on-late-child rule in `add_child` and `intake_route`'s explicit
  reopen before a re-parent are removed.
- `proxies.create_proxy` (`vrg-epic-create --proxy-for`) no longer reopens a
  closed proxy: it refuses, naming the proxy and saying it is closed. A
  person may reopen it by hand, and a re-run then reuses it.
- A **missing** standing epic is still created on demand
  (`ensure_standing_epic`, for example in a new repository); a **closed** one
  is refused, never reopened or replaced. The lookup reads closed issues as
  well as open ones: an open epic with the canonical title is reused whatever
  closed ones share it, and only when no open one exists and a closed one
  does is the call refused, naming it and saying a person may reopen it by
  hand.
- The audit's `closed_epic_open_child` check stays, but its reopen fix
  (`reopen_epics_with_open_children`) is removed: a closed epic with an open
  child is reported as drift for the human, and the sweep never reopens it.

**The existing public readers become chain-aware.** `child_states`,
`child_states_with_blockers`, `native_children`, `all_children_closed`,
`parent_of` and `native_parent_of` (as seen by callers outside the chain
layer) answer for the chain: children of a head are the chain's, and a task's
parent is its head. The raw, per-container reads stay private to `epics` and
`epic_chain`. The callers that read or write children today, all of which move
to the chain-aware form, are:

- linking, moving and unlinking (every caller of `add_child`, `remove_child`
  or `reparent_child` today): `vrg-epic-link`, `vrg-issue-create`,
  `vrg-intake-create`, `vrg-security-record`, `vrg-epic-move`, `intake_route`
  (its links, re-parents, the unlink before a transfer and the stale-listing
  cleanup), `ops_report`, the epic sweep's orphan pass and its re-links
  (`epic_audit`);
- reading: `epic_audit` (including `epic_children` with its proxies),
  `queue_view`, `roadmap`, `planning_close`, `security_records`,
  `merge_review/request`, `vrg-submit-pr`, and `vrg-security-record`'s resume
  lookup (`_find_resumed_issue`, which reads the intake epic's sub-issues
  with a raw REST call today);
- the rollup and the standing drain (§4.6).

**No raw parent or child queries outside the chain layer.** Every GraphQL or
REST selection of an issue's `parent` or `subIssues` outside `epics` and
`epic_chain` moves behind the chain layer too, not only the callers of the
public readers above. That includes `epic_audit`'s `_OPEN_ISSUES_QUERY`,
whose `parent` field `misplaced_adhoc` keys on (it maps a native parent
through `head_of`), and `merge_review/request`'s own queries.

Whether a given caller wants a reader that also exposes containers (the audit,
for drift) is the builder's call; the default is the flattened, head-level
view.

### 4.3 Rollover

- **First fit.** A writer links an item into the first container, in chain
  order, that holds fewer than 99 items. Items are never in any order across
  containers, and nothing reads them in one: every reader sees the flattened
  list. A slot freed by a drain or a move is therefore reused before the chain
  grows.
- **The count is cheap.** A container's item count is its sub-issue total
  (GitHub's count, without listing the children), less one when it holds a
  continuation. GitHub counts **every** child, open and closed, so the count is
  of all children, not only open ones.
- **When:** a writer rolls over only when every container in the chain holds
  99 items.
- **How:** find-or-create the next continuation, then link it into the tail's
  last slot, then link the item into it. Find-or-create first looks for an
  existing continuation child of the tail (a concurrent writer's), then for an
  **orphaned continuation** of this head (below), and creates one only when it
  finds neither, so two rollovers do not normally make two.
- **Order matters for safety:** the item is linked only after its container is
  in place, so a writer that dies mid-rollover never leaves an unlinked item.
  It can leave an empty, correctly linked continuation (which later links fill,
  by first fit), or a continuation that was created but never linked.
- **Orphaned continuations.** A continuation that is open but cannot be
  reached from the head its `Continues:` line names (created by a writer that
  died before linking it, by a rollover or by a §4.4 heal, which may leave an
  item in it) is an orphan. Find-or-create adopts one by linking it into the
  tail's last slot, and the epic sweep does the same for any it finds (§4.7).
  An orphan is adopted only when it passes the §4.1 trust rules other than
  the native link: the label, a trusted body and a `Continues:` line naming
  this head. Once it is linked, its `Container:` line and title are updated
  to its place in the chain, for display (§4.1). One that fails a rule is reported as chain drift and never
  adopted.

### 4.4 Races

GitHub offers no lock, so a chain is made safe by reading back, not by
locking:

- **GitHub's limit error is "full".** A link that fails with GitHub's
  sub-issue limit error is never surfaced raw: the writer re-reads the tail,
  rolls over and retries, a bounded number of times. Only when the retries run
  out does it fail, naming the limit, the container and the item.
- **A full tail with no continuation** (two writers each saw 98 items in the
  tail and both linked, taking slot 100). Only the tail can reach this state:
  any other container already holds its continuation, so a racing link there
  would be a 101st child, which GitHub refuses (the bullet above). After every
  link into the tail the writer reads it back, and any writer, or the epic
  sweep, that finds the tail at 100 children with no continuation heals it,
  with no step that leaves an item outside a container:
  1. find-or-create the continuation (unlinked for the moment);
  2. move one item from the tail into it with an atomic re-parent
     (`replaceParent`), which frees a slot: the item this writer just linked,
     or, for a writer with no item of its own (the sub-issue list can lag
     behind writes, so the filler's read-back may have missed the state),
     the tail's newest item;
  3. link the continuation into the freed slot.
- **Two continuations under one container** (two writers both created one):
  the writer that finds it after its own link keeps the lower-numbered issue,
  moves any items from the other into it, unlinks the other, and only then
  closes it as not planned with a comment naming the kept one, so no closed
  duplicate stays in the chain or holds a slot. The audit reports any it finds (§4.7).
- The exact retry bound, and how each heal is reported, are the builder's
  call; every heal prints one line saying what it did.

### 4.5 The depth limit

The chain is linear, so container `k` sits at depth `k` below the head (plus
the head's own depth, if it is itself a sub-issue). When the next continuation
would exceed GitHub's nesting limit, `link_into` refuses **before** anything is
created or linked, naming the epic, its item count, its container count and
the depth limit, and saying the epic must be split. GitHub's own depth error,
if it comes first, is reworded the same way. The exact headroom (eight levels
counting the head or not) is confirmed by a live probe during implementation
and fixed as a constant.

**Create-then-link commands check first.** `vrg-issue-create`,
`vrg-intake-create` and `vrg-security-record` create an issue and then link
it. They run the chain layer's capacity check before creating, so the depth
refusal leaves nothing orphaned. Rollover itself never needs a pre-check: it
cannot fail for capacity short of the depth limit.

**`vrg-epic-move` stops orphaning.** It moves with `move_into` (one atomic
re-parent) instead of removing the current link and then adding the new one,
so a refused move leaves the task where it was.

### 4.6 Rollup and drain

- **Rollup.** `rollup(task)` resolves the task's head (`head_of(parent)`). A
  finite epic closes only when every item across its chain is closed and
  trusted-closed, as today; it then closes the chain with `close_chain`
  (below). **Before judging, it looks for orphans** of the head (§4.3, the
  search §4.7 defines): it adopts each that passes the trust rules and judges
  the whole chain with it, and it refuses to close while an orphan it cannot
  adopt exists, reporting it. An orphan of a head that is already closed is
  reported as drift for the human and never adopted (closed is closed, §4.2). A continuation closing never rolls anything up on its own: a close
  event whose issue is a continuation is a no-op for the rollup.
- **Every close closes the chain.** The chain layer's `close_chain(head)`
  closes the head first, then every continuation. Every tooling path
  that closes an epic goes through it: the rollup, the audit's `close_drift`
  and `close_proxies`. A close interrupted partway, or an epic closed any
  other way (by hand, say), leaves a **closed head with an open
  continuation**: the audit reports it as drift and the epic sweep heals it
  by closing those continuations. Closing them is not a reopen, so closed is
  closed (§4.2) still holds.
- **Standing drain.** `drain_standing_child` judges the **head**'s title and
  kind, not the continuation's, so a closed item drains to its archive from
  any container. The batch drain (`apply_standing_drain`) reads the whole
  chain.
- **No shrink.** Nothing removes a continuation that drains empty. It stays
  linked, and first fit (§4.3) reuses its slots once the containers before it
  are full again. Items are never moved between containers to compact a
  chain.

### 4.7 Visibility

- **A development epic's rollover is loud.** When `link_into` creates a
  continuation of an epic that is not a standing queue, the command prints a
  plain line, for example: `epic vergil-project/.github#1384 rolled over into
  continuation vergil-project/.github#N at 99 children; consider splitting
  it`. It also comments the same on the head, so the epic's record shows it.
- **It stays visible.** `vrg-epic-audit --epic` shows a chained epic's
  container count. The epic sweep reports every open development or proxy
  epic with a continuation, each run, as a report line (not a failure).
- **A queue's rollover is the review nudge.** For a standing queue the
  rollover prints a line saying the queue has rolled over and to run its
  review. The sweep's orphan pass keeps its existing cap of 95 open children
  as the "run intake review" signal, but no longer refuses at it: it links
  through the chain and reports the nudge instead.
- **Drift is reported.** The audit and the sweep report chain drift (§4.1),
  a full tail with no continuation, duplicate continuations, and a closed head
  with an open continuation (§4.6), each
  naming the containers, as link drift is reported today.
- **Orphans are found.** The audit and the sweep list every open issue
  labelled `continuation` in the home repo and check that each can be reached
  from the head its `Continues:` line names. The sweep adopts each orphan that
  passes the trust rules (§4.3), reports it, and then runs the rollup again
  for that head, since an item that closed inside the orphan fired no rollup
  the head could see; the audit reports it; one that fails them is reported
  as chain drift.

### 4.8 Re-homing #1384's tasks

Once the chain layer is released and installed, the 14 tasks of
vergil-project/.github#1384's Task 13 filed under ad-hoc epics (listed on the
planning issue) are moved into #1384's chain with `vrg-epic-move`, which rolls
that epic over. This is the first live rollover of a development epic and so its
end-to-end check: afterwards `vrg-epic-audit --epic vergil-project/.github#1384`
lists all of them, and the epic's driver drops the hand tracking. This runs
only while #1384 is open. If it has closed by then, closed is closed
(`move_into` refuses it, §4.2): the 14 tasks stay under their ad-hoc epics,
the retrospective of #1384 records them, and the live check of a development
epic's rollover is made another way, which the plan names.

### 4.9 Documentation

The human-facing docs that describe the 100-child cap, the sweep's 95 cap and
link drift (`docs/site/docs/standards/github-issues.md` in vergil-tooling,
among others) are brought in line: the chain, the continuation label and
lines, the rollover messages, the depth refusal and the drift checks. Skills
that tell an agent what to do at the cap (for example the workaround used for
that epic) are updated to say the tooling now rolls over.

## 5. Constraints

- GitHub: 100 sub-issues per parent, eight levels of nesting, a sub-issue has
  one parent, and `replaceParent` moves a sub-issue atomically.
- Every read of issue content goes through the trust layer; a continuation
  whose body is untrusted is drift, never followed.
- No silent failures: every refusal, heal and rollover is reported.
- Existing chains do not exist yet, so there is no migration beyond §4.8.

## 6. Left to the builder

- The module's exact function names and how the chain-aware readers wrap the
  raw ones.
- The retry bound and the heal reporting (§4.4).
- The depth headroom constant, fixed by a live probe (§4.5).
- Whether `vrg-epic-audit` exposes a per-container view as well as the flat
  one.
- How tests fake a chain (the existing GitHub fakes in the test suite are the
  starting point).
