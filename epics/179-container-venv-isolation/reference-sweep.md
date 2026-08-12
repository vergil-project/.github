# Fleet reference sweep: stale `.venv-host` / dual-venv references

**Issue:** vergil-project/.github#182 (fleet reference sweep bookend of
epic vergil-project/.github#179)
**Type:** research / catalog (read-only sweep; no code changes)
**Date:** 2026-07-23

## Summary

Epic #179 retired the `.venv-host` dual-venv model: the dev container now
masks `/workspace/.venv` with an anonymous volume, so a single host `.venv`
is safe and the `UV_PROJECT_ENVIRONMENT=.venv-host` bootstrap is gone. This
sweep catalogs every remaining fleet reference to the obsolete model
(`.venv-host`, the dual-venv split, `UV_PROJECT_ENVIRONMENT=.venv-host`,
host-vs-container venv setup guidance).

Headline finding: **the biggest live surface is `vergil-vm`.** Every
Vergil-built VM (Lima agent VM, Azure, GCP, and the shared provision
script) still actively provisions `export UV_PROJECT_ENVIRONMENT=.venv-host`
into `~/.zshrc`, and the VM test suite asserts that export is present. That
is not stale prose — it is live code that configures the retired model onto
every new VM, and it needs a follow-up task (not just a doc edit).

Counts:

- **Live hits needing remediation:** 3 clusters —
  (1) `vergil-vm` provisioning + tests (in-org, code change),
  (2) `vergil-tooling` / `docs` human-facing docs + gitignore leftovers (in-org),
  (3) `logical-minds-foundry` consumer `.gitignore` files (other-org).
- **Historical hits (leave as-is):** CHANGELOGs, release notes, dated
  plan/spec files, completed-plan records, and the epic's own docs.
- **Cross-org triage handoffs:** `logical-minds-foundry` (5 repos) and
  `wphillipmoore/the-infrastructure-mindset` (article/research content).

## Coverage note (what was searched, what was not)

**Searched (local checkouts, grep for `.venv-host`, `UV_PROJECT_ENVIRONMENT`,
`dual-venv`, plus phrase variants `host venv` / `host-side venv` /
`own venv` / `separate venv`; excluded `.worktrees/`, `.venv/`,
`.venv-host/`, `node_modules/`, `.git/`):**

- vergil-project: `docs`, `vergil-actions`, `vergil-claude-plugin`,
  `vergil-containers`, `vergil-vm`, `vergil-tooling`, `.github` — all present
  locally and swept.
- Also swept the other org checkouts present on this host:
  `logical-minds-foundry` (7 sub-repos), `diogenes-project`,
  `mq-rest-admin-project`, `vergils-nemesis`, `wphillipmoore`.

**Could NOT search — logged, no silent cap:**

- **Cross-org GitHub code search was unavailable.** `vrg-gh search code`
  is not in the wrapper allowlist (`vrg-gh` permits only `issue`, `label`,
  `pr`, `repo`, `run`; raw `gh` is denied by the permission model). So any
  in-scope reference living in a repo that is **not checked out on this
  host** was not searched. Coverage of `logical-minds-foundry` and other
  orgs is limited to the repos present locally. A human with `gh` access
  should run, to close the gap:

  ```bash
  gh search code 'venv-host' --owner vergil-project
  gh search code 'venv-host' --owner logical-minds-foundry
  gh search code 'UV_PROJECT_ENVIRONMENT=.venv-host' --owner vergil-project
  ```

- The voice-to-text note ("VM" sometimes meaning the Python venv) was
  considered; the literal in-scope tokens above were used, plus the phrase
  variants. No additional venv-meaning-VM references surfaced beyond the
  token hits catalogued here.

## Live hits (recommend REMOVE / UPDATE)

Canonical reference for the single-`.venv` model is **vergil-tooling**
(`CLAUDE.md` "Environment Setup", already collapsed to a single `.venv` by
task T3 / vergil-tooling#2491; and the container anonymous-volume masking,
vergil-tooling #2486).

### Cluster 1 — `vergil-vm` (in-org; live provisioning + tests)

Every VM flavor sets the retired env var in `~/.zshrc` and the test suite
pins it. All five are the same block. This is **live code that provisions
the dual-venv model onto every new VM**, so it is the highest-priority
remediation and is a **code change, not a doc edit**.

| Path | Line | Context | Recommendation |
|---|---|---|---|
| `templates/provision/30-toolchain.sh` | 50 | `export UV_PROJECT_ENVIRONMENT=.venv-host` (+ rationale comment ll. 33–49) | Remove the export + comment; interactive host uv can use plain `.venv` now that the container masks `/workspace/.venv`. |
| `templates/agent.yaml` | 365 | `export UV_PROJECT_ENVIRONMENT=.venv-host` (+ comment ll. 344–364) | Same removal. |
| `opentofu/modules/azure/vm/cloud-init.yaml` | 322 | `export UV_PROJECT_ENVIRONMENT=.venv-host` (+ comment) | Same removal. |
| `opentofu/modules/gcp/vm/cloud-init.yaml` | 333 | `export UV_PROJECT_ENVIRONMENT=.venv-host` (+ comment) | Same removal. |
| `tests/test_base.sh` | 69–70 | `grep -q 'UV_PROJECT_ENVIRONMENT=.venv-host' "$HOME/.zshrc"` asserts the export is present | Invert/remove the assertion once provisioning drops the export. |
| `.gitignore` | 13 | `.venv-host/` | Replace with `.venv/` (or drop) — leftover ignore for a dir that is no longer produced. |

**Caveat for the implementer:** confirm the epic's anonymous-volume masking
behaves the same on VMs before deleting the export — the original rationale
(interactive host uv clobbering the container's `.venv`) is exactly what the
masking neutralizes, but VM provisioning should be validated, not assumed.
This is why it is a task, not a mechanical doc edit.

**Remediation owner:** in-org (`vergil-project/vergil-vm`). Spawn a
follow-up **implementation task under epic #179** (or a sibling epic) — this
is beyond the documentation-review gate's remit because it edits provisioning
code and a test.

### Cluster 2 — `vergil-tooling` + `docs` (in-org; human-facing docs & gitignore leftovers)

Per epic #179 plan.md, these human-facing docs were **explicitly deferred to
the documentation-review gate (#2477)**, which sweeps all human-facing docs.

| Repo | Path | Line | Context | Recommendation | Owner |
|---|---|---|---|---|---|
| vergil-tooling | `docs/site/docs/guides/consuming-repo-setup.md` | 79 | "use the `.venv-host` dev-tree override described in the vergil-tooling `CLAUDE.md`" | Stale + now dangling: `CLAUDE.md` no longer describes `.venv-host` (T3 collapsed it to single `.venv`). Update to the single-`.venv` dev-tree override. | doc-review gate #2477 |
| vergil-tooling | `docs/specs/host-level-tool.md` | 174, 179–202, 514, 527, 564, 653–669 | "Why `.venv-host`, not `.venv`" section + dev-tree bootstrap | Epic §4.4 flags "remove the 'Why `.venv-host`' rationale"; explicitly deferred to #2477. | doc-review gate #2477 |
| vergil-tooling | `.gitignore` | 12 | `.venv-host/` (line 11 already adds `.venv/` via T3) | Optional cleanup: drop the dead `.venv-host/` line. Epic mandated *adding* `.venv/`, not removing this; low priority. | in-org, vergil-tooling |
| docs (vergil-project/docs) | `.gitignore` | 13 | `.venv-host/` | Leftover ignore; replace with `.venv/` or drop. | in-org, vergil-project/docs |

### Cluster 3 — `logical-minds-foundry` consumer `.gitignore` files (OTHER ORG)

These are consumer repos generated from the `repo_init` `.gitignore`
template. The template bug the epic fixed (ignored `.venv-host/` but not
`.venv/`, leaving consumers' host venv untracked) is visible in their
committed `.gitignore` files. Re-running `repo_init` / `vrg-update` after the
fixed template ships would correct them; until then they are stale.

| Repo | Path | Line | Context | Recommendation |
|---|---|---|---|---|
| logical-minds-foundry/docs | `.gitignore` | 13 | `.venv-host/`, **no** `.venv/` | Ignore `.venv/`; drop `.venv-host/`. |
| logical-minds-foundry/.github | `.gitignore` | 13 | `.venv-host/`, **no** `.venv/` | Ignore `.venv/`; drop `.venv-host/`. |
| logical-minds-foundry/mq-resiliency-observability | `.gitignore` | 13 | `.venv-host/` (also has `.venv/` at 22) | Drop the dead `.venv-host/` line. |
| logical-minds-foundry/mq-gateway-replay-lab | `.gitignore` | 13 | `.venv-host/` (also has `.venv/` at 22) | Drop the dead `.venv-host/` line. |
| logical-minds-foundry/mq-resiliency-lab-for-linux | `.gitignore` | 35 | `.venv-host/` under a "host venv" comment | Update comment + drop `.venv-host/`. |

**Remediation owner:** other-org (`logical-minds-foundry`). Hand off via a
**triage issue into that org**, referenced by comment only — cross-org
linkage to epic #179 is out of scope. Note the fix is largely mechanical and
flows from re-running the fixed `repo_init` template.

## Historical hits (leave as-is — dated records of past decisions)

These merely record the dual-venv model as history and must not be edited
(consistent with the epic's own "historical plan/spec/release documents are
left as-is" rule).

- **vergil-tooling**
  - `CHANGELOG.md` (l. 28 retiring it; l. 3426 introducing it, #240)
  - `releases/v1.3.0.md`, `releases/v1.3.1.md`, `releases/v1.3.2.md`,
    `releases/v2.1.159.md` (the retirement release note)
  - `docs/plans/completed/2026-05-20-github-repo-init-plan.md`,
    `docs/plans/completed/2026-05-11-vergil-rename.md`,
    `docs/plans/completed/host-level-tool-plan.md`
  - `docs/plans/2026-06-16-vm-shared-from.md`,
    `docs/plans/in-progress/2026-05-20-p1-vergil-vm-repository.md`
  - `docs/specs/2026-05-20-github-repo-init-design.md`,
    `docs/specs/git-url-dev-dependency.md`, and the dated per-task plan specs
    under `docs/specs/2026-*` (batch-pr-pipeline, vrg-update-deps,
    claude-share-set-audit, vm-instance-name-length-limit) — all use
    `UV_PROJECT_ENVIRONMENT=.venv-host uv run …` as the per-step TDD command
    of a completed/dated plan.
  - `docs/superpowers/plans/2026-06-06-finalize-pr-progress-adoption.md`
  - `paad/pushback-reviews/*` and `paad/alignment-reviews/*` (review records)
  - **Not stale, keep as correct:** `tests/vergil_tooling/test_repo_init.py:348`
    `assert ".venv-host/" not in content` — this is the post-T3 assertion that
    the template no longer emits `.venv-host/`. Correct as written.
- **vergil-vm**
  - `CHANGELOG.md` (l. 91) and `releases/v2.1.27.md` (introduced the model, #206)
- **.github**
  - `epics/179-container-venv-isolation/{spec,plan}.md` — the epic's own
    design docs (in-scope terms appear only as the thing being retired).
    Excluded per task scope.

## Out-of-scope (Lima agent-VM) — for separate triage

No stale **Lima agent-VM (`vrg-vm`) setup** docs surfaced that are distinct
from the venv concern. The `vergil-vm` `templates/agent.yaml` hit above is an
agent-VM template, but the specific stale line is the `.venv-host` export
(venv concern, in scope), not agent-VM provisioning generally. Nothing else
to route here.

## Out-of-scope (other-org article content) — for separate triage

- **wphillipmoore/the-infrastructure-mindset** (personal org, article repo):
  - `articles/R0004-claims.md:27` — "Phase 7 (dual-venv bootstrap, …) was
    introduced in commit fea4a79 (#240)."
  - `articles/A0003-…/research/containerization-changes-snapshot.md`
    (ll. 119–196, 293) — narrates the dual-venv model, including the
    `UV_PROJECT_ENVIRONMENT=.venv-host` bootstrap, as part of the
    containerization story.

  These are editorial/research narrative, not fleet guidance that provisions
  anything. They read as historical, but an article presenting the model as
  current could mislead readers now that #179 retired it. **Author's call** —
  route as a note to the article owner, not an epic-linked task. Other-org,
  out of scope for #179 linkage.

- **False positives (not the dual-venv model, no action):**
  `vergil-claude-plugin/docs/development/skills-architecture.md:361` (generic
  "host venv" mention); `vergil-vm/build/mempalace/**` CONTRIBUTING files
  (third-party build artifact, generic "pip in your own venv").

## Split remediation plan

**In-org (vergil-project) — spawnable under / adjacent to epic #179:**

1. **`vergil-vm` provisioning task (code change).** Remove
   `export UV_PROJECT_ENVIRONMENT=.venv-host` from
   `templates/provision/30-toolchain.sh`, `templates/agent.yaml`,
   `opentofu/modules/azure/vm/cloud-init.yaml`,
   `opentofu/modules/gcp/vm/cloud-init.yaml`; invert/remove the assertion in
   `tests/test_base.sh`; replace `.venv-host/` with `.venv/` in `.gitignore`.
   Validate the single-`.venv` model on a real VM before landing. → new task.
2. **Human-facing docs (doc-review gate #2477).**
   `vergil-tooling/docs/site/docs/guides/consuming-repo-setup.md` and
   `vergil-tooling/docs/specs/host-level-tool.md` — already claimed by #2477.
3. **Gitignore leftovers (low priority, in-org).**
   `vergil-tooling/.gitignore:12` and `docs/.gitignore:13` — drop the dead
   `.venv-host/` line / ensure `.venv/` is ignored.

**Other-org — triage handoff (referenced by comment, never epic-linked):**

1. **`logical-minds-foundry` consumer `.gitignore` files** (5 repos above) —
   file a triage issue in that org; the fix rides the corrected `repo_init`
   template.
2. **`wphillipmoore/the-infrastructure-mindset` article content** — note to
   the article owner that the dual-venv narrative is now historical.
