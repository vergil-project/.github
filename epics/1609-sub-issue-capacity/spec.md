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
6. A drained queue's chain shrinks again.
7. The 14 tasks of vergil-project/.github#1384 filed under ad-hoc epics are
   re-homed under #1384's chain.

### Non-goals

- **Archives.** Quarterly archives already roll over into numbered segments
  (`ensure_writable_standing_archive`, segment cap 95). They are append-only
  buckets that nothing reads as one list, and their scheme works; this epic
  leaves it as it is.
- **Unbounded size.** The chain is linear, so the depth limit bounds it at
  roughly seven or eight containers (about 700 items). Past that the tooling
  refuses (§4.5); nothing lifts the bound.
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
  tooling.
- A continuation carries **none** of the head's kind labels (`intake`,
  `planning`, `deferred`, `ad-hoc`, `proxy`, `hoshin`, `operations`,
  `epic-development`, …). Every lookup that finds a canonical epic by label or
  title (`find_standing_epic`, `ensure_standing_epic`, `queue_kind`,
  `standing_kind_of`, `roadmap.gather`, the planning queue view) must
  therefore never find a continuation, and a continuation's kind is always its
  head's.
- **Trust.** A container is accepted as a continuation only when it carries
  the label, its body passes the trust layer, its `Continues:` line names the
  head being walked, its `Container:` number is the next in sequence, and it is
  a native child of the container before it. Any mismatch is **chain drift**: a
  reader raises it, naming the container and the rule it broke, and the audit
  reports it (§4.7). A reader never guesses past a broken link.

### 4.2 The chain layer

A new module, `lib/epic_chain.py`, owns the chain. Its public surface (names
are the builder's to refine):

- `chain_of(head) -> list[IssueRef]`: the head then each continuation.
- `head_of(container) -> IssueRef`: the head of any container (itself, for a
  head). It walks native parents while the container is a continuation, and
  checks the `Continues:` line agrees.
- `chain_child_states(head)`: the flattened children across the chain, in
  container order, continuations excluded.
- `link_into(head, task)`: link *task* into the chain's tail, rolling over
  first when the tail is full (§4.3, §4.4). It keeps `add_child`'s
  reopen-on-late-child rule: a closed head is reopened, and so is a closed
  continuation it writes into.
- `move_into(head, task)`: atomically move *task* from wherever it is into the
  chain's tail (`replaceParent`, as `reparent_child` does today), with the same
  rollover.
- `chain_contains(head, task)`: whether *task* is linked anywhere in the chain.

**The existing public readers become chain-aware.** `child_states`,
`child_states_with_blockers`, `native_children`, `all_children_closed`,
`parent_of` and `native_parent_of` (as seen by callers outside the chain
layer) answer for the chain: children of a head are the chain's, and a task's
parent is its head. The raw, per-container reads stay private to `epics` and
`epic_chain`. So every existing caller is correct without each one being
rewritten. The callers that read or write children today, all of which move
to the chain-aware form, are:

- linking: `vrg-epic-link`, `vrg-issue-create`, `vrg-intake-create`,
  `vrg-security-record`, `vrg-epic-move`, `intake_route`, `ops_report`, the
  epic sweep's orphan pass and its re-links (`epic_audit`);
- reading: `epic_audit` (including `epic_children` with its proxies),
  `queue_view`, `roadmap`, `planning_close`, `security_records`,
  `merge_review/request`, `vrg-submit-pr`;
- the rollup and the standing drain (§4.6).

Whether a given caller wants a reader that also exposes containers (the audit,
for drift) is the builder's call; the default is the flattened, head-level
view.

### 4.3 Rollover

- **When:** a writer rolls over before linking when the tail already holds 99
  children. GitHub counts **every** child, open and closed, so the count is of
  all children, not only open ones.
- **How:** find-or-create the next continuation, then link it into the tail's
  last slot, then link the item into it. Find-or-create first looks for an
  existing continuation child of the tail (a concurrent writer's), so two
  rollovers do not normally make two.
- **Order matters for safety:** the item is linked only after its container is
  in place, so a writer that dies mid-rollover leaves at worst an empty,
  correctly linked continuation (which the shrink pass removes, §4.6), never an
  unlinked item.

### 4.4 Races

GitHub offers no lock, so a chain is made safe by reading back, not by
locking:

- **GitHub's limit error is "full".** A link that fails with GitHub's
  sub-issue limit error is never surfaced raw: the writer re-reads the tail,
  rolls over and retries, a bounded number of times. Only when the retries run
  out does it fail, naming the limit, the container and the item.
- **A full container with no continuation** (two writers each saw 98 and both
  linked, taking slot 100): after every link the writer reads the container
  back. If it holds 100 children and no continuation, the writer whose item it
  just linked moves that item out (unlink), links a continuation into the freed
  slot (find-or-create), and links its item there. Whoever finds the state
  heals it.
- **Two continuations under one container** (two writers both created one):
  the writer that finds it after its own link keeps the lower-numbered issue,
  moves any items from the other into it, and closes the other as not planned
  with a comment naming the kept one. The audit reports any it finds (§4.7).
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

### 4.6 Rollup, drain and shrink

- **Rollup.** `rollup(task)` resolves the task's head (`head_of(parent)`). A
  finite epic closes only when every item across its chain is closed and
  trusted-closed, as today; it then closes every continuation, tail first, and
  the head last. A continuation closing never rolls anything up on its own: a
  close event whose issue is a continuation is a no-op for the rollup.
- **Standing drain.** `drain_standing_child` judges the **head**'s title and
  kind, not the continuation's, so a closed item drains to its archive from
  any container. The batch drain (`apply_standing_drain`) reads the whole
  chain.
- **Shrink.** The epic sweep closes any continuation that holds no items
  (only, at most, its own successor), re-linking its successor under its
  predecessor with an atomic re-parent first, then closing it as completed with
  a comment. The head is never removed. New items always go into the tail, so
  a shrunken chain is simply shorter; items are never moved back toward the
  head to fill freed slots.

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
  a full container with no continuation, and duplicate continuations, each
  naming the containers, as link drift is reported today.

### 4.8 Re-homing #1384's tasks

Once the chain layer is released and installed, the 14 tasks of
vergil-project/.github#1384's Task 13 filed under ad-hoc epics (listed on the
planning issue) are moved into #1384's chain with `vrg-epic-move`, which rolls
that epic over. This is the first live rollover of a development epic and so its
end-to-end check: afterwards `vrg-epic-audit --epic vergil-project/.github#1384`
lists all of them, and the epic's driver drops the hand tracking. If #1384 has
closed by then, the move reopens it (the reopen-on-late-child rule) and its
rollup closes it again once they are all closed.

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
