# Collapse PR-linkage enforcement to construction time — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Guarantee the "one task → one PR → one auto-close" linkage invariant at PR-body construction and retire the redundant `vrg-pr-issue-linkage` CI gate.

**Architecture:** A single shared guard (`find_linkage_keyword`) rejects issue-linkage keywords in the free-text `--notes`/`--summary` fields at two construction points — `report-ready` (agent-facing, immediate feedback) and `build_pr_body` (the chokepoint every PR body flows through). With free text keyword-free, the builder's `## Issue Linkage` line is the body's *only* keyworded linkage, so "exactly one, targeting the primary task" holds by construction. The CI gate is then dead code and is removed across `vergil-actions` (the invocation) and `vergil-tooling` (the script).

**Tech Stack:** Python 3.12 (CI) / 3.14 (dev container), `re`, pytest; GitHub Actions reusable workflows (YAML); `vrg-*` tooling.

**Spec:** `epics/82-linkage-construction-time/spec.md` (this epic). **Epic:** vergil-project/.github#82.

## Global Constraints

- **Git/GitHub:** use `vrg-git` / `vrg-gh` / `vrg-commit` only — raw `git`/`gh` are denied. All work flows through a feature-branch worktree; the main worktree is read-only.
- **Validation (the only command):** `vrg-container-run -- vrg-validate` (in `vergil-tooling` this expands to `uv run vrg-validate` via the `[validation]` override). Run it before each task's final commit.
- **Single-test runs during red/green:** `vrg-container-run -- uv run pytest <path>::<name> -v`.
- **Python:** every module starts with `from __future__ import annotations`.
- **No silent failures:** the guard *raises* with an actionable message; it never strips or swallows.
- **Commits:** conventional-commit subjects; body ends with `Ref #<task-issue>` (managed tasks auto-link `Closes` at submit).
- **Task issues:** A = vergil-tooling#2117, B = vergil-actions#746, C = vergil-tooling#2116.

## Sequencing & release (read before starting)

The tasks are **order-dependent**; ship each and let it propagate before the next:

1. **Task A → release a `vergil-tooling` 2.1.x.** The guard goes live; PRs get immediate feedback. The old CI gate still runs (now redundant, harmless).
2. **Task B → release/tag `vergil-actions`.** The standards job is removed org-wide; `vrg-pr-issue-linkage` is no longer invoked anywhere. **Non-breaking:** the `run-standards` input stays *accepted-but-ignored*, so no consumer's reusable-workflow call errors.
3. **Task C → release a `vergil-tooling` 2.1.x.** Delete the now-dead script/entry/test and the local `ci.yml` passthrough. Safe because no CI calls the script after B, and the vestigial input means removing our passthrough cannot error.

**Why this order:** A before C so there is never a window with *no* enforcement; B before C so the script is never deleted while a workflow still calls it (`command not found`).

**Assumption to verify (maintainer):** `ci.yml` pins `vergil-actions@v2.1` (a moving major.minor tag), and consumers pass `run-standards`. If that holds, keeping the input vestigial in Task B is mandatory to avoid breaking every consumer. If consumers instead pin exact patch tags and none pass the input, the input *could* be removed outright — but the vestigial path is safe either way, so the plan takes it unconditionally.

**Follow-up (out of scope, track separately):** once every consumer has dropped its `run-standards` passthrough, remove the vestigial input from `ci-security.yml`.

---

## Task A: Construction-time linkage guard (`vergil-tooling#2117`)

**Files:**
- Modify: `src/vergil_tooling/lib/linkage.py` (add `find_linkage_keyword` + `freetext_linkage_error`)
- Modify: `src/vergil_tooling/bin/vrg_pr_workflow.py` (guard in `cmd_report_ready`)
- Modify: `src/vergil_tooling/lib/pr_body.py` (guard in `build_pr_body`)
- Test: `tests/vergil_tooling/test_linkage.py`
- Test: `tests/vergil_tooling/test_pr_body.py`
- Test: `tests/vergil_tooling/pr_workflow/test_cli_e2e.py`

**Interfaces:**
- Produces:
  - `find_linkage_keyword(text: str) -> str | None` — first `Ref`/`Close[sd]`/`Fix(es|ed)`/`Resolve[sd]` + `#N`/`owner/repo#N` substring in *text*, mid-line; `None` for bare `#N` or keyword-free prose.
  - `freetext_linkage_error(found: str, primary_issue: str) -> str` — user-ready rejection message with the lossless-redirect guidance.
- Consumes: `commit_message.AUTOCLOSE_RE` pattern shape (as a model; not imported).

### A1 — the guard helper

- [ ] **Step 1: Write the failing tests** in `tests/vergil_tooling/test_linkage.py` (append; add `find_linkage_keyword` to the existing import block):

```python
import pytest
from vergil_tooling.lib.linkage import find_linkage_keyword


@pytest.mark.parametrize(
    "text,expected",
    [
        ("Ref #157", "Ref #157"),
        ("Ref: #157", "Ref: #157"),
        ("Closes #42", "Closes #42"),
        ("- Fixes #9", "Fixes #9"),
        ("this also Resolves #9 in passing", "Resolves #9"),
        ("Ref owner/repo#3", "Ref owner/repo#3"),
        ("Closes vergil-project/.github#82", "Closes vergil-project/.github#82"),
    ],
)
def test_find_linkage_keyword_matches(text: str, expected: str) -> None:
    assert find_linkage_keyword(text) == expected


@pytest.mark.parametrize(
    "text",
    [
        "See #200 for background",
        "Part of epic vergil-project/.github#82",
        "referenced the earlier work",
        "no linkage here",
        "",
    ],
)
def test_find_linkage_keyword_ignores_bare_and_plain(text: str) -> None:
    assert find_linkage_keyword(text) is None
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_linkage.py -k find_linkage_keyword -v`
Expected: FAIL — `ImportError: cannot import name 'find_linkage_keyword'`.

- [ ] **Step 3: Implement the helper** in `src/vergil_tooling/lib/linkage.py` (add after `LINKAGE_VALUE_RE`):

```python
# Any issue-linkage keyword (Ref, or the close/fix/resolve family) followed by an
# issue reference, matched mid-line (not anchored) so a smuggled "... this also
# Closes #999" is caught. A bare "#200" does NOT match, so a lightweight
# cross-reference in free text is still allowed. This is the guard for the
# free-text PR fields (--notes/--summary): broader than LINKAGE_RE (anchored,
# Ref/Close only) and commit_message.AUTOCLOSE_RE (close family only).
_FREETEXT_LINKAGE_RE = re.compile(
    r"\b(?:ref|close[sd]?|fix(?:e[sd])?|resolve[sd]?):?\s+"
    r"(?:[a-zA-Z0-9._-]+/[a-zA-Z0-9._-]+)?#[0-9]+",
    re.IGNORECASE,
)


def find_linkage_keyword(text: str) -> str | None:
    """Return the first issue-linkage keyword+ref in *text*, or None.

    Matches Ref / Close[sd] / Fix(es|ed) / Resolve[sd] followed by an issue
    reference (#N or owner/repo#N), anywhere in the line. A bare "#200" does not
    match. The returned substring (e.g. "Ref #157") names the offending text in
    the guard's error message.
    """
    match = _FREETEXT_LINKAGE_RE.search(text)
    return match.group(0) if match else None
```

- [ ] **Step 4: Run to verify pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_linkage.py -k find_linkage_keyword -v`
Expected: PASS (all parametrized cases).

- [ ] **Step 5: Write the failing test for the message helper** (append to `test_linkage.py`):

```python
from vergil_tooling.lib.linkage import freetext_linkage_error


def test_freetext_linkage_error_names_offender_and_redirects() -> None:
    msg = freetext_linkage_error("Ref #157", "83")
    assert "Ref #157" in msg
    assert "vrg-gh issue comment 83" in msg
    assert "added for you" in msg
```

- [ ] **Step 6: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_linkage.py -k freetext_linkage_error -v`
Expected: FAIL — `ImportError: cannot import name 'freetext_linkage_error'`.

- [ ] **Step 7: Implement the message helper** in `linkage.py` (after `find_linkage_keyword`):

```python
def freetext_linkage_error(found: str, primary_issue: str) -> str:
    """User-ready error for a linkage keyword smuggled into --notes/--summary.

    Rejects rather than strips: a keyword the agent typed is a signal that a real
    relationship exists, so the message redirects that reasoning to a lossless
    home — a comment on the primary issue — instead of discarding it.
    """
    return (
        f"notes/summary must not contain an issue-linkage keyword (found {found!r}). "
        "A PR links exactly one task, and that link is added for you automatically. "
        "If this change genuinely relates to another issue, record it — and why — "
        "as a comment on the primary issue: "
        f'vrg-gh issue comment {primary_issue} --body "Related to #N — <the reason>". '
        "Don't encode it as a bare linkage in notes, where the reasoning is lost."
    )
```

- [ ] **Step 8: Run to verify pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_linkage.py -k freetext_linkage_error -v`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
vrg-git add src/vergil_tooling/lib/linkage.py tests/vergil_tooling/test_linkage.py
vrg-commit --type feat --scope linkage \
  --message "add find_linkage_keyword free-text guard helper" \
  --body "Shared guard for the free-text PR fields: matches any Ref/Close/Fix/Resolve + issue ref mid-line, ignores bare #N. freetext_linkage_error redirects the reasoning to a comment on the primary issue rather than stripping it. Ref #2117"
```

### A2 — guard `report-ready`

- [ ] **Step 1: Write the failing test** in `tests/vergil_tooling/pr_workflow/test_cli_e2e.py` (mirror `test_report_ready_allows_non_epic_task`):

```python
def test_report_ready_rejects_linkage_keyword_in_notes(
    in_git_repo: Path, capsys: pytest.CaptureFixture[str]
) -> None:
    with (
        patch("vergil_tooling.bin.vrg_pr_workflow.github.current_repo", return_value="org/repo"),
        patch("vergil_tooling.bin.vrg_pr_workflow.epics.is_epic_linkage", return_value=False),
    ):
        rc = vrg_pr_workflow.main(
            ["--base", "develop", "report-ready",
             "--issue", "42", "--title", "t", "--summary", "s", "--notes", "Ref #157"]
        )
    assert rc == 1
    assert "Ref #157" in capsys.readouterr().err
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/pr_workflow/test_cli_e2e.py -k rejects_linkage_keyword -v`
Expected: FAIL — `rc == 0` (guard not wired yet).

- [ ] **Step 3: Wire the guard** in `src/vergil_tooling/bin/vrg_pr_workflow.py`. Add the import and the check in `cmd_report_ready`, immediately after the `normalize_linkage` block (before `state = transport.read()`):

```python
from vergil_tooling.lib.linkage import (
    find_linkage_keyword,
    freetext_linkage_error,
    normalize_linkage,
)
```

```python
    for value in (args.notes, args.summary):
        found = find_linkage_keyword(value)
        if found:
            raise WorkflowError(freetext_linkage_error(found, str(args.issue)))
```

- [ ] **Step 4: Run to verify pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/pr_workflow/test_cli_e2e.py -k rejects_linkage_keyword -v`
Expected: PASS.

- [ ] **Step 5: Regression — the whole report-ready suite still passes**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/pr_workflow/test_cli_e2e.py -v`
Expected: PASS (existing `--notes n` cases carry no keyword, so they are unaffected).

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/bin/vrg_pr_workflow.py tests/vergil_tooling/pr_workflow/test_cli_e2e.py
vrg-commit --type feat --scope pr-workflow \
  --message "reject linkage keywords in report-ready notes/summary" \
  --body "Agent-facing construction point: report-ready now raises with an actionable, lossless-redirect message when --notes/--summary carry a linkage keyword, instead of letting it duplicate the builder's linkage line and fail in CI later. Ref #2117"
```

### A3 — guard `build_pr_body` (the chokepoint)

- [ ] **Step 1: Write the failing tests** in `tests/vergil_tooling/test_pr_body.py` (append):

```python
def test_build_pr_body_rejects_linkage_keyword_in_notes() -> None:
    with pytest.raises(SystemExit) as exc:
        pr_body.build_pr_body(summary="s", linkage="Closes", issue_ref="#42", notes="Ref #99")
    assert "Ref #99" in str(exc.value)


def test_build_pr_body_rejects_linkage_keyword_in_summary() -> None:
    with pytest.raises(SystemExit) as exc:
        pr_body.build_pr_body(summary="Closes #7", linkage="Closes", issue_ref="#42", notes="")
    assert "Closes #7" in str(exc.value)


def test_build_pr_body_allows_bare_reference_in_notes() -> None:
    body = pr_body.build_pr_body(
        summary="s", linkage="Closes", issue_ref="#42", notes="See #99 for background"
    )
    assert "See #99 for background" in body
```

- [ ] **Step 2: Run to verify failure**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_pr_body.py -k "rejects_linkage or allows_bare" -v`
Expected: FAIL — the two `rejects_` tests do not raise (guard absent).

- [ ] **Step 3: Wire the guard** in `src/vergil_tooling/lib/pr_body.py`. Add the import and check at the top of `build_pr_body`:

```python
from vergil_tooling.lib.linkage import find_linkage_keyword, freetext_linkage_error
```

```python
def build_pr_body(*, summary: str, linkage: str, issue_ref: str, notes: str) -> str:
    """Render the canonical PR body from validated fields."""
    for value in (summary, notes):
        found = find_linkage_keyword(value)
        if found:
            raise SystemExit(freetext_linkage_error(found, issue_ref.lstrip("#")))
    notes_section = notes or "-"
    ...
```

- [ ] **Step 4: Run to verify pass**

Run: `vrg-container-run -- uv run pytest tests/vergil_tooling/test_pr_body.py -v`
Expected: PASS (new tests pass; existing `build_pr_body` tests use keyword-free notes, unaffected).

- [ ] **Step 5: Full validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (lint, typecheck, full test suite, audit).

- [ ] **Step 6: Commit**

```bash
vrg-git add src/vergil_tooling/lib/pr_body.py tests/vergil_tooling/test_pr_body.py
vrg-commit --type feat --scope pr-body \
  --message "reject linkage keywords in build_pr_body notes/summary" \
  --body "Chokepoint guard: every PR body flows through build_pr_body (submit CLI mode, submit-from-state, pr-fix-body), so guarding here closes the direct-CLI path that never touches report-ready. With free text keyword-free, the builder's Issue Linkage line is the body's only keyworded linkage. Ref #2117"
```

---

## Task B: Remove the redundant CI standards gate (`vergil-actions#746`)

**Prerequisite:** Task A released. **Files** (in `vergil-actions`):
- Delete: `actions/ci/security/standards-compliance/action.yml` (and the now-empty `standards-compliance/` dir)
- Modify: `.github/workflows/ci-security.yml` (remove the `standards` job; keep the `run-standards` input, marked vestigial)
- Modify: `.github/workflows/README.md` (drop the standards-compliance mention if present)

- [ ] **Step 1: Remove the standards job** from `.github/workflows/ci-security.yml` — delete the whole `standards:` job block (the `Checkout` / `Install vergil-tooling` / `Validate standards` steps). Leave the `codeql` job and workflow-level `permissions` intact.

- [ ] **Step 2: Neutralize (do NOT delete) the `run-standards` input.** In the `on.workflow_call.inputs` block, keep `run-standards` and add a comment so it is not mistaken for live wiring:

```yaml
      run-standards:
        description: >-
          Deprecated / vestigial. The standards-compliance gate
          (vrg-pr-issue-linkage) was retired — linkage is now enforced at
          PR-body construction (epic vergil-project/.github#82). Kept as an
          accepted-but-ignored input so consumers passing it do not error;
          remove once no consumer passes it.
        required: false
        type: boolean
        default: false
```

(Preserve the input's existing `type`/`required`; only the `description`/`default` and the removal of any use change.)

- [ ] **Step 3: Delete the action** directory `actions/ci/security/standards-compliance/` (its only step was `vrg-pr-issue-linkage`).

- [ ] **Step 4: Drop stale references.** Update `.github/workflows/README.md` (and any docs) that describe the standards-compliance action/job to reflect that linkage is enforced at construction (epic #82).

- [ ] **Step 5: Validate**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — `actionlint` is clean (an unused `workflow_call` input is valid; no job references the removed action).

- [ ] **Step 6: Commit**

```bash
vrg-git add .github/workflows/ci-security.yml .github/workflows/README.md
vrg-git add -A actions/ci/security/standards-compliance
vrg-commit --type chore --scope ci \
  --message "retire standards-compliance linkage gate" \
  --body "Linkage is now guaranteed at PR-body construction (epic vergil-project/.github#82), so the standards job and its vrg-pr-issue-linkage action are removed. The run-standards input is kept accepted-but-ignored so no consumer's reusable-workflow call breaks; removing the input is a separate coordinated follow-up. Ref #746"
```

---

## Task C: Delete the dead script and local CI passthrough (`vergil-tooling#2116`)

**Prerequisite:** Task B released and propagated (no CI invokes `vrg-pr-issue-linkage`). **Files:**
- Delete: `src/vergil_tooling/bin/vrg_pr_issue_linkage.py`
- Modify: `pyproject.toml` (remove the `vrg-pr-issue-linkage` entry point, line ~44)
- Delete: `tests/vergil_tooling/test_vrg_pr_issue_linkage.py`
- Modify: `.github/workflows/ci.yml` (remove the `run-standards:` passthrough, line ~48)

- [ ] **Step 1: Delete the script and its test.**

```bash
vrg-git rm src/vergil_tooling/bin/vrg_pr_issue_linkage.py tests/vergil_tooling/test_vrg_pr_issue_linkage.py
```

- [ ] **Step 2: Remove the entry point** from `pyproject.toml` — delete the line:

```toml
vrg-pr-issue-linkage = "vergil_tooling.bin.vrg_pr_issue_linkage:main"
```

- [ ] **Step 3: Remove the `ci.yml` passthrough.** In `.github/workflows/ci.yml`, delete the line inside the `security` job's `with:` block:

```yaml
      run-standards: ${{ inputs.run-release != 'false' }}
```

(Safe: Task B keeps the input accepted, so the reusable call does not error whether or not it is passed. Leave `run-security`, `container-tag`, `container-suffix`.)

- [ ] **Step 4: Confirm nothing else references the script.**

Run: `vrg-container-run -- uv run bash -c "grep -rn 'vrg-pr-issue-linkage\|vrg_pr_issue_linkage' src tests pyproject.toml .github || echo CLEAN"`
Expected: `CLEAN` (only the now-removed files matched before).

- [ ] **Step 5: Full validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS — the entry-point removal is consistent, the remaining suite (including the untouched `extract_tracking_issue` / `vrg-resolve-tracking-issue` / `epic_audit` tests) is green.

- [ ] **Step 6: Commit**

```bash
vrg-git add -A src/vergil_tooling/bin tests/vergil_tooling pyproject.toml .github/workflows/ci.yml
vrg-commit --type chore --scope linkage \
  --message "delete dead vrg-pr-issue-linkage script and CI passthrough" \
  --body "After the standards gate was retired (epic vergil-project/.github#82, vergil-actions#746), the script, its entry point, its test, and the ci.yml run-standards passthrough are dead code. lib/linkage.py stays (shared regexes, normalize_linkage, extractors, find_linkage_keyword); the extractors' ValueError-on-multiple remains as a now-unreachable defensive assertion. Ref #2116"
```

---

## Self-review

- **Spec coverage:** guard helper (A1) ✓; both construction points — report-ready (A2) + build_pr_body (A3) ✓; lossless-redirect message (A1/A2/A3) ✓; extractors/`vrg-resolve-tracking-issue`/`epic_audit` untouched (verified in C5 regression) ✓; delete script + entry + test (C) ✓; remove CI gate in vergil-actions (B) ✓; remove ci.yml passthrough (C3) ✓; staged sequencing ✓.
- **Refinement vs. spec/Task B issue:** the spec/issue say "remove the `run-standards` input"; the plan keeps it *vestigial* to avoid breaking every consumer's reusable-workflow call, with outright removal deferred to a tracked follow-up. This is a safety refinement, not a scope change.
- **Type/name consistency:** `find_linkage_keyword(text) -> str | None` and `freetext_linkage_error(found, primary_issue) -> str` are used with identical signatures in `linkage.py`, `vrg_pr_workflow.py`, and `pr_body.py`.
- **Placeholder scan:** none — every step carries concrete code/commands. (`<the reason>` / `#N` inside the *error message string* are intentional user-facing template tokens, not plan placeholders.)
