# Fleet `secrets: inherit` audit — Implementation Plan

> **For agentic workers:** tasks are GitHub issues under epic
> vergil-project/.github#197. The discovery task is a research/catalog PR; the
> remediation is split (vergil-project residual under this epic; other orgs via
> per-org triage handoffs — cross-org is never epic-linked).

**Goal:** find and fix every `secrets: inherit` across all six fleet orgs,
driving each to explicit least-privilege (or removal).

## Global constraints

- Validation: `vrg-container-run -- vrg-validate`. Git via `vrg-git`/`vrg-commit`.
- **Cross-org linkage is prohibited** — other-org work is referenced by comment /
  filed as triage in that org, never a sub-issue of #197.
- Fleet is checked out under `~/dev/projects/<org>/<repo>` — use existing
  checkouts; no redundant clones.
- **`cd.yml` / workflow-file edits can't be pushed by the agent** (no `workflow`
  token scope) — the agent edits/validates/commits, a human pushes (as in #189).

---

### Task 1 — enabler: `vrg-gh search code` (#2505, ad-hoc)

Not under this epic (it lives in the ad-hoc epic), but it **gates discovery**.
Add `search` to the `vrg-gh` allowlist so `vrg-gh search code … --owner <org>`
works. Merge + release + install before Task 2 runs.

### Task 2 — discovery / fleet catalog (#199, blocked-by #2505)

- [ ] For each of the six orgs: `vrg-gh search code 'secrets: inherit' --owner <org>`.
- [ ] Cross-check against `~/dev/projects/<org>/*` local checkouts; log coverage
  (search vs local; any repo not reached).
- [ ] For each hit, classify the fix (§4 of the spec) by inspecting the workflow
  it calls and what `secrets.*` that workflow reads.
- [ ] Write `epics/197-fleet-secrets-inherit-audit/catalog.md`: org, repo,
  workflow, quote, recommended fix, remediation owner.
- [ ] Catalog PR into `.github` (closes #199 on merge).

### Task 3 — vergil-project residual (from the catalog)

For any vergil-project repo the catalog flags that #189 missed: an impl task under
#197, one `cd.yml`/workflow fix per repo (explicit map or removal), human-pushed.

### Task 4 — per-org handoffs (from the catalog)

For each *other* org with hits: `vrg-triage-create --repo <org>/.github` carrying
that org's catalog subset + the per-repo recommended fix; reference it from #197
by comment. Each org grooms it into its own epic/tasks. (We can offer to drive a
given org's remediation as its own epic, separately.)

### Task 5 — doc-review (#2519, closing gate)

Light: confirm the catalog is published, every affected org has a tracked handoff,
and the explicit-secrets guidance (already shipped in #189) is accurate. Spawn a
per-repo doc task only if a real gap is found.

## Self-review

Spec §2 shape → Tasks 3/4 split; §3 discovery → Task 2 (blocked-by #2505/Task 1);
§4 classification → Task 2; §6 bookends → Tasks 1(doc)/5(doc-review), no
brainstorm/validation by design. Cross-org handled by triage, never linked.
