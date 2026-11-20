# Standalone ad-hoc queues — retrospective

- **Epic:** vergil-project/.github#1417
- **Planning issue:** vergil-project/.github#1146
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Retrospective task:** vergil-project/.github#1419
- **Span:** the epic opened at 2026-11-19T18:14Z. Execution began when the
  spec and plan merged, at 2026-11-19T22:33Z. The last task before this
  retrospective closed at 2026-11-20T20:21Z. That is about 26 hours from
  opening and about 22 hours of execution.

## 0. At a glance

**What we set out to do.** Make an ad-hoc epic hold only standalone,
single-issue work, so that being placed in one proves a task is safe to
work alone. Two kinds of task sat there without belonging:

- tasks of another org's epic, because GitHub sub-issues cannot cross orgs;
- the `Logbook:` tasks of `/vergil:hoshin`.

The design gives each its own home: **proxy epics** for cross-org work,
driven only from the primary, and a standing **hoshin epic** for the
logbook. It then re-homes every misplaced task.

**What shipped.** All of it:

- **The hoshin epic** is a standing kind with its own queue view.
  `/vergil:hoshin` files and reads its tasks there. `epic-implement` and
  `issue-implement` refuse it, and the `Logbook:` title skip in the adhoc
  mode is gone.
- **Proxy epics** are made with `vrg-epic-create --proxy-for`, linked by a
  `Primary:` line and trusted `Proxy:` comments.
  - They are exempt from rollup and from the development-epic checks.
  - The daily sweep closes them once their primary closes.
  - `vrg-epic-audit --epic` lists their tasks and fails on link drift.
  - Merge review resolves a proxy task's open tasks and design sources
    through the primary.
  - `epic-create` and `epic-review` route cross-org work to them, and
    `vrg-security-record retarget` moves an advisory to its fix's org.
- **A misplacement report**, in a human-run `vrg-epic-audit`, flags ad-hoc
  tasks that look misplaced. A trusted `Standalone-confirmed` comment
  clears a flag.
- **The migration** surveyed all six orgs' ad-hoc and deferred queues and
  found one real primary, logical-minds-foundry/.github#319.
  - Its proxy is vergil-project/.github#1443, and it took
    vergil-project/vergil-tooling#3188 and #3189.
  - 50 flagged tasks were confirmed standalone.
  - The primary's evolution-entry task is logical-minds-foundry/.github#412.
  - No `Logbook:` task was open in any ad-hoc epic, so the move step
    moved nothing.

**Work delivered** (PR, what it did, merge time in UTC):

- **Spec and plan:** .github #1430, 2026-11-19 22:33.
- **vergil-tooling** (all on 2026-11-20):
  - Task 2a, #4808: the `HOSHIN` standing kind, its sentinel, drain and
    intake-route refusal. 00:00.
  - Task 7, #4809: the proxy model (`is_proxy`, `lib/proxies.py`) and its
    exemptions. 00:15.
  - Task 2b, #4813: `vrg-queue hoshin`. 11:36.
  - Task 8, #4814: `vrg-epic-create --proxy-for`. 11:36.
  - Task 11, #4816: the cross-org advisory move. 11:38.
  - Tasks 9a–9c (batch), #4815: proxy children and link checks, the proxy
    close pass, and the misplacement report. 12:02.
  - Task 10, #4820: merge review's open tasks through a proxy. 12:17.
  - Documentation review, #4839: the docs brought in line with hoshin and
    proxy epics. 20:21.
- **vergil-claude-plugin** (all on 2026-11-20):
  - Task 4, #1217: hoshin files and reads under the hoshin epic, and
    `epic-implement` and `issue-implement` refuse it. 14:17.
  - Task 13a, #1218: `epic-implement` and `epic-retrospective` drive proxy
    tasks from the primary. 14:18.
  - Task 13b, #1219: merge review takes a proxy task's design sources from
    the primary. 14:18.
  - Task 13c, #1220: `epic-create` and `epic-review` place cross-org work
    under proxies. 14:18.
  - Task 6, #1226: the `Logbook:` skip retired. 19:11.
  - Per-repo doc task #1230, #1231: proxy epics in the skills index. 20:19.
- **docs:** per-repo doc task #148, #149: the management pages. 2026-11-20
  20:19.
- **.github:** the plan's evolution entry (new task #1438), #1439.
  2026-11-20 14:17.
- **Ad-hoc fixes this epic's work surfaced** (not tasks of the epic):
  - vergil-tooling #4829 (#4828): `consumer-refresh` runs when only
    `update-deps` failed. 13:02.
  - vergil-tooling #4831 (#4825): landed the blocked 2.1.331 dependency
    update. 13:31.
  - vergil-claude-plugin #1221 (#1216): `epic-create` checks release
    contents with git ancestry. 14:18.
  - vergil-claude-plugin #1222 (#1213): the agent runs `pr-watch` itself
    on a PR reported as not merged. 18:32.

**Releases** (published, UTC, 2026-11-20):

- vergil-tooling v2.1.330 (Tasks 2a–11), 12:27. It was published but not
  installed; see §2.
- vergil-tooling v2.1.331 (adds #4829 and #4831, and was installed), 13:43.
  It served Tasks 3 (#1423) and 12 (#1426).
- vergil-claude-plugin v2.1.82 (Tasks 4 and 13a–13c, plus #1221 and
  #1222), 18:36. It served Tasks 5 (#1424) and 14 (#1427).
- vergil-claude-plugin v2.1.83 (Task 6), 19:23. It served Task 6d (#1425).

**Counts:**

- **Children:** 27. They break down as:
  - the planned 21 tasks;
  - the spec task #1418;
  - the two bookends;
  - #1438, added during execution;
  - the two per-repo doc tasks the sweep spawned.
- **PRs closing them:** 17, plus this retrospective's. Four more were ad
  hoc.
- **Repos touched:** vergil-tooling, vergil-claude-plugin, .github and
  docs. The migration also wrote issue comments in five other orgs:
  logical-minds-foundry, mnemosys-project, mq-rest-admin-project,
  diogenes-project and vergils-nemesis.
- **Security records:** the epic was a security epic. Its advisories are
  GHSA-xj7j-xg99-xhhx (Task 8), and GHSA-8fmf-cp85-xh7r,
  GHSA-pm2g-vjv4-pmvh, GHSA-rhfx-48qr-xfqw, GHSA-rwxm-5qfw-7jmj,
  GHSA-rvjx-26r5-6255 and GHSA-r3mh-rr3j-6qmh (Task 11).
- **Rulings:** one security ruling during execution, `fix`, carried out on
  #4805 under a scope change, plus four epic-level rulings (N0.1–N0.4).

## 1. How the plan evolved

The task list held. All 21 planned tasks ran in the planned order, and the
two groups shipped separately as the plan said they would. The plan's
evolution log has one entry. Execution brought several other changes that
the log does not record, because they touched the tooling and skills
around the epic rather than its task list.

- **Release checks moved from release notes to git ancestry** (the one
  logged entry). The deployment tasks asked the runs to confirm that "the
  release notes list the PRs". The GitHub release body is only a docs link,
  so the step could be met only from the CHANGELOG. Meanwhile the runs were
  already deciding with `merge-base --is-ancestor`, as `issue-deploy`
  prescribes. The human ruled that the git-history check is the one that
  counts:
  - Tasks 6d and 14 were reworded by scope changes;
  - the entry was recorded through #1438;
  - the source wording in `epic-create` was fixed in #1216.
- **Tasks 3 and 12 shared one release.** The hoshin group and the proxy
  group were planned with separate tooling releases. By the time Task 3's
  release was cut, every proxy tooling task had merged too. Task 10 was
  submitted first so that one release, v2.1.330, carried all of it, and
  then v2.1.331 replaced it (see §2).
- **The sweep spawned two doc tasks**, which the plan had foreseen for the
  docs repo:
  - docs#148 for the management pages;
  - vergil-claude-plugin#1230 for proxy gaps in the skills index.

  One proxy item also stayed open in the tooling sweep until #1230: the
  `epic-implement` description.
- **Task 9a–9c ran as one batch.** All three change
  `epic_audit.py` and `vrg_epic_audit.py`, so separate PRs would have
  conflicted on rebase.

## 2. Lessons learned

- **A release that publishes is not a release that installs.** v2.1.330
  was tagged and published, but its `update-deps` step failed on the next
  version's dependency update. `consumer-refresh` then skipped itself, so
  nothing was installed. The human attested "installed on the host and the
  VMs" from the release's overall result, and the next deploy run found
  the VM still on v2.1.329.
  - Fixed in #4828: `update-deps` is optional, and the refresh runs
    whenever only it failed.
  - The lesson for deployment tasks: verify the install where you can,
    because an attestation is only as good as the output it rests on.
- **Check release contents with git, not prose.** Release notes and
  changelogs are a lossy view: a batch PR appears there as its issues, and
  the GitHub release body is only a link. `merge-base --is-ancestor`
  decides it deterministically. It was already in `issue-deploy`, but
  `epic-create` had no rule for the step, so the planner improvised
  (#1216).
- **One sentence of skill prose can override a user's evident intent.**
  `epic-implement` said "Do not run `pr-watch` — … the human invokes" it,
  and the same line was in its description. That made the agent hand the
  `/vergil:pr-watch` command back to the human even after the human had
  pasted the failure. Rewording three places fixed it (#1213). The skill's
  name is the larger fix, and it is in intake as #1214 (pr-repair).
- **Parallel PRs into one file always conflict.** Tasks 2a and 7 both
  extended `labels.json` and the label-count test. Tasks 8 and 9c both
  changed the imports in `lib/proxies.py`. Both conflicts were real and
  were repaired through `pr-watch`, each with a merge review before the
  push. Planning these as independent was right in substance and wrong
  about the merge. When a plan knows two tasks touch the same registry
  file, a batch or an ordering costs nothing.
- **Time-box a flake.** A gpg-agent "Bad passphrase" failure under
  parallel test load blocked the 2.1.331 dependency update.
  - About 3,000 reproduction attempts never reproduced it.
  - The human capped the investigation at the cost of a workaround.
  - #4825 landed a visible, single, test-only retry that names the race,
    with the investigation recorded in the test.
  - The lesson: decide early whether a rare race is worth more than a
    loud workaround.
- **`!` runs where the session runs.** This session ran inside a Lima VM,
  so a `! …` line given for a host install would have run in the VM. The
  deploy run caught it. Host-only steps need plain commands for a Mac
  terminal.
- **The tooling's ordering guards hold up under real use.** Two examples:
  - `vrg-scope-change` refused while a ruling exchange was still open, and
    the scope change went in once the run responded;
  - `vrg-ruling draft` caught an item with no `security` flag.

  Both refusals were correct, and each cost the human only a short wait.

## 3. Compromises and tradeoffs

- **The gpg race is worked around, not fixed.** The root cause is
  unproven. The retry is limited to the test, fires once, and only on that
  exact error, and it raises `GpgAgentBadPassphraseRace` each time it
  fires, so a recurrence is visible. Production signing is unchanged.
- **The proxy create has a short double-create window.** If the proxy is
  created, then the `Proxy:` comment fails, and a re-run follows within
  seconds, a second proxy could be created. The next reader stops with
  "two open proxies" rather than choosing one, so the failure is loud.
  It is recorded in #4800's notes and was not fixed.
- **The misplacement report is a report, never a guard**, by design (spec
  S1.3 and S4.2). A hand-filed task can still reach an ad-hoc epic. The
  report and `Standalone-confirmed` are the backstop, and it runs only
  when a human runs the audit, because the scheduled sweep has no
  trusted-authors configuration (P1.1, intake vergil-tooling#4793).
- **Two releases did the work of one** (v2.1.330 and v2.1.331). v2.1.330
  was published but never installed. An extra release was cheaper than an
  out-of-band install.
- **The migration's 50 `Standalone-confirmed` comments** were posted on
  the strength of a reading of each body. Every one names a closed epic or
  an upstream issue that is not an epic. They were confirmed in bulk on
  the human's ruling, not one by one.

## 4. New problems and opportunities

**The unaddressed-findings sweep.** The sweep covered:

- the 26 children other than this task, their comments, and the merged PRs
  that closed them;
- the epic issue's own comments;
- planning issue #1146's design-review rounds.

No PR carried a review in state COMMENTED or CHANGES_REQUESTED. Every
`Awaiting a ruling` item has a trusted ruling line, and every rejected or
dropped design finding has its reason.

The items, and where each went:

- **The daily sweep has no trusted-authors configuration (P1.1).** Found:
  intake vergil-tooling#4793.
- **`pr-watch`'s name versus its job.** Found: intake
  vergil-claude-plugin#1214.
- **The agent handed the `pr-watch` command back.** Filed and fixed:
  vergil-claude-plugin#1213.
- **`epic-create` worded release checks as release-notes checks (N0.3).**
  Filed and fixed: vergil-claude-plugin#1216, with the plan entry in #1438.
- **`consumer-refresh` skipped when only `update-deps` failed (N0.2).**
  Filed and fixed: vergil-tooling#4828.
- **The blocked 2.1.331 dependency update and the gpg race.** Fixed with a
  workaround in vergil-tooling#4825. The root-cause hunt was dropped on the
  human's ruling.
- **The multi-line `consumer-refresh` script in `vergil.toml`.** Found: the
  human's own tasks vergil-tooling#4827 and #4830.
- **Four stale release tracking issues (N0.1).** Closed on the human's
  ruling.
- **mnemosys-project/.github#26 missing its `archive` label (N0.4).** Fixed
  on the human's ruling.
- **A security follow-up from Task 11.** Ruled `fix` and carried out on
  #4805 under a scope change (counts only).
- **A proxy's title did not name its primary (#1443).** Filed and fixed:
  vergil-tooling#4840 and docs#150. #1443 was retitled by hand.
- **The `--proxy-for` double-create window (#4800).** Dropped on the
  human's ruling, because a second proxy fails loudly.
- **The seven advisories stayed `draft` after their tasks closed.** Fixed:
  the human ran `vrg-security-record sync`, which closed them.

**Opportunities:**

- A plan that knows two tasks touch the same registry file
  (`labels.json`, a label-count test) could say so, and `epic-implement`
  could order or batch them.
- The deployment-task scaffold's attestation could ask the human to paste
  `uv tool list` output rather than assert an install.

## 5. What's next

- **The primary's side:** logical-minds-foundry/.github#319's own driver
  works #412, the plan-evolution entry for the re-homed tasks. Proxy #1443
  stays open until #319 closes, and then its org's daily sweep closes it.
- **The pr-repair rename:** intake vergil-claude-plugin#1214.
- **Trusted authors as central configuration:** intake
  vergil-tooling#4793.

## Appendix A — Operational notes

- **Order of operations that worked:**
  1. the tooling release;
  2. the attestation;
  3. the tooling deploy checks (#1423, #1426);
  4. the plugin tasks;
  5. the plugin release;
  6. the plugin deploy checks (#1424, #1427);
  7. the migration (#1428);
  8. the validation (#1429).

  Task 6 and its release (v2.1.83, #1425) ran after Task 5, in parallel
  with the migration.
- **The release blocks:** a failed `update-deps` leaves a dependency PR
  open plus a fix task, which together gate the next `vrg-release`. Here
  that was #4824 and #4825. Merging the fix PR closes the task, and the
  next release closes the dependency PR itself.
- **Host versus VM:** the agent session runs in `lima-vergil-user`. A host
  install (`uv tool install …`, `vrg-vm update --all`) and attestations
  are run from a Mac terminal.
- **The migration's survey** covered six orgs from their local `.github`
  clones. One quirk: `vrg-queue adhoc` needs every repo with ad-hoc tasks
  to have a main clone under `--root`. Point `--root` at the directory that
  holds every org's clones, not one org's directory.
- **`Standalone-confirmed` comments** count only from a trusted author.
  The agent identity `wphillipmoore-vergil-user` is in the trusted agents
  list, so its comments clear the report.
