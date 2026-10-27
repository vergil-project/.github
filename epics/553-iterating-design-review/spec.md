# Iterating design review in `epic-create`

- **Epic:** vergil-project/.github#553
- **Planning issue:** vergil-project/.github#469 (review history is in its
  comments)
- **Umbrella:** vergil-project/.github#500 (child C). Child F depends on this
  epic.
- **Status:** draft, in design review
- **Brainstorm source:** none. The umbrella spec (§3.1, §3.2, §4.C) settled
  the design, so the human skipped the brainstorm.

## 1. Intent

`epic-create` runs `paad:pushback` on the spec once and `paad:alignment` once.
Running them again finds real breakage the first pass missed. Without rules,
though, repeated passes make a design worse. In the four-round experiment
behind #469, each round treated earlier decisions as settled and only added to
them. Complexity compounded, later rounds mostly repaired earlier rounds'
additions, and the epic was abandoned.

The umbrella spec (`epics/500-paad-first-engineering/spec.md`) settled the
rules for repeating a review: §3.1 is the common core and §3.2 is mode 1,
human-gated design review. This epic builds mode 1 into `epic-create` and
writes the shared doctrine document that child D extends with mode 2.

## 2. Inherited rules (binding, not reopened here)

Umbrella §3.1 and §3.2 are binding on this epic as written (umbrella §4,
"How binding section 3 is"). In short:

- each round is a fresh subagent that reviews the whole artifact, including
  what earlier rounds added, and removing an earlier addition is a legitimate
  finding;
- the orchestrator (the main session) classifies each finding as **new**,
  **known** or **repair**; the reviewer never does;
- the reviewer assigns severity, and the orchestrator maps it onto
  high/medium/low without re-grading it;
- reviewer discards are recorded only as a count plus one line each;
- the stop and continue signals, and their precedence (continue wins),
  narrowed for mode 1 by the human's decision recorded in 3.6;
- the stages: spec pushback rounds, the plan is written, plan pushback rounds,
  alignment rounds;
- the reviewer runs review-only: it collects every finding, edits and writes
  nothing (except paad's own `paad/security/` file), asks nothing, and returns
  the full ranked list with its discards;
- each round: review, a walk-through with the human, commit, checkpoint report;
- the human decides every further round, and stopping after one round is
  normal;
- no review history inside the reviewed document; the full record goes in
  planning-issue comments;
- every finding that was not addressed is accepted, rejected with a reason,
  dropped by the human with a reason, or filed in intake (umbrella §2.4);
- security details appear in public records only as counts.

The rest of this spec is mechanism. Under umbrella §4 the mechanisms are this
epic's to design, as long as the rules still hold.

## 3. Design

### 3.1 The doctrine document

A new reference document, `vergil-claude-plugin/docs/paad-review-doctrine.md`,
holds the shared doctrine:

- **§1 Common core:** umbrella §3.1, rewritten as operating instructions for an
  orchestrator, plus the canonical severity mapping for every paad scale, the
  classification procedure and its trusted-authors rule (all 3.6). Where a mode
  narrows a §1 signal, §1 says so and points to that mode's section, as the
  umbrella does for mode 2.
- **§2 Mode 1, human-gated:** umbrella §3.2 plus the mechanisms in this spec:
  the reviewer prompt contract, output checks, the round comment format, the
  checkpoint report and recommendation logic (including mode 1's narrowed
  continue signal), and security handling.
- **§3 Mode 2, autonomous:** a stub saying child D
  (epic vergil-project/.github#568) writes it.

It lives next to `docs/paad-policy.md`. The plugin ships `docs/` (the cached
plugin directory includes it), and skills already link there
(`skills/handoff/SKILL.md` → `../../docs/paad-policy.md`). `epic-create` keeps
only the step-by-step procedure and links to the doctrine for the rules, so
`epic-create` and D's `issue-implement` share one source.

`docs/paad-policy.md` gains a short section pointing to the doctrine, and the
deviations register gains the rows in 3.12.

### 3.2 The pipeline in `epic-create`

The workflow steps change as follows. Steps 0–3 and 8–10 are unchanged except
where noted.

| Step | Today | New |
|---|---|---|
| 3 | Write the spec | Write the spec. Its header follows 3.9. |
| 4 | `paad:pushback` once | **Stage 1: spec pushback rounds** (3.3). |
| 5 | Human review | Human review of the spec (unchanged). |
| 6 | `superpowers:writing-plans` | Unchanged: one automated pass, no rounds. |
| 7 | `paad:alignment` once | **Stage 2: plan pushback rounds**, then **Stage 3: alignment rounds** (3.3). |

The four-stage interaction-doctrine table in `epic-create` becomes:

| Stage | Mode | Contract |
|---|---|---|
| `superpowers:brainstorming` | interactive | Unchanged. |
| Design review (pushback on the spec, pushback on the plan, alignment) | **human-gated rounds** | A fresh review-only subagent per round. The main session walks the human through the findings, and the human decides every further round. |
| `superpowers:writing-plans` | automated | Unchanged. |

At the end of each stage the human chooses to stop that stage, and the
pipeline moves to the next one. Stage 3 may change both the spec and the plan,
as alignment does today.

### 3.3 One round

Every round in every stage runs the same procedure. `<stage>` is one of
`spec pushback`, `plan pushback`, `alignment`. Rounds are numbered from 1 within
each stage.

1. **Dispatch.** The main session starts a fresh `general-purpose` subagent
   with the reviewer prompt (3.4). It runs in the background, because the
   human cares about the result, not the work (umbrella §2.2).
2. **Check the output** (3.5). A failed check is a broken round (3.5).
3. **Classify and map severity** (3.6).
4. **Walk the human through the findings,** one at a time and in severity
   order, the way `paad:pushback` does today: the problem, options from best to
   worst, and a recommendation. "A response is not a decision" applies: a
   question gets an answer and the same options again. Each finding ends as:
   - **accepted:** the main session applies the change;
   - **rejected:** the human gives a reason, and the artifact is not changed;
   - **dropped:** the human gives a reason, or the main session files it in
     intake (`vrg-intake-create`), linking the planning issue and the round.

   The main session writes the requirement, never the weakness (paad's own
   rule), and leaves anything not discussed alone. Security findings follow
   3.8.
5. **Commit** the round's changes in the worktree, as one commit:
   `vrg-commit --type docs --scope epic-<N> --message "<what changed>"`. The
   message describes only the change, never the round, the stage or finding
   IDs: paad's reality check reads the branch's history, and naming the review
   there would tell the next reviewer about earlier rounds (the same rule as
   mode 2). The round comment records the commit's SHA (3.7). The docs branch
   is not rebased between rounds, so those SHAs stay valid until
   `vrg-submit-pr` rebases it at hand-off. A round that changes nothing makes
   no commit.
6. **Post the round comment** on the planning issue (3.7), including the
   checkpoint report and the recommendation (3.6).
7. **Ask the human:** another round of this stage, or move on. The human's
   answer is final. The recommendation is advice.

### 3.4 The reviewer prompt contract

The doctrine gives a prompt template. Every reviewer prompt must:

- name the skill to invoke through the Skill tool (`paad:pushback` or
  `paad:alignment`) and its arguments, as absolute paths in the docs worktree:
  the spec for spec pushback, the plan for plan pushback (with the spec named
  as context, not as the document under review), and the spec and the plan
  for alignment;
- **work in the docs worktree:** every Bash command starts
  `cd <docs-worktree> && …`, and every subagent the skill dispatches is told
  the same. paad then writes its default paths (`paad/security/`) in the
  worktree, where the main session collects them. The prompt says nothing else
  about where paad writes: adapting to paad's defaults in caller code is more
  reliable than overriding the skill's prose (the same rule as mode 2);
- use `vrg-git`, since raw `git` is blocked;
- give the **review-only directives** from umbrella §3.2:
  - collect every finding;
  - edit nothing and write nothing, except paad's own `paad/security/` file;
  - ask nothing. Where the skill would ask (input confirmation, a
    source-control conflict, a scope split, which update to make), report it
    as a finding, or state the assumption it used. paad's "hold or ordinary
    report" security question is always answered **hold**, so security
    details never reach the ordinary findings;
  - skip pushback's report file, and skip alignment's TDD rewrite: instead,
    report each code task that is not in red/green/refactor form as a minor
    finding;
  - return the full ranked list with discards;
- say to review the **whole** artifact, including mechanisms that look like
  recent additions, and that removing a mechanism is a legitimate finding;
- give the **git history rule** (this resolves umbrella §3.2's open
  provenance point): the reality check reads the recent history of the
  **member repos the artifact names** (their `origin/develop`), for conflicts
  between the artifact and the code. It must not read the history, diffs or
  commit messages of the epic's own `epics/<N>-<slug>/` directory, and must not
  read the planning issue. Anchoring on earlier rounds is what §3.1 rules out;
- pass no information about earlier rounds: no round number, no earlier
  findings, no decisions;
- give the **return format:** a first line `Running paad:<skill> v<version>`,
  a `Config:` line when a config file was followed, then one block per finding
  with its paad severity, its section reference, the "Y happens, because Z"
  claim, and its options; then `Discards: <k>` followed by one line each; then
  `Security: <k> finding(s)` or `Security: 0`; then every file paad wrote,
  each as an absolute path.

### 3.5 Output checks and failed rounds

Before dispatch, the main session lists the `paad/` of the session's starting
directory. After every review, it checks structure, not prose:

- the version line is present. A version other than the one the doctrine
  records is a warning in the round comment, not a failure;
- there is no `Config:` line while no fallback (3.11) is adopted; once one is
  adopted, the `Config:` line names exactly the expected file;
- every finding carries a paad severity and a section reference, and the
  severity is a word in that skill's column of the severity table (3.6);
- the reviewer did not record a security hold;
- the `Discards:` and `Security:` lines are present;
- every file paad wrote, as an absolute path, is under the docs worktree's
  `paad/security/`, and when `Security:` reports findings, that file exists;
- nothing new appeared under the `paad/` of the session's starting directory;
- `vrg-git status --porcelain` in the worktree is empty. `paad/security/` is
  ignored by the managed `.gitignore` block, so the security file does not
  show.

A failed check is a **broken round**: the reviewer did not produce a usable
review, which says nothing about the design. It is never treated as a pass,
and its findings are not acted on. The main session:

1. reverts anything the reviewer changed: `vrg-git checkout -- <file>` for
   each changed tracked file, and `rm` for each untracked path that
   `vrg-git status --porcelain` lists;
2. re-runs the round once, automatically, with a fresh reviewer;
3. on a second broken run, stops and asks the human, who chooses: retry,
   run the skill interactively in the main session as `epic-create` does
   today (that round is recorded as a deviation), or skip the stage. A skip
   is recorded in the round comment with the human's reason.

Nothing moves on to the next stage without the human's decision. Repeated
broken rounds are evidence for the fallback (3.11).

A security hold keeps its details only in the terminal conversation with the
human.

### 3.6 Classification, severity and the recommendation

**Classification.** After each round, the main session matches every finding
against:

- earlier round comments on the planning issue for this epic (all stages);
- the diffs of earlier round commits, found by the SHAs those comments record
  (`vrg-git show <sha>`); and
- for security findings, which public records show only as counts, the
  advisories already on this planning issue, read with
  `vrg-security-record show` or `list --details` in the main session's terminal
  only.

Only comments whose author passes `vrg-trusted-authors` count (doctrine §1;
the command comes from child D, vergil-project/vergil-tooling#3741). Anyone can
comment on a public issue, and a forged "decision" must not turn a finding into
a known one.

A finding is **known** if an earlier round raised it or the human deliberately
accepted it. It is **repair** if the text it criticises was added by an
earlier round's commit. Otherwise it is **new**. Where a finding fits both
known and repair, it is known. Round 1 of stage 1 has only new findings.

**Severity mapping.** paad's skills use several severity scales. The vergil
workflow maps every one of them onto **one canonical scale**, GitHub's
(`critical`, `high`, `medium`, `low`), so that a severity means the same thing
in a checkpoint count, a security record (3.8) and anywhere else it is used.

- **One table, in the doctrine.** The canonical mapping lives in
  `docs/paad-review-doctrine.md` §1 and nowhere else. Other docs link to it
  and do not copy it. The security-records guide's own table
  (`vergil-tooling/docs/site/docs/guides/security-records.md`, "Severity")
  is replaced by a link.
- **The reviewer assigns severity** in its skill's own terms. The
  orchestrator maps it with the table and never re-grades.
- **Design reviews use only high, medium and low** (umbrella §3.1), so their
  scales fold onto those three.

The pushback and alignment columns are fixed by this design. Every column was
checked against its paad 2.0.2 skill's own text on 2026-10-27 (agentic-architecture
calls it impact, not severity). paad is unpinned, so a later paad release that
changes a scale needs a doctrine change:

| Canonical | pushback | alignment | agentic-review | agentic-dedup | agentic-owasp | agentic-architecture | agentic-a11y |
|---|---|---|---|---|---|---|---|
| `critical` | n/a | n/a | Critical | Critical | Critical | n/a | critical |
| `high` | critical, serious | Critical | Important | Important | High | High | serious |
| `medium` | moderate | Important | Suggestion | Suggestion | Medium | Medium | moderate |
| `low` | minor | Minor | n/a | n/a | Low | Low | minor |

Child D owns the agentic-review column for mode 2 and may revise it. Changing
any column later is a doctrine change.

A finding is **structural** when it is about scope (bundled features, a split),
or when the change it asks for adds or removes a whole section, mechanism or
task. The main session tags structural findings in the round comment.

**Recommendation.** The recommendation is advice to the human, who decides
every round (3.3 step 7). It applies umbrella §3.1's signals, with continue
signals winning, but **only new findings argue for another round**:

- a **known** finding has already been decided, and a fresh reviewer will
  raise a rejected finding again;
- a **repair** finding is about an earlier round's own addition. A high repair
  is still walked through and fixed in its round, but another round would
  mostly re-review the repairs, which is the #469 pattern the stop signal
  exists to catch.

This narrows umbrella §3.1's continue signal ("any high-severity or
structural finding") for mode 1. Changing a rule is a human decision (umbrella
§4), and the human made it while reconciling this spec with children A′, D and
E; it is recorded on the planning issue (vergil-project/.github#469). It rests
on the ad hoc trials of mode 1 on children E (planning issue
vergil-project/.github#506) and D (vergil-project/vergil-claude-plugin#677). In
both, mostly-repair rounds recommended *continue* only because some repairs
were high. Doctrine §1 notes the narrowing and points here.

- *continue* if any **new** finding is high or structural, or if this round's
  commit changed the artifact materially (an accepted new structural or high
  finding);
- otherwise *stop* if the round raised nothing new, or only one or two low
  findings remain, or more than half the findings are repairs (even when some
  are high), or the artifact grew in each of the last three rounds of this
  stage;
- otherwise *continue*.

**Repair concentration** is the largest number of this round's repairs that
trace back to one earlier addition. It makes the #469 pattern visible at a
glance.

**Growth** is the change in the artifact's line count and its count of `##`
and `###` headings, from before the round to after its commit. For alignment,
it is the sum over spec and plan.

From round 4 of a stage on, the report says the loop is past the typical
number of rounds.

### 3.7 The round comment

One comment per round on the planning issue, posted with
`vrg-gh issue comment <M> --repo <owner>/<repo> --body-file <tmp>`.

- **First line** (shown by `vrg-queue planning` as the latest note):
  `Design review: <stage> round <n>: <k> findings, <a> accepted; recommend <continue|stop> (<signal>)`.
  The fixed prefix `Design review: <stage> round <n>` is how a resumed session
  rebuilds the stage and round count.
- **Checkpoint:** raised, accepted, rejected and dropped counts; the mapped
  severity mix; new, known and repair counts; the repair concentration; the
  growth delta; the round commit's SHA, or "no commit"; the recommendation and
  its signal; the round-4 note when it applies; and the broken-round note
  (including any version warning or recorded skip) when it applies.
- **Findings:** one entry per finding, with its round ID (`<stage-letter><n>.<i>`,
  for example `S2.3` for spec pushback round 2, finding 3; `P` for plan
  pushback, `A` for alignment), paad and mapped severity, classification,
  structural tag, a summary of the claim, the decision, and the reason or
  intake ref.
- **Discards:** the count and one line each.
- **Security:** a count only (3.8).
- The comment carries no marker lines (`Epic:`, `Docs task:`,
  `Tasks filed:`), so the queue view's marker parsing is unaffected.

A comment that would pass GitHub's 65,536-character limit is split into parts.
Each part's first line is the same first line plus `(part <i>/<m>)`.

Stage-boundary progress comments, which `epic-create` already requires, still
post. When the human stops a stage, the round comment's recommendation line is
followed by a progress comment that names the next stage.

### 3.8 Security findings

paad writes security findings under the docs worktree's ignored
`paad/security/` (3.4). paad names the file by date and document only, so a
later round on the same day would overwrite it. Once the output checks pass,
the main session therefore moves the round's security file to a private scratch
directory outside the worktree, named for the round, and works from that copy.
It walks the human through each finding in the terminal, like any other.

Only **new** and **repair** security findings are recorded. A **known** one
already has its record (3.6). Recording follows child A′'s contract (epic
vergil-project/.github#573, spec §4.4), under which one issue can name several
advisories:

- **accepted** (the spec or plan now states the requirement): record it with
  `vrg-security-record create --repo <affected repo> --issue <planning issue>
  …`. A planning issue may carry several;
- **rejected** (the human decided it is not a problem): record it with plain
  `vrg-security-record create` (no `--issue`), then close the neutral intake
  issue it creates as "not planned". The sweep then closes the advisory. The
  human's reason goes into the advisory details, not into a public record;
- **dropped to intake:** record it with plain `vrg-security-record create`,
  and leave its intake issue open.

**Step 9 handoff.** Handing each advisory on the planning issue to its fix
task, or to the documentation task, is already in `epic-create`
(vergil-project/vergil-claude-plugin#756). This epic does not change it; the
`epic-create` rewrite keeps it as it is.

**Cleanup.** Once every finding in the scratch copy has been recorded, the main
session deletes the copy, so no scratch details are left behind. If any record
fails, the copy is kept and the round comment reports the failure. A copy is
never deleted while a finding in it is unrecorded.

The round comment shows only the count of security findings and how many were
accepted. No GHSA IDs, titles or descriptions appear in the round comment, the
spec, the plan or commit messages. The advisory's link to the planning issue is
A's mechanism and is neutral by design.

### 3.9 No review history in specs and plans

A spec or plan header may carry only: `Epic`, `Planning issue` (with "review
history is in its comments"), `Umbrella` when there is one, `Status` (one of
`draft, in design review`, `approved`, or `superseded by <ref>`), and
`Brainstorm source`. It must not carry:

- a `Review:` line, a per-round log, or a count of findings;
- inline notes saying which round, review or decision produced which text
  (`(added in pushback round 2)`, `**Alignment (paad, …):**` bullets, "revised
  after pushback");
- a "review" or "pushback" section.

`epic-create` states this rule and the reason (umbrella §3.2). Existing
published specs and plans are not rewritten.

### 3.10 The planning issue

The skill already runs every session through a planning issue. This epic makes
the remaining pieces explicit:

- every epic has a planning issue; when `epic-create` starts from a free-text
  idea, step 0 creates one (unchanged), and the planning issue is the permanent
  record of the design review;
- links run both ways: the epic body's `Planning:` line, the spec header's
  `Planning issue` line, and the planning issue's closing comment
  `Planned as <epic ref>`;
- the planning issue is never moved to the epic it produced. It closes in the
  planning epic and drains into its archive.

### 3.11 Fallback when prompting is not enough

If the validation spike (3.13) shows that the prompt alone cannot keep the
reviewer review-only, the fallback is paad's experimental config:
`paad/config/pushback.md` and `paad/config/alignment.md`, holding the
review-only directives. paad reads config relative to the reviewer's working
directory, which is the docs worktree (3.4), so the files are committed in the
epic's home repo (`vergil-project/.github`, and any self-homed private repo
that runs `epic-create`). Each file names the paad version it was written for
(paad-policy §1, §4).

paad applies a config file to every run of that skill in the repo, including a
human's own interactive `/paad:pushback`. So the files must act only on the
review-only reviewer, for example by applying only when the prompt carries a
review-only marker. If that cannot be done, the side effect is documented in
paad-policy before the files are adopted.

If config is also not enough, the next step is asking the paad authors for a
report-only mode (umbrella §3.2); that request is outside this epic.

### 3.12 Deviations register additions

Add rows to `vergil-claude-plugin/docs/paad-policy.md` §7:

| Deviation from paad | Reason |
|---|---|
| pushback and alignment run review-only in a fresh subagent, and the main session presents the findings and edits the document; paad runs them interactively in the session that edits | Each round needs a context that has not seen earlier rounds (umbrella §3.1). The human still sees one finding at a time with options. |
| alignment's TDD rewrite step is skipped | `superpowers:writing-plans` writes tasks in red/green/refactor form. The reviewer reports each code task that is not as a minor finding (3.4), and it is fixed in the walk-through. |
| The reviewer's reality check does not read the history of the document under review | Commit history of earlier rounds carries the provenance that umbrella §3.1 keeps out of the reviewer's context. |
| Design-review rounds are repeated under explicit stop and continue signals, with the human deciding each round | The authors say "revise and repeat as needed" without stop rules. The #469 experiment showed that unruled repetition compounds complexity. |

### 3.13 Validation

- **Spike, first:** a `validation` task runs the reviewer prompt (3.4) on an
  already published spec and plan (`epics/517-security-records/`) twice for
  each of the three stage configurations, and applies the output checks (3.5).
  One extra run reviews a copy of the spec seeded with a security-shaped
  weakness, from a session started in a different repo from the docs
  worktree, so the security-file path (3.4, 3.5) and its cleanup (3.8) are
  exercised. The seeded finding is recorded with plain
  `vrg-security-record create` and a neutral `PROBE:` summary (as A's guide
  does for its own probe), and its intake issue is then closed as "not
  planned". Success: every run passes the checks and leaves the worktree
  clean, and the scratch copy is deleted. The result decides whether the
  fallback (3.11) is built. If it fails, its result
  goes back to the human before the skill rewrite starts.
- **Dogfood, last:** once the plugin release containing the new `epic-create`
  is installed (a human-gated release), run the full mode 1 pipeline on the next
  real `epic-create` and record the round comments. The task cites that
  planning issue.

## 4. Goals, non-goals, anti-goals

**Goals.**

- `epic-create` runs the authors' full design loop under the written doctrine.
- Each round's record is complete on the planning issue, and the spec and plan
  stay free of review history.
- One doctrine document, ready for D to extend.

**Non-goals.**

- Mode 2 and the retrospective's unaddressed-findings sweep (child D).
- Tooling to compute classification or growth. The main session does it by
  reading the comments and the diffs; tooling can come later if the records
  show a need.
- Rewriting review history out of already published specs and plans.
- Changing paad itself.

**Anti-goals** (inherited from umbrella §5): rounds that only add; a loop that
cannot be stopped after one round on simple work; findings silently dropped;
security details in any public record.

## 5. Deliverables by repo

| Repo | Deliverable |
|---|---|
| `vergil-claude-plugin` | `docs/paad-review-doctrine.md`; the `epic-create` rewrite; `docs/paad-policy.md` pointer and register rows; the skills index page; the pipeline description in `CLAUDE.md` |
| `vergil-project/.github` | this spec and plan; `paad/config/` files only if the fallback is needed (3.11) |
| `vergil-tooling` | `docs/site/docs/standards/github-issues.md` pipeline steps, and the "Severity" table in `docs/site/docs/guides/security-records.md` replaced by a link to the doctrine (both via the documentation review) |
| `docs` | `management/workflow.md` and `management/tools-and-skills.md` (via the documentation review) |

## 6. Constraints

- Public repos: planning-issue comments, commit messages, PR bodies and
  report-ready notes are public.
- Agents use `vrg-git` and `vrg-gh` only; agents never run `vrg-submit-pr`.
- The reviewer subagent needs the Skill tool and paad installed where
  `epic-create` runs. B Task 4 (vergil-project/vergil-tooling#3668) verified
  paad on the agent VMs.
- paad's config mechanism is experimental and paad is unpinned (paad-policy
  §1, §4).
- Prerequisites from sibling children: the release and deployment of A′'s
  one-to-many security records (vergil-project/vergil-tooling#3733, epic
  vergil-project/.github#573) for 3.8, and D's `vrg-trusted-authors`
  (vergil-project/vergil-tooling#3741) for classification (3.6).
