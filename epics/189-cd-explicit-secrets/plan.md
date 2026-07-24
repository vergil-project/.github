# CD explicit-secrets — Implementation Plan

> **For agentic workers:** each task below is a GitHub issue under epic
> vergil-project/.github#189, implemented via `vergil:issue-implement` (USER
> agent). **`cd.yml` tasks cannot be pushed by the agent** (no `workflow` token
> scope) — the agent edits/validates/commits, a human pushes.

**Goal:** replace blanket `secrets: inherit` in CD workflows with explicit,
language-aware least-privilege secrets, and stop `repo_init` re-introducing it.

**Tech stack:** GitHub Actions YAML, Python (`repo_init.py`), pytest.

## Global constraints

- Validation: `vrg-container-run -- vrg-validate` (green before report-ready).
- Git/GitHub via `vrg-git`/`vrg-gh`/`vrg-commit`.
- **Workflow-file pushes are human-only** (agent identity lacks `workflow`
  scope). For each `cd.yml` task: edit → `vrg-container-run -- vrg-validate` →
  `vrg-commit` → **hand the human a `git push` command to run from their shell**.
  Do not report-ready a `cd.yml` task through the agent push path; it will be
  refused.
- `cd-release.yml` secret map by ecosystem: python/go → none (OIDC); rust →
  `CARGO_REGISTRY_TOKEN`; ruby → `RUBYGEMS_API_KEY`; java → `CENTRAL_USERNAME`,
  `CENTRAL_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`.

---

### Task A — `cd.yml` per repo (#656 claude-plugin, #447 containers, #284 vm)

**Files:** `.github/workflows/cd.yml` in each repo.

- [ ] Inspect the repo's `cd.yml`: what `language:` (if any) does each
  `cd-release.yml` job pass? Does it publish anything needing a token?
- [ ] Replace `secrets: inherit` with an explicit `secrets:` map of exactly the
  needed secrets, or **remove the `secrets:` line** if none (python/OIDC, go,
  non-publishing, container-build via `GITHUB_TOKEN`).
- [ ] `vrg-container-run -- vrg-validate` (actionlint/yamllint) green.
- [ ] `vrg-commit --type fix --scope ci` with a message explaining the explicit
  map / OIDC rationale, `Ref` epic #189.
- [ ] **Hand the human the push command** (`git push origin <branch>` from the
  worktree, their shell). CI re-scan → semgrep clears `secrets-inherit`.
- [ ] Verify the acceptance: no `secrets: inherit`; map matches the real publish
  path (noting release runs only on `main`).

*(vergil-tooling's cd.yml is already done via #179 PR #2506 — proof-of-concept.)*

---

### Task B — `repo_init` template (#2508, vergil-tooling)

**Files:** `src/vergil_tooling/lib/repo_init.py` (release-job block, ~line 624);
tests in `tests/vergil_tooling/test_repo_init.py`.

- [ ] **RED:** extend the render-gitignore/workflow test — assert the generated
  `cd.yml` has **no** `secrets: inherit`; for a python fixture, no `secrets:`
  line; for java/rust/ruby fixtures, an explicit map listing exactly that
  ecosystem's secrets. Run; confirm it fails.
- [ ] **GREEN:** replace the unconditional `"    secrets: inherit\n"` with a
  language-aware explicit `secrets:` block keyed on `ctx.primary_language`
  (python/go → omit; rust/ruby/java → the map above, each value
  `${{ secrets.NAME }}`). Run; confirm green.
- [ ] **REFACTOR:** if the per-language map is non-trivial, extract a small helper
  (e.g. `_cd_release_secrets(language) -> list[str]`) so the mapping has one
  home; keep 100% branch coverage.
- [ ] `vrg-container-run -- vrg-validate` green → report-ready (this one is pure
  Python, so the normal agent push/report-ready flow works).

---

### Task C — documentation review (#2507, closing gate, vergil-tooling)

- [ ] Sweep for stale `secrets: inherit` / cd-secrets guidance in
  vergil-actions cd-release + semgrep docs, `repo_init` docs, and site CI/CD
  guides; align to the explicit per-language model.
- [ ] Spawn per-repo doc tasks (born linked under #189) for any docs that live in
  other repos; do not cross a repo boundary with one PR.
- [ ] Closes the epic once merged (final gate).

## Self-review

- Spec §4 design → Tasks A (per-repo) + B (template). §6 tasks all filed
  (#656/#447/#284/#2508/#2507; vergil-tooling via #2506). §5 workflow-push
  constraint reflected in Global constraints + Task A. No follow-on-brainstorm
  bookend, by recorded decision.
