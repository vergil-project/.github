# Every Issue Belongs to an Epic — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. In this framework each task below is a filed issue implemented with `issue-implement` (operational tasks with `issue-validate` / `issue-deploy`), driven by `epic-implement`.

> **Revision 2 (2026-10-23, task #430).** Rewritten for spec revision 2 (the intake → planning model absorbed from #429). Revision 1's tasks had not started; their issue numbers are reused where the scope survives (see *Task ↔ issue map*).

**Goal:** Make every issue in a managed repo either a legitimate top-level kind or a child of an epic, with unreviewed work in one intake queue, epic creation queued and tracked in one planning queue, and a parent-based PR refusal that keeps both queues un-buildable.

**Architecture:** All model logic extends `vergil_tooling/lib/epics.py`, which already has the ad-hoc standing-epic pattern, the archive-and-drain machinery, sub-issue linking, and the visibility-based epic home. We generalize "standing epic" into a small `StandingKind` table (ad-hoc, intake, planning) shared by creation, rollup, drain, and roadmap; add an org-wide orphan pass to the existing `vrg-epic-audit --close` sweep; add two CLIs (`vrg-intake-route`, `vrg-queue`); rename `vrg-triage-create` → `vrg-intake-create`; and rewrite the skills in `vergil-claude-plugin`.

**Tech Stack:** Python 3.12+ (`vergil-tooling`), `pytest` with 100% branch coverage, `gh`/GraphQL via `vergil_tooling.lib.github`, Claude Code skills (Markdown) in `vergil-claude-plugin`.

**Spec:** `epics/176-every-issue-in-epic/spec.md` (revision 2). Executors read both.

## Global Constraints

- Validation is exactly `vrg-container-run -- vrg-validate` (expands to `uv run vrg-validate` in vergil-tooling). Never run linters standalone. For red/green, run focused tests with `vrg-container-run -- uv run pytest <path>::<test> -v`.
- **100% branch coverage** is enforced. Every new line and branch needs a test.
- Git/GitHub only via `vrg-git` / `vrg-gh`. Commits via `vrg-commit --type <t> --scope <s> --message <m> [--body <b>]`.
- One PR per task, filed and landed in the repo named in its heading. No PR `Closes` an issue across repos.
- **No silent failures:** every fallback (transfer → recreate, best-effort guard) prints what it did. A full queue stops and reports, never skips.
- Standing epics: titles `Epic (intake): <name>` / `Epic (planning): <name>`, labels `epic`+`intake` / `epic`+`planning`. Archives: `Archive (intake): <name> — YYYY-Qn` / `Archive (planning): <name> — YYYY-Qn`, labels `archive`+`intake` / `archive`+`planning`. `<name>` is the org for the `.github` pair, the bare repo name for a private repo's self-homed pair.
- Legitimate top-level kinds: `epic` label, `archive` label, or `release.is_release_tracking_issue(title, body)`.
- Retired labels `triage` / `idea` / `research`: no code or skill applies them after this epic. They stay in `data/labels.json`.
- Single per-item planning label: `in-progress`.
- Batch size for intake review: 4. Cap: reuse `_ADHOC_ARCHIVE_SEGMENT_CAP` (95) for open children of intake/planning.
- Reuse, don't duplicate: `resolve_epic_home`, `_find_epic_by_title`, `add_child`, `reparent_child`, `parent_of`, `child_states`, `_labels`, `ensure_labels`, `github.create_issue`, `github.list_org_repos`, `github.is_public`, `remote_config.fetch_config_from_ref`, `release.is_release_tracking_issue`, and the `_reject_operational_issue` / `_reject_if_operational_task` guard pattern.

## Review Focus

1. **A release in flight during a sweep:** an open `release: X.Y.Z` issue with the `<!-- vrg-release:progress -->` marker but no parent must not be attached. Pinned by Task 5's `test_orphan_drift_skips_release_tracking_issue`.
2. **A private repo's orphan or capture:** must land in that repo's own intake epic, never `.github`. Pinned by Task 1's `test_queue_home_private_repo_self_homes` and Task 3's `test_intake_create_private_repo_files_in_repo`.
3. **A transfer that GitHub refuses** (permissions, private → public, API error): must fall back to recreate-and-close and say so on stderr, never leave the item half-moved. Pinned by Task 6's `test_route_one_off_falls_back_to_recreate_when_transfer_fails`.
4. **A planning issue whose epic exists but whose docs PR has not merged:** must show as in progress (waiting on docs), not ready to close. Pinned by Task 7's `test_planning_ready_requires_docs_task_closed`.
5. **The intake epic at the cap:** the migration must stop with a message naming the epic and the remaining count, not attach some and drop the rest. Pinned by Task 5's `test_attach_orphans_stops_at_cap`.

## Task ↔ issue map

| Task | Repo | Issue |
| --- | --- | --- |
| 1 Standing kinds + resolver + predicates | vergil-tooling | rewrites `#2481` |
| 2 Generalized archive drain | vergil-tooling | new |
| 3 `vrg-intake-create` rename + capture | vergil-tooling | merges `#2484` + tool half of `#2482` |
| 4 PR refusal by parent | vergil-tooling | rewrites `#2485` (closes `.github#175` via `Ref`) |
| 5 Orphan pass + migration in `vrg-epic-audit` | vergil-tooling | rewrites `#2483` |
| 6 `vrg-intake-route` | vergil-tooling | new |
| 7 `vrg-queue` | vergil-tooling | new |
| 8 Skills | vergil-claude-plugin | skill half of `#2482` (new issue in plugin repo) |
| 9 Deployment: release tooling + plugin | vergil-tooling | new, `--kind deployment` |
| 10 Validation: one-shot migration per org | .github | rewrites `#185` |

Order: 1 → 2 → (3, 4, 5, 6, 7 in any order) → 8 → 9 → 10. Task 2 must ship in the same release as Task 1 or before any standing intake/planning epic exists, because until it lands the rollup does not drain their closed children.

---

### Task 1: Standing kinds, queue homes, resolver, and predicates [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add `StandingKind`, kinds table, `queue_home`, `ensure_standing_epic`, `ensure_intake_epic`, `ensure_planning_epic`, `is_standing`, `queue_kind`, `is_legitimate_top_level`, `is_managed_repo`, `native_parent_of`, `resolve_orphan_epic`; extend `resolve_epic_ref`; route `ensure_adhoc_epic` through `ensure_standing_epic`; make `rollup` treat every standing kind as perpetual)
- Modify: `src/vergil_tooling/lib/roadmap.py:49-53` (`_is_perpetual` → `epics.is_standing`)
- Modify: `src/vergil_tooling/lib/epic_audit.py:376` (perpetual check → `epics.is_standing`)
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/test_roadmap.py`

**Interfaces:**

- Produces:
  - `@dataclass(frozen=True) class StandingKind: key: str; title_prefix: str; labels: tuple[str, str]; archive_prefix: str; archive_labels: tuple[str, str]; body: str`
  - `ADHOC: StandingKind`, `INTAKE: StandingKind`, `PLANNING: StandingKind`, `STANDING_KINDS: tuple[StandingKind, ...]`
  - `queue_home(target_repo: str) -> tuple[str, str]` — `(home "owner/repo", name)`: public → `("<org>/.github", "<org>")`; private self-homed → `("<org>/<bare>", "<bare>")`
  - `ensure_standing_epic(kind: StandingKind, home: str, name: str) -> IssueRef`
  - `ensure_intake_epic(target_repo: str) -> IssueRef`, `ensure_planning_epic(target_repo: str) -> IssueRef`
  - `is_standing(labels: set[str]) -> bool` — True iff `"epic"` in labels and any of `ad-hoc` / `intake` / `planning`
  - `queue_kind(epic: IssueRef) -> str | None` — `"intake"` / `"planning"` for a live intake/planning epic, else None
  - `is_legitimate_top_level(labels: set[str], title: str, body: str) -> bool`
  - `is_managed_repo(repo: str) -> bool` — `vergil.toml` present on the default branch
  - `native_parent_of(task: IssueRef) -> IssueRef | None` — native sub-issue parent only (no body fallback)
  - `resolve_orphan_epic(repo: str) -> IssueRef` — `ensure_intake_epic(repo)`
  - `resolve_epic_ref` accepts `"intake"` and `"planning"` sentinels in addition to `"adhoc"`

- [ ] **Step 1: Failing tests**

```python
def test_queue_home_public_repo_uses_org_dotgithub(monkeypatch):
    monkeypatch.setattr(epics, "resolve_epic_home", lambda org, bare: "org/.github")
    assert epics.queue_home("org/tool") == ("org/.github", "org")

def test_queue_home_private_repo_self_homes(monkeypatch):
    monkeypatch.setattr(epics, "resolve_epic_home", lambda org, bare: "org/secret")
    assert epics.queue_home("org/secret") == ("org/secret", "secret")

def test_ensure_intake_epic_reuses_by_title_and_labels(monkeypatch):
    seen = {}
    def fake_find(home, title, *, prefer_oldest=False, labels=("epic", "ad-hoc")):
        seen.update(home=home, title=title, labels=labels)
        return IssueRef("org", ".github", 7)
    monkeypatch.setattr(epics, "queue_home", lambda r: ("org/.github", "org"))
    monkeypatch.setattr(epics, "_find_epic_by_title", fake_find)
    assert epics.ensure_intake_epic("org/tool") == IssueRef("org", ".github", 7)
    assert seen == {"home": "org/.github", "title": "Epic (intake): org", "labels": ("epic", "intake")}

def test_ensure_planning_epic_creates_when_absent(monkeypatch): ...
    # create_issue called with title "Epic (planning): org", labels ["epic", "planning"]

def test_ensure_planning_epic_ensures_in_progress_label(monkeypatch): ...
    # ensure_labels(home, ("in-progress",)) called even when the epic already exists

def test_ensure_adhoc_epic_unchanged_through_standing_helper(): ...
    # existing ensure_adhoc_epic tests keep passing unmodified

def test_is_standing():
    assert epics.is_standing({"epic", "intake"}) and epics.is_standing({"epic", "planning"})
    assert epics.is_standing({"epic", "ad-hoc"})
    assert not epics.is_standing({"epic"}) and not epics.is_standing({"archive", "intake"})

def test_is_legitimate_top_level():
    assert epics.is_legitimate_top_level({"epic"}, "x", "")
    assert epics.is_legitimate_top_level({"archive", "ad-hoc"}, "Archive (ad hoc): t — 2026-Q4", "")
    assert epics.is_legitimate_top_level(set(), "release: 2.1.258", "")
    assert not epics.is_legitimate_top_level({"bug"}, "fix: thing", "")

def test_queue_kind_reads_parent_labels(monkeypatch): ...
    # {"epic","intake"} -> "intake"; {"epic","planning"} -> "planning"; {"epic","ad-hoc"} -> None;
    # {"archive","intake"} -> None (archives are not queues)

def test_is_managed_repo_true_when_vergil_toml_present(monkeypatch): ...
def test_is_managed_repo_false_on_404(monkeypatch): ...
def test_is_managed_repo_propagates_other_api_errors(monkeypatch): ...

def test_native_parent_of_ignores_body_parent_line(monkeypatch): ...
    # graphql parent null + body "Parent: org/.github#5" -> None

def test_resolve_epic_ref_intake_and_planning_sentinels(monkeypatch): ...
    # "intake" -> ensure_intake_epic(repo); "planning" -> ensure_planning_epic(repo)

def test_rollup_never_closes_intake_or_planning_epic(monkeypatch): ...
    # parent labels {"epic","intake"}, all children closed -> no "issue close" call

def test_roadmap_skips_intake_and_planning_epics(): ...  # in test_roadmap.py
```

- [ ] **Step 2: Run them — expect FAIL**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_epics.py tests/vergil_tooling/test_roadmap.py -v -k "queue_home or intake or planning or standing or legitimate or managed or native_parent or sentinels"`
Expected: FAIL (`AttributeError: module 'vergil_tooling.lib.epics' has no attribute 'queue_home'`).

- [ ] **Step 3: Implement in `src/vergil_tooling/lib/epics.py`**

  - `ADHOC` carries the existing `_ADHOC_*` constants. `INTAKE`/`PLANNING` carry the Global Constraints titles and labels, with bodies `"Perpetual intake queue for {name}: every captured, unreviewed item and every orphan the sweep finds. Reviewed oldest first by intake-review.\n"` and `"Perpetual planning queue for {name}: items that need a full epic, each tracked through its epic-create session.\n"`.
  - `ensure_standing_epic` is the body of today's `ensure_adhoc_epic` with `kind` substituted (`_find_epic_by_title(home, title, labels=kind.labels)`, `ensure_labels(home, kind.labels)`, `create_issue(..., labels=list(kind.labels))`). `ensure_adhoc_epic` calls it with `(ADHOC, resolve_epic_home(owner, bare), bare)`, so its behavior and tests are unchanged.
  - `ensure_planning_epic` also calls `ensure_labels(home, ("in-progress",))`, so `epic-create` can apply the label in any planning home.
  - `queue_home`: `home = resolve_epic_home(owner, bare)`; name is `owner` when `home == f"{owner}/.github"`, else `bare`.
  - `is_managed_repo`: `github.read_json("api", f"repos/{repo}/contents/vergil.toml")`; `GitHubAPIError` with `remote_config.is_not_found(exc)` → False; other errors propagate.
  - `native_parent_of`: the GraphQL half of `parent_of`; refactor `parent_of` to call it, then fall back to the body regex.
  - `rollup`: replace `if "ad-hoc" in _labels(parent):` with a lookup of the parent's standing kind. Ad-hoc keeps today's per-repo drain path. Intake/planning delegate to Task 2's `drain_standing_child` (until Task 2 lands, they return without closing).
  - `roadmap._is_perpetual` and `epic_audit` line 376 call `epics.is_standing`.

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate` (all green, 100% coverage).

- [ ] **Step 4b: Refactor.** Fold `_ADHOC_EPIC_*` constants into `ADHOC` so there is one source per kind (keep the old names only where tests or other modules import them). Check that no `"ad-hoc" in labels` perpetual check remains outside `is_standing` (`grep -rn '"ad-hoc" in' src/`). Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope epics --message "add intake and planning standing epics, queue homes, and the legitimate-top-level predicate" --body "Ref #2481"`

---

### Task 2: Generalized archive drain for intake and planning [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (parameterize the archive helpers by `StandingKind`; add `drain_standing_child`, `drain_standing_org`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (call `drain_standing_org` beside `drain_adhoc_org` in `--close` and preview)
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: Task 1's `StandingKind`, `INTAKE`, `PLANNING`, `queue_home`.
- Produces:
  - `archive_title(kind: StandingKind, name: str, quarter: str, segment: int = 1) -> str` (ad-hoc output identical to today's `_archive_title`)
  - `ensure_writable_standing_archive(kind: StandingKind, home: str, name: str, quarter: str) -> IssueRef`
  - `drain_standing_child(kind: StandingKind, parent: IssueRef, task: IssueRef) -> None` — the single-child rollup path
  - `drain_standing_org(org: str, *, apply: bool, now: datetime) -> list[DrainPlan]` — the intake and planning epics of `.github` and of every private repo's self-home

- [ ] **Step 1: Failing tests**

```python
def test_archive_title_intake():
    assert epics.archive_title(epics.INTAKE, "org", "2026-Q4") == "Archive (intake): org — 2026-Q4"
    assert epics.archive_title(epics.INTAKE, "org", "2026-Q4", segment=2) == "Archive (intake): org — 2026-Q4 (2)"

def test_archive_title_adhoc_unchanged():
    assert epics.archive_title(epics.ADHOC, "tool", "2026-Q4") == epics._archive_title("tool", "2026-Q4", 1)

def test_rollup_drains_closed_intake_child_into_quarter_archive(monkeypatch): ...
    # parent {"epic","intake"} titled "Epic (intake): org"; task closedAt 2026-10-23 ->
    # reparent_child(<Archive (intake): org — 2026-Q4>, task); parent never closed

def test_rollup_leaves_intake_archive_terminal(monkeypatch): ...
    # parent {"archive","intake"} -> no reparent, no close

def test_drain_standing_org_covers_dotgithub_and_private_self_homes(monkeypatch): ...
    # list_org_repos -> [".github", "tool", "secret"]; secret private -> plans for
    # Epic (intake)/(planning): org in org/.github and Epic (intake)/(planning): secret in org/secret

def test_drain_respects_segment_cap(monkeypatch): ...
    # head archive at 95 children -> next child goes to segment 2

def test_epic_audit_close_runs_standing_drain(monkeypatch): ...  # test_vrg_epic_audit.py
```

- [ ] **Step 2: Run them — expect FAIL** (`AttributeError: ... 'archive_title'`).

- [ ] **Step 3: Implement.** Thread `kind` (default `ADHOC`) through `_archive_title`, `_adhoc_archive_segments`, `ensure_adhoc_archive`, `_adhoc_archive_head`, and `ensure_writable_adhoc_archive`, keyed by `(home, name)` instead of re-deriving from `target_repo`. Keep the `ensure_adhoc_*` names as thin `ADHOC` wrappers so every existing call site and test is untouched. Build the archive recognizer per kind from `kind.archive_prefix` (same shape as `_ADHOC_ARCHIVE_RE`). Keep the "SIBLING RE-PARENT SITE" comments accurate for both paths.

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Collapse `_ADHOC_ARCHIVE_RE` into the per-kind recognizer, and `_adhoc_archive_segments` / `list_open_adhoc_archives` into kind-parameterized helpers, so the batch drain and the rollup single-child path share one segment rule. Remove any `ADHOC` wrapper nothing calls. Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope epics --message "drain closed intake and planning children into per-quarter archives"`

---

### Task 3: `vrg-intake-create` (rename + capture) [`vergil-tooling`]

**Files:**

- Rename: `src/vergil_tooling/bin/vrg_triage_create.py` → `src/vergil_tooling/bin/vrg_intake_create.py`
- Create: `src/vergil_tooling/bin/vrg_triage_create.py` (deprecated alias)
- Modify: `pyproject.toml:73` (add `vrg-intake-create = "vergil_tooling.bin.vrg_intake_create:main"`; keep `vrg-triage-create` pointing at the alias)
- Modify: `src/vergil_tooling/bin/vrg_gh.py:45` (denial message names `vrg-intake-create`, "intake issue linked under the intake epic")
- Rename/modify: `tests/vergil_tooling/test_vrg_triage_create.py` → `test_vrg_intake_create.py`; add alias tests

**Interfaces:**

- Consumes: Task 1's `ensure_intake_epic`, `add_child`.
- Produces: CLI `vrg-intake-create --title T (--body B | --body-file F) [--repo owner/repo] [--label L ...] [--assignee A ...]`. Prints `Created <url> (intake, under <epic slug>).`

- [ ] **Step 1: Failing tests**

```python
def test_intake_create_files_in_org_dotgithub_and_links_under_intake(...):
    # detect_org -> "org"; queue_home -> ("org/.github","org"); create_issue(repo="org/.github", labels=[])
    # add_child(IssueRef("org",".github",<intake>), IssueRef("org",".github",321)); rc == 0

def test_intake_create_private_repo_files_in_repo(...):
    # cwd repo "org/secret" private (queue_home -> ("org/secret","secret")) -> create_issue(repo="org/secret")
    # and linked under "Epic (intake): secret"

def test_intake_create_applies_no_kind_label(...):
    # labels passed to create_issue contain none of {"triage","idea","research"}

def test_intake_create_rejects_kind_flag():
    with pytest.raises(SystemExit): parse_args(["--title", "T", "--kind", "idea"])

def test_intake_create_link_failure_is_loud(...):
    # add_child raises -> rc == 1, stderr names the created URL and "not linked"

def test_triage_alias_warns_ignores_kind_and_forwards(capsys, ...):
    # vrg_triage_create.main(["--kind","idea","--title","T"]) -> stderr contains
    # "vrg-triage-create is deprecated; use vrg-intake-create" and "--kind is ignored";
    # vrg_intake_create.main called with ["--title","T"]
```

- [ ] **Step 2: Run them — expect FAIL** (`ModuleNotFoundError: vergil_tooling.bin.vrg_intake_create`).

- [ ] **Step 3: Implement.** Move the module (keep the `target_org` scoping and `ensure_labels_or_report` for any extra `--label`). Default repo: when `--repo` is absent, use the current repo if it is private (its own queue), else `<org>/.github`. After `create_issue`, `add_child(ensure_intake_epic(repo), created)`. If linking fails, exit 1 naming the URL (the sweep will heal it, but the operator must know). The alias strips `--kind <v>` / `--kind=<v>`, prints both warnings to stderr, and calls `vrg_intake_create.main(rest)`. Update docstrings to remove "unlinked".

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Make the alias a thin shim (no duplicated argument parsing beyond stripping `--kind`). Drop the now-unused `_KINDS`. Reuse the `target_org` / default-repo resolution between `vrg-intake-create` and `vrg-issue-create` if they now duplicate each other. Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope intake --message "rename vrg-triage-create to vrg-intake-create and link captures under the intake epic" --body "Ref #2484, Ref #2482"`

---

### Task 4: PR refusal for intake and planning items [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/epics.py` (add `queue_item_kind`)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (add `_reject_queue_issue`, call after `_reject_operational_issue` at line 213)
- Modify: `src/vergil_tooling/bin/vrg_submit_pr.py` (add `_reject_if_queue_item` beside `_reject_if_operational_task`, same call sites)
- Test: `tests/vergil_tooling/test_epics.py`, `tests/vergil_tooling/pr_workflow/…` (existing report-ready guard tests), `tests/vergil_tooling/test_vrg_submit_pr.py`

**Interfaces:**

- Consumes: Task 1's `parent_of`, `queue_kind`.
- Produces: `queue_item_kind(ref: str, *, default_repo: str) -> str | None` — `"intake"` / `"planning"` when the issue's parent is a live intake/planning epic; None otherwise, and for an unparseable ref.

- [ ] **Step 1: Failing tests**

```python
def test_queue_item_kind_intake_child(monkeypatch): ...   # parent labels {"epic","intake"} -> "intake"
def test_queue_item_kind_planning_child(monkeypatch): ... # -> "planning"
def test_queue_item_kind_routed_task_is_none(monkeypatch): ...  # parent {"epic","ad-hoc"} -> None
def test_queue_item_kind_unparseable_ref_is_none(): ...

def test_report_ready_refuses_intake_item(...):
    # WorkflowError message contains "is in the intake queue" and "route it with intake-review first"
def test_report_ready_refuses_planning_item(...):
    # message contains "is in the planning queue" and "expand it with epic-create first"
def test_report_ready_defers_when_queue_check_cannot_run(...):
    # CalledProcessError -> no raise (mirrors _reject_operational_issue)
def test_submit_pr_refuses_queue_item(...):  # authoritative: exits non-zero with the same messages
```

- [ ] **Step 2: Run them — expect FAIL.**

- [ ] **Step 3: Implement.** Mirror the operational guards exactly, including the best-effort `(subprocess.CalledProcessError, OSError)` deferral in `report-ready` and the authoritative refusal in `vrg-submit-pr`. Message copy: `--issue (<ref>) is in the intake queue; it is not buildable until it is routed. Route it with intake-review first.` / `... is in the planning queue; it is not buildable until its epic is expanded. Expand it with epic-create first.`

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Put the two refusal messages in one helper in `epics.py` (e.g. `queue_refusal_message(ref, kind) -> str`) that `vrg-pr-workflow`, `vrg-submit-pr`, and the `issue-implement` skill text all quote, so the copy cannot drift. Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope pr-workflow --message "refuse PRs against intake and planning queue items" --body "Ref #2485. Ref vergil-project/.github#175"`

---

### Task 5: Orphan pass, migration, and visibility checks in `vrg-epic-audit` [`vergil-tooling`]

**Files:**

- Modify: `src/vergil_tooling/lib/epic_audit.py` (add `Orphan`, `orphan_drift`, `attach_orphans`, `StandingEpicFull`, `retired_label_drift`, `strip_retired_labels`, `body_only_parents`, `nativize_parents`, `public_self_homed_queues`, `move_public_queues`, `private_under_public_epic`; replace both `_INTAKE_LABELS` uses; delete `_INTAKE_LABELS`; extend `render` / `render_closed`)
- Modify: `src/vergil_tooling/bin/vrg_epic_audit.py` (preview lists everything; `--close` applies, after the existing passes, in the order nativize → strip labels → move public queues → attach orphans; `StandingEpicFull` → stderr message and exit 1)
- Test: `tests/vergil_tooling/test_epic_audit.py`, `tests/vergil_tooling/test_vrg_epic_audit.py`

**Interfaces:**

- Consumes: Task 1's `is_legitimate_top_level`, `is_managed_repo`, `native_parent_of`, `resolve_orphan_epic`, `queue_kind`, `queue_home`, `ensure_intake_epic`, `ensure_planning_epic`; `child_states`, `add_child`, `reparent_child`, `_ADHOC_ARCHIVE_SEGMENT_CAP`.
- Produces:
  - `@dataclass(frozen=True) class Orphan: ref: IssueRef; target: IssueRef`
  - `orphan_drift(org: str) -> list[Orphan]` — open issues from `gh search issues --owner <org> --state open "no:parent-issue"` in managed repos, minus legitimate top-level kinds and minus issues `parent_of` resolves (body-declared parents count as parented)
  - `class StandingEpicFull(RuntimeError)` with `epic: IssueRef`, `open_children: int`, `pending: list[IssueRef]`
  - `attach_orphans(orphans: list[Orphan]) -> int` — attaches in order; before each attach, raises `StandingEpicFull` if the target's open children ≥ `_ADHOC_ARCHIVE_SEGMENT_CAP`
  - `retired_label_drift(org: str) -> list[IssueRef]` and `strip_retired_labels(refs: list[IssueRef]) -> int` — open issues carrying `triage`/`idea`/`research` (parented or not). Stripping only removes labels.
  - `body_only_parents(org: str) -> list[tuple[IssueRef, IssueRef]]` and `nativize_parents(pairs) -> int` — `(task, epic)` where `native_parent_of(task)` is None but `parent_of(task)` is an epic
  - `public_self_homed_queues(org: str) -> list[IssueRef]` and `move_public_queues(epics_: list[IssueRef]) -> int` — intake/planning epics outside `.github` in a repo that is now public. Moving reparents their open children to the `.github` epic of the same kind, then closes the empty epic with the comment `Repo is now public; queue items moved to <slug>.`
  - `private_under_public_epic(org: str) -> list[str]` — slugs of open issues in private repos whose parent epic lives in `<org>/.github`, rendered under the heading `**Private repo work under public .github epics** — an unsupported public → private flip? Resolve by hand (spec: Private repos and visibility):`. Report only.

- [ ] **Step 1: Failing tests**

```python
def test_orphan_drift_targets_intake_epic_of_home(monkeypatch): ...
    # public tool#5 -> Orphan(tool#5, .github intake); private secret#3 -> Orphan(secret#3, secret intake)
def test_orphan_drift_skips_release_tracking_issue(monkeypatch): ...
    # title "release: 2.1.258", body with "<!-- vrg-release:progress -->" -> not an orphan
def test_orphan_drift_skips_archives_and_epics(monkeypatch): ...
def test_orphan_drift_skips_unmanaged_repos(monkeypatch): ...      # is_managed_repo False -> skipped
def test_orphan_drift_treats_body_parent_as_parented(monkeypatch): ...
def test_attach_orphans_is_idempotent(monkeypatch): ...  # second orphan_drift after attach -> []
def test_attach_orphans_stops_at_cap(monkeypatch):
    # target has 95 open children -> raises StandingEpicFull(epic, 95, [all three pending]); add_child never called
def test_strip_retired_labels_keeps_existing_parent(monkeypatch): ...
    # .github#428 under #139 with "idea" -> edit --remove-label idea; no reparent
def test_nativize_parents_adds_native_link(monkeypatch): ...
def test_move_public_queues_reparents_then_closes(monkeypatch): ...
def test_private_under_public_epic_reports_only(monkeypatch): ...
def test_task_drift_skips_pr_ref_to_queue_item(monkeypatch): ...  # replaces the _INTAKE_LABELS skip
def test_stray_check_allows_queue_items_without_kind_labels(monkeypatch): ...
def test_vrg_epic_audit_preview_lists_orphans_without_mutating(monkeypatch): ...
def test_vrg_epic_audit_close_applies_in_order(monkeypatch): ...
    # call order: nativize, strip, move public queues, attach
def test_vrg_epic_audit_close_reports_full_queue_and_exits_1(monkeypatch, capsys): ...
    # stderr: "intake epic org/.github#N is full (95 open); 3 orphan(s) not attached: ... Run intake-review."
```

- [ ] **Step 2: Run them — expect FAIL.**

- [ ] **Step 3: Implement.** Scope every mutation under `github.target_org(org)`, as the ad-hoc drain does. In `task_drift`, replace the `labels & _INTAKE_LABELS` skip with `epics.queue_kind(parent) is not None` for the task's `parent_of`. In `stray_dotgithub_issue`, drop the intake-label exemption: queue items are parented, so they already pass the parent check. Memoize `is_managed_repo` and `is_public` per run.

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Share the org-wide issue enumeration (search + managed-repo filter + memoized visibility) between `orphan_drift`, `retired_label_drift`, `body_only_parents`, and `private_under_public_epic` instead of running four searches. Confirm `_INTAKE_LABELS` is gone (`grep -rn _INTAKE_LABELS src/`). Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope epic-audit --message "attach orphans to intake, retire intake kind labels, and check queue visibility in the sweep" --body "Ref #2483"`

---

### Task 6: `vrg-intake-route` [`vergil-tooling`]

**Files:**

- Create: `src/vergil_tooling/lib/intake_route.py`, `src/vergil_tooling/bin/vrg_intake_route.py`
- Modify: `pyproject.toml` (`vrg-intake-route = "vergil_tooling.bin.vrg_intake_route:main"`)
- Test: `tests/vergil_tooling/test_intake_route.py`, `tests/vergil_tooling/test_vrg_intake_route.py`

**Interfaces:**

- Consumes: Task 1's `queue_kind`, `parent_of`, `ensure_planning_epic`, `resolve_epic_ref`, `queue_home`; `add_child`, `reparent_child`, `github.is_public`.
- Produces: CLI `vrg-intake-route <issue-ref> <route> …` where `<route>` is one of:
  - `drop --reason TEXT` — comment `Dropped in intake review: TEXT` and close
  - `defer --reason TEXT` — comment `Deferred YYYY-MM-DD: TEXT`; no other change
  - `one-off --repo OWNER/REPO [--epic REF]` — move to the repo (if different) and link under `--epic` (default `adhoc`)
  - `planning` — reparent from intake to the planning epic of the same home
  - `split --into REF [--into REF …] --reason TEXT` — comment `Split into: <refs>. TEXT` and close (tasks are created beforehand with `vrg-issue-create`)
  - Library: `route(ref: IssueRef, action: RouteAction) -> RouteResult` with `@dataclass(frozen=True) class RouteResult: final: IssueRef; moved_by: Literal["none", "transfer", "recreate"]`
- Refuses (exit 1) when the issue is not an intake child (`queue_kind(parent_of(ref)) != "intake"`), except `planning`, which also accepts an unparented issue (the `epic-create` free-text path), and when a private-repo item would be routed to a public repo or epic.

- [ ] **Step 1: Failing tests**

```python
def test_route_drop_comments_and_closes(...): ...
def test_route_defer_comments_with_date_only(...): ...     # no close, no reparent
def test_route_planning_reparents_to_planning_epic(...): ...
def test_route_one_off_same_repo_reparents_only(...): ...  # .github item whose PR lands in .github
def test_route_one_off_transfers_then_links(...):
    # gh issue transfer 352 org/tool -> new url .../tool/issues/900; add_child(<tool ad-hoc>, tool#900);
    # RouteResult(final=tool#900, moved_by="transfer")
def test_route_one_off_falls_back_to_recreate_when_transfer_fails(capsys, ...):
    # transfer raises CalledProcessError -> create_issue(repo="org/tool", body contains "Moved from <orig url>"),
    # add_child, comment "Moved to <new url>" + close original; stderr "transfer failed (...); recreated as <url>";
    # moved_by == "recreate"
def test_route_refuses_private_item_to_public_target(...): ...
def test_route_refuses_non_intake_item(...): ...
def test_route_split_closes_with_links(...): ...
```

- [ ] **Step 2: Run them — expect FAIL.**

- [ ] **Step 3: Implement.** Transfer with `github.read_output("issue", "transfer", str(n), dest, "--repo", src)`, then parse the new URL. Always re-link after a move, because a transfer may not keep the sub-issue link. Recreate copies the title and body (plus `Moved from <url>`) and the labels other than retired ones. Run the whole route under `github.target_org(owner)`.

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Keep the CLI a thin argparse layer over `intake_route.route`. Make the transfer and recreate paths share one "re-link and report" tail. Move the comment templates (`Dropped in intake review: …`, `Deferred …`, `Split into: …`, `Moved to/from …`) into module constants. Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope intake --message "add vrg-intake-route for deterministic intake routing"`

---

### Task 7: `vrg-queue` (intake and planning views) [`vergil-tooling`]

**Files:**

- Create: `src/vergil_tooling/lib/queue_view.py`, `src/vergil_tooling/bin/vrg_queue.py`
- Modify: `pyproject.toml` (`vrg-queue = "vergil_tooling.bin.vrg_queue:main"`)
- Test: `tests/vergil_tooling/test_queue_view.py`, `tests/vergil_tooling/test_vrg_queue.py`

**Interfaces:**

- Consumes: Task 1's `queue_home`, `ensure_intake_epic`, `ensure_planning_epic`; `child_states`.
- Produces:
  - CLI `vrg-queue {intake|planning} [--repo OWNER/REPO] [--json]`. The default home is the org's `.github` pair; `--repo` selects that repo's home (a private repo's own queue).
  - `@dataclass(frozen=True) class QueueItem: ref: IssueRef; title: str; created_at: str; state: Literal["waiting", "in-progress", "ready-to-close"]; note: str; epic: IssueRef | None`
  - `intake_items(home: str, name: str) -> list[QueueItem]` — open children, `created_at` ascending, state always `"waiting"`
  - `planning_items(home: str, name: str) -> list[QueueItem]`
  - `parse_planning_markers(comments: list[str]) -> PlanningMarkers` with `@dataclass(frozen=True) class PlanningMarkers: epic: IssueRef | None; docs_task: IssueRef | None; tasks_filed: bool` — the latest of each line wins: `Epic: owner/repo#N`, `Docs task: owner/repo#N`, `Tasks filed: yes`
- Planning state: `in-progress` label absent → `waiting`. Present → `ready-to-close` iff markers give `epic` and `docs_task`, the docs task is CLOSED, and `tasks_filed` is set; otherwise `in-progress`. `note` is the first line of the latest comment.
- Text output: one line per item, `#N  <age>  <state>  <title>  — <note>`. JSON output: a list of `QueueItem` dicts.

- [ ] **Step 1: Failing tests**

```python
def test_intake_items_oldest_first(...): ...
def test_parse_planning_markers_latest_wins():
    m = queue_view.parse_planning_markers(["Epic: org/.github#10", "Docs task: org/.github#11", "Epic: org/.github#12"])
    assert m.epic == IssueRef("org", ".github", 12) and m.docs_task == IssueRef("org", ".github", 11) and not m.tasks_filed
def test_planning_waiting_without_label(...): ...
def test_planning_ready_requires_docs_task_closed(...):
    # in-progress + markers + tasks filed + docs task OPEN -> "in-progress"; CLOSED -> "ready-to-close"
def test_planning_ready_requires_tasks_filed(...): ...
def test_vrg_queue_json_output(...): ...
def test_vrg_queue_repo_flag_selects_private_home(...): ...
```

- [ ] **Step 2: Run them — expect FAIL.**

- [ ] **Step 3: Implement.** Read comments with `gh issue view --json comments,labels,createdAt,title`. Read-only: no command in this task mutates anything.

- [ ] **Step 4: Run them — expect PASS**, then `vrg-container-run -- vrg-validate`.

- [ ] **Step 4b: Refactor.** Define the marker line formats (`Epic:`, `Docs task:`, `Tasks filed:`, `Planning:`) as constants in `queue_view.py`, which the `epic-create` skill (Task 8) quotes verbatim. Fetch each item's comments once, not once per marker. Re-run `vrg-container-run -- vrg-validate`.

- [ ] **Step 5: Commit** — `vrg-commit --type feat --scope intake --message "add vrg-queue to view the intake and planning queues"`

---

### Task 8: Skills [`vergil-claude-plugin`]

**Files:**

- Rename + rewrite: `skills/triage-capture/` → `skills/intake-capture/SKILL.md`
- Rename + rewrite: `skills/triage-review/` → `skills/intake-review/SKILL.md`
- Modify: `skills/epic-create/SKILL.md`, `skills/issue-implement/SKILL.md`, `skills/migrate-repo/SKILL.md`, `skills/memory-audit/SKILL.md`, `skills/issue-validate/SKILL.md`, `skills/issue-validate/references/operational-task-lifecycle.md`, `skills/issue-deploy/SKILL.md`, `docs/development/skills-architecture.md`, `docs/site/docs/skills/index.md`

**Interfaces:**

- Consumes: CLIs from Tasks 3, 6, 7 (`vrg-intake-create`, `vrg-intake-route`, `vrg-queue`) and `vrg-issue-create --epic planning` (Task 1 sentinel).

- [ ] **Step 1: `intake-capture`.** Same triggers as `triage-capture`. Steps: paraphrase voice input → write body to a temp file → `vrg-intake-create --title … --body-file …` → report the URL and stop. No kind step. Note that a private repo's capture lands in its own queue automatically.

- [ ] **Step 2: `intake-review`.** Triggers: "review intake", "groom the backlog", "process the intake queue", "what's in intake", plus the old triage phrasings. Workflow:
  1. `vrg-queue intake`. Report the count and the one-line list, oldest first. For a private repo, add `--repo`.
  2. Batches of 4. For each item: **check current state first** (search for an overlapping or closed issue, read the code when it is a bug), then write one paragraph covering what it is, why it matters, where it stands, and a recommended route.
  3. The human confirms or redirects the batch in one reply.
  4. Apply with `vrg-intake-route` (drop / defer / one-off / split / planning). Split: create each task with `vrg-issue-create` first, then `vrg-intake-route … split --into …`.
  5. Stop when the queue is empty or the human stops. Report the tallies.

  The skill must state that **the agent closes dropped and split items itself** (via `vrg-intake-route`) after the human confirms. The old "closing is a human action" text is removed.

- [ ] **Step 3: `epic-create`.** Add a *Planning issue* section ahead of Workflow, covering:
  - the four entry points from the spec table, using `vrg-queue planning` for the no-argument summary;
  - `vrg-intake-route <n> planning` to pull an intake item into planning;
  - `vrg-issue-create --epic planning --repo <home> …` for free text.

  Make it the first workflow step: add the `in-progress` label (`vrg-gh issue edit --add-label in-progress`). Add progress comments at each stage boundary (brainstorm approved, epic created, spec committed, pushback done, plan written, alignment done, docs PR handed off, tasks filed), plus whenever the session pauses ("blocked on …" / "resume at …"). Each comment includes the marker lines `Epic: …`, `Docs task: …`, `Tasks filed: yes` once known. When creating the epic, add `Planning: <ref>` to the epic body. Final step: if `vrg-queue planning` shows the item `ready-to-close`, comment `Planned as <epic>` and close it. Otherwise record "waiting on docs PR merge". A planning item abandoned during brainstorming is closed with the reason.

- [ ] **Step 4: Other producers and consumers.** `migrate-repo`: the "triage" bucket becomes "intake". Link uncertain items under the intake epic of their home (`vrg-epic-link --epic <intake epic> --task …`) instead of labelling them `triage`, and update the summary line. `issue-implement`: before starting, refuse an issue whose parent is an intake or planning epic, using the Task 4 message (the upstream half of the PR refusal). `memory-audit`: `vrg-intake-create` with no `--kind`. `issue-validate`, `issue-deploy`, the lifecycle reference, and the two docs pages: `triage-capture` → `intake-capture`, `triage-review` → `intake-review`.

- [ ] **Step 5: Verify** with `vrg-container-run -- vrg-validate` (markdownlint). Then check that `grep -rnE 'triage-(capture|review)|vrg-triage-create|--kind (triage|idea|research)|label `?triage' skills docs` returns nothing outside `CHANGELOG.md`.

- [ ] **Step 6: Commit** — `vrg-commit --type feat --scope skills --message "replace triage skills with intake-capture and intake-review; track epic-create through planning issues" --body "Ref vergil-project/vergil-tooling#2482"`

---

### Task 9: Deployment — release the tooling and plugin [`vergil-tooling`, `--kind deployment`]

Created with `vrg-issue-create --epic vergil-project/.github#176 --repo vergil-project/vergil-tooling --kind deployment --title "Deploy: release vergil-tooling and vergil-claude-plugin with the intake/planning model" --blocked-by …#<Tasks 1–7> --blocked-by vergil-project/vergil-claude-plugin#<Task 8>`.

- **Preconditions (human-attested):** vergil-tooling and vergil-claude-plugin releases containing Tasks 1–8 are published. The agent never cuts a release.
- **Procedure:** confirm `vrg-intake-create --help`, `vrg-intake-route --help`, and `vrg-queue --help` resolve on the Mac host and in an agent VM (`vrg-vm update`). Confirm the plugin lists `intake-capture` and `intake-review` and not `triage-*`. Confirm the `ops-epic-sweep` and `epic-rollup` workflows install the new tooling version (they pin the `v2.1` line).
- **Acceptance:** all of the above hold. Record `Outcome: SUCCESS`.

### Task 10: Validation — run the one-shot migration per org [`.github`, `--kind validation`]

Rewrites `#185`, blocked by Task 9.

- **Preconditions:** Task 9 closed with SUCCESS (probe: `vrg-intake-create --help` exits 0 in the sweep runner).
- **Procedure, per org (`vergil-project`, then `logical-minds-foundry`):**
  1. Preview: `vrg-epic-audit` (no `--close`) from a repo in the org. Record the orphan, retired-label, body-parent, public-queue, and private-under-public counts.
  2. A human triggers `ops-epic-sweep` via `workflow_dispatch`.
  3. Re-run the preview.
- **Acceptance:**
  - The second preview shows 0 orphans, 0 retired-label issues, 0 body-only parents, and 0 public self-homed queues.
  - `vrg-queue intake` lists the migrated items oldest first.
  - Every open `release: X.Y.Z` issue and every open archive is still unparented.
  - A second `workflow_dispatch` changes nothing (idempotence).
- A `StandingEpicFull` failure is a FAIL: record it, run `intake-review` to drain the queue, and rerun.

---

## Bookends (not implementation tasks)

- `#430` (this revision's docs PR) closes on merge.
- After merge: close `#429` with "absorbed into #176 (spec revision 2)". Retarget `#178` to "Follow-on brainstorm: review the wider label set" and keep it as the follow-on bookend. Seed the retrospective task `Retrospective: every-issue-in-epic` in `.github` if it is not already present.
- `vergil-tooling#2474` (docs review) sweeps `docs/site/docs/standards/github-issues.md`, `docs/site/docs/guides/agent-vm-claude-share-set.md`, the `docs` repo's `management/tools-and-skills.md`, and the plugin site, spawning per-repo doc tasks as needed. It must document the one-way visibility rule (public → private means a new repo and a code migration).
