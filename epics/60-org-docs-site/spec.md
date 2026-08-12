# Org documentation site (`docs` repo) & published roadmap

- **Epic:** vergil-project/.github#60
- **Answers:** idea vergil-project/.github#59
- **Status:** Approved design (2026-06-30)

## 1. Problem

The epic/task observability (`vrg-roadmap`, `vrg-activity-log`, `vrg-epic-audit`)
is generated, but has **no published home**. The data lives in `<org>/.github`,
which is GitHub's *special* repo — hidden, profile/community-health-focused, not
something most people know to visit, and not meant to be a versioned/released
documentation product. There is no obvious, discoverable place to see what a
project is and where it is going.

## 2. Decision

Create a generic, per-org **`<org>/docs`** repository — a normal, versioned,
released repo that publishes the organization's documentation **site**.

- **Name `docs`** — chosen for genericity: it travels to every org regardless of
  org naming (`vergil-project/docs`, `logical-minds-foundry/docs`,
  `mqrest-admin-project/docs`), it is honest about its contents, and it is the
  obvious place to look. (`<org>/<org>` was rejected — doubled, fugly, and orgs
  are inconsistently named; brand names like `vergil` don't generalize; `common`
  collides with the shared-code meaning in mqrest-admin.)
- **Complements `.github`**, does not replace it. `.github` keeps the profile
  README and community-health files (which GitHub requires there); `docs` is the
  discoverable, published, versioned site.

## 3. What `docs` contains

1. **Hand-written org-wide narrative** — what `<project>` is, architecture,
   getting-started, conventions.
2. **The generated roadmap + activity-log**, as published pages — the org's plan
   and recent throughput, finally findable.

The repo is strictly documentation; it is built and published like the existing
per-repo sites (mkdocs + the reusable docs-publish workflow), and released on a
cadence with its own version/changelog.

## 4. Data flow — generators run in `docs`

A CI job in the `docs` repo, **nightly + on each release**, runs `vrg-roadmap` /
`vrg-activity-log` (which read the epics from `<org>/.github` via the GitHub API,
not local files), writes them as mkdocs pages, builds, and publishes (Pages).

`.github` remains the **data home** (epics + their `epics/<N>-<slug>/` docs);
`docs` is the **published view**. This **subsumes the deferred "nightly publish"
task** — there is no longer any need to commit `roadmap.md`/`activity-log.md`
into `.github`.

## 5. Per-repo Planning section

Each content repo's existing site gains a **Planning** page, beside its release
notes/changelog: the same roadmap/activity-log **filtered to that repo**, so a
developer sees that repo's slice of the org plan where they already look.

Requires a `--repo <owner>/<repo>` filter on the reporting CLIs
(`vrg-roadmap`, `vrg-activity-log`). Phaseable: the org `docs` site ships first;
the per-repo Planning sections roll out second.

## 6. Portability

The recipe — `<org>/docs` + the publish job + the `--repo` filter + the per-repo
Planning convention — is documented so it drops, unchanged, onto every org
(logical-minds-foundry, mqrest-admin-project, nemosys-project). Each org gets its
own `docs` site from the same template.

## 7. Non-goals

- Not replacing `.github` (profile / community-health stays there).
- Not a per-org code or config repo (`docs` is strictly documentation).
- Cross-org aggregation is out of scope — each org publishes its own `docs`
  site; epics never span orgs.
