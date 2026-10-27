# Deferred Queue — Retrospective

Epic: [vergil-project/.github#508](https://github.com/vergil-project/.github/issues/508) ·
Spec: [`spec.md`](spec.md) · Plan: [`plan.md`](plan.md)

## 0. At a glance

**What we set out to do.** Intake and planning were filling up with items that had been
evaluated and consciously parked: waiting on an upstream release, another epic, or a date. They
were mixed in with live work, so every review re-read them, and nothing recorded why they were
parked or what would bring them back. The epic set out to:

- add a third standing queue, `Epic (deferred): <name>`, in every queue home;
- make defer a route in `vrg-intake-route` that requires a reason and revival conditions, from
  intake, planning or ad hoc only;
- list the queue with `vrg-queue deferred`;
- add the `deferred-review` and `planning-review` skills, and update `intake-review`,
  `epic-create` and `issue-implement`;
- sweep every live queue once, moving everything already parked into the deferred queue.

**What shipped.** All of it, plus one unplanned fix:

- vergil-tooling v2.1.270 added the `DEFERRED` kind, the defer and revive routes, and the
  `vrg-queue deferred` view.
- vergil-claude-plugin v2.1.42 added the two review skills and updated the three existing ones.
- The sweep (#520) reviewed all 96 intake and planning items in three queue homes and deferred
  28 of them.
- vergil-tooling v2.1.272 fixed a race that created duplicate standing epics. The sweep found
  it; see §1.
- The docs review updated the vergil-tooling standards pages and spawned three doc tasks in other
  repos.

| Repo | PR | Task | What it did |
|---|---|---|---|
| .github | [#521](https://github.com/vergil-project/.github/pull/521) | #513 | Spec and plan |
| vergil-tooling | [#3684](https://github.com/vergil-project/vergil-tooling/pull/3684) | #3673 (T1) | The `DEFERRED` standing queue kind |
| vergil-tooling | [#3686](https://github.com/vergil-project/vergil-tooling/pull/3686) | #3674 (T2) | `vrg-intake-route` routes into and out of the deferred queue |
| vergil-tooling | [#3687](https://github.com/vergil-project/vergil-tooling/pull/3687) | #3675 (T3) | `vrg-queue deferred` |
| vergil-claude-plugin | [#733](https://github.com/vergil-project/vergil-claude-plugin/pull/733) | #728 (T5) | `intake-review`, `epic-create` and `issue-implement` handle the deferred queue |
| vergil-claude-plugin | [#734](https://github.com/vergil-project/vergil-claude-plugin/pull/734) | #729 (T6) | New `deferred-review` skill |
| vergil-claude-plugin | [#735](https://github.com/vergil-project/vergil-claude-plugin/pull/735) | #730 (T7) | New `planning-review` skill |
| vergil-tooling | [#3701](https://github.com/vergil-project/vergil-tooling/pull/3701) | #3695 *(added)* | `ensure_standing_epic` waits until a new standing epic is listed |
| vergil-tooling | [#3713](https://github.com/vergil-project/vergil-tooling/pull/3713) | #3669 | Documentation review: standards pages, CLI overview and `CLAUDE.md` |
| vergil-claude-plugin | [#749](https://github.com/vergil-project/vergil-claude-plugin/pull/749) | #748 *(spawned)* | README Skills table and skills catalogue |
| .github | [#543](https://github.com/vergil-project/.github/pull/543) | #540 *(spawned)* | Comment in `epic-rollup.yml` on why queue epics never roll up |
| docs | [#66](https://github.com/vergil-project/docs/pull/66) | #63 *(spawned)* | Management pages describe the three-queue model |

- **Repos touched by PRs (4):** .github, vergil-tooling, vergil-claude-plugin, docs. The sweep
  also changed issues in logical-minds-foundry/.github, mq-gateway-replay-lab and
  mq-resiliency-lab-for-linux.
- **Children (16):**
  - 12 planned: T1–T9 and three bookends (#513, #3669, #515).
  - 4 added during execution: the race fix (#3695) and three doc tasks spawned by the docs
    review (#748, #540, docs#63).
  - 3 operational: deployments vergil-tooling#3676 (T4) and vergil-claude-plugin#731 (T8), and
    the migration sweep .github#520 (T9).
- **PRs merged:** 12, including the spec/plan PR and excluding this one.
- **Releases cut (by the human):**
  - vergil-tooling **v2.1.270** (T1–T3) and **v2.1.272** (the race fix). v2.1.271, between
    them, held unrelated security-records work.
  - vergil-claude-plugin **v2.1.42** (T5–T7). T8 installed **v2.1.43**, which was released
    before the install succeeded and adds nothing from this epic.
- **Span:** epic opened 2026-10-26 17:13Z; the last task before this one (docs#63) closed
  2026-10-27 11:55Z, about 19 hours.

## 1. How the plan evolved

The plan has no "Evolution during execution" log, so this section is reconstructed from the issue
record and the session. Tasks T1–T7 shipped as planned. The deviations came in the operational
tasks.

- **The sweep found a race and paused for a real fix.** The first two defers in vergil-project ran
  eight seconds apart. Each called `ensure_standing_epic`, which finds the deferred epic by
  listing issues and creates one when the list is empty. GitHub's issue list had not yet caught
  up with the first new epic, so the second call created a duplicate (#531 and #532). The human
  merged the two by hand. Pre-creating the deferred epic in every org would have avoided the
  race for this sweep only. The human chose a proper fix instead: the creating call knows the
  number it created, so it now waits, with a bounded budget, until that epic is listed, and
  raises if it never is (#3695, v2.1.272).

  The sweep held the logical-minds-foundry and gateway defers until the release, and each
  home's first defer then served as the live test. #388 was listed after about 11 seconds and
  gateway #329 after about 5.5, and each home ended with exactly one deferred epic.
- **T9 finished before T8.** The plan put the plugin install (T8) before the sweep (T9). The
  sweep needed only the tooling routes, so it ran on v2.1.270 and v2.1.272 while the plugin
  install was still pending. Intake review was not run in that window, as the plan required. The
  sweep used `vrg-intake-route` directly, with every batch confirmed by the human.
- **T8 was blocked by plugin installation, not by the plugin.** Updating to v2.1.42 failed with
  "its source doesn't match its extraKnownMarketplaces entry". The user settings declared the
  marketplace with `ref: main`, but it was registered without a ref, and Claude Code now drops a
  marketplace when the two disagree. The repos have declared `ref: main` since June 2026, so they
  were not the cause.

  Two things made it slow to untangle:
  - The macOS host and the Lima VM each have their own `~/.claude`, so fixing one left the
    other broken.
  - `claude plugin marketplace remove` also deleted the marketplace declaration from the
    checked-in `.claude/settings.json` of the repo it ran in. That happened in two repos, and the
    dirty main worktree then failed `vrg-finalize-pr` cleanup for #749.

  The fix that worked on both machines can be scripted. Remove the marketplace from a directory
  outside any repo, add it again as `vergil-project/vergil-claude-plugin@main`, then reinstall.
- **The docs review spawned three tasks.** The vergil-tooling standards pages were the planned
  scope. A read-only survey of the other repos found the docs repo's `management/intake.md`
  wrong, not just incomplete: it said a deferred item "keeps its age and comes up first next
  time". It also found the plugin README listing 7 of 17 skills, and a misleading workflow
  comment in `.github`. Each became a task under the epic. docs#63 was blocked until #3669
  merged, because #3669 renamed the anchor that docs#63 links to.

## 2. Lessons learned

- **A real fix, validated in production, beat a workaround.** Pre-creating the epics would have
  hidden the race until the next home's first creation hit it. Holding the remaining defers
  until v2.1.272 turned the sweep into the fix's acceptance test at almost no cost.
- **List-based find-or-create is unsafe on GitHub without a wait.** The same problem had been
  handled for archives (`prefer_oldest`, #2698) but not for live standing epics. Any new
  find-or-create over `gh issue list` should wait for its own write to become visible.
- **The plugin install path is part of the deployment and needs to be scriptable.** T8's own
  steps were trivial. All of the time went into marketplace state that is spread across user
  settings, project settings and `known_marketplaces.json`, on two machines. The steps that
  worked are now known and can be run from the command line (§1); they should be written down
  once, not rediscovered.
- **Docs reviews should check the other repos' text, not just their structure.** The
  docs-repo page that contradicted the new behavior would have survived a check of headings and
  links.
- **Re-derive state from GitHub, not from the session.** Early in implementation the agent
  misreported the spec/plan branch's state, and the human corrected it. The skill already says
  to reconstruct state from GitHub; following it each time avoids this.

## 3. Compromises & tradeoffs

- **Four queue locations were not swept.** dmorel69 has no GitHub App installation;
  wphillipmoore/career-strategy and wphillipmoore/local-llm could not resolve their epic home
  (`gh repo view … --json visibility` failed); wphillipmoore/.github does not exist. The human
  ruled all four out of scope. Any parked items there stay where they are.
- **The sweep needed a lot of human attention.** 96 items in batches of up to 4 meant 15
  approval rounds. That was deliberate (every route is a human decision), but it is the cost of
  a one-time migration done item by item.
- **Some items were kept live rather than deferred.** For example, vergil-project #174 is parked
  on capacity, not an outside condition, and the children of umbrella #500 are already
  sequenced by #500. These were judgment calls recorded in the #520 results, not rule
  violations, but they mean intake and planning are not purely live work.
- **Duplicate deferred epic #532 remains, closed as not planned.** It was consolidated by hand
  rather than by tooling.

## 4. New problems & opportunities

| Item | Where it went |
|---|---|
| Standing-epic creation race | Fixed: vergil-tooling#3695 (v2.1.272) |
| `vrg-queue` raises an uncaught traceback when a queue home (`wphillipmoore/.github`) does not exist | Logged in the #520 results; not yet acted on |
| Archive containers (logical-minds-foundry/.github#180, mq-gateway-replay-lab#261) were linked under intake epics; unlinked, cause unknown | Logged in the #520 results; not yet acted on |
| Plugin refresh is spread across two machines and three files, and `marketplace remove` edits project settings | Logged here; not yet acted on. A scripted refresh command was suggested. |
| Issue template still offers the retired `research` kind | Intake: vergil-project/.github#541 |
| Two vergil-tooling bugs filed in the wrong org | Re-filed as vergil-project/.github#533 and #534 |
| DB-labs direction split out of mq-gateway-replay-lab#227 | logical-minds-foundry/.github#385 |

## 5. What's next

- The first periodic `deferred-review` pass over the 28 deferred items, the first real use of
  that skill.
- The first `planning-review` pass over the 25 remaining planning items.
- No follow-on brainstorm or epic was opened from this one.

## Appendix A — Operational notes

**Release and install order.** Tooling release (v2.1.270) → T4 install → sweep, with the
plugin release (v2.1.42) alongside. Then the race fix (v2.1.272) mid-sweep, and finally the
plugin install (T8, v2.1.43).

**Sweep results (#520).**

| Home | Intake before → after | Planning before → after | Deferred after |
|---|---|---|---|
| vergil-project/.github | 31 → 16 | 27 → 25 | 21 (#531) |
| logical-minds-foundry/.github | 34 → 22 | none | 4 (#388) |
| logical-minds-foundry/mq-gateway-replay-lab | 4 → 0 | none | 3 (#329) |

vergil-project's intake-after count includes four items filed during the sweep. Full decisions
are in the #520 results comment.

**Plugin marketplace recovery (scriptable).** Run on each machine that runs Claude Code:

```bash
cd /tmp   # outside any repo, so project settings are not rewritten
claude plugin marketplace remove vergil-marketplace
claude plugin marketplace add vergil-project/vergil-claude-plugin@main
claude plugin install vergil@vergil-marketplace
```
