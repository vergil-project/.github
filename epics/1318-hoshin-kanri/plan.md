# Hoshin kanri: a review cadence and logbook for the fleet — implementation plan

- **Epic:** vergil-project/.github#1318
- **Planning issue:** vergil-project/.github#471 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** each code task is one issue and one PR in the repo
> named in its heading, worked with `issue-implement`; the deployment task is
> worked with `issue-deploy`, and the bootstrap validation with
> `issue-validate`.

**Goal:** The human runs `/vergil:hoshin start` each morning (and, when they
choose, `/vergil:hoshin end [<level>]`), and each run reviews the fleet at the
right levels and leaves dated entries in a public logbook on the docs site.

**Architecture:** Two pieces of Markdown. The docs repo gets the logbook tree,
its entry format and the hoshin background page. The plugin gets one skill
whose `SKILL.md` carries the run procedure and whose reference files carry
the calendar rules, the logbook and PR rules, the security rules and one
checklist per level. There is no code: the agent follows the skill, and every
data check is written so it can later become a `vrg-*` tool.

**Tech Stack:** Markdown, MkDocs (Material) for the docs site, Claude Code
plugin skills.

**Spec:** [`spec.md`](spec.md) in this directory.

## Global Constraints

- Every path this epic creates is ASCII. Japanese terms appear only in
  content.
- The logbook lives at `docs/site/docs/logbook/` in `vergil-project/docs`,
  with `daily/`, `weekly/`, `monthly/`, `quarterly/` and `yearly/`, each with
  an `index.md`.
- File names: `daily/YYYY-MM-DD-<mode>.md`, `weekly/<Monday's
  YYYY-MM-DD>-<mode>.md`, `monthly/YYYY-MM-<mode>.md`,
  `quarterly/YYYY-Qn-<mode>.md`, `yearly/YYYY-<mode>.md`; `<mode>` is `start`
  or `end`.
- Weeks run Monday to Sunday. Every period comes from the week's Monday.
- Every date and time is computed in the IANA time zone recorded in
  `logbook/index.md`, never from the host clock.
- A run's task title and commit description are `Logbook: <date> <mode>`; its
  task body and `report-ready` fields name only the entries written, by path.
  Commits are typed `docs(logbook)`.
- The logbook is public: the "Security" rules of the spec apply to every
  public record a run creates.
- The skill runs as the `user` agent and never submits, merges or releases.
- Validation runs through `vrg-container-run -- vrg-validate` in the task
  repo's worktree.
- No fixture runs: the epic changes no language support, image, shared
  workflow or release tooling.

## Review Focus

Each line names an input the spec implies and the behaviour expected; the
owning task pins it as a worked example in the skill's calendar or logbook
reference, which its reviewer checks by hand.

- **The year-end week.** On Thursday 2026-01-01, the current week is the week
  of Monday 2025-12-29, which belongs to December 2025, Q4 2025 and 2025; no
  January, Q1 or 2026 review is due until Monday 2026-01-05 (Task 3a,
  `references/calendar.md`, example 1).
- **An `end` after midnight.** `end` at 00:30 on Friday, with Thursday's
  `start` the latest daily entry and no Thursday `end`, writes
  `daily/<Thursday>-end.md` covering Thursday to midnight, with
  `late: false`, and writes nothing for Friday (Task 3a, example 2).
- **An `end` on the same day.** `end` at 18:00 on Thursday writes
  `daily/<Thursday>-end.md` with `window_to` 18:00, never midnight; Friday's
  `start` window opens at 18:00 Thursday, so the evening is reviewed (Task
  3a, example 4).
- **Monday after a weekend off, Thursday left open.** `start` on Monday finds
  Thursday's `start` unclosed (offers the one-keystroke "not performed"),
  finds the week unclosed (offers late or not performed), names no days as
  gaps, and opens the new week; the daily window runs from Thursday's
  `start` window end (Task 3a, example 3).
- **A preflight conflict.** An open logbook PR, or a logbook task still open
  in the docs repo's ad-hoc epic, stops the run before any review, with the
  PR or task named (Task 3a, `SKILL.md` preflight).
- **A failed run in a private repo, and a failed security job in a public
  one.** The entry shows "1 failed run in a private repo" with no name, and
  "a security-related failure" with a security-record ref, never the cause
  (Task 3a, `references/security.md`, examples A and B).
- **An entry that breaks a security rule.** Before any commit, the screen
  stops a run whose entry text holds a `GHSA-` or `CVE-` ID, an alert number,
  a private repo's name or a per-repo alert count, naming the match; nothing
  is committed or pushed (Task 3a, `references/security.md`, example C).

---

## Tasks

| Task | Repo | Kind | Blocked by |
| --- | --- | --- | --- |
| 1 | .github | task (#1319) | — |
| 2 | docs | task | 1 |
| 3a | vergil-claude-plugin | task | 1 |
| 3b | vergil-claude-plugin | task | 3a |
| 4 | vergil-claude-plugin | deployment | 2, 3b |
| 5 | docs | validation | 4 |
| — | docs | bookend (#136) | 5 |
| — | .github | bookend (#1320) | all |

- **Task 1:** the spec and plan (this PR).
- **Task 2:** the logbook tree, entry format, fleet roster, background page
  and nav.
- **Task 3a:** the `hoshin` skill's procedure, calendar, logbook, PR and
  security rules.
- **Task 3b:** the `hoshin` checklists and checks, one reference per level.
- **Task 4:** a plugin release with the skill, installed on the host and VMs,
  and the merged check briefs posted to their planning issues.
- **Task 5:** the bootstrap, the first `/vergil:hoshin start` on the empty
  logbook.
- **Bookends:** the documentation review (docs#136) and the retrospective
  (#1320).

Tasks 2 and 3a are independent and land in either order. 3b is split from 3a
because the checklists are content a reviewer can reject while accepting the
run machinery.

### Task 2: The logbook tree and background page (docs)

**Files:**

- Create: `docs/site/docs/logbook/index.md`
- Create: `docs/site/docs/logbook/hoshin.md`
- Create: `docs/site/docs/logbook/fleet.md`
- Create: `docs/site/docs/logbook/{daily,weekly,monthly,quarterly,yearly}/index.md`
- Create: `docs/site/docs/management/hoshin-kanri.md`
- Modify: `docs/site/mkdocs.yml` (nav), `docs/site/docs/management/index.md`
  (link the new page)

**Interfaces:**

- Produces, for Task 3a: the entry header and the index line formats below,
  and the time-zone line in `logbook/index.md`.

The entry header is YAML front matter, exactly these keys:

```yaml
level: daily            # daily | weekly | monthly | quarterly | yearly
mode: start             # start | end
period: 2026-11-18      # the file name's period part
window_from: 2026-11-17T18:42-05:00   # omitted on a not-performed entry
window_to: 2026-11-18T08:05-05:00     # omitted on a not-performed entry
ran_at: 2026-11-18T08:05-05:00
late: false
not_performed: false
```

An index line is `- [<period> <mode>](<file name>)`, newest first, under a
level heading; an entry that ran late or was not performed adds a space and
`(late)` or `(not performed)` at the end of its line.

- [ ] **Step 1: Ask the human for the fleet's org names**, each with active or
  inactive, and the IANA time zone. Record their answer in a comment on the
  task before using it.
- [ ] **Step 2: Write `logbook/index.md`:** what the logbook is (one
  paragraph, from the spec's Intent); a link to the background page, to
  `hoshin.md` and to each level's index; the line `Time zone: <IANA zone>`;
  the entry header and index line formats above; the file-name rules from
  Global Constraints; the rule that a run never edits another run's entry.
- [ ] **Step 3: Write `hoshin.md`** with a heading, one paragraph on what the
  vital few are, and the line `No objectives yet: the first yearly and
  quarterly start reviews set them.`
- [ ] **Step 4: Write `fleet.md`:** one table row per org (`Org`, `Status`
  active or inactive, `Notes`), from Step 1.
- [ ] **Step 5: Write the five level `index.md` files**, each a heading, one
  sentence on the level (with its Japanese term where the spec gives one),
  and `No entries yet.`
- [ ] **Step 6: Write `management/hoshin-kanri.md`:** the spec's "The model"
  background and references, the terms list, the two halves, the calendar in
  brief, and how Vergil scales the method to one person and agents. Check
  the Bridgestone date and the term 社長診断 against the references; state
  each as the sources give it.
- [ ] **Step 7: Add the nav:** a top-level `Logbook` section listing
  `logbook/index.md`, `hoshin.md`, `fleet.md` and the five level indexes;
  `Hoshin kanri: management/hoshin-kanri.md` under Management.
- [ ] **Step 8: Validate.** Run `vrg-container-run -- vrg-validate`.
  Expected: pass, including the strict MkDocs build.
- [ ] **Step 9: Commit** with `vrg-commit --type docs --scope logbook
  --message "add the hoshin logbook and the hoshin kanri background page"`.

### Task 3a: The `hoshin` skill, run machinery (vergil-claude-plugin)

**Files:**

- Create: `skills/hoshin/SKILL.md`
- Create: `skills/hoshin/references/calendar.md`
- Create: `skills/hoshin/references/logbook.md`
- Create: `skills/hoshin/references/security.md`
- Modify: `README.md` (skill list), `docs/site/docs/skills/index.md` (an
  entry for the skill)

**Interfaces:**

- Consumes: Task 2's header keys, index line format and time-zone line.
- Produces, for Task 3b: the per-level reference names
  `references/{daily,weekly,monthly,quarterly,yearly}.md`, which `SKILL.md`
  step 4 loads for each review it runs, and the check format in
  `references/logbook.md` (a `Check: <name>` heading with Answers, Sources,
  Window, Output and Tool fields).

- [ ] **Step 1: Write `SKILL.md` frontmatter.** `name: hoshin`; a
  description that names `/vergil:hoshin start`, `end` and `end <level>`, the
  triggers ("start the day", "hoshin start", "end of day", "close the
  week/month"), and that it runs as the user agent.
- [ ] **Step 2: Write the procedure,** the spec's seven run steps in order:
  preflight (user agent; docs clone current with `origin/develop`; refuse on
  an open logbook PR or open `Logbook:` task in the docs ad-hoc epic, naming
  it; refuse if the entry to write exists); read the logbook (front matter
  of each level's latest entries); decide what is due (by
  `references/calendar.md`); close then open in the spec's order (closing
  ends smallest first; on `start`, daily first, then yearly down to weekly);
  file follow-ups (private-repo follow-ups in that repo); write entries and
  index lines; run the pre-commit gate; hand off one PR with the fixed
  wording.
  - **The pre-commit gate,** before `vrg-commit`: first the screen in
    `references/security.md` over the final text of every entry (a hit stops
    the run and names the match); then show the human the full final text
    of every entry and wait for approval. Only then commit, validate and run
    `report-ready`.
  - **No merge review.** State that a logbook run skips `merge-review` on
    purpose: the review conversation is the review, the gate's screen and the
    human's approval of the final text are the content check, and the human's
    submit is the sign-off. The handoff notes carry no eligibility line.
- [ ] **Step 3: Write `references/calendar.md`:** the spec's calendar rules
  (time zone, days, weeks, periods from the Monday, boundaries, closing an
  open period with the daily one-keystroke default, gaps at week and above,
  windows, `end` targeting the latest open period, the first run), then the
  four worked examples named in Review Focus, each as given date and time,
  logbook state, and the exact files written with their windows. The `end`
  window rule: an `end` the human runs covers from its matching `start`'s
  `window_to` to whichever is earlier, now or the end of its period, with
  `late: false`; an `end` run late by a `start` covers to the end of its
  period, with `late: true`.
- [ ] **Step 4: Write `references/logbook.md`:** where the logbook is, the
  file-name rules, the header (Task 2), the index line, the "not performed"
  entry's shape, the fixed wording for the task, commit and handoff, the
  kaizen prompt that ends every review, and the check format.
- [ ] **Step 5: Write `references/security.md`:** the spec's Security rules,
  each as a rule plus one example of the allowed wording, with example A (a
  failed run in a private repo) and example B (a failed security job in a
  public repo) from Review Focus; then the pre-commit screen (patterns
  `GHSA-`, `CVE-`, alert numbers, each private repo name from the visibility
  check, per-repo alert counts) with example C, a hit and the stop it
  produces.
- [ ] **Step 6: Register the skill** in `README.md` and
  `docs/site/docs/skills/index.md`, matching the existing entries' shape.
- [ ] **Step 7: Validate.** `vrg-container-run -- vrg-validate`. Expected:
  pass.
- [ ] **Step 8: Commit** with `vrg-commit --type feat --scope hoshin
  --message "add the hoshin skill's run procedure, calendar, logbook and
  security rules"`.

### Task 3b: The `hoshin` checklists and checks (vergil-claude-plugin)

**Files:**

- Create: `skills/hoshin/references/{daily,weekly,monthly,quarterly,yearly}.md`

**Interfaces:**

- Consumes: Task 3a's reference names and check format.
- Produces, for Task 4: the `Tool: none yet` checks, each naming its planning
  issue.

- [ ] **Step 1: Write one reference per level,** each with a `start` and an
  `end` section carrying the spec's first checklist for that review (its
  Japanese name in the heading), and each data step written as a check in
  Task 3a's format. Where a `vrg-*` tool answers a check (`vrg-repo-status`,
  `vrg-queue`, `vrg-epic-audit`, `vrg-roadmap`, `vrg-activity-log`), `Tool:`
  names it; otherwise `Tool: none yet` and the check names the `vrg-gh` /
  `vrg-git` queries that answer it today and the planning issue it belongs to
  (#1316 fleet state, #1140 org state, #1317 history).
- [ ] **Step 2: Validate.** `vrg-container-run -- vrg-validate`. Expected:
  pass.
- [ ] **Step 3: Commit** with `vrg-commit --type feat --scope hoshin
  --message "add the hoshin checklists and checks for each level"`.

### Task 4: Plugin release with the skill (deployment)

- **Precondition (human-attested):** a vergil-claude-plugin release
  containing Tasks 3a and 3b is published; Task 2 is merged. The version is
  named in a comment on the task.
- **Procedure:** the agent-safe install: `vrg-host plugins` on the host and
  `vrg-vm update --all` for the VMs, then a new session. Then post each
  `Tool: none yet` check from Task 3b, as merged, as a comment on the
  planning issue it names (#1140, #1316 or #1317), linking the reference file
  at the release tag.
- **Checks:**
  1. In the new session, the skill list includes `vergil:hoshin`.
  2. The installed plugin's `skills/hoshin/` holds `SKILL.md` and the eight
     reference files.
  3. `https://vergil-project.github.io/docs/` shows the Logbook section.
  4. Each `Tool: none yet` check has its comment on its planning issue, and
     each link resolves.
- **On failure:** the task stays open; fix forward with a new task.

### Task 5: Bootstrap the logbook (validation, docs)

- **Precondition:** Task 4 closed.
- **Procedure:** in the human's dedicated planning session, run
  `/vergil:hoshin start`. With the logbook empty, it runs the daily `start`,
  then the yearly, quarterly, monthly and weekly `start` reviews, and sets
  the first `hoshin.md` by catchball; the human submits its PR.
- **Acceptance:**
  1. The logbook PR merged, holding five `start` entries for the current
     day, week, month, quarter and year, their index lines, and a
     `hoshin.md` with three to five objectives.
  2. No `end` entry and no gap was written.
  3. The entries, task, commit and handoff follow the fixed wording and the
     Security rules (checked by the human reading the PR).
- **Results:** recorded as a SUCCESS or FAILURE comment; any rule the run
  broke is a new task under the epic.

## Evolution during execution
