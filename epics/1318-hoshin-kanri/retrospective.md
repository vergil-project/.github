# Hoshin kanri: a review cadence and logbook for the fleet — retrospective

- **Epic:** vergil-project/.github#1318
- **Spec:** [`spec.md`](spec.md) · **Plan:** [`plan.md`](plan.md)
- **Planning issue:** vergil-project/.github#471

## 0. At a glance

We set out to give the fleet a regular way to review what ran unattended and to
revisit direction: one skill, `/vergil:hoshin start|end [<level>]`, that runs
daily and boundary reviews on the 方針管理 hōshin kanri model and leaves dated
entries in a public logbook on the docs site. All of it shipped. The logbook
section and background page are live. The `hoshin` skill was released in
vergil-claude-plugin v2.1.79, with its security rules tightened in v2.1.85. The
first `/vergil:hoshin start` bootstrapped the logbook with five `start` entries
and a `hoshin.md` of four objectives. Two late fixes, vergil-claude-plugin#1277
and #1278, are merged but not yet in a plugin release.

Work delivered, by PR (task in brackets, merge date last):

- **.github#1334** (#1319): spec and plan. 2026-11-18.
- **docs#140** (docs#138): logbook tree, entry format, fleet roster, background
  page, nav. 2026-11-18.
- **vergil-claude-plugin#1172** (#1166): `hoshin` run procedure, calendar,
  logbook and security rules, logbook guards. 2026-11-18.
- **vergil-claude-plugin#1173** (#1168): per-level checklists and checks; fixed
  3a's reference screen. 2026-11-18.
- **No PR** (#1170): deployment of v2.1.79 on the host and VMs, check briefs
  posted, .github#1354 filed. 2026-11-18.
- **docs#155** (docs#154, not an epic child): the bootstrap run, five `start`
  entries and `hoshin.md`. 2026-11-22.
- **No PR** (docs#139): validation of the bootstrap, SUCCESS, two findings fixed
  forward. 2026-11-23.
- **vergil-claude-plugin#1268** (#1263): security rules keep a failure's effect
  and onset, and the kind of finding, out of entries. 2026-11-23.
- **vergil-claude-plugin#1269** (#1266): two wording fixes in the security
  reference and the skill's table. 2026-11-23.
- **docs#162** (docs#136): documentation review, `hoshin` and `pr-watch` entries
  on tools-and-skills. 2026-11-23.
- **vergil-claude-plugin#1279** (#1277): `pr-watch` fixes logbook PRs, skipping
  merge review for them. 2026-11-23.
- **vergil-claude-plugin#1280** (#1278): "Epic drift" treats exit 1 as an audit
  problem, not a partial report. 2026-11-23.
- **docs#165** (docs#164): the docs side of #1277. 2026-11-23.

- **Repos touched:** vergil-project/.github, vergil-project/docs,
  vergil-project/vergil-claude-plugin.
- **Counts:**
  - 13 tasks, counting the retrospective: 8 planned and 5 added during
    execution.
  - 12 PRs: 11 for epic tasks, plus the bootstrap logbook PR.
- **Plugin releases carrying the work:**
  - v2.1.79: #1166 and #1168.
  - v2.1.85: #1263 and #1266.
  - Not yet released: #1277 and #1278. The next release is held up by another
    epic.
- **Span:** opened 2026-11-18, closed 2026-11-23 with this retrospective. The
  bootstrap waited for a morning set aside for it.

## 1. How the plan evolved

The plan's shape held: logbook tree and skill machinery in parallel, then
checklists, release, bootstrap, and the bookends. What moved was the depth of
the checklists and how the security rules read in practice.

**The checklists were cut back to a skeleton.** Task 3b's first merge review
judged the checklists as if they were final, and asked for exhaustive queries
and per-repo detail. The human ruled that this pass sets shape and structure
only: fine detail comes from iterating on real reviews and from the tools that
will mechanize the checks (#1140, #1316, #1317). That ruling went onto #1168 as
a scope change. It also carried a fix to Task 3a's screen, whose reference check
called a command the user agent cannot run, and a correction when "drop
`--limit`" turned out to shorten the list it was meant to complete.

**Deployment needed the human more than the plan said.** Task 4 listed the
plugin refresh commands as agent-safe; both refuse the agent identity, so the
human ran them. That labelling is fixed outside the epic (#1177).

**The bootstrap surfaced two security-wording gaps.** The first real run's
entries passed the mechanical screen but carried two lines the rules mean to
keep private. Merged entries are never edited, so validation was recorded as
SUCCESS and the skill was fixed forward (#1263, then #1266 for two older wording
problems its merge review found). The human installed that release by hand, with
no deployment task.

**One spec decision was reversed.** The spec had `pr-watch` refuse any
`Logbook:` PR, so that no fix could push text past the run's screen. At the
documentation review the human ruled that wrong: a failed PR needs fixing, and
its fixes are mechanical. #1277 lets `pr-watch` fix logbook PRs, skipping merge
review for them. docs#164 followed because the documentation review had already
merged the old wording.

**Outside changes moved two details.** Another epic removed `epic-implement`'s
adhoc-mode `Logbook:` skip (vergil-claude-plugin#1226), and logbook tasks now
live under their own hoshin epic rather than the docs repo's ad-hoc epic the
spec names. The documentation review found the docs already matched.

The plan's "Evolution during execution" log was left empty while the epic ran.
Its entries are written with this retrospective.

## 2. Lessons learned

- **Say the pass's depth up front.** A first pass of review content needs its
  level of detail written into the task. Without it, merge review judges against
  an imagined final version, and the human spends gate time on details meant for
  later. Next time, the plan states "skeleton" for any task whose content will
  be refined by use.
- **The first real run is the security test.** The mechanical screen caught
  identifiers, but the leaks were in prose. Running the skill for real, and
  having a validation task read its output against the rules, found what no
  review of the skill's text did.
- **Keep the plan's evolution log during the run, not after.** The log stayed
  empty while five tasks and four scope changes arrived. The record survived in
  issue comments, but writing each entry when it happens is cheaper than
  reconstructing them.
- **Check command identity in the plan.** Two deploy steps were labelled
  agent-safe from memory. A one-line check of each command under the agent
  identity, when writing a deployment task, would have avoided a stop.
- **A guard that blocks repair has to say how repair happens.** The `pr-watch`
  rule protected the screen but left a failing logbook PR with no fix short of
  redoing the review. Any "never do X" rule needs a stated path for the failure
  it creates.

## 3. Compromises & tradeoffs

- **Skeleton checklists, by choice.** Several checks are knowingly incomplete:
  release windows capped at 1000, archived repos not tracked, checks with no
  tool yet that rely on ad hoc `vrg-gh` queries. They will be refined through
  use and mechanized under #1140, #1316, #1317 and #1354.
- **The bootstrap's entries stay as merged.** Two lines in the first daily entry
  break the tightened rules. The logbook never edits a merged entry, so they
  remain. The rules were fixed forward, not the record.
- **Logbook PRs skip merge review, now including `pr-watch`'s fixes.** We rely
  on the run's own screen and the human's approval before the PR exists, and
  treat later fixes as mechanical. A fix that changes meaning would pass
  unreviewed; we accept that for a documentation run.
- **The retrospective ships before the release of #1277 and #1278.** Until the
  next plugin release, installed copies still refuse to fix logbook PRs and
  still read "Epic drift" exit 1 the old way.
- **Some checks rest on the human or are unverifiable.** The user identity
  cannot read issue timelines or Dependabot alerts, so the bootstrap's finalize
  step was checked indirectly. The security checks read the nightly ops results
  instead.

## 4. New problems & opportunities

**Filed during the epic:**

- **Open:**
  - .github#1354: planning issue for a logbook reader tool.
  - vergil-claude-plugin#1265: operational state directories must be created at
    mode 700. Ad-hoc epic.
- **Closed:**
  - vergil-claude-plugin#1177: deploy steps that refuse the agent belong under
    Human-run steps. Ad-hoc epic.
- **Check briefs** posted on #1140, #1316 and #1317 for the checks that have no
  tool yet.

**Security items:**

- 1 raised in this session, filed as #1265.
- 2 from the bootstrap validation, fixed forward through #1263.
- 11 from design review, all accepted into #1166 and #1168.

**Unaddressed-findings sweep:**

- **Scope:** 12 child tasks, the epic issue and planning issue #471.
- **Rulings and PRs:** every ruling and filing was confirmed. No PR had review
  comments.
- **Design review:** every finding on #471 was accepted, and its one out-of-spec
  item is intake docs#137.

Five items were untracked:

1. The plan's evolution log was empty. **Fixed:** this PR adds the entries.
2. The spec names the docs ad-hoc epic for logbook tasks; a per-org hoshin epic
   is used. **Dropped by the human:** recorded in the evolution log, and the
   spec stays as written.
3. The bootstrap's yearly repo counts for one org differ from today's. **Dropped
   by the human:** a repo changed after the run.
4. The user identity cannot read issue event timelines, so finalize runs cannot
   be verified directly. **Dropped by the human:** a known limit of the `gh api`
   denial.
5. Merge review sees only the human's comments, so it could not check an answer
   the agent recorded on the human's behalf (docs#138). **Filed:** intake
   .github#1534.

## 5. What's next

- Run `/vergil:hoshin` daily. The skill itself will be refined from the
  logbook's kaizen notes.
- Cut the plugin release that carries #1277 and #1278 once the other epic's
  blocker clears.
- The status and history tools the checks point to: #1140 (`vrg-org-status`),
  #1316 (`vrg-fleet-status`), #1317 (`vrg-*-history`), and #1354 (logbook
  reader).
