# coherent-docs-site — implementation plan

- **Epic:** vergil-project/.github#1668
- **Planning issue:** vergil-project/.github#1666 (review history is in its comments)
- **Status:** draft

> **For agentic workers:** each task is one issue and one PR in the repo named
> in its heading, worked with `issue-implement`. Operational tasks
> (deployment, validation) are worked with `issue-deploy` and
> `issue-validate`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every page of every docs site in vergil-project,
logical-minds-foundry and mnemosys-project is reachable from its site's home
page, every site follows one navigation standard, and a check in
`vrg-validate` and PR CI enforces it.

**Architecture:**

- **The hub first.** `vergil-project/docs` is restructured to the six-tab
  layout with a home-page map and the standard's page (Task 2). It depends
  on nothing else.
- **The ignore entries first.** The new staging outputs are added to the
  managed `.gitignore` (Task 1a), released and synced to every repo of the
  in-scope orgs (Task 1b) before any code that writes them can reach a
  repo.
- **vergil-tooling** gains the org's lists (`lib/doc_sites.py`, Task 3),
  the navigation check (`lib/docs_nav.py` and `vrg-docs-nav-check`, Tasks
  4 and 5), and two staging steps in `vrg-docs-stage`: the hub's generated
  Repositories page (Task 6a) and the member sites' header link to the hub
  (Task 6b). The site template is brought to the standard (Task 7). Every
  org is report-only until Task 13.
- **vergil-actions** runs the check after PR CI's strict build and on the
  docs deploy (Task 8).
- **Rollout.** Release report-only (Tasks 9, 10), audit and file one fix
  task per short site (Task 11), fix (Task 12-*), re-audit (Task 12z),
  switch the in-scope orgs to enforcing (Task 13), release (Task 14) and
  validate (Task 15).

**Tech Stack:** Python 3 and pytest, MkDocs Material, the standard library's
`html.parser` (vergil-tooling); GitHub Actions YAML (vergil-actions);
Markdown (the docs sites).

**Spec:** [`spec.md`](spec.md) in this directory. Executors read both.

## Global Constraints

- **A docs site is a published MkDocs site built from
  `docs/site/mkdocs.yml`.** `packages` is not one; the check, the audit and
  the standard skip it (spec §4.1).
- **Tab cap: at most 7 top-level entries, Home included,** on a site using
  `navigation.tabs`. More than 7 means layout B: no tab row, full tree in
  the side panel (spec §4.1 rule 1).
- **Checked rules:** orphans, the home-page map (rule 2), the tab cap
  (rule 1) and the hub link (rule 5). Rules 3 and 4 are reviewed, never
  checked; rule 6 holds by construction (spec §4.1).
- **The hub link** points at `https://<org>.github.io/docs/`, sits in the
  site header on every page, is the same on every member site, and is
  labelled for the project's docs (on vergil-project, `Vergil project
  docs`). It applies only in an org whose lists say it has a hub (spec
  §4.1 rule 5).
- **The standard is intra-org.** No rule requires or forbids a link to
  another org's docs (spec §1).
- **The gate needs no network.** Only the audit's crawl of a published site
  does (spec §4.3).
- **A site the check cannot judge is a finding,** never a pass (spec §2,
  anti-goals).
- **Two modes, set in vergil-tooling:** report-only (findings are visible
  warnings, the stage passes) and enforcing (any finding fails the stage).
  Enforcing applies only in an org the lists mark `enforcing=True`; every
  other org, and any org not in the lists, is report-only. The deploy run
  follows the same mode as the gate, before the publish step (spec §4.3).
- **A failure of the check itself is never a finding.** An internal error
  (a traceback, a missing input, a failed build of the tree to judge)
  exits non-zero in either mode and fails its stage, the deploy included.
  Report-only softens findings only.
- **Every staging output is gitignored before it is written.** The managed
  `.gitignore` carries them (Task 1a), and the local gate, the audit and CI
  stage through the same code, so their outputs are the same.
- **The Pages owner is the org of the repo being checked,** never the
  constant `vergil-project` (spec §4.3).
- **In-scope orgs:** `vergil-project`, `logical-minds-foundry`,
  `mnemosys-project`. Inactive org with docs sites: `mq-rest-admin-project`
  (spec §4.4).
- **The bar is the strict build and the published `dev` version** (or the
  published site, for a site mike does not version). `latest` and older
  versions are out of scope (spec §2, non-goals).
- **The hub's six tabs, in order:** Home, Logbook, Management, Platform,
  Repositories, Getting started (spec §4.2).

## Review Focus

1. **A repo in an org the lists do not know** (a new org, a fork, a clone
   with an unparseable `origin`). The check runs report-only, says the org
   is not in the lists, and never fails the stage. Pinned in Task 3
   (`test_org_docs_unknown_org_is_report_only`) and Task 4
   (`test_gate_unknown_org_reports_and_passes`).
2. **A page reachable only from the tab row** (the hub's Logbook today).
   The check reports the section as missing from the home-page map even
   though no page is an orphan. Pinned in Task 4
   (`test_home_map_tab_only_section_is_finding`).
3. **Links the crawl must not follow or count** (an anchor `#x` on the same
   page, a link to another site of the org, a `mailto:`, a `../` link that
   leaves the site, the search page, `404.html`, `sitemap.xml`). In-site
   targets resolve; everything else is ignored; generated non-content files
   are never orphans. Pinned in Task 4
   (`test_crawl_ignores_external_and_special_targets`,
   `test_crawl_never_reports_special_files_as_orphans`).
4. **A mike-versioned published site.** The audit crawls under `/dev/`,
   not the root redirect and not `latest`, and does not report the other
   versions' pages. Pinned in Task 5
   (`test_audit_published_crawl_stays_under_dev`).
5. **A site that fails its strict build in the audit.** It is a finding
   for that site, the audit goes on to the next site, and the exit is
   non-zero. Pinned in Task 5 (`test_audit_build_failure_is_finding`,
   `test_merge_keeps_every_site_and_coverage`).

## Delivery table

Each line gives the task, its repo and kind, what it delivers, and what it
is blocked by.

- **1** (.github, task #1669): spec and plan, this PR.
- **1a** (vergil-tooling): the managed `.gitignore` gains the new staging
  outputs (`docs/site/docs/repositories.md`, `docs/site/overrides.staged/`).
  Blocked by nothing.
- **1b** (.github, deployment): the vergil-tooling release carrying 1a,
  and the managed `.gitignore` synced to every repo of the in-scope orgs
  at once, not left to the nightly job. Blocked by 1a.
- **2** (docs): the hub restructure: six tabs in the spec's order, the
  home page as a map, the navigation standard's page, the
  vergil-claude-plugin title fix. Blocked by nothing.
- **3** (vergil-tooling): `lib/doc_sites.py`, the org's lists filled from
  an inventory of the in-scope orgs, every org `enforcing=False`;
  `doc_links` and the docs-link check also judge each listed org's own
  Pages links, report-only until the org enforces.
- **4** (vergil-tooling): `lib/docs_nav.py` (crawl, home map, tab cap, hub
  link) and `vrg-docs-nav-check` gate mode, run in `vrg-validate`'s
  `docs` stage after the strict build; report-only. Blocked by 3.
- **5** (vergil-tooling): `vrg-docs-nav-check --audit`: the strict build
  and the published `dev` crawl of each site, and the lists' coverage
  report. Blocked by 4.
- **6a** (vergil-tooling): docs staging (CI's `vrg-docs-stage` and the
  local gate's) writes the hub's `repositories.md` from the lists. Blocked
  by 1b, 3.
- **6b** (vergil-tooling): docs staging adds the header link to the hub on
  every member site of an org with a hub. Blocked by 1b, 3.
- **7** (vergil-tooling): the `vrg-github-repo-init` site template brought
  to the standard, with a test that its built site passes the check.
  Blocked by 4, 6b.
- **8** (vergil-actions): `ci-docs.yml` runs the check after its strict
  build; `cd-docs.yml` builds and runs it after the pre-deploy command and
  staging, before the publish, in the org's mode. Blocked by 9.
- **9** (.github, deployment): the vergil-tooling release carrying 3–7,
  report-only, installed on the host and every VM. Blocked by 3, 4, 5, 6a,
  6b, 7.
- **8b** (vergil-actions): `ops-docs-coverage.yml`, a reusable ops
  workflow that runs `vrg-docs-nav-check --coverage <calling org>` daily,
  with an `ops-report` job that reports a failure to the org's operations
  queue. Blocked by 9.
- **10** (.github, deployment): the vergil-actions release carrying 8 and
  8b, on the `v2.1` series. Blocked by 8, 8b.
- **2b** (docs): the hub's Repositories tab from the generated page,
  replacing the home-page table. Blocked by 2, 9.
- **11** (.github, validation): the audit of every docs site in the
  in-scope orgs with the released check, recorded per site; the human's
  review; one fix task filed per short site, and each added to 12z's
  `Blocked-by:`. Blocked by 2b, 9, 10.
- **12-*** (each site's repo; filed by 11, not by `file-tasks`): one fix
  task per site the audit finds short of the standard; and, if any of them
  is in vergil-tooling, a deployment task (`Plan task: 12 release`) that
  releases and installs it and redeploys the hub, blocked by those
  vergil-tooling fixes.
- **12z** (.github, validation): the re-audit: every in-scope site passes
  on its strict build and its published `dev` version. Blocked by 11 and
  every 12-* task, the `12 release` task included.
- **13** (vergil-tooling): the in-scope orgs marked `enforcing=True`.
  Blocked by 12z.
- **14** (.github, deployment): the vergil-tooling release carrying 13,
  installed on the host and every VM. Blocked by 13.
- **15** (.github, validation): the enforcing gate passes in every docs
  site's repo of the in-scope orgs, and each org's daily coverage job has
  run green. Blocked by 14, 17a, 17b, 17c.
- **17a** (vergil-project/.github), **17b**
  (logical-minds-foundry/.github, under that org's proxy, by full ref) and
  **17c** (mnemosys-project/.github, under that org's proxy, by full ref):
  the org's `ops.yml` calls `ops-docs-coverage.yml@v2.1` as a
  `docs-coverage` job. Each blocked by 10 and 12z, so the first daily run
  judges the corrected lists.
- **16** (fixture runs, via `fixture-run`): `python-lib`, `python-app`,
  `rust-lib`, `rust-app`, `ts-lib`, `ts-app`; `--validates "the
  report-only docs navigation check: vergil-tooling <9's release> and
  vergil-actions <10's release>"`; `--after` 9 and 10. Each fixture is
  also a site the audit judges, so a run's change task may carry that
  fixture's fix (Task 11 says how).

### Bookends

- **doc-review** (vergil-tooling, `Plan task: doc-review`): blocked by 15.
  Its sweep covers vergil-tooling's `docs/site/docs/reference/` pages for
  `vrg-docs-nav-check`, `vrg-validate`'s docs stage and `vrg-docs-stage`;
  the hub's navigation-standard page and its Platform/CI pages; and
  vergil-actions' docs for `ci-docs.yml`, `cd-docs.yml` and
  `ops-docs-coverage.yml`, each a spawned
  per-repo doc task.
- **retrospective** (.github, `Plan task: retrospective`): blocked by
  everything else.

### Filed outside the epic

- **mq-rest-admin-project follow-up.** In `file-tasks`, one issue in
  `mq-rest-admin-project/.github`, titled `Bring the docs sites up to the
  Vergil navigation standard`, its body naming this epic, the standard's
  page and the two steps (fix each site; mark the org `enforcing=True` in
  vergil-tooling's `lib/doc_sites.py`). Filed with `vrg-intake-create
  --repo mq-rest-admin-project/.github`, then routed with
  `vrg-intake-route <ref> defer --reason "outside coherent-docs-site's
  scope: the org is inactive" --revisit-when "the org returns to active
  development"`. It is not a child of this epic, so the key pass cannot see
  it: the stage's progress comment records it on a `Follow-up filed:
  <ref>` line, and a re-entry that finds a trusted one files nothing. If
  the org has no intake or deferred queue, stop for the human.

Notes:

- **No enforcing release before every site passes.** vergil-tooling
  releases from develop often and every repo pins only `v2.1`, so any task
  may reach the fleet before Task 9. Every vergil-tooling task up to 13
  keeps every org `enforcing=False`, so no release before Task 14 can fail
  a repo's docs stage. Task 13 is blocked by the re-audit (12z).
- **The ignore entries ship before the code that needs them.** For the
  same reason, Tasks 6a and 6b could reach the fleet in any release once
  merged, and staging refuses to write a path that is not gitignored
  (`unignored_staging_error`). So 6a and 6b are blocked by 1b, the
  deployment that has the entries in every in-scope repo, not only by 1a's
  merge.
- **Task 8 needs the command released.** vergil-actions' CI and every
  caller install the newest vergil-tooling `v2.1.x` tag, so a workflow that
  calls `vrg-docs-nav-check` before Task 9's release would fail every docs
  job. Task 8 is therefore blocked by Task 9, not by Task 4.
- **The hub's Repositories tab waits for the generator's release** (2b
  blocked by 9) for the same reason: the hub's strict build stages the
  page through the released `vrg-docs-stage`.
- **Fix tasks are filed by the audit,** since only the audit knows which
  sites are short. Task 11 files each in the site's own repo, under this
  epic for vergil-project and under the org's proxy otherwise
  (`vrg-epic-create --proxy-for vergil-project/.github#1668 --repo
  <org>/<repo>`), keyed `Plan task: 12 <org>/<repo>` (a fix to the lists
  is `Plan task: 12 coverage`, and the release after any vergil-tooling
  fix `Plan task: 12 release`), and adds a
  `Blocked-by:` line for each to Task 12z's body. Task 12z's precondition
  re-lists the epic's and proxies' children and refuses while any `Plan
  task: 12 …` task is open, so a fix task added later still gates it.
- **Fixture runs.** The epic changes shared CI workflows (`ci-docs.yml`,
  `cd-docs.yml`), so it seeds a fixture run on every fixture. The runs
  judge the report-only release; the enforcing release is judged by
  Task 15 across every site, fixtures included.
- **No `dev-` image validation:** no step depends on freshly published
  images.
- **No follow-on brainstorm task:** no enabling chain is known. A hub for
  an org with none, and cross-site search, are non-goals, accrued as intake
  items if wanted.
- **Docs travel with behaviour.** Each vergil-tooling and vergil-actions
  task updates its repo's reference docs in the same PR (named in its
  Files); the doc-review bookend checks the whole.

---

### Task 1a: the managed ignore entries (vergil-tooling)

Spec §4.4 step 1 (no repo meets a failing docs gate).

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/base` (the docs staging
  block gains `docs/site/docs/repositories.md` and
  `docs/site/overrides.staged/`)
- Test: the existing managed-gitignore tests, extended for the two entries

- [ ] **Step 1: Write the failing test:** the rendered managed
  `.gitignore` ignores both paths.
- [ ] **Step 2: Run it to verify it fails.**
- [ ] **Step 3: Add the two entries** beside the existing staging entries,
  with a comment naming the staging step that writes each.
- [ ] **Step 4: Run the full gate** — PASS.
- [ ] **Step 5: Commit.**

### Task 1b: release and sync the ignore entries (.github, deployment)

`vrg-issue-create --kind deployment --blocked-by` 1a.

- **Human-gated precondition:** a vergil-tooling release carrying Task 1a,
  checked with `vrg-release-record check vergil-project/vergil-tooling#<1a>`,
  exit 0.
- **Human-run steps (need a human identity):** upgrade the host and the VMs
  to the release (`vrg-host upgrade`, `vrg-vm update --all`), then sync the
  managed files at once rather than waiting for the nightly job: either
  dispatch the GitHub config job of each in-scope org's `ops.yml`, or run
  `vrg-github-repo-config update` in each repo of the in-scope orgs. The
  sync PRs auto-merge.
- **Agent-safe check:** in each repo of the three in-scope orgs that has
  `docs/site/mkdocs.yml`, after `vrg-git fetch origin`, `vrg-git show
  origin/develop:.gitignore` carries both entries.
- **Acceptance:** every such repo's `develop` ignores both paths.

### Task 2: the hub restructure (docs)

Spec §4.2, §4.1 rules 2–4, §4.5.

**Files:**

- Modify: `docs/site/mkdocs.yml` (`nav`)
- Modify: `docs/site/docs/index.md` (the home-page map)
- Create: `docs/site/docs/platform/index.md`; move `languages.md`, `ci.md`,
  `packages.md` under `docs/site/docs/platform/`
- Create: `docs/site/docs/management/navigation-standard.md`
- Modify: every page linking to a moved page (a grep for `languages.md`,
  `ci.md`, `packages.md`)

**Interfaces:**

- Produces: the page `management/navigation-standard.md`, at
  `https://vergil-project.github.io/docs/management/navigation-standard/`,
  which Task 4's messages and Task 7's template link to.

- [ ] **Step 1: Restructure `nav`**

  Top level, in order: `Home: index.md`; `Logbook` (unchanged entries);
  `Management` (unchanged entries, plus `Navigation standard:
  management/navigation-standard.md`); `Platform` (`platform/index.md`,
  `Supported languages`, `Continuous integration`, `Package repository`);
  `Getting started: getting-started.md`. Six tabs come with Task 2b's
  Repositories; until then there are five.

- [ ] **Step 2: Write the home page as a map**

  One section per top-level tab, each linking the section's index and its
  key pages: Logbook links its five level indexes (daily first) and Hoshin
  and Fleet; Management its six pages, the Roadmap and Activity; Platform
  its three pages. Keep the repository table and the closing generated-pages
  note until Task 2b. In the table, the vergil-claude-plugin row drops any
  "Standard Tooling Plugin" wording.

- [ ] **Step 3: Write the navigation standard's page**

  Rules 1–6 of spec §4.1 as written there, the definition of a docs site,
  layout B, which rules the check enforces and which are reviewed, and the
  intra-org scope. `management/hoshin-kanri.md` and `logbook/index.md`
  keep their mutual links (rule 4).

- [ ] **Step 4: Validate**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS (the strict build resolves every moved link).

- [ ] **Step 5: Commit** with `vrg-commit --type docs`.

### Task 2b: the hub's Repositories tab (docs)

Spec §4.1 rule 6, §4.2.

**Files:**

- Modify: `docs/site/mkdocs.yml` (`Repositories: repositories.md` between
  Platform and Getting started)
- Modify: `docs/site/docs/index.md` (drop the repository table; the map
  gains a Repositories card linking `repositories.md`; the `packages` row's
  link moves to the Platform card)

**Interfaces:**

- Consumes: Task 6a's staged `repositories.md` (released in Task 9),
  gitignored through the managed `.gitignore` (Task 1b).

- [ ] **Step 1: Make the nav and home-page changes above**
- [ ] **Step 2: Validate**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS, and the staged site has a `repositories/index.html`
  listing every repo of vergil-project, the TypeScript fixtures included.

- [ ] **Step 3: Commit.**

### Task 3: the org's lists (vergil-tooling)

Spec §4.3 (the org's lists), §4.4 step 1.

**Files:**

- Create: `src/vergil_tooling/lib/doc_sites.py`
- Modify: `src/vergil_tooling/lib/doc_links.py` (the site sets and
  `PAGES_OWNER` give way to `doc_sites`; `_SITE_URL` matches
  `<owner>.github.io` for the owner passed in)
- Modify: `src/vergil_tooling/bin/validate_common.py:190-260` (the own-site
  and docs-link check use the checked repo's org)
- Test: `tests/vergil_tooling/test_doc_sites.py`,
  `tests/vergil_tooling/test_doc_links.py`,
  `tests/vergil_tooling/test_validate_common.py`

**Interfaces:**

- Produces, in `lib/doc_sites.py`:
  - `@dataclass(frozen=True) class Repo: name: str; docs_site: bool;
    versioned: bool` (`versioned` implies `docs_site`; a repo with a Pages
    output that is not a docs site, such as `packages`, is
    `docs_site=False` and is named in `OrgDocs.other_pages`).
  - `@dataclass(frozen=True) class OrgDocs: org: str; has_hub: bool;
    hub_label: str; enforcing: bool; repos: tuple[Repo, ...]; other_pages:
    frozenset[str]`. `hub_label` is the header link's label (on
    vergil-project, `Vergil project docs`), recorded in the inventory.
  - `ORGS: dict[str, OrgDocs]`, keyed by lower-case org, with
    vergil-project, logical-minds-foundry, mnemosys-project and
    mq-rest-admin-project, every `enforcing=False`.
  - `org_docs(org: str) -> OrgDocs | None` (case-insensitive).
  - `is_enforcing(org: str | None) -> bool`: `False` for an unknown or
    `None` org.
  - `hub_url(org: str) -> str`: `https://<org>.github.io/docs/`.
  - `site_url(org: str, repo: str, *, version: str | None) -> str`:
    `https://<org>.github.io/<repo>/` plus `<version>/` when given.
  - `published_sites(org: str) -> frozenset[str]` and
    `versioned_sites(org: str) -> frozenset[str]`, which `doc_links` uses.
  - `repo_org(repo_root: Path) -> str | None`: the owner from `origin` on
    github.com, `None` otherwise.

- [ ] **Step 1: Take the inventory**

  For each of the four orgs, list its non-archived repos (`vrg-gh repo
  list <org> --json name,isArchived`) and, for each, whether
  `docs/site/mkdocs.yml` exists on `develop` and whether its `cd.yml`
  deploys versioned docs. Record it in `ORGS`, with a comment giving the
  date of the inventory. `has_hub` is whether the org has a `docs` repo
  with a docs site. `hub_label` is written into the inventory, not derived
  from the hub's `site_name` (vergil-project's is `"Vergil"`):
  `Vergil project docs` for vergil-project, `Logical Minds Foundry docs`
  for logical-minds-foundry and `Mnemosys Project docs` for
  mnemosys-project; for mq-rest-admin-project, if it has a hub, its
  `site_name` followed by `docs`.

- [ ] **Step 2: Write the failing tests**

  - `test_org_docs_vergil_project_matches_today`: vergil-project's
    `versioned_sites` equals today's `VERSIONED_SITES`, and
    `published_sites` adds `docs` and `packages`.
  - `test_org_docs_unknown_org_is_report_only`: `org_docs("nope")` is
    `None`, `is_enforcing("nope")` and `is_enforcing(None)` are `False`.
  - `test_no_org_enforces_yet`: every `ORGS` entry has `enforcing=False`.
  - `test_versioned_implies_docs_site`: no `Repo` has `versioned` without
    `docs_site`.
  - `test_vergil_project_links_judged_from_every_org` (doc_links): a link
    to an unpublished `https://vergil-project.github.io/<repo>/` is still a
    hit when checked from a logical-minds-foundry repo, and still fails the
    check, as today.
  - `test_own_org_links_judged_for_listed_org` (doc_links): a link to
    `https://logical-minds-foundry.github.io/nope/` is unpublished when the
    checked repo is in logical-minds-foundry.
  - `test_own_org_links_unjudged_for_unknown_org` (doc_links): a repo in an
    org not in `ORGS` has its own-org Pages links left unjudged, never
    flagged against an empty set.
  - `test_own_org_hits_warn_until_enforcing` (validate_common): a hit on a
    repo's own-org Pages host, in an org that is not enforcing and is not
    vergil-project, is a warning and the check passes; with the org
    patched to enforcing, it fails.

- [ ] **Step 3: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_doc_sites.py tests/vergil_tooling/test_doc_links.py tests/vergil_tooling/test_validate_common.py -v`
  Expected: FAIL (no module `doc_sites`).

- [ ] **Step 4: Implement** the module and switch `doc_links` and
  `validate_common` to it. `doc_links`' functions take the owner of the
  host being judged. Links to `vergil-project.github.io` are judged from
  every org and enforced, exactly as today. In addition, a repo in another
  org listed in `ORGS` has its links to its own org's Pages host judged,
  and those hits follow `is_enforcing(org)`: warnings that leave the check
  passing until Task 13. A repo in an org not in `ORGS` gets only today's
  check. Links to any other org's Pages host are left unjudged.

- [ ] **Step 5: Run the full gate**

  Run: `vrg-container-run -- vrg-validate`
  Expected: PASS.

- [ ] **Step 6: Commit.**

### Task 4: the check, gate mode (vergil-tooling)

Spec §4.1, §4.3 (gate; what it judges; modes).

**Files:**

- Create: `src/vergil_tooling/lib/docs_nav.py`
- Create: `src/vergil_tooling/bin/vrg_docs_nav_check.py` (and its
  `pyproject.toml` entry point)
- Modify: `src/vergil_tooling/bin/vrg_validate.py:1289-1340`
  (`_run_docs_build` keeps the built site and passes it to the check)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (the command;
  the docs stage's new step)
- Test: `tests/vergil_tooling/test_docs_nav.py`,
  `tests/vergil_tooling/test_vrg_docs_nav_check.py`,
  `tests/vergil_tooling/test_vrg_validate.py`, fixture sites under
  `tests/vergil_tooling/data/docs_nav/`

**Interfaces:**

- Consumes: `doc_sites.org_docs`, `is_enforcing`, `hub_url` (Task 3).
- Produces, in `lib/docs_nav.py`:
  - `@dataclass(frozen=True) class Finding: rule: Literal["orphan",
    "home-map", "tab-cap", "hub-link", "unjudged"]; path: str; message:
    str`.
  - `check_site(site_dir: Path, *, org: str | None, repo: str) ->
    list[Finding]`: judges a built site directory.
  - `crawl(pages: Mapping[str, str], start: str) -> set[str]`: the pages
    reached from `start`, over in-site `href`s; `pages` maps a site-relative
    path to its HTML. Task 5 reuses it on fetched pages.
  - `STANDARD_URL = "https://vergil-project.github.io/docs/management/navigation-standard/"`,
    named in every finding's message.
- Produces: `vrg-docs-nav-check --site-dir <dir> [--report-only]
  [--org <org>] [--repo <name>]`, `--org` and `--repo` defaulting from the
  checkout's `origin`. Exit 0 with no findings, or with findings when
  report-only applies (`--report-only`, or `not is_enforcing(org)`), each
  printed as `WARNING: docs-nav: <rule>: <path>: <message>`; exit 1 with
  findings when enforcing, each printed as `ERROR: …`. An internal error
  (an exception, an unreadable argument) exits 2 in either mode, never 0.

- [ ] **Step 1: Write the failing tests**

  Each against a small built fixture site (built in the test with the dev
  image's mkdocs, or stored as HTML under `data/docs_nav/`):

  - `test_orphan_page_reported`: a page in the build but in neither `nav`
    nor any link is an `orphan` finding naming its path.
  - `test_linked_page_outside_nav_not_orphan`: a page only linked from
    another page is reached.
  - `test_home_map_tab_only_section_is_finding`: a top-level section
    reached only through the tab row gives a `home-map` finding naming the
    section.
  - `test_tab_cap_eight_entries_is_finding`: 8 top-level entries with
    `navigation.tabs` → `tab-cap`; 7 → none; 8 without tabs → none.
  - `test_hub_link_missing_on_member_site`: in an org with `has_hub`, a
    member site with no header anchor to `hub_url(org)` gives `hub-link`;
    the hub itself (`repo == "docs"`) and a site in an org without a hub
    give none.
  - `test_crawl_ignores_external_and_special_targets`: `#x`, `mailto:`,
    `https://<org>.github.io/other/`, and a `../` link that leaves the site
    are not followed and raise no error.
  - `test_crawl_never_reports_special_files_as_orphans`: `404.html`,
    `search/`, `sitemap.xml`, assets and `objects.inv` are never orphans.
  - `test_unreadable_site_is_unjudged`: a missing site dir or no
    `index.html` gives one `unjudged` finding, never an empty list.
  - `test_gate_unknown_org_reports_and_passes` (CLI): findings, `--org
    nope` → exit 0, `WARNING:` lines, and a line saying the org is not in
    the lists.
  - `test_gate_enforcing_org_fails` (CLI): with `doc_sites.ORGS` patched to
    mark the org enforcing, findings → exit 1, `ERROR:` lines.
  - `test_report_only_flag_overrides_enforcing` (CLI).
  - `test_internal_error_exits_2_in_report_only` (CLI): an exception inside
    the check exits 2 with `--report-only`, never 0.
  - `test_validate_docs_stage_runs_check_after_build` (vrg_validate): the
    stage builds, then calls `check_site` on the built directory; a finding
    in a report-only org leaves the stage passing with the warnings in its
    output; a failed build skips the check (the build failure already fails
    the stage).

- [ ] **Step 2: Run them to verify they fail**

  Run: `vrg-container-run -- pytest tests/vergil_tooling/test_docs_nav.py tests/vergil_tooling/test_vrg_docs_nav_check.py tests/vergil_tooling/test_vrg_validate.py -k "docs" -v`
  Expected: FAIL.

- [ ] **Step 3: Implement**

  Parse with `html.parser`. The start page is `index.html`. Navigation
  entries are ordinary anchors in the built HTML, so the crawl needs no
  theme knowledge beyond: the tab row is `nav.md-tabs`, and the home
  page's own content is `article.md-content__inner` (both MkDocs Material
  markup). The home-map rule takes the top-level sections from
  `nav.md-tabs` (or, with no tab row, the side panel's top level) and
  requires an anchor in the home page's content into each. The hub link is
  an anchor with class `vrg-hub-link` and `href` equal to `hub_url(org)`
  inside `header.md-header`, on every page. A directory URL resolves to its
  `index.html`.

- [ ] **Step 4: Run the full gate** — `vrg-container-run -- vrg-validate`,
  expected PASS. vergil-tooling's own site gives warnings only.

- [ ] **Step 5: Commit.**

### Task 5: the audit mode (vergil-tooling)

Spec §4.3 (as an audit), §4.4 step 3.

**Files:**

- Modify: `src/vergil_tooling/lib/docs_nav.py` (`fetch_published`,
  `audit_site`, `coverage`)
- Modify: `src/vergil_tooling/bin/vrg_docs_nav_check.py` (`--audit`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_docs_nav.py`,
  `tests/vergil_tooling/test_vrg_docs_nav_check.py`

**Interfaces:**

- Consumes: `crawl`, `check_site`, `Finding` (Task 4); `doc_sites`
  (Task 3); `vrg_validate`'s staging and strict build, factored into a
  callable `build_site(repo_root: Path) -> Path` that Task 4's stage also
  uses (and that carries Tasks 6a and 6b's staging once they land).
- Produces: `vrg-docs-nav-check --audit [--json]`, run inside one checkout
  through `vrg-container-run`, which mounts only that checkout: the
  checkout's strict-build findings and its published-site findings
  (labelled `build` and `published`). The published crawl is plain HTTPS
  to a public Pages site and needs no credentials.
- Produces: `vrg-docs-nav-check --coverage <org> [--json]`, run on the
  host, where `vrg-gh` has its credentials: the lists' coverage findings
  for the org.
- Produces: `vrg-docs-nav-check --merge <json> [<json> …]`: one report
  from the per-checkout and coverage results. Every mode exits 1 on any
  finding and 0 otherwise; an audit always reports, never enforces.
- Produces: `coverage(org: str, repos: Iterable[str], sites:
  Iterable[str]) -> list[Finding]` with rule `"coverage"`: a repo of the
  org not in the lists, and a repo with `docs/site/mkdocs.yml` the lists
  do not mark `docs_site`.

- [ ] **Step 1: Write the failing tests**

  - `test_audit_published_crawl_stays_under_dev`: with a fake fetcher, a
    versioned site is crawled from `site_url(org, repo, version="dev")`;
    links into `/latest/` or `/2.1/` are not followed, and pages there are
    not reported.
  - `test_audit_unversioned_site_crawls_root`.
  - `test_audit_published_generated_page_orphan`: a page present only in
    the published fetch (the hub's Activity tree) and reached from nowhere
    is reported under `published`.
  - `test_audit_build_failure_is_finding`: the checkout's build fails →
    an `unjudged` finding for it, exit 1.
  - `test_merge_keeps_every_site_and_coverage`: merging two checkouts'
    results and a coverage result reports every finding of each; a missing
    or unreadable input is an internal error (exit 2).
  - `test_audit_fetch_failure_is_unjudged`: an HTTP error on the published
    home page → `unjudged`, never a pass.
  - `test_coverage_reports_unlisted_repo_and_site`.
  - `test_audit_reports_every_finding`: two orphans and a tab-cap finding
    on one site are all reported in one run.

- [ ] **Step 2: Run them to verify they fail.**
- [ ] **Step 3: Implement.** Fetch with `urllib.request`, following
  redirects, within the site's URL prefix only, with a page cap of 5000
  that, when hit, is itself an `unjudged` finding. The repo list for
  `--coverage` comes from `vrg-gh repo list`, and whether a repo has
  `docs/site/mkdocs.yml` on `develop` from a `vrg-gh api` contents lookup.
- [ ] **Step 4: Run the full gate** — PASS.
- [ ] **Step 5: Commit.**

### Task 6a: the hub's generated Repositories page (vergil-tooling)

Spec §4.1 rule 6, §4.2.

**Files:**

- Modify: `src/vergil_tooling/lib/docs.py` (`render_repositories_page`;
  `stage_docs` takes the org and repo and writes it when the repo is its
  org's hub; `staging_outputs` lists it only then)
- Modify: `src/vergil_tooling/bin/vrg_docs_stage.py` (passes the org and
  repo)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (`_stage_docs_sources`,
  and so Task 5's `build_site`, passes the checked repo's org and repo)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`
- Test: `tests/vergil_tooling/test_docs.py`,
  `tests/vergil_tooling/test_vrg_docs_stage.py`,
  `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Consumes: `doc_sites.org_docs`, `site_url` (Task 3); the ignore entry
  (Task 1b).
- Produces: `render_repositories_page(org_docs: OrgDocs) -> str`; staging,
  from CI's `vrg-docs-stage` and from the local gate and the audit alike,
  writes `<docs-dir>/repositories.md` when `repo == "docs"` and the org
  `has_hub`, and nothing otherwise. `staging_outputs` lists the file only
  when staging writes it (as it lists the changelog only when there is
  one), so `unignored_staging_error` requires it to be gitignored on a
  hub and nowhere else.

- [ ] **Step 1: Write the failing tests**

  - `test_local_staging_writes_repositories_on_hub` (vrg_validate): the
    local gate's staging of a vergil-project `docs` fixture writes the same
    `repositories.md` as `vrg-docs-stage`, and its strict build has
    `repositories/index.html`.
  - `test_repositories_page_lists_every_repo`: one row per `Repo`, sorted
    by name; a `docs_site` repo links its bare site root, `site_url(org,
    name, version=None)`, which mike redirects to the default version (a
    repo that has never released has no `latest`); any other repo links
    `https://github.com/<org>/<name>`; a repo in `other_pages` links its
    Pages URL too.
  - `test_stage_writes_repositories_only_on_hub`: staged in `docs` of
    vergil-project → written; in vergil-tooling → not; in an unknown org →
    not, with a notice.
  - `test_repositories_page_unignored_is_error`.
  - `test_staging_outputs_without_new_entries_outside_hubs`: a repo in an
    unknown org, and a member repo, whose `.gitignore` lacks the new
    entries, stage without error.

- [ ] **Step 2: Run them to verify they fail.**
- [ ] **Step 3: Implement.** The page opens with a line saying it is
  generated from vergil-tooling's `lib/doc_sites.py`.
- [ ] **Step 4: Run the full gate** — PASS.
- [ ] **Step 5: Commit.**

### Task 6b: the member sites' header link to the hub (vergil-tooling)

Spec §4.1 rule 5, §6 (one mechanism on every member site).

**Files:**

- Create: `src/vergil_tooling/data/docs_overrides/partials/source.html`
- Modify: `src/vergil_tooling/lib/docs.py` (`stage_hub_link`;
  `staging_outputs` lists `docs/site/overrides.staged/` only when staging
  writes it: on a member site of an org with a hub)
- Modify: `src/vergil_tooling/bin/vrg_docs_stage.py`,
  `src/vergil_tooling/bin/vrg_docs_patch_nav.py` (the staged config)
- Modify: `src/vergil_tooling/bin/vrg_validate.py` (`_stage_docs_sources`,
  and so Task 5's `build_site`, stages the hub link too)
- Test: `tests/vergil_tooling/test_docs.py`,
  `tests/vergil_tooling/test_vrg_docs_patch_nav.py`,
  `tests/vergil_tooling/test_vrg_validate.py`

**Interfaces:**

- Consumes: `doc_sites.org_docs`, `hub_url` (Task 3); the ignore entry
  (Task 1b).
- Produces: on a member site of an org with a hub, from CI's staging and
  the local gate's and the audit's alike, the staged config sets
  `extra.vergil_hub: {url: <hub_url>, label: <org_docs.hub_label>}` and
  `theme.custom_dir: overrides.staged` (each merged into the config's
  existing `extra` and `theme` mappings), a directory staged at
  `docs/site/overrides.staged/` that renders an anchor `<a
  class="vrg-hub-link" href="{{ config.extra.vergil_hub.url }}">` with the
  label inside `header.md-header`. A site with its own `custom_dir` has it
  copied into the staged directory with the override added, unless it
  already overrides `partials/header.html` or `partials/source.html`: then
  staging
  adds nothing, and Task 4's `hub-link` rule reports the site if its own
  header lacks the anchor (a report-only finding until Task 13, never a
  staging error). Task 4's `hub-link` rule checks the result.

- [ ] **Step 1: Write the failing tests**

  - `test_hub_link_staged_on_member_site`: the staged config of a
    vergil-project member site carries `extra.vergil_hub`, with the label
    `Vergil project docs` from `hub_label`, and the `custom_dir`; a strict
    build's pages contain the anchor inside `header.md-header`.
  - `test_local_staging_stages_hub_link` (vrg_validate): the local gate's
    staging of a member fixture gives the same staged config and
    `overrides.staged/` as `vrg-docs-stage` and `vrg-docs-patch-nav`.
  - `test_hub_link_not_staged_on_hub_or_hubless_org`: nor listed in
    `staging_outputs`, so a repo in an unknown or hubless org whose
    `.gitignore` lacks `overrides.staged/` stages without error.
  - `test_hub_link_merges_existing_custom_dir`.
  - `test_hub_link_merges_existing_extra_and_theme`: on a fixture whose
    config already has `extra.version.provider: mike` and a `theme:` block
    with `features`, the staged config keeps both and adds
    `extra.vergil_hub` and `theme.custom_dir`, and has exactly one
    top-level `extra:` key and one `theme:` key. Staging merges into the
    existing mappings; it never appends a second top-level key.
  - `test_hub_link_existing_header_override_left_to_rule`: a site that
    already overrides `partials/header.html` or `partials/source.html`
    stages without error, and
    `check_site` gives a `hub-link` finding when its header lacks the
    anchor.
  - `test_built_member_site_passes_hub_link_rule`: Task 4's `check_site`
    gives no `hub-link` finding on the built fixture.

- [ ] **Step 2: Run them to verify they fail.**
- [ ] **Step 3: Implement.** Material is not pinned (the images install
  it unversioned), so copy as little of it as possible. Material's
  `partials/header.html` has no inner block, and `base.html`'s
  `{% block header %}` only includes it, so an override of that block
  places what it adds after `</header>`. Instead, override
  `partials/source.html`, which `partials/header.html` includes inside
  `nav.md-header__inner` whenever `repo_url` is set (every member site
  sets it). The override cannot include the stock partial by its own name
  (the template search path resolves that name to the override itself),
  so it is a copy of Material's 12-line `source.html` with the anchor
  added after it. Add a vergil-tooling test that compares the copy with
  the installed Material's `partials/source.html`, apart from the anchor,
  and fails on drift. A site with no `repo_url` gets no anchor, and Task
  4's `hub-link` rule reports it. The strict-build test above runs with the
  image's current Material.
- [ ] **Step 4: Run the full gate** — PASS.
- [ ] **Step 5: Commit.**

### Task 7: the site template to the standard (vergil-tooling)

Spec §2 (new repos start compliant), §4.4 step 1.

**Files:**

- Modify: `src/vergil_tooling/lib/repo_init.py:873-935`
  (`render_mkdocs_yml`, `render_docs_index`, and the getting-started page
  `step_docs_site` writes)
- Test: `tests/vergil_tooling/test_repo_init.py`

**Interfaces:**

- Consumes: `check_site` (Task 4); the hub-link staging (Task 6b).

- [ ] **Step 1: Write the failing test**

  `test_rendered_site_passes_nav_check`: render the template for a repo in
  vergil-project, stage it (staging always writes `releases/index.md`, and
  `changelog.md` when there is a changelog), build it strictly, and
  `check_site` returns no findings. The rendered `index.md` links every
  top-level `nav` entry from its own content.

- [ ] **Step 2: Run it to verify it fails** (the home page links nothing,
  and the staged release pages are orphans).
- [ ] **Step 3: Implement.** `render_mkdocs_yml`'s `nav` gains
  `Changelog: changelog.md` and a `Release Notes` section holding
  `releases/index.md`, as on vergil-tooling's and vergil-actions' sites,
  so `patch_nav` adds each release under it. `render_docs_index` writes a
  home page with a map section linking every top-level `nav` entry.
- [ ] **Step 4: Run the full gate** — PASS.
- [ ] **Step 5: Commit.**

### Task 8: the check in PR CI and the docs deploy (vergil-actions)

Spec §4.3 (gate in `ci-docs.yml`; on every docs deploy).

**Files:**

- Modify: `.github/workflows/ci-docs.yml` (a `Check docs navigation` step
  after `Build docs`: build to a known directory, then
  `vrg-docs-nav-check --site-dir <dir>`)
- Modify: `.github/workflows/cd-docs.yml` (after `Stage documentation
  sources`, before `Deploy docs`: a `Resolve mkdocs command` step as in
  `ci-docs.yml` (`uv run mkdocs` for Python repos, so the judged build
  uses the same Material as the deploy action's `uv run mike`), then a
  strict build with that command to a temporary directory and
  `vrg-docs-nav-check --site-dir <dir>`)
- Modify: the workflows' README and the smoke test for `cd-docs`
  (`smoke-cd-docs.yml`)

**Interfaces:**

- Consumes: `vrg-docs-nav-check` as released in Task 9.

- [ ] **Step 1: Add the steps.** `Build docs` gains `-d
  "$RUNNER_TEMP/site"`; the new step runs only when `present == 'true'`.
  Neither step passes `--report-only`: the mode comes from the released
  tooling and the org's lists, so a finding fails the deploy only where
  the org enforces (after Task 13). Neither step carries
  `continue-on-error` or `|| true`: a non-zero exit fails the job, so an
  internal error (exit 2) or a failed build always does. The deploy step
  runs before `Deploy docs`, so a failing site is not published. The
  `smoke-cd-docs` test pins this: a report-only finding passes, and a
  forced exit 2 fails the job.
- [ ] **Step 2: Validate** — `vrg-container-run -- vrg-validate`, PASS
  (actionlint included).
- [ ] **Step 3: Confirm in CI.** The PR's own docs job (and the
  `smoke-cd-docs` run) shows the `docs-nav` output.
- [ ] **Step 4: Commit.**

### Task 8b: the daily coverage check (vergil-actions)

Spec §4.1 rule 6 (the hub links every member site; generated from the
lists, so the lists must match the org).

**Files:**

- Create: `.github/workflows/ops-docs-coverage.yml`, modelled on
  `ops-github-config.yml`: on `workflow_call`, in the `automation`
  environment with the org App's token (verified with `vrg-whoami --mode`
  before use, as there), one `docs-coverage` job running
  `vrg-docs-nav-check --coverage "${{ github.repository_owner }}"`, and an
  `ops-report` job that opens or comments on the org's ops issue on a
  failure or cancel and records the first green run after one.
- Modify: the workflows' README; a smoke test of the layout, as
  `actions/ops/agent-audit/tests/agent-audit.test.sh` guards its workflow.

**Interfaces:**

- Consumes: `vrg-docs-nav-check --coverage` as released in Task 9 (exit 1
  on any coverage finding, 2 on an internal error; both fail the job).
- Produces: a reusable workflow each in-scope org's `.github/ops.yml`
  calls (Tasks 17a–17c).

- [ ] **Step 1: Write the workflow and its layout test.**
- [ ] **Step 2: Validate** — `vrg-container-run -- vrg-validate`, PASS
  (actionlint included).
- [ ] **Step 3: Commit.**

### Task 9: release vergil-tooling, report-only (.github, deployment)

`vrg-issue-create --kind deployment --blocked-by` 3, 4, 5, 6a, 6b, 7.

- **Human-gated precondition:** a vergil-tooling release on `v2.1`
  carrying Tasks 3–7, checked with `vrg-release-record check
  vergil-project/vergil-tooling#<T>` for each, exit 0.
- **Human-run steps:** `vrg-host upgrade` on the host, `vrg-vm update
  --all` for the VMs.
- **Acceptance:** `vrg-docs-nav-check --help` runs on the host and a VM;
  `vrg-container-run -- vrg-validate` in `vergil-project/docs` shows
  `WARNING: docs-nav:` lines (or none) and passes.

### Task 10: release vergil-actions (.github, deployment)

`--blocked-by` 8, 8b.

- **Human-gated precondition:** a vergil-actions `v2.1.x` tag carrying
  Tasks 8 and 8b (`vrg-release-record check
  vergil-project/vergil-actions#<T>` for each).
- **Agent-safe step:** none beyond the check; callers pick the tag up
  through `@v2.1`.
- **Acceptance:** a PR in any vergil-project repo with a docs site runs
  the `docs / docs` job with the `Check docs navigation` step, passing.

### Task 11: the audit (.github, validation)

Spec §4.4 steps 3–4. `--blocked-by` 2b, 9, 10.

- **Precondition:** Task 9's release is installed (`vrg-docs-nav-check
  --help` exits 0).
- **Procedure:**
  1. Clone or fetch every repo `doc_sites.ORGS` marks `docs_site` in the
     three in-scope orgs (the host's existing checkouts under the projects
     volume, `vrg-git fetch origin`), each checked out at a detached
     `origin/develop`.
  2. For each checkout, `cd` into it and run `vrg-container-run --
     vrg-docs-nav-check --audit --json`, writing the result to a file in
     the running session's scratch directory. `vrg-container-run` mounts
     only the checkout it runs in, so each site is audited in its own
     container, with its own environment.
  3. On the host, run `vrg-docs-nav-check --coverage <org> --json` for each
     of the three orgs, into the same directory.
  4. Run `vrg-docs-nav-check --merge <files…>`, and post the merged report
     as a comment (as a fenced block, or an attached file if long).
  5. A `coverage` finding (a repo or site the lists miss) is fixed by a
     vergil-tooling task filed now, keyed `Plan task: 12 coverage`, before
     any site's fix is judged. It is its own task and PR, separate from any
     fix to vergil-tooling's own site (step 6), which keeps the
     `Plan task: 12 vergil-project/vergil-tooling` key. Each docs site a
     `coverage` finding names is then audited too: fetch its repo as in
     step 1, run step 2 on it, and merge its result into the report
     (step 4), so its findings reach the review in step 6 with the rest.
     The released lists do not name it yet, so a finding that depends on
     them (its org mode, its hub link) is judged again by 12z.
  6. Stop for the human's review of the findings. For each site the human
     confirms short of the standard, file one fix task in its repo, keyed
     `Plan task: 12 <org>/<repo>`, its body listing that site's findings,
     the standard's page, layout B if the site needs more than 7 tabs, and
     the removal of any remaining "standard tooling" naming (spec §4.5).
     For a fixture site, the fix may ride that fixture's run (Task 16): if
     its change task is still open, its body gains the fix and the key, and
     no separate task is filed. In logical-minds-foundry and
     mnemosys-project, file under the org's proxy (`vrg-epic-create
     --proxy-for vergil-project/.github#1668 --repo <org>/<repo>`).
  7. If any fix task filed in steps 5–6 is in vergil-tooling, file one
     deployment task in `.github`, keyed `Plan task: 12 release`, blocked
     by every vergil-tooling fix task: the human-gated precondition is a
     vergil-tooling release carrying them (`vrg-release-record check
     vergil-project/vergil-tooling#<T>` for each, exit 0); the human-run
     steps are `vrg-host upgrade` on the host and `vrg-vm update --all`;
     the agent-safe step re-runs the hub's `cd-docs` on `develop`
     (`workflow_dispatch`), so its published Repositories page is staged
     from the released lists. Its acceptance: `vrg-docs-nav-check
     --coverage` for each org runs from the installed release, and the
     hub's `cd-docs` run succeeded after the release.
  8. Add a `Blocked-by:` line for each fix task, and for the `12 release`
     task if one was filed, to Task 12z's body.
- **Acceptance:** every in-scope docs site has its findings recorded, and
  every short site has a keyed fix task that 12z is blocked by.

### Task 12-* (filed by Task 11): fix one site

Each: restructure the site to the standard (rules 1–5 of spec §4.1, tab
grouping and wording the builder's), clear every orphan, remove "standard
tooling" naming, and pass `vrg-container-run -- vrg-validate` with no
`docs-nav` warning. The PR lands on `develop`. A member site's fix needs
no release (the bar is `dev`). A vergil-tooling fix does: the audit, the
gate and the hub's Repositories page all run from the installed release,
so Task 11 files the `12 release` deployment task after it (step 7).

### Task 12z: the re-audit (.github, validation)

Spec §4.4 step 6. `--blocked-by` 11 and every 12-* task.

- **Precondition:** listing the epic's and its proxies' children
  (`vrg-epic-audit --epic vergil-project/.github#1668`), no `Plan task: 12
  …` task is open; and each fixed site's `dev` version was deployed after
  its fix merged (the site's last `cd-docs` run on `develop` succeeded
  after the merge).
- **Procedure:** rerun Task 11's procedure steps 1–4, after a fresh fetch,
  over the repos the installed release's `doc_sites.ORGS` marks
  `docs_site` (after any `12 release`, the updated lists), not only the
  repos Task 11 audited.
- **Acceptance:** exit 0: no finding on any site's build or published
  `dev` version, and no coverage finding.

### Task 13: enforce in the in-scope orgs (vergil-tooling)

Spec §4.3 (modes), §4.4 step 6.

**Files:**

- Modify: `src/vergil_tooling/lib/doc_sites.py` (`enforcing=True` for
  vergil-project, logical-minds-foundry, mnemosys-project)
- Modify: `tests/vergil_tooling/test_doc_sites.py`
  (`test_no_org_enforces_yet` becomes `test_only_in_scope_orgs_enforce`:
  exactly those three, and mq-rest-admin-project not)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (the mode, and
  the recovery: a patch release setting the orgs back to report-only)

- [ ] **Step 1: Update the test; run it; expect FAIL.**
- [ ] **Step 2: Flip the three entries; run the full gate; expect PASS**
  (vergil-tooling's own site has no findings after its Task 12 fix).
- [ ] **Step 3: Commit.**

### Task 14: release vergil-tooling, enforcing (.github, deployment)

`--blocked-by` 13. As Task 9, for Task 13
(`vrg-release-record check vergil-project/vergil-tooling#<13>`).

### Task 15: the enforcing gate holds (.github, validation)

`--blocked-by` 14, 17a, 17b, 17c.

- **Precondition:** Task 14's release installed; `vrg-docs-nav-check`
  run in a vergil-project checkout prints `ERROR:` (not `WARNING:`) for a
  planted orphan in a scratch copy.
- **Procedure:** in every docs site's repo of the three orgs, the latest
  `docs / docs` job on `develop` after the release succeeded with the
  check step run (or, where none ran since, trigger one by a
  `workflow_dispatch` or the next PR). In each org's `.github`, the latest
  `Ops` run's `docs-coverage` job succeeded (or trigger one by
  `workflow_dispatch`).
- **Acceptance:** every one passed. A failure is recovered as spec §4.4
  says: a patch release returning the org to report-only, a fix task for
  the site, and this task stays open. A failed `docs-coverage` job is
  recovered by a `Plan task: 12 coverage`-style lists fix and its release,
  and this task stays open.

### Tasks 17a, 17b, 17c: wire the daily coverage check (each org's .github)

`--blocked-by` 10, 12z. 17a in `vergil-project/.github`, under this epic;
17b in `logical-minds-foundry/.github` and 17c in
`mnemosys-project/.github`, each under its org's proxy
(`vrg-epic-create --proxy-for vergil-project/.github#1668 --repo
<org>/.github`).

**Files:**

- Modify: `.github/workflows/ops.yml`: a `docs-coverage` job calling
  `vergil-project/vergil-actions/.github/workflows/ops-docs-coverage.yml@v2.1`
  with `secrets: inherit`, after the existing jobs and run whatever their
  result (`if: ${{ !cancelled() }}`), as `agent-audit` is in
  vergil-project's.

- [ ] **Step 1: Add the job.**
- [ ] **Step 2: Validate** — `vrg-container-run -- vrg-validate`, PASS.
- [ ] **Step 3: Confirm** after merge: a `workflow_dispatch` of `Ops`
  runs `docs-coverage` green.
- [ ] **Step 4: Commit.**
