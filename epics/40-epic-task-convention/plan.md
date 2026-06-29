# Epic / Task Convention — Implementation Plan (epic task map)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **This is a multi-repo epic**: each task below is a separate child issue and a separate PR in the repo named in its heading. The bite-sized red/green/refactor steps for the foundational tasks (1–4) are written out in full; tasks 5–9 carry exact paths, signatures, and acceptance criteria, and are expanded to full step granularity by the implementing subagent at execution time (each is its own task issue in its own repo).

**Goal:** Establish a two-tier epic/task issue convention with mechanical, deterministic closing, a lossless intake queue, a standardized label namespace, epic-owned docs, and generated roadmap/activity-log observability — then migrate every `vergil-project` repo into it.

**Architecture:** Finite epics live in `vergil-project/.github` as native cross-repo sub-issue parents; tasks live in member repos, 1:1 with a finalizing PR. `vrg-finalize-pr` gains a close+rollup stage (programmatic close at finalize, `Ref`-only linkage preserved). Umbrella links use GitHub native sub-issues with a cross-repo reflink fallback (forge-portable). Labels seed through the existing `vrg-ensure-label` registry. Observability is mechanically generated from issue/PR metadata.

**Tech Stack:** Python 3.12 (`vergil_tooling` package, pytest, subprocess-`gh` via `lib/github.py`); bash 3.2 hooks (`vergil-claude-plugin`); GitHub GraphQL via `gh api graphql`; Markdown specs/plans.

## Global Constraints

- Linkage stays **`Ref`-only**; `Fixes`/`Closes`/`Resolves` remain forbidden (enforced by `block-autoclose-linkage.sh`, `linkage.AUTOCLOSE_RE`, and the `pr-issue-linkage` CI gate).
- All git/gh through `vrg-git` / `vrg-gh`; commits via `vrg-commit`; PRs via `vrg-submit-pr`. Raw `git commit` / `gh pr create` are hook-denied.
- No heredocs for multi-line CLI args; write to a temp file and use `--body-file` / `--input`.
- All edits flow through a `.worktrees/<name>/` worktree on a `feature/<N>-<slug>` branch — the main worktree is write-guarded in **every** managed repo (incl. `.github`).
- New `vrg-*` CLIs register via `[project.scripts]` in `vergil-tooling/pyproject.toml`.
- Python: subprocess-`gh` only (no PyGithub/httpx); reuse `lib/github.py` helpers; mock with `unittest.mock.patch` + autouse fixtures.
- Bash hooks: bash 3.2-compatible (no associative arrays); table-driven `.test.sh`.
- No cross-org links; epics are per-org (`vergil-project/.github`).

**TDD / refactor note (applies to every code task — T2, T3, T4, T5, T8):** each
follows red→green→**refactor**. After the test passes, before the commit, do a
refactor pass: extract duplicated logic, move hard-coded values (label names,
GraphQL query strings, repo slugs) to module constants, consolidate with existing
`lib/github.py` / `lib/epics.py` helpers, and align naming with the interfaces
declared here. The refactor step is explicit because it is the one most often
skipped.

---

## Task map & dependency order

```
T1 docs task (.github)            ── publishes spec + this plan; first child of the epic
T2 label taxonomy (vergil-tooling)── extend vrg-ensure-label registry            [blocks enforcement, T5]
T3 umbrella links (vergil-tooling)── github.graphql() + lib/epics.py + vrg-gh    [blocks T4]
T4 finalize close+rollup (vergil-tooling) ── new stage in vrg-finalize-pr        [needs T3]
T5 linkage enforcement (vergil-tooling)   ── vrg-pr-issue-linkage + vrg-submit-pr[needs T2]
T6 intake skills (vergil-claude-plugin)   ── triage capture + triage-review      [needs T2]
T7 docs-lifecycle + on-ramp (vergil-claude-plugin + .github) ── conventions/docs  [needs T1,T2]
T8 observability (vergil-tooling + .github)── roadmap + activity-log generators   [needs T2,T3]
T9 migration passes (per repo)            ── one child task per repo, brainstorm-driven [needs T2-T5]
```

T2 and T3 are the unblockers; do them first after the docs task. T9 keeps the epic open until every repo is migrated.

---

## Task 1: Documentation task — publish spec + plan (`.github`)

**Repo:** `vergil-project/.github` · **Epic:** #40 · **Task:** #41 · **Branch:** `feature/41-epic-task-convention-docs`

**Files:**
- Create: `.github/epics/40-epic-task-convention/spec.md` (the reviewed design doc)
- Create: `.github/epics/40-epic-task-convention/plan.md` (this file)

**Source:** spec + plan are in this session's scratchpad —
`…/scratchpad/2026-06-28-epic-task-issue-and-docs-lifecycle-convention-design.md`
and `…/scratchpad/2026-06-28-epic-task-convention-implementation-plan.md`.

**Interfaces — Produces:** the canonical `.github/epics/<N>-<slug>/` directory shape that T7 documents and T8 reads.

- [ ] **Step 0:** Confirm `.github`'s base branch: `vrg-git -C <.github> symbolic-ref refs/remotes/origin/HEAD` (it is `library-release` model — likely `develop`; use whatever this prints as `<base>`).
- [ ] **Step 1:** From the `.github` repo root, create the worktree: `vrg-git worktree add -b feature/41-epic-task-convention-docs .worktrees/issue-41-epic-task-convention-docs origin/<base>` then `cd` into it.
- [ ] **Step 2:** Copy the scratchpad spec → `.github/epics/40-epic-task-convention/spec.md` and the scratchpad plan → `.github/epics/40-epic-task-convention/plan.md`.
- [ ] **Step 3:** Validate: `vrg-container-run -- vrg-validate` (markdownlint must pass). Expected: PASS.
- [ ] **Step 4:** Commit: `vrg-commit --type docs --scope epics --message "publish epic/task convention spec and plan" --body "Ref #41"`.
- [ ] **Step 5:** Submit: `vrg-submit-pr --issue 41 --title "docs(epics): epic/task convention spec + plan" --summary "Publish the reviewed design and implementation plan" --linkage Ref`.

**Acceptance:** spec.md + plan.md exist under `.github/epics/<EPIC_N>-…/`; PR `Ref`s the docs task; markdownlint green.

---

## Task 2: Label taxonomy — extend the `vrg-ensure-label` registry (`vergil-tooling`)

**Repo:** `vergil-project/vergil-tooling` · **Branch:** `feature/<N>-epic-task-labels`

**Files:**
- Modify: the label registry consumed by `load_labels()` in `src/vergil_tooling/bin/vrg_ensure_label.py` (locate the registry data file it reads — JSON/TOML under `src/vergil_tooling/`; grep `def load_labels`).
- Test: `tests/vergil_tooling/test_vrg_ensure_label.py`

**Interfaces:**
- Consumes: existing `sync_repo(repo)`, `load_labels() -> dict` (keys `labels: list[{name,color,description}]`, `delete: list[str]`).
- Produces: canonical labels **added additively** — `epic`, `standing`, `triage`, `idea`, `hotfix` (Role/Stage/Kind/Exception axes), with the `Kind` set aligned to the `.github` issue template (`feature`,`bug`,`documentation`,`refactor`,`chore`,`research`) plus `idea`.

**Transition constraint (spec §5):** this task is **additive only** — it does **not**
touch the `delete` list. Retiring default cruft (`help wanted`,`question`,`wontfix`,
`duplicate`,`invalid`) happens **per repo, after that repo's migration pass** (T9),
so no label is orphaned off a still-unmigrated issue.

- [ ] **Step 1: Write the failing test.**
```python
def test_registry_includes_convention_labels() -> None:
    reg = load_labels()
    names = {l["name"] for l in reg["labels"]}
    for required in {"epic", "standing", "triage", "idea", "hotfix"}:
        assert required in names

def test_label_addition_is_additive_only() -> None:
    # This change must not enqueue deletions; cruft retirement is deferred to T9.
    reg = load_labels()
    assert {"help wanted", "wontfix", "question"}.isdisjoint(set(reg.get("delete", [])))
```
- [ ] **Step 2: Run to verify it fails.** `uv run pytest tests/vergil_tooling/test_vrg_ensure_label.py -k "convention or additive" -v` → FAIL (labels absent).
- [ ] **Step 3: Add the labels** to the registry data file (each with `name`, 6-hex `color`, `description`). Do **not** add anything to `delete`.
- [ ] **Step 4: Run to verify it passes.** Same command → PASS.
- [ ] **Step 5: Refactor.** Per the TDD/refactor note — e.g. group the convention labels in the registry with a comment naming each axis; no behavior change. Re-run tests → PASS.
- [ ] **Step 6: Commit.** `vrg-commit --type feat --scope labels --message "add epic/task convention labels (additive)" --body "Ref #<N>"`.

**Acceptance:** `vrg-ensure-label sync` provisions the five new labels additively on a repo; the `delete` list is unchanged; tests green. Cruft retirement is T9's job.

---

## Task 3: Umbrella links — `github.graphql()` + `lib/epics.py` + `vrg-gh` (`vergil-tooling`)

**Repo:** `vergil-project/vergil-tooling` · **Branch:** `feature/<N>-umbrella-links`

**Files:**
- Modify: `src/vergil_tooling/lib/github.py` (add `graphql()` helper)
- Create: `src/vergil_tooling/lib/epics.py` (umbrella relationship, mechanism-agnostic)
- Test: `tests/vergil_tooling/lib/test_epics.py`, `tests/vergil_tooling/lib/test_github_graphql.py`

**Interfaces — Produces (consumed by T4 & T8):**
- `github.graphql(query: str, **variables) -> dict` — wraps `gh api graphql -f query=… -F k=v`, returns parsed `data`.
- `epics.child_states(epic: IssueRef) -> list[ChildState]` where `ChildState = {"ref": IssueRef, "state": "OPEN"|"CLOSED"}`; uses native `subIssues` GraphQL, falls back to reflink scan (`Parent: <org>/.github#<N>` cross-references) when native returns none.
- `epics.parent_of(task: IssueRef) -> IssueRef | None`
- `epics.add_child(epic: IssueRef, task: IssueRef) -> None` — native `addSubIssue` mutation; on a **closed** epic, reopen it first (`github.run("issue","reopen",…)`), then link.
- `epics.all_children_closed(epic: IssueRef) -> bool`
- `IssueRef = {"owner": str, "repo": str, "number": int}`

- [ ] **Step 1: Write failing test for `graphql()`** (mock `github._run_with_retry`, assert it shells `gh api graphql` and returns `data`).
- [ ] **Step 2:** Run → FAIL. `uv run pytest tests/vergil_tooling/lib/test_github_graphql.py -v`.
- [ ] **Step 3:** Implement `graphql()` in `lib/github.py` using the existing `_run_with_retry` + `json.loads`, raising on `errors`.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Write failing tests for `epics.child_states` / `all_children_closed`** (mock `github.graphql` to return a `subIssues` payload; assert states; add a fallback-path test where native returns `[]` and a reflink search returns children).
- [ ] **Step 6:** Run → FAIL.
- [ ] **Step 7:** Implement `lib/epics.py` (`IssueRef`, the GraphQL query for `subIssues`/`parent`, the reflink fallback via `github.read_json("issue","list","--search", f'"Parent: {org}/.github#{n}"' …)`, and the predicates).
- [ ] **Step 8:** Run → PASS.
- [ ] **Step 9: Write failing test for `add_child` reopen-on-closed** (mock parent state CLOSED; assert `issue reopen` called before `addSubIssue`).
- [ ] **Step 10:** Run → FAIL → implement → Run → PASS.
- [ ] **Step 11: Confirm the wrapper boundary (do NOT loosen it).** `vrg-gh api` is
  denied for all interactive identities (verified during epic-#40 bootstrap: "gh api
  is denied … broad write-capable escape hatch"). That is correct and stays. The
  native sub-issue mutation runs **only** through the internal `github.graphql()`
  helper, which shells `gh` directly as trusted tooling (the same way `lib/github.py`
  already calls `gh`) — never via the `vrg-gh` agent wrapper. Add a test asserting
  `github.graphql()` shells `gh api graphql` directly; do not add any `vrg-gh api`
  allowance. If agents ever need to link sub-issues, expose a *narrow* dedicated
  `vrg-gh`-allowlisted verb later — not the raw `api` escape hatch.
- [ ] **Step 11b: Backfill the bootstrap link.** Once `epics.add_child` exists, run
  it to convert the `#40 ← #41` reflink into a native sub-issue (idempotent: the
  reflink remains the fallback).
- [ ] **Step 12: Commit.** `vrg-commit --type feat --scope epics --message "umbrella link helpers (native sub-issues + reflink fallback)" --body "Ref #<N>"`.

**Acceptance:** `epics.all_children_closed()` is correct for native and reflink-linked epics; `add_child` reopens a closed parent; `vrg-gh` gates `graphql` by identity; tests green.

---

## Task 4: Finalize close + epic rollup — new `vrg-finalize-pr` stage (`vergil-tooling`)

**Repo:** `vergil-project/vergil-tooling` · **Branch:** `feature/<N>-finalize-close-rollup`

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_finalize_pr.py` (add `_stage_close` + register in `build_stages`)
- Modify: `src/vergil_tooling/lib/epics.py` (add `rollup(task: IssueRef)` + `premature_close_guard`)
- Test: `tests/vergil_tooling/test_vrg_finalize_pr.py` (extend), `tests/vergil_tooling/lib/test_epics.py`

**Interfaces:**
- Consumes: `linkage.extract_tracking_issue(pr_body) -> int|None`; `epics.parent_of`, `epics.all_children_closed`, `github.run`.
- Produces: `_stage_close(ctx)` — after `cd-check` succeeds, resolve the Ref'd task from the merged PR body, `github.run("issue","close",N,"--repo",repo)`, then `epics.rollup(task)` which closes the parent finite epic iff it lacks `standing` and `all_children_closed`. Runs **only** when prior fail_fast/fail_defer stages succeeded.

- [ ] **Step 1: Write failing test** `test_close_stage_closes_task_then_rolls_up_epic` — patch `linkage.extract_tracking_issue`→N, `epics.parent_of`→epic, `epics.all_children_closed`→True, `github.run`; assert `issue close N` then `issue close <epic>` called; assert a `standing`-labelled epic is **not** closed.
- [ ] **Step 2:** Run → FAIL. `uv run pytest tests/vergil_tooling/test_vrg_finalize_pr.py -k close -v`.
- [ ] **Step 3:** Implement `_stage_close` + `epics.rollup`; append `Stage("close", _stage_close, "fail_fast")` to `build_stages` **after** post-checks, guarded so it runs only on overall success.
- [ ] **Step 4:** Run → PASS.
- [ ] **Step 5: Write failing test** `test_close_skipped_when_cd_check_failed` (cd-check deferred-fail ⇒ task stays open). Run → FAIL → implement guard → PASS.
- [ ] **Step 6: Write failing test** `test_premature_close_guard_reopens_on_unmet_criteria` (stub criteria check) → implement `premature_close_guard` (acceptance-criteria presence check; flag/reopen) → PASS.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope finalize --message "close task + roll up epic at end of finalize" --body "Ref #<N>"`.

**Acceptance:** finalize closes the task only after merge+post-checks succeed; rolls up non-standing epics; leaves standing epics open; stragglers stay open on post-check failure. Tests green.

---

## Task 5: Linkage enforcement — reject epic-linked & multi-task PRs (`vergil-tooling`)

**Repo:** `vergil-project/vergil-tooling` · **Branch:** `feature/<N>-linkage-epic-guard`

**Files:**
- Modify: `src/vergil_tooling/bin/vrg_pr_issue_linkage.py` (CI validator) and `src/vergil_tooling/lib/linkage.py`
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (pre-submit check)
- Test: `tests/vergil_tooling/test_linkage.py`, `tests/vergil_tooling/test_vrg_pr_issue_linkage.py`

**Interfaces:**
- Consumes: `extract_tracking_issue()` (already raises on >1 Ref); `github.read_json("issue","view",N,"--json","labels")`.
- Produces: a validator rule — the single Ref'd issue must exist and must **not** carry the `epic` label; scoped to new-taxonomy issues so legacy issues don't trip it (skip the epic-label check when the issue has none of the convention labels).

**Execution note:** TDD steps mirror Task 2/4 — write `test_pr_linking_epic_is_rejected` and `test_legacy_issue_without_convention_labels_passes` first (mock `github.read_json` labels), run-fail, implement the label query + rejection in `vrg_pr_issue_linkage.main`, run-pass, then the matching `vrg-submit-pr` pre-check, then commit `feat(linkage): reject epic-linked and multi-task PRs`.

**Acceptance:** a PR `Ref`ing an `epic`-labelled issue fails the gate; multi-Ref already fails; legacy unlabeled issues pass. Tests green.

---

## Task 6: Intake skills — `triage` capture + `triage-review` (`vergil-claude-plugin`)

**Repo:** `vergil-project/vergil-claude-plugin` · **Branch:** `feature/<N>-intake-skills`

**Files:**
- Create: `skills/triage-capture/SKILL.md`, `skills/triage-review/SKILL.md`
- Modify: `docs/repository-standards.md` (document the `triage` intake convention)

**Interfaces:** SKILL.md frontmatter `{name, description}`; description leads with triggers (per `docs/development/skills-architecture.md`). Invoked `/vergil:triage-capture`, `/vergil:triage-review` (namespace `vergil`).

**Execution note:** no code/TDD — these are skill-definition docs. `triage-capture`: low-friction "create a `triage`-labelled issue in the most-relevant repo (or `.github` if project-level)"; description triggers on "don't forget", "capture this idea", "note a todo". `triage-review`: collect `label:triage` across repos + `.github` (via `vrg-gh issue list --label triage` per repo), walk each through the four dispositions (drop / assign-to-epic / route-to-standing / promote-to-epic→brainstorm), removing `triage` on disposition. Validate with `vrg-validate` (markdownlint) and commit `feat(skills): triage capture + review`.

**Acceptance:** both skills load (`/vergil:` namespace); descriptions trigger on natural phrasings; standards doc describes the intake queue.

---

## Task 7: Docs-lifecycle convention + on-ramp (`vergil-claude-plugin` + `.github`)

**Repo(s):** `vergil-claude-plugin` (standards/on-ramp) and `.github` (epics/ README) · two child tasks if split per repo.

**Files:**
- Modify: `vergil-claude-plugin/CLAUDE.md` + `docs/repository-standards.md` (epic/task model on-ramp; `.github/epics/<N>-<slug>/` doc home; durable-knowledge integration step; correct namespace to `vergil`).
- Create: `vergil-project/.github/epics/README.md` (explains the epics directory + roadmap derivation).

**Execution note:** documentation only. Capture, in `repository-standards.md` /
`CLAUDE.md` / `epics/README.md`:

- **Docs lifecycle:** epics own specs/plans in `.github/epics/<N>-<slug>/`;
  member-repo `docs/specs|plans/` are ad-hoc/standing only; epic open=in-flight /
  closed=complete (no `status:*`).
- **Transition / cutover (spec §5):** new work follows the model on day one; the
  existing backlog is grandfathered and reconciled by T9; enforcement applies
  only to new-taxonomy issues (legacy issues don't trip it); label ordering is
  **additive first (T2), retire cruft per-repo after migration (T9)**.
- **Reopen invariants (spec §3.3):** a task is **never** reopened; a revert or
  follow-up is a **new `bug`** (optionally `Ref`-linked to the culprit change);
  **only epics** are ever reopened.
- **`hotfix` emergency path (spec §3.4):** allowed under production pressure —
  create the task under the repo's standing epic, label `hotfix`, ship it; the
  label flags it for retroactive scrutiny so the formal path stays preferred.
- **Durable-knowledge integration (spec §3.6):** a **manual close-step** — before
  closing a finite epic, integrate durable rationale/contracts into the public
  surface (`profile/README.md` or site). Documented as a checklist item; not
  auto-enforced (a future enhancement may add a close-time prompt).

Validate markdownlint; commit `docs(convention): epic/task model, docs lifecycle, transition & reopen rules`.

**Acceptance:** a newcomer can read CLAUDE.md + repository-standards + epics/README
and follow the model end-to-end, including what happens to legacy issues, how
reverts and hotfixes are handled, and that tasks never reopen.

---

## Task 8: Observability — roadmap + activity-log generators (`vergil-tooling` + `.github`)

**Repo(s):** `vergil-tooling` (generators) + `.github` (output + nightly workflow).

**Files:**
- Create: `vergil-tooling/src/vergil_tooling/bin/vrg_roadmap.py` (+ `vrg-activity-log` or one CLI with subcommands), `lib/roadmap.py`
- Add: `[project.scripts]` entries in `vergil-tooling/pyproject.toml`
- Create: `vergil-project/.github/.github/workflows/observability.yml` (nightly `schedule`) + output `roadmap.md` / `activity-log.md`
- Test: `tests/vergil_tooling/lib/test_roadmap.py`

**Interfaces:** Consumes `epics.child_states`, `github.read_json` for issue/PR metadata (created/closed timestamps, repos, counts). Produces deterministic markdown: roadmap = open finite epics × (affected repos, child counts, created date, rolled-up last-activity timestamp); activity-log = finalized PRs / closed tasks-epics over day/week/month windows, drillable.

**Open decision (resolve in this task's detailed plan):** publish surface — generate into `.github/` and surface a digest in `profile/README.md` (no MkDocs site exists). Recommend: markdown files + a generated section in `profile/README.md`.

**Execution note:** TDD the pure-function generators first (feed fixture metadata, assert rendered markdown), then wire the nightly workflow. Commit per CLI.

**Acceptance:** `vrg-roadmap` and the activity-log generator produce stable markdown from fixtures; nightly workflow regenerates and commits them; timestamps make epic active-vs-parked visible without a stored status.

---

## Task 9: Migration passes — one child task per repo (per repo, brainstorm-driven)

**Repos:** every `vergil-project` member repo (start with the lab repos with auto-closeable backlogs).

**Not TDD-plannable** — each repo's pass is its own brainstorm-and-reconcile
session. Per-repo ordered sequence:

1. **Seed labels additively** — `vrg-ensure-label sync <repo>` (T2 labels; still no deletions).
2. **Create the standing epic** — one `Ad-hoc maintenance` issue in the repo,
   labels `epic`+`standing`, title `Epic (standing): Ad-hoc maintenance`. (Use a
   small `epics.create_standing(repo)` helper if tooled, else `vrg-gh issue create`.)
3. **Reconcile** — retro-create the implicit finite epics in `.github`, fold this
   repo's issues into them (as sub-issues / reflinks), label remaining loose
   small work into the standing epic or `triage`.
4. **Batch auto-close** already-done issues (`vrg-gh issue close` / a batch helper
   over `epics` lib), applying premature-close judgement.
5. **Retire cruft (now safe)** — apply this repo's `delete` list via
   `vrg-ensure-label` (the deletion deferred from T2 per §5), since every open
   issue now carries the new taxonomy.

**Per-repo acceptance:** every open issue is an epic, a task under an epic, or
`triage`; done issues closed; standing epic created; cruft labels retired;
new-taxonomy enforcement active.

**Epic done-definition:** this epic closes only when the last repo-migration child task closes — long-lived by design (dog-foods rollup + reopen-on-late-child).

---

## Self-review

- **Spec coverage:** §3.1–3.7 → T2 (labels, additive), T3/T4 (hierarchy+close+rollup+reopen), T5 (1:1 enforcement), T6 (intake + `hotfix`), T7 (docs lifecycle + **transition §5** + **reopen invariants §3.3** + `hotfix` §3.4 + durable-knowledge §3.6), T8 (observability), T9 + §5 (migration; cruft retirement). #681 remains out of scope. ✓
- **Placeholders:** tasks 1–4 are bite-sized; 5–9 carry exact paths/signatures/acceptance with explicit execution notes (expanded per-task at run time), not "TBD." ✓
- **Type consistency:** `IssueRef`, `ChildState`, `epics.child_states/parent_of/add_child/all_children_closed/rollup`, `github.graphql`, `linkage.extract_tracking_issue` used consistently across T3/T4/T8. ✓
- **Bootstrap — DONE (2026-06-29):** `epic` label created in `vergil-project/.github`;
  **epic = #40** (labeled `epic`); **docs task = #41** (child of #40 via `Parent:`
  reflink); slug = `40-epic-task-convention`. Native sub-issue link deferred to T3
  (`vrg-gh api` is correctly denied; tooling does it via `github.graphql()`).
