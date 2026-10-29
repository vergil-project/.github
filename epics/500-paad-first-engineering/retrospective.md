# Retrospective: paad-first AI-assisted engineering (umbrella)

**Epic:** vergil-project/.github#500 · **Planning issue:**
vergil-project/.github#499 · **Spec:** [spec.md](spec.md) ·
**Plan:** [plan.md](plan.md)

## 0. At a glance

We set out to make paad a consistent, first-class part of the vergil workflow,
following the paad authors' recommended loop by default. There were three
parts:

- one written doctrine for repeated paad reviews, with a human-gated mode in
  `epic-create` and an autonomous mode in `issue-implement`;
- every deviation from paad written down with its reason;
- the umbrella's own hygiene work (B).

The umbrella owned B. It drove six children through gate tasks and text links,
because native epic nesting doesn't exist yet (.github#88).

**What shipped.** Every child closed.

- **The doctrine.** Both modes are live in vergil-claude-plugin, and the
  deviations register is maintained in `docs/paad-policy.md`.
- **Security findings.** They now have a private, durable home: a draft
  advisory paired with a neutral issue, with one issue able to name several
  advisories.
- **Steering and settings files.** They are owned, cleaned and audited across
  18 repos.
- **Honestly, the main result.** Mode 2 did not converge on its dogfood run,
  and the doctrine's complexity is mostly interface work around paad. Both
  lead to the follow-on epic .github#699 (paad unattended mode).

| Child | Delivered as | Gate | What it delivered |
|---|---|---|---|
| A: security-finding handling | epic .github#517 | #509 | Private advisory paired with a neutral issue; `vrg-security-record`; sweep; audit of past issues |
| A′: one-to-many security records | epic .github#573 | #572 | Several advisories per issue across the tooling |
| B: paad 2.0 hygiene | this epic's own tasks | none | Managed `paad/security/` ignore, fleet sync, VM check, paad policy |
| C: design review in `epic-create` | epic .github#553 | #510 | Doctrine common core and mode 1; `epic-create` review rounds |
| D: agentic review in `issue-implement` | epic .github#568 | #511 | Doctrine mode 2; the gate in `issue-implement`; the sweep in `epic-retrospective` |
| E: steering and settings review | epic .github#558 | #512 | Ownership model; fleet cleanup; nightly audit; `steering-audit` |
| F: wider paad adoption | single task plugin#823 | #514 | Decisions on the remaining paad skills, recorded in the policy |

**The umbrella's own PRs (7):**

| PR | Merged | Task | What it did |
|---|---|---|---|
| .github#516 | 2026-10-26 | .github#501 | Published the spec and plan |
| vergil-tooling#3692 | 2026-10-26 | vergil-tooling#3666 | Managed `.gitignore` base ignores `paad/security/` |
| vergil-claude-plugin#736 | 2026-10-26 | plugin#727 | paad usage policy and deviations register |
| vergil-claude-plugin#744 | 2026-10-26 | plugin#743 | Skills reference names `vergil:handoff` with its prefix and notes `paad:handoff` |
| docs#62 | 2026-10-26 | docs#61 | Workflow page links the paad usage policy |
| vergil-tooling#3706 | 2026-10-27 | vergil-tooling#3658 | Documentation review: `paad/security/` in the managed base docs, policy links |
| vergil-claude-plugin#826 | 2026-10-29 | plugin#823 | Child F's decisions recorded in the policy |

- **Fleet sync (vergil-tooling#3667):** 17 gitignore sync PRs in three orgs,
  all merged and audited compliant.
- **Children:** 16 in total:
  - 7 tasks with a PR;
  - 2 operational B tasks (the sync deployment and the VM validation);
  - 6 gate tasks;
  - this retrospective.
- **Child epics:**
  - A, 15 PRs;
  - A′, 14 PRs;
  - C, 8 PRs;
  - D, 11 PRs;
  - E, 23 task PRs plus 9 sweep PRs and 4 ad-hoc fixes.

  Their releases are listed in each child's retrospective.
- **Releases from the umbrella's own tasks:** vergil-tooling v2.1.271 contains
  #3666, as attested in #3667. Which plugin release first carried #736, #744
  and #826 is **unknown**.
- **Span:** planning issue and epic opened on 2026-10-26; the last gate
  closed on 2026-10-29. That is four days for the umbrella and all six
  children.

## 1. How the plan evolved

`plan.md` has no evolution log and was not edited after it merged. The
changes below come from the issue record.

- **A child was dropped at planning.** In plan pushback round 2, the human
  dropped child G (minimal nested-epic support). The gate-task approach tests
  nesting by hand instead, and the real mechanism stays with .github#88.
- **A child was added mid-flight.** A′ (#571, then epic #573) appeared on
  2026-10-27. C and D would raise several security findings against one
  issue, and A's tooling read only one advisory per issue. It got its own
  gate, #572, and C and D gained a dependency on it.
- **The children were planned in parallel and then reconciled.** The planning
  sessions for A′, C, D and E ran close together and kept settling the same
  questions: where paad writes, how security findings are recorded, and how
  comments are trusted. C paused partway through spec pushback and adopted
  the answers from A′, D and E (see epic 553's retrospective).
- **Mode 1 was trialled before it shipped.** E's planning and C's own later
  rounds ran an ad hoc version of the human-gated rounds, and E's evidence
  retuned C's recommendation rule. Planning used the process that was still
  being built.
- **D's dogfood did not converge.** The gate shipped as specified, but a real
  task ran 5 rounds without converging, and the human dropped it. Together
  with C's doctrine review (plugin#811), that moved the fragile parts into a
  proposed change to paad itself, epic .github#699. The spec had allowed for
  that: its non-goals permit an authors-side change "for example, an
  unattended mode".
- **F collapsed to one task.** Evaluating the six remaining skills one by one
  found one adoption (`rethink`), five used by hand, and one deviation
  (`backlog`), with nothing to build. F closed as a single doc task, which is
  the case the plan's gate text anticipated.
- **Gate bookkeeping lagged once.** Child A's planning item, #505, was never
  closed after epic #517 finished. Gate #509 stayed blocked until the human
  closed it as "Planned as #517" on 2026-10-29.

## 2. Lessons learned

- **Gate tasks plus text links worked as a stand-in for nesting.** Six
  children, one added late, closed through mechanical gate checks with no
  native nesting. The cost was manual upkeep: the table in #500, the planning
  item closes, and one missed close (#505). That's useful input for .github#88.
- **Children that share a mechanism should settle it once, early.** A′, C, D
  and E each re-derived "where does paad write" and "how are security
  findings recorded". A short shared-contract step after the umbrella spec
  would have saved C's pause and restart.
- **Steering paad from outside costs more than it looks.**
  - The doctrine has a long reviewer prompt that overrides the skill, output
    checks, a stray-write check, and a security self-test.
  - Each spike or dogfood added another guard.
  - Some guards then misfired: the stray-write check gave false positives
    under parallel runs, and the security self-test sent meta findings about
    a docs-only change into the private path (#721 and #722, ruled false
    positives).

  The lesson is to fix the interface at the source (.github#699) rather than
  keep layering guards.
- **Dogfood on real work.** C's dogfood found the "hold" wording problem and
  the brainstorm problems. D's dogfood found that the gate doesn't converge.
  No synthetic check found either.
- **Not every paad skill belongs in the daily loop.** F's main finding is that
  pushback, alignment and agentic-review fit the per-change workflow, while
  architecture, test-roadmap, OWASP and dedup are strategic tools for a
  periodic review run by hand. That avoided building tooling around
  experimental skills.

## 3. Compromises and tradeoffs

- **The doctrine shipped knowing much of it was workaround.** It stays in use
  until .github#699 lands. That includes the stray-write check, which the
  human has ruled overboard and is removing, and which this session overrode
  on every gate run.
- **Mode 2 is live but not proven to converge.** The gate pauses for the human
  after 3 rounds, so it is safe. But its dogfood was dropped, not passed.
- **No paad version drift detection.** paad is unpinned and nothing detects
  drift between releases (paad policy §1), deferred until real use needs it.
- **Some task records predate the gate.** Plugin tasks in C were implemented
  before mode 2 was loaded in their sessions, so they have no gate records.
  The human accepted this as a side effect of the rollout.
- **Security records are heavy.** Every security-class finding creates a
  draft advisory and a public neutral issue. Over the umbrella that came to
  many records (the spike alone produced 11), including two false positives.

## 4. New problems and opportunities

| Item | Where it went |
|---|---|
| paad needs an unattended mode so the doctrine can shed its interface work | Epic .github#699, planned in .github#698 (in progress) |
| Mode 2 stray-write sweep fails on harness outputs and parallel scratch copies | .github#707; the check is being removed |
| Security self-test over-routes meta findings about security handling | .github#723 |
| Mode 1 checkpoint recommendation leans toward continue | .github#716 |
| Two `epic-create` sessions can run on one planning issue | .github#715 |
| Mode 1 paths never exercised live: resume and untrusted-comment classification | .github#717 |
| Intake epic hit GitHub's 100-sub-issue cap | .github#714 (queue drained; tooling handling still open) |
| Remaining paad skills not covered by F (`agentic-a11y`, `vibe`, `makefile`, `paad-help`) | .github#719 |
| `test-roadmap`'s branch use, and whether the `fix-architecture` reroute is a deviation | .github#720 |
| A skill for the periodic strategic review (architecture, OWASP, dedup) | Already tracked (per the human) |
| Native epic nesting | .github#88 (non-goal of this epic) |

### Unaddressed-findings sweep

- **Scope:** the umbrella's 15 children other than this retrospective, their
  merged PRs, and planning issue #499. The trusted authors are from
  `vrg-trusted-authors`. No umbrella PR had a review or conversation comment.
  Each child epic ran its own sweep in its own retrospective.
- **With an agentic-review summary:** plugin#823 passed after 1 round.
  - Its unfixed suggestions are tracked as .github#719 and #720.
  - Its 2 security findings were recorded, then ruled false positives by the
    human, and their neutral issues were closed as not planned.
- **Without gate records:**
  - **B's tasks:** vergil-tooling#3666, #3658, plugin#727, #743 and docs#61.
    They were implemented on 2026-10-26 and 27, before mode 2 existed.
  - **Operational tasks:** the B deployment #3667, the B validation #3668,
    and the six gate tasks.
  - **The docs task #501:** it is covered by the design rounds on #499.
- **Planning issue #499:** every finding was accepted, across spec pushback
  (10, 13), plan pushback (7, 10) and alignment (12).
  - Six round-2 findings were handed to child D as required inputs.
  - Dropped child G is covered by .github#88.
- **Other comments:** #3667 noted an unrelated audit failure, already tracked
  as .github#536 (closed).
- **Untracked problems:** none found.

## 5. What's next

- **.github#699, paad unattended mode:** the direct follow-on. It is a proof
  of concept in a paad fork, after which the human decides whether to offer
  it upstream. If it lands, the doctrine is simplified and the stray-write
  check removed.
- **A periodic strategic review** (architecture, OWASP, dedup, test-roadmap,
  run by hand). It is already tracked. Its results feed planning, and
  security findings go through `vrg-security-record`.
- **.github#88, native epic nesting:** this umbrella's gate tasks and text
  links are a worked example of what it should replace.
