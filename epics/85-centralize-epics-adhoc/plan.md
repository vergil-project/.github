# Centralize epics in `.github` — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `.github` the single home of all epics (finite + ad-hoc), rename "standing" → "ad hoc", route the three intake queues to `.github`, and formalize the epic-bookend + `epic-create`-orchestrator conventions — shipped phased and backward-compatible.

**Architecture:** Additive/backward-compatible changes land first (new `ad-hoc` label + code, with `standing` kept as a live alias), then a one-time org data migration relocates existing standing epics into `.github`, then a final task retires the `standing` alias. Library logic lives in `lib/epics.py`; CLIs are thin wrappers; reporting is `lib/roadmap.py` + `lib/epic_audit.py`. Workflow conventions live as prose in `vergil-claude-plugin` skills.

**Tech Stack:** Python 3.12+, `argparse` CLIs, `pytest` (mocking the `github` boundary), `gh` CLI via `lib/github.py`, GitHub native sub-issues (GraphQL) with `Parent:` reflink fallback.

## Global Constraints

- **Epic home is `<org>/.github`**; org auto-detected from the current repo's origin remote (`github.detect_org()`), never hard-coded.
- **Task↔PR 1:1:** every task below is one PR, closed in the repo where its change lands, linked under epic `vergil-project/.github#85` via `vrg-epic-link`.
- **Phased rollout ordering (spec §9):** Phase 1 (labels + code, `standing` kept as deprecated alias) → Phase 2 (data migration) → Phase 3 (retire `standing`). Do **not** remove `standing` handling until Phase 3.
- **`idea` and `research` labels already exist** in `data/labels.json`; do not re-create them.
- **Validation:** `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` here) is the only validation command; run it before every PR handoff.
- **Tests** live at `tests/vergil_tooling/test_<module>.py` and mock `vergil_tooling.lib.github`.
- **Agents do not run `vrg-submit-pr`** — hand off via `vrg-pr-workflow report-ready`.

## File structure (what each task touches)

| Area | Files |
|---|---|
| Core model | `src/vergil_tooling/lib/epics.py`, `src/vergil_tooling/data/labels.json` |
| Ad-hoc CLI | `src/vergil_tooling/bin/vrg_adhoc_epic.py` (renamed), `pyproject.toml` |
| Sentinel callers | `src/vergil_tooling/bin/vrg_issue_create.py`, `vrg_epic_move.py` |
| Intake | `src/vergil_tooling/bin/vrg_triage_create.py` |
| Reporting | `src/vergil_tooling/lib/roadmap.py`, `lib/epic_audit.py`, `bin/vrg_epic_audit.py`, `bin/vrg_epic_rollup.py`, `bin/vrg_epic_create.py` |
| Migration | `src/vergil_tooling/bin/vrg_adhoc_migrate.py` (new, one-shot) |
| Skills (separate repo) | `vergil-claude-plugin`: `epic-create`, `triage-capture`, `migrate-repo` |
| Docs | `vergil-tooling/CLAUDE.md`, relevant `docs/` |

---

## Phase 1 — labels + code (backward-compatible; `standing` stays a live alias)

### Task 1: Core ad-hoc epic model in `lib/epics.py` (+ `ad-hoc` label)

**Repo:** vergil-tooling. **Depends on:** none (foundational).

**Files:**

- Modify: `src/vergil_tooling/data/labels.json` (add `ad-hoc`)
- Modify: `src/vergil_tooling/lib/epics.py`
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/test_labels.py`

**Interfaces:**

- Produces: `ensure_adhoc_epic(target_repo: str) -> IssueRef` — locate/create the ad-hoc epic for `target_repo` in `<org>/.github`, titled `Epic (ad hoc): <bare-name>`, labelled `epic` + `ad-hoc`.
- Produces: `resolve_epic_ref(ref, *, repo)` accepts `"adhoc"` and (alias) `"standing"`, both routing to `ensure_adhoc_epic(repo)`.
- Produces: `rollup()` treats a parent labelled `ad-hoc` **or** `standing` as perpetual (never auto-closes).

- [ ] **Step 1: Add the `ad-hoc` label.** In `data/labels.json`, add after the `standing` entry:

```json
{"name": "ad-hoc", "color": "5319e7", "description": "Perpetual umbrella for ad-hoc work; never auto-closes"},
```

Leave `standing` in place (retired in Task 11). Add/extend `test_labels.py` to assert `ad-hoc` is present.
**Precondition for live use:** `gh issue create --label ad-hoc` fails unless the label already exists in the target repo, so `ensure_adhoc_epic` cannot mint an epic in `.github` until `ad-hoc` is **synced into `<org>/.github`** via the label-sync path (`vrg-ensure-label` / the labels.json apply). This sync must land before Task 10; it is the real-world gate on Phase 2.

- [ ] **Step 2: Write failing tests for `ensure_adhoc_epic`** in `test_epics.py`:

```python
def test_ensure_adhoc_epic_finds_by_title_in_dotgithub(monkeypatch):
    calls = {}
    def fake_read_json(*args, **kw):
        # issue list in <org>/.github filtered by epic+ad-hoc
        return [{"number": 40, "title": "Epic (ad hoc): vergil-tooling"}]
    monkeypatch.setattr(epics.github, "read_json", fake_read_json)
    ref = epics.ensure_adhoc_epic("vergil-project/vergil-tooling")
    assert ref == epics.IssueRef("vergil-project", ".github", 40)

def test_ensure_adhoc_epic_creates_when_absent(monkeypatch):
    monkeypatch.setattr(epics.github, "read_json", lambda *a, **k: [])
    created = {}
    def fake_create_issue(**kw):
        created.update(kw)
        return "https://github.com/vergil-project/.github/issues/99"
    monkeypatch.setattr(epics.github, "create_issue", fake_create_issue)
    ref = epics.ensure_adhoc_epic("vergil-project/vergil-tooling")
    assert ref == epics.IssueRef("vergil-project", ".github", 99)
    assert created["repo"] == "vergil-project/.github"
    assert created["title"] == "Epic (ad hoc): vergil-tooling"
    assert set(created["labels"]) == {"epic", "ad-hoc"}
```

Run: `uv run pytest tests/vergil_tooling/test_epics.py -k adhoc -v` → FAIL (no `ensure_adhoc_epic`).

- [ ] **Step 3: Implement `ensure_adhoc_epic`.** Replace `_STANDING_EPIC_*` with `_ADHOC_EPIC_TITLE_PREFIX = "Epic (ad hoc): "`, `_ADHOC_EPIC_LABELS = ("epic", "ad-hoc")`, and a body constant. New function:

```python
def ensure_adhoc_epic(target_repo: str) -> IssueRef:
    """Return target_repo's ad-hoc epic in <org>/.github, creating it if absent.

    All ad-hoc epics live in the org's .github, one per repo, disambiguated by
    the title 'Epic (ad hoc): <bare repo name>'. Idempotent.
    """
    if "/" not in target_repo:
        raise ValueError(f"cannot resolve repo for ad-hoc epic (repo={target_repo!r})")
    owner, bare = target_repo.split("/", 1)
    dotgithub = f"{owner}/.github"
    title = f"{_ADHOC_EPIC_TITLE_PREFIX}{bare}"
    raw = github.read_json(
        "issue", "list", "--repo", dotgithub,
        "--label", "epic", "--label", "ad-hoc",
        "--state", "open", "--json", "number,title",
    )
    rows = [r for r in raw if isinstance(r, dict) and r.get("title") == title] \
        if isinstance(raw, list) else []
    if len(rows) > 1:
        nums = ", ".join(f"#{r['number']}" for r in rows)
        raise ValueError(f"multiple ad-hoc epics titled {title!r} in {dotgithub} ({nums})")
    if rows:
        return IssueRef(owner=owner, repo=".github", number=int(rows[0]["number"]))
    url = github.create_issue(repo=dotgithub, title=title,
                              body=_ADHOC_EPIC_BODY, labels=list(_ADHOC_EPIC_LABELS))
    number = int(url.rstrip("/").rsplit("/", 1)[-1])
    return IssueRef(owner=owner, repo=".github", number=number)
```

Keep a thin `ensure_standing_epic = ensure_adhoc_epic` module-level alias so external callers survive Phase 1.

- [ ] **Step 4: Sentinel + rollup.** In `resolve_epic_ref`, change the sentinel branch to `if ref in ("adhoc", "standing"): return ensure_adhoc_epic(repo)`. In `rollup`, change the perpetual guard to `if _labels(parent) & {"ad-hoc", "standing"}: return`.

- [ ] **Step 5: Run tests.** `uv run pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_labels.py -v` → PASS. Update any existing standing-epic tests to the new title/label expectations (keep alias assertions).

- [ ] **Step 6: Validate + commit.** `vrg-container-run -- vrg-validate`; `vrg-commit --type feat --scope epics --message "add ad-hoc epic model in .github (deprecate standing) (#<TASK>)"`.

### Task 2: Rename CLI `vrg-standing-epic` → `vrg-adhoc-epic`

**Repo:** vergil-tooling. **Depends on:** Task 1.

**Files:**

- Rename: `src/vergil_tooling/bin/vrg_standing_epic.py` → `bin/vrg_adhoc_epic.py`
- Modify: `pyproject.toml` (`[project.scripts]`)
- Test: rename `tests/vergil_tooling/test_vrg_standing_epic.py` → `test_vrg_adhoc_epic.py`

- [ ] **Step 1:** `git mv` the bin and test modules; update `prog="vrg-adhoc-epic"`, docstring, and help to "ad-hoc" wording; call `epics.ensure_adhoc_epic(repo)`. `ensure` keeps `--repo` (the target repo whose ad-hoc epic to ensure in `.github`).
- [ ] **Step 2:** In `pyproject.toml`, add `vrg-adhoc-epic = "vergil_tooling.bin.vrg_adhoc_epic:main"` and **keep** `vrg-standing-epic = "vergil_tooling.bin.vrg_adhoc_epic:main"` as a deprecated alias (both point to the new module). Removed in Task 11.
- [ ] **Step 3:** Update test to assert the printed `Ad-hoc epic: ...` and that the alias entrypoint still resolves. Run `uv run pytest tests/vergil_tooling/test_vrg_adhoc_epic.py -v` → PASS.
- [ ] **Step 4:** Validate + commit (`--type refactor --scope cli`).

### Task 3: Sentinel-aware cross-org guard in `vrg-issue-create` + `vrg-epic-move`

**Repo:** vergil-tooling. **Depends on:** Task 1.

**Files:** Modify `bin/vrg_issue_create.py`, `bin/vrg_epic_move.py`; tests `test_vrg_issue_create.py`, `test_vrg_epic_move.py`.

- [ ] **Step 1:** Write a failing test: `--epic adhoc` must skip the cross-org owner guard (like `standing`) and route to the `.github` ad-hoc epic. Run → FAIL.
- [ ] **Step 2:** In both files, change the guard predicate `if args.epic != "standing":` → `if args.epic not in ("standing", "adhoc"):`. Update the inline comments to mention both sentinels.
- [ ] **Step 3:** Update `vrg_issue_create.py` docstring line ("pass `--epic standing`") to name `--epic adhoc` as the current form (standing = deprecated alias). Run tests → PASS.
- [ ] **Step 4:** Validate + commit (`--type fix --scope cli`).

### Task 4: Intake `--kind` + default-to-`.github` in `vrg-triage-create`

**Repo:** vergil-tooling. **Depends on:** none (labels exist). **Supersedes #2075's current-repo default.**

**Files:** Modify `bin/vrg_triage_create.py`; test `test_vrg_triage_create.py`.

**Interfaces:** Produces: `vrg-triage-create --kind {triage,idea,research}` (default `triage`), primary label = kind; `--repo` defaults to `<org>/.github`.

- [ ] **Step 1: Failing tests:**

```python
def test_kind_research_labels_and_targets_dotgithub(monkeypatch):
    monkeypatch.setattr(tc.github, "detect_org", lambda: "vergil-project")
    seen = {}
    monkeypatch.setattr(tc.github, "create_issue",
        lambda **kw: seen.update(kw) or "https://github.com/vergil-project/.github/issues/5")
    tc.main(["--title", "x", "--kind", "research"])
    assert seen["repo"] == "vergil-project/.github"
    assert seen["labels"] == ["research"]

def test_default_kind_is_triage(monkeypatch):
    ...  # asserts labels == ["triage"] and repo defaults to <org>/.github
```

Run → FAIL.

- [ ] **Step 2:** Add `--kind` with `choices=["triage", "idea", "research"], default="triage"`. Build `labels = list(dict.fromkeys([args.kind, *args.label]))`. Change default repo: `repo = args.repo or f"{github.detect_org()}/.github"`. Update `prog` description/epilog to describe all three kinds and the `.github` default; note it supersedes the current-repo default.
- [ ] **Step 3:** Run tests → PASS. **Step 4:** Validate + commit (`--type feat --scope cli`).

### Task 5: Reporting rename (alias-aware) — `roadmap.py` + `epic_audit.py`

**Repo:** vergil-tooling. **Depends on:** Task 1.

**Files:** Modify `lib/roadmap.py`, `lib/epic_audit.py`, `bin/vrg_epic_rollup.py` (docstring), `bin/vrg_epic_create.py` (help example); tests `test_roadmap.py`, `test_epic_audit.py`.

- [ ] **Step 1:** In `roadmap.py`, rename `_is_standing` → `_is_perpetual` and match **either** `standing` or `ad-hoc` (alias window): `any(l.get("name") in ("standing", "ad-hoc") for l in labels)`. Update docstring ("skipping standing/ad-hoc buckets"). This preserves the existing "strategic view = `epic AND NOT ad-hoc`" behavior.
- [ ] **Step 2:** In `epic_audit.py`, update `epic_drift` / `task_drift` to treat both labels as perpetual (skip both). Update the module docstring wording.
- [ ] **Step 3:** In `vrg_epic_rollup.py` docstring ("non-standing parent") and `vrg_epic_create.py` help (`--label standing` example → `--label ad-hoc`), swap wording.
- [ ] **Step 4:** Run `uv run pytest tests/vergil_tooling/test_roadmap.py tests/vergil_tooling/test_epic_audit.py -v` → PASS (add a case asserting an `ad-hoc`-labelled epic is skipped from the strategic roll-up). **Step 5:** Validate + commit (`--type refactor --scope reporting`).

### Task 6: Invariant-enforcement checks in `vrg-epic-audit`

**Repo:** vergil-tooling. **Depends on:** Task 5.

**Files:** Modify `lib/epic_audit.py`, `bin/vrg_epic_audit.py`; test `test_epic_audit.py`, `test_vrg_epic_audit.py`.

**Interfaces:** Produces two report-only checks (human-gated close/fix stays as-is):

1. `epic_outside_dotgithub(org) -> list[IssueRef]` — open `epic`-labelled issues in any repo other than `.github` (violates invariant 1).
2. `stray_dotgithub_issue(org) -> list[IssueRef]` — open `.github` issues that are **not** an epic, **not** intake (`triage`/`idea`/`research`), and **not** a self-referential task (one whose closing PR is in `.github`) (violates invariant 2). For the self-referential exemption, treat any issue linked under a `.github` ad-hoc epic (or any non-epic issue with an open/merged `.github` PR ref) as allowed; when uncertain, report it (fail-loud, human decides).

- [ ] **Step 1:** Failing tests: seed a fake `github.read_json` returning (a) an `epic`-labelled issue in `vergil-project/vergil-tooling` → flagged by check 1; (b) a plain issue in `.github` with no epic/intake/self-ref → flagged by check 2; (c) a `triage`-labelled `.github` issue → **not** flagged. Run → FAIL.
- [ ] **Step 2:** Implement both functions in `epic_audit.py`; add a `render` section for each (headed "Invariant violations"). Wire into `vrg_epic_audit.py` output alongside drift. Keep read-only default.
- [ ] **Step 3:** Run tests → PASS. **Step 4:** Validate + commit (`--type feat --scope epic-audit`).

### Task 12: Docs + `CLAUDE.md` sweep (standing→ad-hoc; docs demotion)

**Repo:** vergil-tooling (and any doc references). **Depends on:** Tasks 1–6 (wording should match shipped behavior). Can land anytime in Phase 1.

**Files:** `CLAUDE.md`, `docs/**` references to "standing epic"; note `docs` is a normal repo.

- [ ] **Step 1:** Grep `grep -rin "standing epic\|standing" CLAUDE.md docs/`; rewrite to "ad-hoc epic" (note `standing` is a deprecated alias until Task 11). **Step 2:** Add a one-line note that `docs` is an ordinary member repo (no catch-all role). **Step 3:** Validate + commit (`--type docs`).

> **Coverage note (alignment):** spec §5 says "audit tooling/skills for docs-as-catch-all assumptions and remove them." The catch-all was **never implemented in code** — it was an aspirational idea dropped during brainstorming — so there is nothing to remove; this docs-only task is full coverage of §5. If a grep for `docs` special-casing in `src/` does surface real code, file it as a follow-up task rather than expanding this one.

---

## Phase 1 (skills — separate repo `vergil-claude-plugin`)

### Task 7: `epic-create` skill — orchestrator inversion, bookends, entry point, interaction doctrine

**Repo:** vergil-claude-plugin. **Depends on:** conceptually on this spec. **Kept as one task per the epic's scope decision; it decomposes into the four concrete parts below as commits within one PR, not separate issues — unless a part grows large enough to warrant its own PR.**

**Deliverable:** Rewrite `skills/epic-create/SKILL.md` so `epic-create` is the **outer** orchestrating workflow, covering four concrete parts:

- **7-A — Orchestration sequence (spec §7):** epic-create runs `superpowers:brainstorming` → initialize epic + seed bookend tasks → write spec → `paad:pushback` → human review → `superpowers:writing-plans` → `paad:alignment` → single docs PR (closes the docs task) → file implementation tasks. Name the exact handoff points and which skill runs at each.
- **7-B — Bookend convention (spec §6):** every epic ≥ 2 tasks (docs-first, review-last); the review task gates closure via the existing rollup.
- **7-C — Default entry point (spec §7.1):** epic-create is the default start for non-trivial work (not brainstorming directly); a trivial single-PR design drops onto the target repo's ad-hoc epic instead of minting a finite epic.
- **7-D — Four-stage interaction doctrine (spec §7.2):** document which stages are interactive (brainstorm, pushback, alignment) vs automated (writing-plans), and the human-judgment principle — gate only material judgment calls; batch minor no-brainers into a single end-of-stage review.

- [ ] **Step 1:** Draft 7-A (sequence + handoff points). **Step 2:** Add 7-B bookend prose. **Step 3:** Add 7-C default-entry and 7-D interaction doctrine (the stage table + human-judgment principle). **Step 4:** Add a worked example (this epic #85 — brainstorm→pushback→plans→alignment→docs-PR). **Step 5:** Verify via the plugin's skill-doc checks (markdownlint / `vrg-validate` in that repo). **Step 6:** Commit; PR against `vergil-claude-plugin`.

### Task 8: Intake-creating skills — route intake to `.github` with `--kind`

**Repo:** vergil-claude-plugin. **Depends on:** Task 4 (tool must accept `--kind`, default `.github`).

**Deliverable:** Update every skill that creates intake so it files into `<org>/.github` (never a member repo), choosing `--kind {triage,idea,research}` by the captured shape:

- `skills/triage-capture/SKILL.md` — remove the "most-relevant repo" routing; always target `.github`; add `--kind` selection guidance.
- `skills/memory-audit/SKILL.md` — surfaced in alignment (it also calls `vrg-triage-create`). Update its intake creation to the same `.github` + `--kind` routing so it doesn't silently file into the current repo.

- [ ] **Step 1:** Rewrite `triage-capture` routing + `--kind` guidance. **Step 2:** Update `memory-audit`'s intake call to `.github` + `--kind`. **Step 3:** Validate + PR (one PR covering both skill edits).

### Task 9: `migrate-repo` skill — align to ad-hoc model + intake routing

**Repo:** vergil-claude-plugin. **Depends on:** Tasks 1, 4.

**Deliverable:** Update `skills/migrate-repo/SKILL.md` so backlog grooming buckets ad-hoc work under the repo's **`.github`-resident** ad-hoc epic (via `--epic adhoc`), and routes intake to `.github` with `--kind`. Reflect that member repos hold only single-PR tasks.

- [ ] **Step 1:** Update the standing→ad-hoc references and routing. **Step 2:** Validate + PR.

---

## Phase 2 — data migration (after Phase 1 code is deployed)

### Task 10: One-time migration of existing standing epics into `.github`

**Repo:** vergil-tooling (ship a one-shot `bin/vrg_adhoc_migrate.py`), executed by a human/USER agent against the org. **Depends on:** Tasks 1, 2 **deployed** (labels + `ensure_adhoc_epic` available).

**Deliverable:** For each member repo (+ `.github`, `docs`) with an `Epic (standing): Ad-hoc maintenance`:

1. `ensure_adhoc_epic(<owner>/<repo>)` → create `Epic (ad hoc): <repo>` in `.github`.
2. Enumerate the old standing epic's **open** children (`epics.child_states`) and re-link each to the new epic (`epics.add_child`); leave closed children with the retired epic for provenance.
3. Close the old per-repo standing epic with a comment pointing at the new `.github` epic.

- [ ] **Step 1:** Write `vrg_adhoc_migrate.py` with `--dry-run` (default) that prints the planned create/relink/close actions per repo, and `--apply`. Mock-tested in `test_vrg_adhoc_migrate.py`.
- [ ] **Step 2:** **Dry-run across the org**; a human reviews the plan. Verify cross-repo native sub-issue linking works on one repo first (spec §9 note).
- [ ] **Step 3:** `--apply`, repo by repo; verify each new epic's children and that roadmap/audit render correctly. **Step 4:** Commit the tool; record the migration run in epic #85.

---

## Phase 3 — retire `standing` (after migration verified)

### Task 11: Remove the `standing` alias everywhere

**Repo:** vergil-tooling. **Depends on:** Task 10 complete and verified; no open issues still rely on `standing`.

**Files:** `lib/epics.py`, `bin/vrg_issue_create.py`, `vrg_epic_move.py`, `lib/roadmap.py`, `lib/epic_audit.py`, `pyproject.toml`, `data/labels.json`; tests.

- [ ] **Step 1:** Remove the `ensure_standing_epic` alias and the `"standing"` sentinel branch (keep only `"adhoc"`). Guards become `if args.epic != "adhoc":`. Perpetual checks match only `ad-hoc`.
- [ ] **Step 2:** Drop the `vrg-standing-epic` console-script alias from `pyproject.toml`.
- [ ] **Step 3:** In `data/labels.json`, remove the `standing` entry and add `"standing"` to the `delete` list so `vrg-ensure-label` retires it org-wide.
- [ ] **Step 4:** Update tests to drop alias assertions; `uv run pytest` green. **Step 5:** Validate + commit (`--type refactor --scope epics --message "retire deprecated standing alias (#<TASK>)"`).

---

## Sequencing summary

```bash
Phase 1 (any order, but 1 before 3/5/6):  T1 → {T2, T3, T5→T6}, T4, T12 ; skills T7, T8(after T4), T9
Phase 2:  T10  (requires T1,T2 deployed)
Phase 3:  T11  (requires T10 verified)
```

Bookend review tasks #87–#90 are dispositioned last; epic #85 cannot roll up until then (spec §6, §8).

## Self-review notes

- **Spec coverage:** §3→T1/T2; §4→T4/T8; §5→T12; §6→T7; §7→T7; §9→T1(alias)/T10/T11; §10→T6; §11→T5; migration→T10; component checklist §12 all mapped.
- **Labels:** `idea`/`research` pre-exist (no task creates them); only `ad-hoc` added (T1), `standing` retired (T11). Verified against `data/labels.json`.
- **Names:** `ensure_adhoc_epic`, `resolve_epic_ref` sentinel `"adhoc"`, perpetual label set `{"ad-hoc","standing"}` (Phase 1) → `{"ad-hoc"}` (Phase 3) used consistently across T1/T3/T5/T11.
- **Open sub-decision (spec §4.1):** keep tool name `vrg-triage-create` (T4 does not rename it) — revisit only if alignment disagrees.
