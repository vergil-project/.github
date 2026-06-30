# Org documentation site — implementation plan

Epic: vergil-project/.github#60 · Spec: `./spec.md`

Tasks are filed in their owning member repo and linked under #60 at
implementation time (per the epic/task convention). Phase 1 ships the org site;
phase 2 adds per-repo Planning.

## Phase 1 — the `docs` site

### Task 1: Create & scaffold `vergil-project/docs` *(new repo)*
Create the repo; add `docs/site/` (mkdocs) mirroring the existing per-repo site
layout; wire the reusable docs-publish workflow + Pages; seed initial org
narrative pages (What is Vergil, architecture, getting-started, conventions);
`vergil.toml` as a documentation-profile repo (single-branch, no release
artifact, or its own release cadence per §3).

### Task 2 (vergil-tooling): roadmap/activity → site pages
Add a generator (or a thin wrapper over `vrg-roadmap` / `vrg-activity-log`) that
emits **mkdocs-ready** pages for the `docs` site, reading epics from
`<org>/.github` via the API. Output: a Roadmap page and an Activity-log page.

### Task 3 (`docs` repo): nightly + on-release publish job
CI in `docs` that runs Task 2's generator and republishes the site **nightly**
and **on each release**. Retires the deferred "commit roadmap.md into .github"
approach.

## Phase 2 — per-repo Planning

### Task 4 (vergil-tooling): `--repo` filter on the reporting CLIs
Add `--repo <owner>/<repo>` to `vrg-roadmap` and `vrg-activity-log`, returning
only the epics/activity touching that repo.

### Task 5: per-repo **Planning** section convention + rollout
Define the convention — each content repo's site gains a `Planning` page
(filtered roadmap/activity, via Task 4) beside release notes/changelog — and roll
it out to the content repos (one task per repo, or a templated sweep).

## Portability

### Task 6: document the portable `<org>/docs` recipe
Capture the recipe (repo + publish job + `--repo` filter + Planning convention)
so it drops onto logical-minds-foundry, mqrest-admin-project, nemosys-project
unchanged. Lives in this repo's convention docs / the `docs` site itself.

## Definition of done

`vergil-project/docs` publishes a discoverable site carrying the org narrative +
the generated roadmap/activity, refreshed nightly and on release; each content
repo surfaces its filtered Planning section; the recipe is documented and
portable to the other orgs.
