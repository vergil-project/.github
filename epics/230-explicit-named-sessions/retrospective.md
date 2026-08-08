# Retrospective — Explicit, purpose-named sessions (deprecate slots + archive)

- **Epic:** vergil-project/.github#230
- **Spec / Plan:** `epics/230-explicit-named-sessions/spec.md`, `plan.md`
- **Span:** 2026-08-06 → 2026-08-08

## 0. At a glance

We set out to replace `vrg-vm`'s slot-numbered, auto-resume-most-recent, archive-on-a-timer
session model with **explicit, purpose-named sessions** (`label:workspace`), reconnect **by
exact name**, and the archive concept **deleted** in favour of a recency display-filter — so a
session can track an epic or ad-hoc problem for its whole life and become a navigable
implementation-history record. That is exactly what shipped. Along the way the design was
**grounded in a live audit of Claude Code's supported surface** (the reverse-engineered
internals are quarantined behind a `SessionStore` seam), the Agent-SDK backend was **evaluated
and deliberately deferred**, `--fork` was **removed**, and **three defects that only the live
lab could surface** were fixed before sign-off.

**Work delivered**

| PR | Repo | What it did |
|---|---|---|
| [.github#233](https://github.com/vergil-project/.github/pull/233) | .github | spec + plan (the epic's docs bookend) |
| [#2621](https://github.com/vergil-project/vergil-tooling/pull/2621) | tooling | T0 — audit the Claude session surface; GO/NO-GO on the SDK |
| [#2622](https://github.com/vergil-project/vergil-tooling/pull/2622) | tooling | T1 — the `SessionStore` seam + `ScrapeStore` backend (the spine) |
| [#2633](https://github.com/vergil-project/vergil-tooling/pull/2633) | tooling | T2 — `--label` named create |
| [#2635](https://github.com/vergil-project/vergil-tooling/pull/2635) | tooling | T3 — `--resume` exact-name attach; remove `--slot` + auto-resume |
| [#2636](https://github.com/vergil-project/vergil-tooling/pull/2636) | tooling | T4 — delete archive/staleness; add the recency filter |
| [#2637](https://github.com/vergil-project/vergil-tooling/pull/2637) | tooling | T5 — `--fresh` as supported retire-rename (never delete) |
| [#2638](https://github.com/vergil-project/vergil-tooling/pull/2638) | tooling | T6 — #2602 selection-correctness regression (test-only) |
| [#2639](https://github.com/vergil-project/vergil-tooling/pull/2639) | tooling | T7 — one-time cosmetic `archived@` strip script |
| [#2649](https://github.com/vergil-project/vergil-tooling/pull/2649) | tooling | doc-sweep sibling — fix stale `--fork` help/error text |
| [#2657](https://github.com/vergil-project/vergil-tooling/pull/2657) | tooling | **fix** — `--resume` bare label + workspace-match; **remove `--fork`** |
| [#2659](https://github.com/vergil-project/vergil-tooling/pull/2659) | tooling | **fix** — detect `--label` collision before the name registers |
| [#2670](https://github.com/vergil-project/vergil-tooling/pull/2670) | tooling | **fix** — resume a created-but-unused session (re-enter at id) |
| [vergil-vm#292](https://github.com/vergil-project/vergil-vm/pull/292) | vergil-vm | doc-review — rewrite session docs to the explicit-named model |
| [vergil-vm#297](https://github.com/vergil-project/vergil-vm/pull/297) | vergil-vm | doc-fix — reflect bare-label resume + `--fork` removal |

- **Children:** 18 (16 code/doc + 1 validation + 1 retrospective). **PRs merged:** 15.
- **Repos:** vergil-tooling (code), vergil-vm (docs), .github (epic docs).
- **Not-a-PR closures:** T8 (#2612) closed **won't-do** (deferred); Validation (#2613) closed on a
  live-lab `Outcome: SUCCESS` comment.
- **Releases cut during the epic:** vergil-tooling 2.1.175 → 2.1.181 (each fix round required a
  release + install before the live re-validation).

## 1. How the plan evolved

The plan's shape (T0 spike → T1 seam → T2/T3 create/attach → T4 archive-delete → T5/T6/T7 →
validation → doc-review → retrospective) held. The **deltas** were concentrated in two places:
the SDK decision, and everything the **live validation** shook loose.

- **The SDK backend was deferred, not built.** T0's spike came back **GO** (the Agent SDK's
  session functions are real, local, and see interactive sessions), yet the human declined it:
  a **pre-1.0 runtime dependency** whose payoff was only partial (it still rides Claude's
  internal transcript format and still needs the roster for liveness) was not worth it on the
  most fragile seam. T8 (#2612) was closed won't-do. **The seam earned its keep here** — it made
  "not now" a one-line future swap rather than a foreclosed path.

- **`--fork` went from "kept" to "removed."** The spec kept `--fork`, but T3's `--slot` removal
  orphaned its only targeting mechanism (it silently always-refused). Rather than re-plumb it,
  the human deprecated it outright — it fights the new *self-contained, start-fresh-often*
  session philosophy. Removed in the fix PR (#2657).

- **The live lab drove three unplanned fixes.** Unit tests were **100% green the entire time**,
  yet the first real create/resume on the Mac exposed defects the unit layer structurally
  could not: (a) `--resume` required the full `label:workspace` while `--label` took the bare
  label — an asymmetry that also contradicted the spec's own §4.2 intent (fixed, #2657);
  (b) the `--label` uniqueness check missed a collision inside a name-registration window
  (fixed with a reserve-name-before-launch mechanism, #2659); and (c) that very reservation
  mechanism then failed to resume a *created-but-unused* session ("No conversation found") —
  because Claude persists transcripts lazily — fixed by re-entering at the reserved id (#2670).
  Each was a full **validate → fix → release → re-validate** cycle.

- **The doc bookend needed a sibling.** The doc-review PR (#292) merged before the late
  behaviour changes (bare-label resume, `--fork` removal) existed, so a follow-up doc task
  (#296 → #297) was spawned to bring the docs to the final shipped state.

## 2. Lessons learned

- **Live-lab validation is not optional for host/TTY behaviour.** 100% unit coverage held
  throughout and still missed three real defects — all in the seam between our code and Claude's
  actual runtime (interactive launch, name registration, lazy transcript persistence). The
  validation gate was the single most valuable step in the back half of the epic. Repeat this
  pattern for anything that shells out to an interactive external tool.
- **A registration-latency race is a *class*, not an incident.** The same "the store isn't
  updated the instant you act" bug bit twice in two days — the GitHub finalize race (#2623) and
  the `--label` collision race (#2659) — plus a third latency-shaped variant (resume-unused,
  #2670). Worth codifying a reusable "wait-for-registration / tolerate not-yet-registered"
  pattern.
- **Reverse-engineered internals demand a seam *and* a spike.** Auditing Claude's supported
  surface up front (T0) and confining the unsupported scrape to one `SessionStore` backend is
  what let the whole epic ship without betting on the pre-1.0 SDK — and kept every later
  Claude-internals surprise contained to one module.
- **New machinery earns extra live scrutiny.** The reservation mechanism was clever and
  unit-tested, but its interaction with Claude's lazy persistence was invisible to unit tests.
  Novel mechanisms should get a deliberate live edge-case pass, not just green tests.

## 3. Compromises & tradeoffs

- **Deferred the SDK backend (T8).** We knowingly stay on the reverse-engineered scrape backend
  rather than a supported-but-pre-1.0 SDK. Debt: the scrape rides Claude's internal transcript
  format and can break on a Claude release. Mitigation: it's isolated behind the seam, and the
  swap is a single backend class when the SDK reaches 1.0.
- **The reservation mechanism adds vrg-owned local state.** `~/.claude/vrg-session-reservations/`
  (TTL-swept) is new surface that must track Claude's lazy-persistence behaviour — the source of
  the #2670 edge. Accepted as the price of race-free collision detection; it is self-cleaning and
  now covered both ways.
- **Orphaned `--slot` machinery left in `session.py`.** `select()`/`_select_explicit`/
  `_select_default` are dead after `--slot` removal but were left in place (out of scope of the
  fixes). Logged as a small follow-up (see §4), not cleaned here.
- **Fix-then-refix cost real cycles.** T3's resume drifted from spec → #2657; #2659's reservation
  introduced #2670. Catching these live (rather than up front) meant ~6 releases and three
  re-validations across two days. The tradeoff was deliberate: the live lab is where these were
  *findable at all*.

## 4. New problems & opportunities

- **#2623 — finalize registration-race (fleet-blocking, fixed & deployed).** Surfaced *because*
  this epic's parallel-agent PR flow hammered `vrg-submit-pr → vrg-finalize-pr` the day GitHub's
  post-outage timing widened a latent check-registration window. Root-caused and fixed
  (ad-hoc, org `.github#99`), unblocking every automated PR in the fleet — a direct dividend of
  running the epic at volume.
- **#2634 — `vrg-worktree-status` hid detached-HEAD / mid-rebase worktrees (fixed).** Found when
  a rebase left a worktree detached and the tool reported "none," making a live branch look
  lost. Filed and fixed (ad-hoc, `.github#99`).
- **Tech debt logged, not yet acted:** the orphaned `--slot` machinery in `session.py` (§3).
- **Opportunity — codify the registration-latency pattern.** Three latency-shaped bugs in two
  days argues for a shared helper/idiom.
- **Realized goal — the epic-keyed session archive.** Sessions are now nameable by epic and
  preserved for their whole life; this is the structured corpus the spec's §9 flagged for a
  future Mem Palace.

## 5. What's next

- **SDK backend (Stage E / T8)** — revisit when `claude-agent-sdk` reaches 1.0; the seam makes it
  a contained swap. (Deferred, `.github#230` §9.)
- **Epic-state-bound session visibility** — a session's default listing could track its epic's
  open/closed state; deferred to avoid coupling `vrg-vm` to live GitHub state (spec §9).
- **Mem Palace indexing** of the epic-keyed transcript archive (spec §9).
- **Small cleanup** — remove the orphaned `--slot` machinery in `session.py`.

## Appendix A — Operational notes

This was a **validation-heavy** epic: the `impl → deploy → validate → fix` loop ran three times,
each requiring a human-gated **release + `uv tool install`** before the live re-test (agents
cannot cut releases or observe the terminal title/TTY). Sequence per round: merge the fix →
release vergil-tooling (2.1.17x bump) → install on the Mac → human runs the `vrg-vm session`
live checks from the projects root → agent records `Outcome:` on #2613. The doc bookend landed
as **two** PRs (the sweep #292, then the post-fix correction #297) because behaviour was still
changing when the first merged — a reminder to hold a doc-review bookend until the code frontier
is truly closed on a validation-driven epic.
