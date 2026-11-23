# Remove private-repo support — retrospective

- **Epic:** vergil-project/.github#1476
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Planning issue:** vergil-project/.github#1474

## 0. At a glance

We set out to remove the fleet's last two private repos and then every
private-repo special case in vergil-tooling, vergil-actions and
vergil-claude-plugin, so that every epic and queue homes in `<org>/.github`
and nothing branches on repo visibility. That shipped.
logical-minds-foundry/mq-gateway-replay-lab is public and its queues and
ad-hoc work live in logical-minds-foundry/.github.
logical-minds-foundry/mq-protocol-gateway is archived. vergil-actions v2.1.89
drops the private-repo SARIF paths and every `actions: read` request.
vergil-tooling v2.1.336 removes self-homing, the visibility guards, GHAS
inference, the token-coverage guard and the visibility primitives.
vergil-claude-plugin v2.1.85 and v2.1.86 carry the skill changes. 20 of the 28
caller repos dropped their now-unneeded `actions: read` grant. The other 8 were
blocked by problems that predate the epic and were moved, open, to their repos'
ad-hoc epics.

Work delivered, by PR. The task is in brackets. Every PR merged on the date
shown.

| PR | Task | What it did | Merged |
| --- | --- | --- | --- |
| .github#1484 | .github#1477 | Spec and plan | 2026-11-22 |
| lmf/mq-gateway-replay-lab#344 | #342 (Task 3) | Lab stops setting `upload-sarif` | 2026-11-22 |
| lmf/mq-resiliency-lab-for-linux#1752 | #1751 (added) | Lock upgrades so the docs job installs under Python 3.15, plus 1 security update | 2026-11-22 |
| lmf/mq-resiliency-lab-for-linux#1750 | #1748 (Task 4) | Stops setting `upload-sarif` and `ghas` | 2026-11-22 |
| vergil-actions#1355 | #1349 (Task 5) | `ci-security` loses the private-repo SARIF paths and every `actions: read` | 2026-11-22 |
| vergil-tooling#4943 | #4901 (Task 8) | Every epic and queue homes in the org `.github` | 2026-11-22 |
| vergil-tooling#4944 | #4906 (Task 13) | Counts-only Dependabot report for every repo | 2026-11-22 |
| vergil-tooling#4948 | #4902 (Task 9) | Epic-audit visibility passes removed, plus unused `home` arguments | 2026-11-22 |
| vergil-tooling#4950 | #4903 (Task 10) | Token-coverage guard removed, plus unused queue-kind branches | 2026-11-22 |
| vergil-tooling#4954 | #4905 (Task 12) | Routing, linking, proxy and security-records visibility guards removed | 2026-11-22 |
| vergil-tooling#4958 | #4904 (Task 11) | `[project].ghas` retired; GHAS inference and private scaffolding removed | 2026-11-22 |
| vergil-tooling#4959 | #4907 (Task 14) | Visibility primitives removed; `repo_exists` gets its own probe | 2026-11-22 |
| vergil-claude-plugin#1249 | #1239 (Task 16) | hoshin reports every repo by name | 2026-11-22 |
| vergil-containers#992 | #991 (added) | Fixes 1 new HIGH image-scan finding in a base-image Python dependency | 2026-11-22 |
| vergil-claude-plugin#1276 | #1240 (Task 17) | Private-home branches removed from the epic skills | 2026-11-23 |
| vergil-tooling#5029 | #4893 (doc review) | Site docs lose the private-repo leftovers | 2026-11-23 |
| vergil-actions#1372 | #1370 (added) | Docs lose the itemized-report, self-homed sweep and GHAS-condition passages | 2026-11-23 |
| vergil-containers#1005 | #1003 (added) | Docs lose a public-repo qualifier | 2026-11-23 |
| docs#168 | docs#166 (added) | Management docs lose the private queue homes and dead links | 2026-11-23 |

Task 7, pass 2 of the caller cleanup: 20 PRs, each removing the
`security:` job's `actions: read` grant and its comment. The six fixture PRs
also carry that fixture's fixture-run change.

| PR | Task | Merged |
| --- | --- | --- |
| lmf/.github#419 | lmf/.github#417 | 2026-11-22 |
| lmf/docs#37 | lmf/docs#36 | 2026-11-22 |
| lmf/mq-gateway-replay-lab#345 | #343 | 2026-11-22 |
| lmf/mq-resiliency-lab-for-linux#1753 | #1749 | 2026-11-22 |
| mnemosys-project/.github#119 | #118 | 2026-11-22 |
| mnemosys-project/docs#32 | #31 | 2026-11-22 |
| .github#1509 | #1483 | 2026-11-22 |
| docs#159 | docs#156 | 2026-11-22 |
| packages#62 | packages#61 | 2026-11-22 |
| vergil-claude-plugin#1254 | #1238 | 2026-11-22 |
| vergil-python#97 | #96 | 2026-11-22 |
| vergil-tooling#4957 | #4900 | 2026-11-22 |
| vergil-vm#443 | #442 | 2026-11-22 |
| vergil-containers#985 | #982 | 2026-11-23 |
| vergil-fixture-python-app#50 | #38 and #39 | 2026-11-22 |
| vergil-fixture-python-lib#57 | #45 and #46 | 2026-11-22 |
| vergil-fixture-rust-app#65 | #53 and #54 | 2026-11-22 |
| vergil-fixture-rust-lib#71 | #59 and #60 | 2026-11-22 |
| vergil-fixture-ts-app#67 | #55 and #56 | 2026-11-22 |
| vergil-fixture-ts-lib#66 | #54 and #55 | 2026-11-22 |

No PR: Task 1 (lmf/.github#416, publish the lab), Task 2 (lmf/.github#415,
archive the gateway), Task 6 (vergil-actions#1350, release), Task 15
(vergil-tooling#4908, release), and the six fixture release and six fixture
validation tasks. All of them recorded `Outcome: SUCCESS`.

`lmf/` is `logical-minds-foundry/`. Unprefixed repos are in `vergil-project`.

- **Repos touched:** 21 with merged PRs:
  - vergil-project: `.github`, `docs`, `packages`, `vergil-actions`,
    `vergil-claude-plugin`, `vergil-containers`, `vergil-python`,
    `vergil-tooling`, `vergil-vm` and the six `vergil-fixture-*` repos;
  - logical-minds-foundry: `.github`, `docs`, `mq-gateway-replay-lab` and
    `mq-resiliency-lab-for-linux`;
  - mnemosys-project: `.github` and `docs`.

  Changed without a PR: logical-minds-foundry/mq-protocol-gateway (archived)
  and the lab's visibility.
- **Counts:**
  - 70 tasks were linked at some point, counting the bookends: 65 planned
    and 5 added during execution (#1751, vergil-containers#991, docs#166,
    vergil-actions#1370, vergil-containers#1003).
  - 8 Task 7 rows were moved out of the epic, unfinished. 62 remain, all
    closed but this retrospective.
  - 39 merged PRs: 38 for tasks and the spec PR.
  - 45 merge-review runs: 2 defects raised and fixed, 1 design question
    ruled keep, no security findings.
  - 2 security items came up during execution, both fixed (see §3).
- **Releases carrying the work:**
  - vergil-actions v2.1.89: Task 5.
  - vergil-tooling v2.1.336: Tasks 8 to 14 and vergil-tooling's Task 7 row.
  - vergil-claude-plugin v2.1.85: Task 16 and the plugin's Task 7 row.
    v2.1.86: Task 17.
  - vergil-containers v2.1.48: #992 and the Task 7 row #985.
  - The fixtures: python-app v0.1.4, python-lib v0.1.5, rust-app v0.1.5,
    rust-lib v0.1.6, ts-app v0.1.7, ts-lib v0.1.7.
  - Not yet in a release: the docs-only PRs vergil-actions#1372,
    vergil-containers#1005 and vergil-tooling#5029.
- **Span:** planning issue opened 2026-11-22 13:28 UTC, epic opened 13:40.
  The last task closed 2026-11-23 16:30. The epic closes when this
  retrospective's PR merges. Most of the work ran in about six hours on
  2026-11-22; the second day was the release, Task 17, the documentation
  review and the moves.

## 1. How the plan evolved

The plan's order held: operational tasks first, then the callers and
vergil-actions in an order that never broke CI, then vergil-tooling one
subsystem per PR, then the skills. Review focus 1 (CI broken by the wrong
removal order) never happened: no caller's CI failed because of what this epic
removed. What moved the plan was everything around the change itself.

**The design shrank before any code.** The first spec had a runtime guard that
refused a private repo and a config-audit check that every repo is public. The
first spec pushback round showed that the audit could not see every repo. The
human removed both: the invariant rests on the human's enumeration of the only
two private repos, and the code is written to it with no check. Most of the
later simplicity came from that ruling.

**The operational tasks met corners the tooling did not cover.** Closing the
gateway's intake epic made the org automation open a fresh archive issue 51
seconds later, so the repo was archived with one open issue (accepted;
race filed). Moving the lab's deferred items found no supported way to move a
deferred item between repos. Routing each one to intake, then re-deferring it
with its original record, worked.

**Pre-existing CI rot caused most of the stops, not the epic's changes.** Four
separate problems predating the epic stopped tasks: a vulnerable locked dev
dependency in the resiliency lab, that repo's docs job failing under Python
3.15 for want of wheels and a compiler, a new image-scan finding that blocked
every vergil-containers PR, and melete's dev container needing an `npm` the
base image no longer has. The first three were fixed in or beside the epic,
through scope changes and two added fix tasks. The lab's two lock PRs
deadlocked until #1751 was widened to merge alone. The last one, and the same
wheel problem in mq-resiliency-observability, were left to their repos.

**Removing private-repo code exposed more code to remove.** Twice a run noticed
generality that had lost its last caller: the `home` arguments and repo-scoped
banners in `lib/epic_audit.py`, then the planning and deferred queue-kind
branches. Each went into the next chained task that edited the same file.
Task 11 also dropped an argument the plan had kept, for the same reason.

**The fixtures were run against the convention, by ruling.** The human granted
this epic the fixture repos while another epic's trios sat open and unstarted,
and approved putting each fixture's Task 7 row in the same PR as its
fixture-run change, though for the app fixtures that adds a command to a
batch. All six trios passed, and CodeQL ran in full on every fixture under
v2.1.89, which is the check the fixture runs existed for.

**The tail was cut rather than waited on.** Eight Task 7 rows could not finish
for reasons outside the epic. The human first ruled that the epic does not
block on them, then moved them to their repos' ad-hoc epics. The vergil-tooling
release waited on another epic's release block, and its install was taken on
the human's attestation rather than per-host evidence.

## 2. Lessons learned

- **A design-review discard is not an instruction.** Plan pushback round 1
  discarded "some callers have no comment above the grant" with "the
  implementer adapts". The task text never said how. It said to delete the
  grant and the comment above it, and never to touch a grant with another
  reason. Faced with a grant and no comment, the agent correctly could not
  tell which case it was in, and six rows stopped. When a reviewer's point is
  discarded because the implementer will handle it, the plan should say how,
  in the task.
- **A fleet-wide caller pass inherits the fleet's CI health.** The epic's own
  changes broke nothing, but touching 28 repos meant running validation in
  repos nobody had worked in for days, and three of them were already broken.
  All three breakages were already filed, on 2026-11-12: melete#265,
  mq-resiliency-observability#72 and mq-resiliency-lab-for-linux#1738. The
  run did not find them and rediagnosed each one.
  Next time, search a repo's open issues for a failing check before
  diagnosing it, and before filing a fix task.
- **Two PRs that each need the other's lock change deadlock.** The way out
  was to make one of them self-sufficient. When a fix task is filed to unblock
  another PR, widen it at once to everything that PR's CI needs.
- **The per-repo shape cost human pushes.** One PR per caller repo kept each
  change trivial to review, but every one touched `.github/workflows/**`, so
  every one needed the human's `vrg-submit-pr` to push it. With four other
  PRs that touched workflows, that was 24 of the epic's 39 PRs, most of them
  one-line deletions. Batching the fixtures halved their share.
  The same choice next time should be weighed against one batch per org or a
  config-sync change.
- **Let the removal keep going.** Folding newly unused code into the next
  chained task was cheap, because that task was already editing the same file.
  Tell implementers to report generality a removal leaves unused, so it can be
  folded in, as these runs did.
- **Generated-issue automation needs to know about retirement.** Closing a
  repo's issues before archiving it can make the automation open new ones.
  Any retire procedure should re-list after the last close, not only before
  it.

## 3. Compromises & tradeoffs

- **No runtime check of the all-public invariant.** By the human's ruling at
  spec round 1, nothing detects a private repo if one appears. A repo made
  private later would meet code that assumes it is public, with no error that
  names visibility. Accepted, because the fleet moves to Forgejo in 2027 and
  the visibility model is rethought there.
- **Eight Task 7 rows left undone, and the spec's goal 1 not fully met.** The
  six mq-rest-admin-project repos, melete and mq-resiliency-observability
  still grant `actions: read` in their `security:` job. The grant is now
  unneeded and harmless, since no workflow requests it. melete has its commit
  on `feature/276-drop-actions-read`; mq-resiliency-observability#83 is open
  with auto-merge on and red. The mq-rest-admin rows record no work on
  their own issues; their stop is recorded on the epic. Their blockers
  predate the epic. The owner of each repo takes on the breakage
  when work resumes there.
- **One open issue stays in an archived repo.**
  logical-minds-foundry/mq-protocol-gateway#48 cannot be closed without
  unarchiving the repo, which the human ruled against.
- **Two security items fixed inside the epic.** A vulnerable locked dev
  dependency flagged by the audit stage (resiliency lab, fixed in #1748's
  branch and #1751) and a new HIGH image-scan finding in a base-image Python
  dependency (vergil-containers#991). Both were fixed, not recorded as
  advisories. One merge review's public step comment named the first fix in
  detail before it merged. The human left it as it was and filed the gap
  (vergil-project/vergil-claude-plugin#1264).
- **The fixture batches broke a guideline.** Batching each Task 7 row with its
  fixture-run change put a new command in a batch for the three app fixtures.
  The human approved it as reasonable for fixture dummy changes; the guideline
  was not amended, and no exception note was filed, by ruling.
- **Evidence rests on attestation in two places.** Task 15's install on the
  host and on the identity VMs other than the one the agent checked rests on
  the human's attestation, by ruling, to avoid overvalidation. Task 3's
  post-merge `vrg-github-repo-config audit` of the lab was not recorded on
  #342. The lab reached public-repo settings through the human's config
  update, also not recorded on a task.
- **The deferral record lost a line in an experiment.** Trying
  `vrg-intake-route defer` on an already-deferred item refreshed
  mq-gateway-replay-lab#225's record in place and dropped its `Deferred from:`
  line before the working route was found.
- **The documentation review kept one near-miss.** A passage in
  vergil-tooling's `guides/account-setup.md` still explains GitHub's
  behaviour on a private repo. The §5 grep misses it because bold markup
  splits the phrase. Its merge review judged it accurate and not a defect.

## 4. New problems & opportunities

**Filed during the epic, by human ruling (both open, minor):**

- vergil-project/vergil-tooling#5005: retiring a repo by closing its intake
  epic opens an archive issue that the archive then freezes open.
- vergil-project/vergil-claude-plugin#1264: merge review can name, in its
  public step comment, a security fix ruled on outside it.

**Declined by the human:** a follow-up on the workflow push rule (an agent's
rebase push to a PR with workflow changes went through, because the rebase
changed no workflow content), and an exception note for the batching
guideline.

**Unaddressed-findings sweep.** 69 tasks swept: the 61 children other than this
retrospective, including the proxy tasks, and the 8 Task 7 rows moved out. Also
swept: the epic's comments, the planning issue's 6 design-review round comments, and
the 39 merged PRs' reviews and conversations. Every `Awaiting a ruling` item has
a closing ruling. The 2 security items are covered by `Human ruling:
security:` counts lines. Every rejected design-review finding has its reason.
No PR has a review in `COMMENTED` or `CHANGES_REQUESTED` state. Problems noted
but not tracked by a task or ruling, each with the human's ruling at review:

1. **Found.** The plan-review delegate closing its security section with a
   second start marker (planning rounds P1 and A1) is
   vergil-project/vergil-claude-plugin#1037, open.
2. **Found.** melete's missing `npm` is mnemosys-project/melete#265, open
   since 2026-11-12. It predates vergil-containers#970, so the base-image
   change suspected during the run is not its cause; the cause is not known.
3. **Found.** mq-resiliency-observability's docs job is
   logical-minds-foundry/mq-resiliency-observability#72, open.
4. **Found, closed.** The resiliency lab's same docs-job failure was
   already filed as logical-minds-foundry/mq-resiliency-lab-for-linux#1738,
   still open though #1752 had fixed it. Closed as completed by #1752, by
   ruling.
5. **Found.** The six mq-rest-admin-project grants with no private-repo
   comment are tracked by their own open Task 7 rows.
6. **Filed.** `vrg-intake-route defer` on an already-deferred item
   refreshes the record in place and silently drops its `Deferred from:`
   line (Task 1). A refresh that loses provenance is a silent data loss.
   Filed in intake as minor: vergil-project/.github#1539.
7. **Dropped.** There is no tooling path to move a deferred item between
   repos' deferred epics (Task 1). The case arose only because a repo stopped
   being self-homed, and no self-homed queue remains.
8. **Dropped.** The `account-setup.md` private-repo passage (§3), because it
   is accurate.
9. **Dropped.** Task 3's post-merge config audit was not recorded (§3),
   because the human's config update brought the lab to public-repo
   settings.

## 5. What's next

- The eight moved Task 7 rows wait in their repos' ad-hoc epics:
  mq-rest-admin-project/.github#52, mq-rest-admin-common#235,
  mq-rest-admin-dev-environment#179, mq-rest-admin-go#331,
  mq-rest-admin-python#535 and mq-rest-admin-rust#130,
  mnemosys-project/melete#276 (after melete#265), and
  logical-minds-foundry/mq-resiliency-observability#82 (after #72).
- The Forgejo migration in 2027 redesigns the public/private model from
  scratch (spec §1.1). This epic leaves it nothing GitHub-specific to port.
- The three filed follow-ups: vergil-project/vergil-tooling#5005,
  vergil-project/vergil-claude-plugin#1264 and vergil-project/.github#1539.

## Appendix A — Operational notes

**Order that kept CI green.**

1. Task 2 archived the gateway; Task 1 confirmed the lab's flip to public,
   then filed the lab's two tasks under the proxy (a proxy refuses a private
   target).
2. Pass 1: the lab (#342) and the resiliency lab (#1748, after #1751) stopped
   passing `upload-sarif` and `ghas`. Dropping an input the reusable workflow
   still declares is harmless.
3. Task 5 removed the input and every `actions: read` request from
   `ci-security.yml`; Task 6 released v2.1.89 and confirmed `v2.1` moved.
4. Pass 2: every caller dropped its grant only after the release, since
   dropping a grant that a workflow still requests fails at startup.
5. vergil-tooling Tasks 8, 9, 10 then 12 ran chained, because they edit the
   same files; 11 and 13 ran in parallel; 14 last; then the v2.1.336 release
   and Task 17.

**Gotchas met.**

- Closing a repo's intake epic triggers the org automation's archive issue
  within a minute. Re-list open issues after the last close, right before
  archiving.
- `vrg-epic-move` refuses a deferred target. To move a deferred item to
  another repo's deferred epic: `vrg-intake-route <ref> intake`, then
  `vrg-intake-route <ref> defer` with the original reason, revisit condition
  and `--date`.
- The nightly sweep had not moved the lab's queues by the time Task 1 ran, so
  the conditional step in Task 1 was needed.
- The docs job resolves the newest uv-managed Python, 3.15 here, because
  `requires-python` has no upper bound. Any locked package with a C extension
  and no cp315 wheel then needs a compiler, which the CI container lacks.
  Check every compiled dependency for a cp315 or abi3 wheel, not only the one
  in the log.
- When two PRs each need the other's lock change, widen one to everything its
  CI needs, merge it, and rebase the other; its duplicate lock commit drops
  out.
- GitHub did not auto-close two tasks when their PRs merged: docs#156
  (docs#159) and vergil-fixture-python-lib#46 (batch PR #57). Each was
  closed with a "Completed by PR" comment, which `vrg-finalize-pr` does by
  design (vergil-tooling#4002).
- An unrelated open worktree in vergil-actions (#1206, mid-rebase) was left
  for the next ad-hoc sweep.
