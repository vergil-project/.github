# sub-issue-capacity — retrospective

- **Epic:** vergil-project/.github#1609
- **Spec:** [`spec.md`](spec.md)
- **Plan:** [`plan.md`](plan.md), with its `## Evolution during execution` log
- **Planning issue:** vergil-project/vergil-tooling#3840

## Span

| Window | Start | End | Elapsed |
| --- | --- | --- | --- |
| Whole epic | Tue 24 Nov 2026, 15:08 UTC | Thu 26 Nov 2026, 00:08 UTC | 1 d 8 h 59 min |
| Planning | Thu 29 Oct 2026, 17:05 UTC | Wed 25 Nov 2026, 12:01 UTC | 26 d 18 h 55 min |
| Implementation | Wed 25 Nov 2026, 11:41 UTC | Thu 26 Nov 2026, 00:08 UTC | 12 h 26 min |

## 0. At a glance

We set out to let an epic of any kind grow past GitHub's 100-sub-issue limit
as a chain of containers behind one chain layer, so that no caller meets the
limit and no item lands outside its epic, and to make a closed epic final.
That is what shipped. Every vergil-tooling reader and writer of sub-issues now
goes through `lib/epic_chain.py`, the raw layer is private and fenced by a
source-scan test, `vrg-gh` refuses the raw sub-issue flags, no tool reopens a
closed epic, and the plugin skills read a task's epic with the new
`vrg-epic-parent`. Rollover shipped switched off, was turned on in a second
release once every reader read through the chain, and the first live rollover
ran on a throwaway probe epic: 100 tasks, a continuation at the 100th, a clean
audit, a sweep that reported the chain, and a rollup that closed it head
first.

### Work delivered

| Repo | PR | Task | What it did |
| --- | --- | --- | --- |
| .github | #1639 | #1610 (1) | The spec and plan. |
| .github | #1646 | #1644 | Rebased the spec and plan onto epic #1384; Task 13 became a probe-epic check. |
| vergil-tooling | #5181 | #5155 (2) | Closed is closed: refuse a closed epic instead of reopening it. |
| vergil-claude-plugin | #1376 | #1371 (2b) | Skills say a closed proxy is refused and the agent stops for the human. |
| vergil-tooling | #5184 | #5156 (3) | `vrg-epic-move` moves in one atomic re-parent and refuses a closed target. |
| vergil-tooling | #5185 | #5157 (5a) | The chain layer's read side, the `continuation` label and the in-memory sub-issue fake. |
| vergil-tooling | #5189 | #5158 (5b) | First-fit linking, moving and unlinking, rollover (off), orphan adoption and the depth refusal. |
| vergil-tooling | #5190 | #5159 (5c) | Heals for racing rollovers (limit retries, full tail, duplicates) and `close_chain`. |
| vergil-tooling | #5194 | #5160 (6) | `epics.py` readers, ref resolution and rollup across the whole chain. |
| vergil-tooling | #5204 | #5163 (10b) | `vrg-epic-parent`: a task's epic and kind through the chain. |
| vergil-tooling | #5205 | #5162 (10) | Queues, roadmaps, planning closes, advisories and merge-review scopes across the chain. |
| vergil-tooling | #5207 | #5161 (9a) | The audit reads chained epics and reports chain drift and orphans. |
| vergil-tooling | #5213 | #5164 (7) | The CLI writers and `ops_report` through the chain, checking capacity before creating. |
| vergil-tooling | #5214 | #5165 (8) | `intake_route` routes and settles items across chained queues. |
| vergil-tooling | #5216 | #5166 (9b) | The sweep adopts orphans, heals chains and closes them whole. |
| vergil-tooling | #5217 | #5167 (11) | The raw sub-issue layer made private, with a boundary test and the `vrg-gh` refusal. |
| vergil-claude-plugin | #1395 | #1372 (14) | Skills read a task's epic and kind with `vrg-epic-parent`. |
| vergil-tooling | #5238 | #5168 (11b) | Rollover turned on; the switch and its disabled paths removed. |
| vergil-tooling | #5253 | #5169 (doc review) | CLI and architecture docs brought in line with chained epics. |
| .github | #1811 | #1810 (doc review) | The Task 4 probe script checks for a `gh` login first. |
| docs | #201 | docs#199 (doc review) | The epic-model summary describes chained epics and closed is closed. |
| vergil-claude-plugin | #1411 | #1406 (doc review) | Three skills describe chained epics and the closed-proxy refusal. |
| vergil-claude-plugin | #1415 | #1414 (follow-up) | Four more passages name chain items and chain drift. |

Operational tasks with no PR: #1632 (Task 4, the nesting-limit probe), the
installed releases #1633, #1634 and #1635 (Tasks 12, 14b and 12b), and the
live rollover #1636 (Task 13).

- **Repos touched:** 4 (vergil-tooling, vergil-claude-plugin, .github, docs).
- **Tasks:** 28 besides this retrospective: 23 closed by a PR, 2
  validation, 3 deployment. 24 were filed with the plan; 5 were added
  during execution (#1644, #1810, docs#199, vergil-claude-plugin#1406 and
  #1414).
- **PRs:** 23 (vergil-tooling 15, vergil-claude-plugin 4, .github 3,
  docs 1).
- **Releases:** vergil-tooling v2.1.348 (Task 2), v2.1.349 (Tasks 3–11)
  and v2.1.350 (Task 11b, rollover on); vergil-claude-plugin v2.1.95
  (Task 2b) and v2.1.97 (Task 14).

## 1. How the plan evolved

The plan's task list held. Two changes are in its evolution log, and a few
more happened inside tasks without changing the list.

- **Rebased onto epic #1384 before implementation.** The epic was planned
  while #1384 (agents submit and finalize their own PRs) was in flight. A
  review against #1384 as shipped found the plan barely encoded the old PR
  boundary; the only real change was Task 13. Its main path re-homed
  #1384's 14 ad-hoc tasks into #1384's chain if #1384 was still open, but
  #1384 was closing, so the probe-epic fallback became the only procedure
  and #1384's own retrospective records the 14 tasks. #1644 made the change
  before any implementation task shipped.
- **Task 4's probe confirmed the plan's assumptions.** The nesting limit is
  8 levels (7 below the top issue), which matches the depth constant, and
  GitHub accepts a link under a closed parent, so Task 5b took its primary
  path (the next continuation may be linked into a closed one) rather than
  the refuse-by-name fallback.
- **The probe script gained a login guard.** Run on a shell with no `gh`
  login it printed plausible but false results; the human ran a guarded
  copy, and the doc-review sweep fixed the plan's copy (#1810).
- **Choices inside tasks.** `vrg-epic-parent` reads only the native parent,
  not `epics.parent_of`'s body `Parent:` fallback, to match what the skills
  and the PR tooling already check. The audit reports a full last container
  only once rollover is on. The `vrg-gh` refusal of the raw sub-issue flags
  applies to the human identity too, not only to agents. Each is in its
  PR's notes.
- **The doc review took three rounds.** The sweep spawned three per-repo
  doc tasks; one of them found four more passages with the same gap, filed
  on the human's ruling as #1414; that run found three more (§4).

## 2. Lessons learned

- **The rollover switch earned its keep.** Shipping the chain layer with
  rollover off let every reader, the plugin skills included, read through
  the chain before any continuation could exist, and the order was
  enforced by blocker edges (11b after 14b after 14 after 12) rather than
  by hope. The cost was a disabled path in several modules, removed in one
  PR (#5238). Do the same next time a change has readers that ship on a
  different release train.
- **Readers before writers paid off.** With the readers (6, 9a, 10) merged
  before the writers (7, 8, 9b), the writer tasks were tested against
  chain-aware reads, and three tasks ran in parallel twice without a
  conflict.
- **Merge review caught real defects.** The first review asked for a fix
  on 7 of the 23 PRs, among them a continuation without its label taken
  for a head (Task 6), a heal error reported as a race (5c), and an orphan
  search that missed an unlabelled continuation (9a). It raised 1 security
  finding (§3).
- **Release steps belong to the release.** The plan listed per-host
  install and refresh steps for Tasks 12, 12b and 14b as human-run steps,
  but a release's install stage already installs on every host and VM. The
  human was asked for work the release had done. This is now intake
  .github#1690, and plan reviews should check for it.
- **A doc sweep should search for the contract, not the files.** The
  sweep and its spawned tasks found the "exit 1 is link drift" wording one
  batch at a time across three rounds. A single search for every mention
  of `vrg-epic-audit --epic` and its exit codes across the touched repos
  would have found them in one pass.
- **Live probes need guards.** A probe script a human runs once should
  check its own preconditions (a login, the right identity) before
  printing results, because a failed run that prints well-formed output
  is worse than one that stops.

## 3. Compromises & tradeoffs

- **Two limitations accepted.** A continuation that is unlinked *and* has
  lost its `continuation` label is invisible to readers and to the orphan
  search; it takes two independent failures, and the code documents it.
  The merge-review request builder no longer checks for a truncated label
  list, which only matters past 100 labels on one epic.
- **`vrg-epic-parent` has no body fallback.** It reads only the native
  parent, so a task linked only by a body `Parent:` line reports `none`.
- **Coverage by attestation.** The installed releases were checked
  directly only on the agent's VM; every other host and VM rests on the
  human's release comment and the release's install record. Task 13's
  "head first" close order rests on close timestamps, because the rollup
  prints nothing when it closes an epic (§4).
- **Rollover turned on with no soak.** Rollover went live in v2.1.350
  under two hours after v2.1.349, with the probe epic as its only live
  test. A real queue or development epic reaching 100 items is the first
  production rollover.
- **Security.** The epic carried 2 security findings, both resolved: 1 from
  alignment (advisory GHSA-3f77-xvqp-p9xw, carried by
  vergil-project/vergil-tooling#5157) and 1 from Task 11's merge review,
  fixed before it was released.

## 4. New problems & opportunities

**Unaddressed-findings sweep.** 28 tasks, the epic issue, the planning
issue and the 23 task PRs were swept. Every ruling request and `Awaiting a
ruling` item has a closing ruling; no `Filed without a
ruling` list exists; no PR has a review comment. The planning issue's
rejected findings all carry a reason, and its two review-process problems
are tracked (vergil-project/vergil-claude-plugin#1401 and #1037).

| # | Problem | Source | Outcome |
| --- | --- | --- | --- |
| 1 | Three more passages describe an exit 1 of `vrg-epic-audit --epic` as link drift only (`issue-implement`, `merge-review`, `skills/index.md`). | #1415's run; epic comment 7178453276 | Filed: vergil-project/vergil-claude-plugin#1416 (ad-hoc). |
| 2 | A tooling release running while agents merge leaves `develop`'s `VERSION` at the latest tag until the merge-back lands, failing `version / version-bump`. | Task 5a's PR | Found: epic vergil-project/.github#1664 (release lock), with the case added as a comment there. |
| 3 | Security checks failed only in their code-scanning upload during a GitHub outage, and the agent identity cannot re-run jobs. | Task 6's PR | Found: the re-run is vergil-project/vergil-tooling#5208 (ad-hoc). The outage itself is dropped: it was GitHub's, and #5208 is the remedy. |
| 4 | `vrg-finalize-pr` refuses to run from inside a worktree. | Task 5a's re-run | Found: vergil-project/vergil-tooling#5192 (closed) made the refusal say what to do. |
| 5 | `vrg-epic-rollup` prints nothing when it closes an epic. | Task 13 | Filed: intake vergil-project/.github#1814. |
| 6 | Task 6's merge-review closing record says `Branch as reviewed: yes` where the rule gives `no`. | #5160's closing comment | Filed: intake vergil-project/.github#1815. |
| 7 | `queue_or_archive_kind` in `lib/epics.py` recognises a continuation by its label only; no effect today. | Task 6's review | Filed: intake vergil-project/.github#1816. |
| 8 | Define fleet-wide what "released" means, install included. | This epic's gates | Found: intake vergil-project/.github#1690. |
| 9 | A plugin release record's `Install:` line names a vergil-tooling version. | #1371 and #1372's release records | Dropped: by design; it names the vergil-tooling version that performed the install. |

## 5. What's next

No follow-on epic. The forward items are the issues filed in §4 and intake
vergil-project/.github#1690, and epic vergil-project/.github#1664 (release
lock), just starting, for the release race. Collecting empty
continuations stays out of scope (spec §2) unless a real chain needs it.

## Appendix A — Operational notes

The human-gated sequence, in order:

1. **Task 4 probe** (#1632): run by the human with their own `gh` login,
   since the agent identity cannot create unparented issues or call
   GraphQL. A first attempt inside the VM had no login and printed false
   results; the guarded copy in `build/` ran on the host.
2. **v2.1.348** carried Task 2 by the normal cadence. Its post-release
   merge-back had not landed when Task 5a submitted, so 5a's
   `version / version-bump` failed until it did; a clean rebase then
   merged it.
3. **Task 6's security checks** failed in their upload step during a
   GitHub outage; the human re-ran the jobs.
4. **v2.1.349** (Task 12): Tasks 3–11, rollover off.
5. **vergil-claude-plugin v2.1.97** (Task 14b): Task 14, promoted.
6. **v2.1.350** (Task 12b): Task 11b, rollover on.
7. **The epic sweep** for Task 13 was dispatched by the human, since
   `vrg-gh` refuses `workflow run`.
