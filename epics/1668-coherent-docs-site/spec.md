# coherent-docs-site: one navigable documentation system — design

- **Epic:** vergil-project/.github#1668
- **Planning issue:** vergil-project/.github#1666 (review history is in its comments)
- **Status:** approved

## 1. Intent

Vergil's documentation is published as a set of separate MkDocs sites: the
project hub (the `docs` repo, at <https://vergil-project.github.io/docs/>) and
one site per member repo (vergil-tooling, vergil-claude-plugin,
vergil-actions, vergil-containers, vergil-vm and the language fixtures),
beside the `packages` repo's package index. The intent, set by epic
[vergil-project/.github#60](https://github.com/vergil-project/.github/issues/60),
was one documentation system: the hub documents the Vergil project as a whole
and links into each repo's own site where a topic belongs to that repo.

That system has drifted. Each site is internally connected, but the sites
are not tied together, and the hub's own navigation does not lead a reader
to what it holds:

- The hub's home page never mentions Management, Hoshin kanri or the
  Logbook. It leaves the left panel empty, since Home has no sub-pages, so a
  reader sees only its prose and its on-page contents.
- The tab row appears only on wide windows. Below that width the only way in
  is the small ☰ menu, which is easy to miss.
- Hoshin kanri (the method) lives under Management while its records live
  under Logbook. The two sections now link to each other, but a reader
  reaches either only through the tab row or the ☰ menu, never from the
  home page.
- The hub does not link to any member site except for one deep link into
  vergil-containers. No member site links back to the hub.

The hoshin review records, which the human reads every day, were hard to
find as a result. The docs are mostly an archival record, and they are also
guidance that agents are sent into, so they must be accurate and reachable:
documentation that cannot be reached from its entry point is, in effect,
unpublished.

This epic sets one navigation standard for every Vergil docs site, builds a
check that enforces it mechanically, audits every docs site of the active
orgs with that check, brings every site up to the standard, and only then
switches the check to an enforcing gate.

The standard is **intra-org**: it makes each org's documentation coherent
within itself. A site in another org may link to vergil-project's docs as a
reference, but no rule requires or forbids such a link.

### Why now

The logbook became a daily working document with the hoshin cadence, and
the human could not find it from the hub. The fix is mostly mechanical, and
doing it once, fleet-wide, while the number of sites is still small, costs
far less than letting each site drift further.

## 2. Goals, non-goals, anti-goals

### Goals

- Every page of every Vergil docs site is reachable from that site's home
  page by in-page links and the site's own navigation (tabs, sections, the
  side panel). A page that is not is an **orphan**, and orphans are defects.
- Every site follows one navigation standard (§4.1), so the sites read as
  one system.
- The hub links to every member site of its org, and every member site
  links back to its org's hub.
- Every rule of the standard that a machine can judge is checked
  mechanically, in `vrg-validate` and in PR CI, in every repo with a docs
  site, and enforced in the in-scope orgs (§4.4). The rest are reviewed
  (§4.1 says which).
- New repos start compliant: the site template that `vrg-github-repo-init`
  produces passes the check.

### Non-goals

- Merging the sites into one MkDocs build. Each member site keeps its own
  build, its own release-versioned docs (mike) and its own publishing.
- Creating a docs hub for an org that has none (mq-rest-admin-project,
  diogenes-project, vergils-nemesis). Such an org may get a hub later; that
  is a separate decision.
- Fixing the sites of the inactive orgs. Only vergil-project,
  logical-minds-foundry and mnemosys-project are in scope (§4.4); each
  inactive org with docs sites gets a follow-up issue instead.
- The versions readers see by default. The bar is the `dev` version:
  `latest` picks up the fixes whenever each repo is next released, and
  older mike versions are left as they are.
- Rewriting the content of the docs beyond what navigation needs. Pages are
  moved, grouped, indexed and linked; their substance is out of scope except
  for the stale details named in §4.5.
- Cross-site search.

### Anti-goals

- **A sudden CI break.** The enforcing check must never reach a repo whose
  docs it would fail. That is why the rollout (§4.4) fixes every in-scope
  site before the check enforces, and enforces only in the in-scope orgs.
- **A check that passes on what cannot actually be reached.** The gate
  judges the strict PR build, not the source files. Pages generated only at
  deploy time (the hub's Activity tree and Roadmap, the mike version
  selector) are not in that build, so the check also judges every docs
  deploy, and the audit judges the published `dev` version (§4.3).
- **A silent skip.** A site the check cannot judge (a build that fails, a
  page it cannot read) is a failure, never a pass.

## 3. Approaches

### Navigation layout

- **A. A short tab row, with the home page as a map (chosen).** The top tab
  row names each section, and the home page shows every section as a card
  listing its key pages, so the home page alone reaches everything even when
  the tabs are hidden on a narrow window. It works well while the number of
  sections is small.
- **B. No tabs; the whole site tree always on the left.** Everything is
  visible on every page, like a book's table of contents. It suits a site
  with many sections, and costs a long side panel.

The human chose A as the standard, and B as the layout a site moves to when
it has too many sections for A (§4.1).

### Site structure

- **A hub linked to separate member sites (chosen).** Keeps each repo's
  versioned docs and publishing, costs little, and carries to every org,
  matching epic #60's design.
- **One merged site.** One search and one tree, but it loses or rebuilds
  each repo's versioned docs, and every docs change would republish the
  whole fleet. Rejected.

### The check

- **Crawl the built site (chosen).** After the strict MkDocs build, follow
  every link and navigation entry from the built home page, and fail on any
  built page never reached. It sees what a reader sees.
- **Read `mkdocs.yml` and the markdown source.** Misses what the build
  actually produces (theme navigation, generated index pages, rewritten
  links). Rejected.
- **An off-the-shelf link checker.** Finds dead links, but not orphans.
  Rejected.

## 4. Design

### 4.1 The navigation standard

A **docs site** is a published MkDocs site, built from a repo's
`docs/site/mkdocs.yml`. A repo that publishes something else on Pages is
not one: `packages` publishes an apt/dnf package index, which the hub links
from its Platform section, and the standard, the check and the audit do
not apply to it.

Every Vergil docs site follows these rules:

1. **A short top tab row.** The site's top-level sections appear as the
   MkDocs Material tab row (`navigation.tabs`), with at most **7** top-level
   entries, Home included. A site that needs more switches to layout B: no
   tab row, with the full tree shown in the side panel.
2. **The home page is a map.** The home page's own content links to every
   top-level section, with each section's key pages, so the home page alone
   reaches every section without the tab row or the ☰ menu.
3. **Every section of more than one page has an index page that lists its
   pages** (`navigation.indexes`), and the side panel lists the same pages
   within the section.
4. **Related sections link to each other**, where one describes what the
   other records or uses (on the hub: Hoshin kanri and the Logbook).
5. **Every member site links back to its org's hub** with one standard link
   in the site header, labelled for the project's docs (for example "Vergil
   project docs"). The rule applies only in an org that has a `docs` repo;
   in an org without one it is skipped.
6. **The hub links to every member site of its org**, from a Repositories
   page that lists every repo of the org: a repo with a docs site links to
   that site's home, and any other repo links to the repo on GitHub. The
   page is generated from the org's lists (§4.3) on every build of the hub,
   so it cannot drift from them, and it replaces the repository table on
   the hub's home page.

The check (§4.3) enforces rules 1, 2 and 5, and the orphan rule of §2. Rule
6 holds by construction, since the page is generated. Rules 3 and 4 are
reviewed, not checked: whether a section's index lists its pages well, and
which sections are related, are judgments.

The standard is written up as a page on the hub (§4.5) and is the reference
the check's messages point to.

### 4.2 The hub's structure

The hub (`vergil-project/docs`) moves to six top-level sections:

| Tab | Holds |
| --- | --- |
| Home | the map of the site (§4.1 rule 2) |
| Logbook | the logbook index, Hoshin, Fleet and the entries by level |
| Management | epics and tasks, intake and planning, workflow, tools and skills, Hoshin kanri, Roadmap, Activity |
| Platform | supported languages, continuous integration, the package repository |
| Repositories | the generated page of every repo of the org (§4.1 rule 6) |
| Getting started | the getting-started guide |

The Logbook comes first after Home because it is the page the human reads
every day. The home page's Logbook card links to the logbook's level
indexes (daily first). The hoshin review (`/vergil:hoshin`) already keeps
those indexes current, newest entry first, so the card never goes stale and
the skill needs no change.

### 4.3 The check

The check lives in `vergil-tooling` and runs in two ways:

- **As a gate.** In `vrg-validate`'s `docs` stage, right after the strict
  MkDocs build that stage already runs, so it runs in every repo with a docs
  site when validating locally. PR CI's `docs / docs` job (`vergil-actions`'
  `ci-docs.yml`) runs its own strict build and does not call `vrg-validate`,
  so that workflow also runs the check after its build, and the change ships
  in a `vergil-actions` release that callers pick up through their `v2.1`
  pin. A repo with no docs site passes as today.
- **As an audit.** A standalone mode pointed at a repo checkout, which
  reports every finding across the site in one pass rather than stopping at
  the first, so the fleet audit (§4.4) can list all the work at once. It
  runs from released `vergil-tooling`: the check is released in report-only
  mode (below) before the audit. The audit judges both the site's strict
  build and its published `dev` version (the published site itself, for a
  site mike does not version), fetched from its Pages URL, so pages that
  only the deploy generates (the hub's Activity tree and Roadmap) are
  judged too. It also reports every repo of the org missing from the org's
  lists, and every docs site the lists do not name. It also reports the
  docs-link check's findings for every repo of the in-scope orgs, a repo
  with no docs site included, since the switch to enforcing (§4.4) makes
  them fatal too.
- **On every docs deploy.** `vergil-actions`' docs deploy runs the check on
  the deploy build, after its pre-deploy command, so the pages generated at
  deploy time are judged on every deploy, not only during the audit. It
  runs before the publish, in the same mode as the gate: where the org
  enforces, a finding fails the deploy and the site is not published.
  Report-only lasts only until the switch to enforcing (§4.4), and its
  reader is the audit, not the deploy log. A failure of the check itself
  (an internal error, or a build of the deploy tree that fails) fails the
  deploy in either mode.
- **Daily, for the lists' coverage.** Each in-scope org's daily operations
  workflow runs the audit's coverage report for that org and reports a
  failure to the org's operations queue. The hub's Repositories page is
  generated from the lists (§4.1 rule 6), so a repo added to the org after
  the epic would otherwise be missing from the hub with nothing to say so.

It judges the built site:

- **Orphans.** Starting from the built home page, follow every in-site link
  and navigation entry. Every built page that is never reached is reported
  by its path.
- **The home-page map.** Every top-level section is linked from the home
  page's own content, not only from the tab row.
- **The tab cap.** A site using the tab row has at most 7 top-level entries.
- **The hub link.** A member site in an org with a hub carries the standard
  header link to it.

**The org's lists.** What the check and the Repositories page need to know
about an org is held in fixed lists in `vergil-tooling`, keyed by org, so
the gate needs no network (only the audit's crawl of a published site
does): whether the org has a hub, every repo of the org and whether it
publishes a docs site, and whether the check enforces there. This is a new
data model. Today's `lib/doc_links.py` knows only vergil-project, with its
Pages owner hard-coded (`PAGES_OWNER`). The Pages owner is instead derived
from the org of the repo being checked, so the existing docs-link check
works in every org too. Links to vergil-project's Pages host stay judged and
enforced from every org, as today. A repo in another listed org also has
its links to its own org's Pages host judged, and those findings follow the
org's mode (below): warnings until the org enforces, then failures. The
lists are filled when the check is built
(§4.4 step 1), from an inventory of every repo of the in-scope orgs, each
`docs/site/mkdocs.yml` included.

The check has two modes, set in `vergil-tooling` itself:

- **Report-only.** The gate reports every finding, and every site it cannot
  judge, as a visible warning in the stage's output, and does not fail the
  stage.
- **Enforcing.** The gate fails the stage on any finding, and on any site it
  cannot judge.

The check is first released in report-only mode. Switching it to enforcing
is one small change, made only once every in-scope site passes (§4.4), and
it enforces only in the orgs the lists mark as enforcing: the in-scope
orgs. Every other org stays report-only, so a repo in an inactive org never
meets a failing gate.

### 4.4 The rollout

The check enforces only once every site passes it, so no repo meets a
failing docs gate. `vergil-tooling` releases from `develop` often, and every
repo pins only its minor line (`vergil = "v2.1"`), so any release reaches
the whole fleet on its next CI run. The check is therefore never merged in a
form that fails a stage until the sites are fixed:

1. **Build the check** in `vergil-tooling` in report-only mode (gate, audit
   and deploy, §4.3), with the org's lists and the Repositories page
   generator, and bring the `vrg-github-repo-init` site template up to the
   standard, so a new repo's site passes it.
2. **Release it, report-only,** and release the `vergil-actions` change
   that runs it in PR CI's `docs / docs` job and in the docs deploy
   (§4.3). The releases reach
   every repo and fail none. The CI job runs whichever mode the released
   `vergil-tooling` sets, so switching to enforcing (step 6) needs no
   second `vergil-actions` release.
3. **Audit the in-scope orgs** with the released check, against every docs
   site in them, and the docs-link check against every repo in them, and
   record the findings per repo.
4. **Review the audit.** The audit is a checkpoint for the human, who
   confirms the fix tasks it leads to. Every in-scope site is fixed in this
   epic.
5. **Fix every site**, one task per site: restructure it to the standard
   and clear its orphans. A repo with docs-link findings gets them cleared
   in the same task, or in a task of its own when it has no docs site. A
   repo in another org is fixed by a task in that org.
6. **Re-audit**, and once every in-scope site passes, **switch the check to
   enforcing** for the in-scope orgs in one small `vergil-tooling` change,
   and release it. The re-audit's bar is the strict build and the published
   `dev` version; no member-repo release is needed first.

If a site still breaks once the check enforces, the recovery is a patch
release that returns the check to report-only, and that site is fixed
before the check enforces again.

**Scope.** The epic covers the three orgs under active development:
vergil-project, logical-minds-foundry and mnemosys-project. For each
inactive org with docs sites, the epic files one follow-up issue in that
org, placed in its deferred queue with "the org returns to active
development" as its revival condition: bringing its sites up to the
standard and marking the org as enforcing in the lists.

The fleet as surveyed when this spec was written (the audit confirms the
exact set of sites):

| Org | Status | In scope | Hub | Docs sites (approximate) |
| --- | --- | --- | --- | --- |
| vergil-project | active | yes | yes | the hub and the 11 mike-versioned member sites |
| logical-minds-foundry | active | yes | yes | the hub and about 2 member sites |
| mnemosys-project | active | yes | yes | about 1 |
| mq-rest-admin-project | inactive | no (follow-up issue) | no | about 6 |
| diogenes-project | inactive | no | no | none |
| vergils-nemesis | inactive | no | no | none |

### 4.5 Documentation and stale details

- The hub gains a page describing the navigation standard (§4.1), which
  `vergil-tooling`'s docs for `vrg-validate` and the check link to.
- `vergil-tooling`'s docs describe the check and its audit mode.
- A stale detail is fixed in passing: the vergil-claude-plugin site's own
  title, `site_name: Standard Tooling Plugin` in its `docs/site/mkdocs.yml`,
  becomes `Vergil Claude plugin`, whatever the audit finds. (The generated
  Repositories page lists the TypeScript fixtures, which the hub's old
  repository table missed.)
- Each site's fix task also removes any remaining "standard tooling" naming
  in that site's docs.

## 5. Delivery outline

1. The check, in `vergil-tooling`, in report-only mode: gate, audit and
   deploy runs, the org's lists with the derived Pages owner, the
   Repositories page generator, and the `vrg-github-repo-init` site
   template brought to the standard; then its release. Alongside it, in
   `vergil-actions`, the change that runs the check in PR CI's docs job and
   in the docs deploy, and its release.
2. The audit: an operational task that runs the released check against
   every site of the in-scope orgs, and the docs-link check against every
   repo of them, and records the findings, then the human's review of
   them.
3. The hub restructure (§4.2), the standard's page and the stale details.
   It depends on neither item 1 nor item 2 and lands first, so the hub is
   navigable without waiting for the check. The generated Repositories page
   replaces the home page's table once item 1's generator is released. The
   audit judges the hub like any other site, and any gap it finds there is
   fixed in item 4.
4. One fix task per site the audit finds short of the standard (the hub
   included), and per repo with docs-link findings, in each repo, under a
   proxy epic for each other org.
5. A re-audit, then the change that switches the check to enforcing in the
   in-scope orgs, its `vergil-tooling` release, and a validation that the
   enforcing gate passes across them. Alongside, a reusable daily coverage
   workflow in `vergil-actions`, wired into each in-scope org's operations
   workflow (§4.3).
6. One follow-up issue in each inactive org with docs sites
   (mq-rest-admin-project), in that org's deferred queue (§4.4).

## 6. Left to the builder

- How the crawl reads the built site (HTML parsing, the theme's navigation
  markup) and how it treats generated pages, redirects and the mike version
  selector.
- How the header link to the hub is added to each site (theme override,
  `extra` configuration or another MkDocs mechanism), as long as it is the
  same on every member site.
- The exact wording of the standard's page and of each site's home-page map.
- How the Repositories page is generated before each build of the hub (a
  `vergil-tooling` command run in staging, as `vrg-roadmap` is, or another
  mechanism), as long as the strict PR build and the deploy build both
  carry it.
- How the check's findings are formatted, and the name of its audit mode.
- The grouping of each member site's sections into at most 7 tabs.
