# Collapse PR-linkage enforcement to construction time

- **Status:** Design — approved, pending implementation plan
- **Date:** 2026-07-02
- **Epic:** [vergil-project/.github#82](https://github.com/vergil-project/.github/issues/82)
- **Supersedes:** [vergil-project/vergil-tooling#2107](https://github.com/vergil-project/vergil-tooling/issues/2107)
  (the original single-task framing; promoted to this epic because the work
  spans `vergil-tooling` and `vergil-actions` across multiple PRs)

## Problem

The "a PR links exactly one task" invariant is enforced **only in CI**, by
`vrg-pr-issue-linkage` inside the `security / standards` gate, and **not** at
the point where the PR body is constructed. Three tools handle the same concept
three different ways:

| Tool | Linkage handling |
| --- | --- |
| `vrg-commit` | **Rejects** auto-close keywords (`close/fix/resolve #N`) in the commit body; allows `Ref #N`. |
| PR-body builders (`vrg-submit-pr` / `report-ready` / `vrg-pr-fix-body`) | **Silently accept** linkage keywords in `--notes`/`--summary` and append them verbatim. |
| CI `vrg-pr-issue-linkage` | **Hard-fails** when the rendered body has more than one linkage line. |

The failure surfaces late (≈10 min into CI) and lands where only a human can
fix it: the GitHub-App identity cannot `updatePullRequest`, so remediation
requires a manual browser edit (the `mq-rest-admin-ruby` PR #162 incident in
vergil-tooling#2107).

Meanwhile the surrounding policy has collapsed to its simplest possible form:
**one task → one PR → one auto-close.** A task is exactly one PR; once it is in
`develop` it is done, and any later change is a new follow-up issue, never a
reopening. Given that, the entire standards gate now does nothing but run
`vrg-pr-issue-linkage` — and the invariant it polices can be guaranteed at
construction instead of audited after the fact.

## Goal

Delete the CI standards gate entirely. Make "one task → one PR → one auto-close"
a **structural** property of PR-body construction, and give the agent an
**immediate, in-session** error when it authors a bad linkage — never a
CI round-trip that lands in a human-only remediation spot.

## The invariant, restated

A PR body carries **exactly one keyworded issue linkage**: the builder-authored
`## Issue Linkage` line, targeting the primary task — `Closes #N` for a managed
task (an issue with an `epic`-labeled parent, auto-closes on merge) or `Ref #N`
for a legacy task. That single line *is* the PR↔code↔task correlation and the
sole auto-close.

Free text (`--notes` / `--summary`) carries **no** linkage keywords. Related
issues, when the relationship matters, are recorded as a **comment on the
primary issue** (see [Error message](#error-message-lossless-redirect)); a bare
`#200` mention still slips past the guard and auto-links on GitHub for a
lightweight cross-reference, but anything with a *reason* belongs in a comment.

### Why this makes the invariant structural

`build_pr_body` is the **sole author** of the `## Issue Linkage` line, and it
always targets the resolved primary `issue_ref`. If free text can never contain
a linkage keyword, then the body has exactly one keyworded linkage — the
builder's — *by construction*. "This PR closes exactly the primary task and
nothing foreign" is then true structurally, with no rendered-body invariant left
to police.

Every check the old CI gate performed becomes structural or construction-guarded:

| Old CI check | Where it goes |
| --- | --- |
| body non-empty | `build_pr_body` always renders content → structural |
| has a linkage | `build_pr_body` always emits the `## Issue Linkage` line → structural |
| exactly one linkage | free text banned from keywords → only the builder's line → guaranteed |
| no banned auto-close | free-text keyword ban + `normalize_linkage` restricts the linkage field to `Ref`/`Closes` → guaranteed |

Nothing is lost by deleting the gate.

## Design

### Guard placement — both construction points

The guard runs at **two** call sites, sharing one helper:

1. **`report-ready`** (`vrg_pr_workflow.py`, `cmd_report_ready`) — the point
   where the **agent** authors `--notes`/`--summary`. Validating here gives the
   agent immediate, in-session feedback. This is the "fail at the earliest
   construction point" win that motivates moving the check off CI.
2. **`build_pr_body`** (`pr_body.py`) — the **true chokepoint**: every
   body-producing path funnels through it, including `vrg-submit-pr` CLI direct
   mode and `vrg-pr-fix-body`, which never call `report-ready`. Without a guard
   here, those paths would be unprotected once the CI gate is gone.

Same rule, two moments: `report-ready` for fast agent feedback, `build_pr_body`
for the closed guarantee.

### New shared guard

Add to `lib/linkage.py`:

```python
def find_linkage_keyword(text: str) -> str | None:
    """Return the first issue-linkage keyword+ref in *text*, or None.

    Matches Ref / Close[sd] / Fix(es|ed) / Resolve[sd] followed by an issue
    reference (#N or owner/repo#N), anywhere in the line (not anchored), so a
    mid-sentence "... this also Closes #999" is caught. A bare "#200" does NOT
    match. The returned substring (e.g. "Ref #157") names the offending text
    in the error message.
    """
```

This is a broader sibling of the existing patterns: `commit_message.AUTOCLOSE_RE`
catches the close-family mid-line; this one additionally catches `Ref`. Bare
`#N` mentions do not match, so a lightweight cross-reference is still possible.

### Error message (lossless redirect)

An agent putting `Ref #157` in notes is a *signal*: it believed a real
relationship exists. Rejecting silently would discard that signal, so the
exception redirects to a durable, lossless home for it:

> **notes must not contain an issue-linkage keyword (found `Ref #157`).**
> A PR links exactly one task, and that link is added for you automatically.
> If this change genuinely relates to **#157**, that relationship — and *why* —
> is worth keeping: record it as a comment on the primary issue, e.g.
> `vrg-gh issue comment <primary-issue> --body "Related to #157 — <the reason>"`
> Don't encode it as a bare linkage in notes, where the reasoning is lost.

`report-ready` raises `WorkflowError`; `build_pr_body` raises the same message
(via `SystemExit` / `ValueError`, matching its existing `resolve_issue_ref`
style). No silent stripping — consistent with `vrg-commit`, which already
**rejects** the analogous case in commit bodies, and with the project's
"no silent failures" principle.

### Unchanged by design

`extract_tracking_issue` / `extract_tracking_ref`, `vrg-resolve-tracking-issue`,
and `epic_audit` stay **byte-for-byte unchanged**. Their "exactly one match"
expectation — which today only holds because the CI gate props it up — is now a
construction guarantee. Their `ValueError`-on-multiple is kept as a
now-unreachable **defensive assertion**: cheap insurance if a hand-edited or
legacy body ever carries two keyworded linkages.

The epic-vs-task keyword decision (`_task_linkage` / `_resolve_linkage`) and the
epic-link rejection (`_reject_if_epic_link`) in `vrg-submit-pr` are unaffected.

## Tasks

Implementation tasks are filed in their member repos at implementation time and
linked to this epic with
`vrg-epic-link --epic vergil-project/.github#82 --task vergil-project/<repo>#<TASK>`.
The staging is additive-then-subtractive so no consuming repo's CI ever calls a
missing command:

- **Task A — `vergil-tooling` (add the guard).** Add `find_linkage_keyword` to
  `lib/linkage.py`; call it in `report-ready` (agent feedback) and
  `build_pr_body` (chokepoint), rejecting linkage keywords in
  `--notes`/`--summary` with the lossless-redirect message. Purely additive;
  safe to ship first.
- **Task B — `vergil-actions` (remove the gate).** Remove the now-redundant
  `standards-compliance` action, the `standards` job in `ci-security.yml`, the
  `run-standards` input plumbing, and the `run-standards` passthrough at
  `ci.yml:48` in `vergil-tooling`. Release/tag so consumers stop calling the
  script.
- **Task C — `vergil-tooling` (delete the script).** Delete
  `bin/vrg_pr_issue_linkage.py`, its `vrg-pr-issue-linkage` entry point in
  `pyproject.toml`, and `tests/.../test_vrg_pr_issue_linkage.py` — **only after
  B has propagated** to consumers.

`lib/linkage.py` stays (it hosts the shared regexes, `normalize_linkage`, the
extractors, and now `find_linkage_keyword`).

## Testing

- **`find_linkage_keyword`** — unit tests: matches `Ref`/`Closes`/`Fixes`/
  `Resolves` + `#N` and `owner/repo#N`, mid-line included; does **not** match a
  bare `#200` or keyword-free prose.
- **`report-ready`** — rejects notes/summary carrying a linkage keyword with the
  actionable message; accepts clean text and bare `#N` mentions.
- **`build_pr_body`** — rejects bad notes/summary, covering the `vrg-submit-pr`
  CLI direct path and `vrg-pr-fix-body`.
- **Delete** `test_vrg_pr_issue_linkage.py`.
- **Regression:** existing extractor / `vrg-resolve-tracking-issue` /
  `epic_audit` tests stay green unchanged.

## What we deliberately give up

CI no longer inspects the rendered PR body. Accepted: no *automated* actor can
produce a body except through `build_pr_body` — the App cannot edit PR bodies
(the exact pain in vergil-tooling#2107), and agents cannot run raw `gh` (hook
guard). A human hand-editing a body post-submit is out of scope, and adding an
extra reference there is now permitted anyway.
