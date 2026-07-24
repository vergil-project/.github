# Retire blanket `secrets: inherit` from CD workflows

- **Epic:** vergil-project/.github#189
- **Status:** Design drafted 2026-07-24. Simple/fast-tracked (no brainstorm — the
  design was settled interactively during the #179 pr-watch triage that surfaced
  it). Full epic convention otherwise.
- **Repo:** vergil-project/vergil-tooling + member repos; epic homed in `.github`.
- **Origin:** surfaced by a `security / semgrep` failure on
  vergil-project/vergil-tooling#2477 (PR #2506) during epic #179 — the
  `yaml.github-actions.security.secrets-inherit` rule, newly live in semgrep's
  floating registry ruleset, flagged our long-standing `secrets: inherit`.

## 1. Summary

Our CD workflows pass **`secrets: inherit`** to the reusable `cd-release.yml`,
handing the called workflow **every** repository secret instead of only the ones
it needs — a least-privilege violation. We replace the blanket inherit with an
**explicit, language-aware secrets map** per repo, and fix the `repo_init`
template so new repos never re-introduce the pattern.

## 2. Why fix, not pin (philosophy)

We **unpin by default and stay on the leading edge**, pinning only when the
leading-edge fix is genuinely too expensive — then documenting the debt.
Pinning-by-default rots into a bigger, costlier upgrade leap later (the trailing
edge stops supporting the pinned version). Here the real fix is cheap, so we take
it. This epic *is* that methodology in action: the leading edge flagged a weak
spot; we evaluated the fix; it was cheap; we fixed rather than pinned.

(Separate observation, **out of scope**: semgrep's gate floats on two axes — the
binary via nightly `prod-base:latest` rebuilds, and the `p/*` rules fetched live
from the registry — so a required check can flip with no change from us. That
determinism concern is noted for a possible follow-on, not solved here.)

## 3. Root cause & scope

`repo_init.py` (~line 624) appends `"    secrets: inherit\n"` unconditionally to
every generated `cd.yml` under `if ctx.publish_release`, so the pattern is
fleet-wide. Confirmed live in **vergil-tooling, vergil-claude-plugin,
vergil-containers, vergil-vm**.

`cd-release.yml` (vergil-actions) declares/uses six secrets, all
ecosystem-specific: `CARGO_REGISTRY_TOKEN` (rust), `RUBYGEMS_API_KEY` (ruby),
`CENTRAL_USERNAME`/`CENTRAL_TOKEN`/`GPG_PRIVATE_KEY`/`GPG_PASSPHRASE` (java).
**Python publishes via OIDC trusted publishing** (`id-token: write`) → **zero**
secrets; go / non-publishing → zero. So each repo passes an explicit map of only
its language's secrets, often **none**.

## 4. Design

- **Per-repo `cd.yml`:** replace `secrets: inherit` with an explicit
  `secrets:` map of exactly the secrets that repo's release path uses — or remove
  the `secrets:` line entirely when none are needed (python/OIDC, go,
  non-publishing).
- **`repo_init` template:** emit a **language-aware** explicit map keyed on
  `ctx.primary_language` (python/go → none; rust → cargo; ruby → rubygems; java →
  central + gpg), never blanket `inherit`; update the render test to cover each
  ecosystem branch.

## 5. Constraints

- **Workflow-push wall (operational reality):** the `user` agent identity lacks
  the GitHub `workflow` token scope, so it **cannot push `.github/workflows/*`
  changes** — it edits, validates, and commits, but a **human maintainer pushes**
  each `cd.yml` change from an elevated shell. Only the template task (pure
  Python) goes through the normal agent report-ready/push flow.
- **Release runs only on `main`:** a `cd.yml` PR's CI does not exercise the
  release job, so the semgrep re-scan proves the *finding* is resolved but
  **publish-correctness** with the explicit map is confirmed at the **next
  release** — verify each map covers the real publish path.

## 6. Tasks

- **#190 — documentation** (`.github`): this spec + plan.
- **cd.yml fixes** (one per repo, explicit map): vergil-claude-plugin#656,
  vergil-containers#447, vergil-vm#284. (vergil-tooling's `cd.yml` was fixed
  **tactically via #179's PR #2506 / #2477** as the proof-of-concept — it lands
  via #179, not as a task here.)
- **#2508 — `repo_init` template** (`vergil-tooling`): language-aware explicit
  map + test.
- **#2507 — documentation review** (`vergil-tooling`, closing gate): align docs
  (vergil-actions cd-release/semgrep docs, `repo_init` docs, site CI/CD guides) to
  the explicit model; spawn per-repo doc tasks if the sweep lands elsewhere.

**No follow-on-brainstorm bookend.** Decision recorded here: nothing downstream
is anticipated beyond the separately-noted semgrep-determinism thread, so the
"what comes next" gate is satisfied without a task.

## 7. Out of scope

- Pinning/vendoring the semgrep binary or ruleset (determinism) — separate concern.
- Any non-CD secret usage.
