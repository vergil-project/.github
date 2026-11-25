# coherent-docs-site: one navigable documentation system — design

- **Epic:** vergil-project/.github#1668
- **Planning issue:** vergil-project/.github#1666 (review history is in its comments)
- **Status:** approved

## 1. Intent

Vergil's documentation is published as a set of separate MkDocs sites: the
project hub (the `docs` repo, at <https://vergil-project.github.io/docs/>) and
one site per member repo (vergil-tooling, vergil-claude-plugin,
vergil-actions, vergil-containers, vergil-vm, packages and the language
fixtures). The intent, set by epic
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
- Management, which describes the process, does not link to the Logbook,
  which records it. Hoshin kanri (the method) lives under Management while
  its records live under Logbook, and neither points at the other.
- The hub does not link to any member site except for one deep link into
  vergil-containers. No member site links back to the hub.

The hoshin review records, which the human reads every day, were hard to
find as a result. Documentation a reader cannot reach from its entry point
is, in effect, unpublished.

This epic sets one navigation standard for every Vergil docs site, builds a
check that enforces it mechanically, audits every docs site in the fleet with
that check, brings every site up to the standard, and only then releases the
check as an enforcing gate.

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
- The standard is checked mechanically, in `vrg-validate` and in PR CI, in
  every repo with a docs site, never by inspection.
- New repos start compliant: the site template that `vrg-repo-init`
  produces passes the check.

### Non-goals

- Merging the sites into one MkDocs build. Each member site keeps its own
  build, its own release-versioned docs (mike) and its own publishing.
- Creating a docs hub for an org that has none (mq-rest-admin-project,
  diogenes-project, vergils-nemesis). Such an org may get a hub later; that
  is a separate decision.
- Rewriting the content of the docs beyond what navigation needs. Pages are
  moved, grouped, indexed and linked; their substance is out of scope except
  for the stale details named in §4.6.
- Cross-site search.

### Anti-goals

- **A sudden CI break.** The enforcing check must never reach a repo whose
  docs it would fail. That is why the rollout (§4.5) fixes every site before
  the check is released.
- **A check that passes on what a reader cannot actually reach.** The check
  judges the built site, as a reader meets it, not the source files.
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

Every Vergil docs site follows these rules:

1. **A short top tab row.** The site's top-level sections appear as the
   MkDocs Material tab row (`navigation.tabs`), with at most **7** top-level
   entries, Home included. A site that needs more switches to layout B: no
   tab row, with the full tree shown in the side panel.
2. **The home page is a map.** The home page's own content links to every
   top-level section, with each section's key pages, so the home page alone
   reaches every section without the tab row or the ☰ menu.
3. **Every section has an index page that lists its pages**
   (`navigation.indexes`), and the side panel lists the same pages within
   the section.
4. **Related sections link to each other**, where one describes what the
   other records or uses (on the hub: Hoshin kanri and the Logbook).
5. **Every member site links back to its org's hub** with one standard link
   in the site header, labelled for the project's docs (for example "Vergil
   project docs"). The rule applies only in an org that has a `docs` repo;
   in an org without one it is skipped.
6. **The hub links to every member site of its org**, from a Repositories
   section, one entry per published site, each linking to that site's home.

The standard is written up as a page on the hub (§4.6) and is the reference
the check's messages point to.

### 4.2 The hub's structure

The hub (`vergil-project/docs`) moves to six top-level sections:

| Tab | Holds |
| --- | --- |
| Home | the map of the site (§4.1 rule 2) |
| Logbook | the logbook index, Hoshin, Fleet and the entries by level |
| Management | epics and tasks, intake and planning, workflow, tools and skills, Hoshin kanri, Roadmap, Activity |
| Platform | supported languages, continuous integration, the package repository |
| Repositories | one entry per member site (§4.1 rule 6) |
| Getting started | the getting-started guide |

The Logbook comes first after Home because it is the page the human reads
every day. The home page's Logbook card links to the latest logbook entries
(§4.4).

### 4.3 The check

The check lives in `vergil-tooling` and runs in two ways:

- **As a gate.** In `vrg-validate`'s `docs` stage, right after the strict
  MkDocs build that stage already runs (the same build as PR CI's
  `docs / docs` check), so it runs in every repo with a docs site, locally
  and in CI, with no new workflow. A repo with no docs site passes as today.
- **As an audit.** A standalone mode pointed at a repo checkout, which
  reports every finding across the site in one pass rather than stopping at
  the first, so the fleet audit (§4.5) can list all the work at once. It
  must run from an unreleased checkout of `vergil-tooling` (through `uv`),
  since the audit runs before the release.

It judges the built site:

- **Orphans.** Starting from the built home page, follow every in-site link
  and navigation entry. Every built page that is never reached is reported
  by its path.
- **The home-page map.** Every top-level section is linked from the home
  page's own content, not only from the tab row.
- **The tab cap.** A site using the tab row has at most 7 top-level entries.
- **The hub link.** A member site in an org with a hub carries the standard
  header link to it.

Whether an org has a hub is read from a fixed list held in
`vergil-tooling`, as its existing docs-link check already does for the
published sites (`lib/doc_links.py`), so the check needs no network. The
check fails the stage on any finding, and on any site it cannot judge.

### 4.4 The latest logbook entries on the home page

The hoshin review (`/vergil:hoshin`) already updates the logbook's index
pages when it writes an entry. It also updates the lines on the hub's home
page that point at the latest entries, in the same docs PR, so the home page
always links to the current day.

### 4.5 The rollout

The check is released only once every site passes it, so no repo meets a
failing docs gate:

1. **Build the check** in `vergil-tooling` (gate and audit modes, §4.3), and
   bring the `vrg-repo-init` site template up to the standard, so a new
   repo's site passes it.
2. **Audit the fleet.** Run the unreleased check against every docs site in
   every org of the fleet, and record the findings per site.
3. **Decide.** The audit is a checkpoint for the human. If the fix work is
   as mechanical as expected, every site is fixed in this epic. If it is
   very large, the human moves part of it to a follow-on epic.
4. **Fix every site**, one task per site: restructure it to the standard
   and clear its orphans. A site in another org is fixed by a task in that
   org.
5. **Re-audit**, and once every site passes, **release** `vergil-tooling`
   and roll it out. The check fails from its first release, everywhere.

If a site still breaks after the release, the rollout pauses and that site
is fixed before going on.

The fleet as surveyed when this spec was written (the audit confirms the
exact set of sites):

| Org | Status | Hub | Sites (approximate) |
| --- | --- | --- | --- |
| vergil-project | active | yes | the hub, packages, and the 11 mike-versioned member sites |
| logical-minds-foundry | active | yes | about 3 |
| mnemosys-project | active | yes | about 1 |
| mq-rest-admin-project | inactive | no | about 5 |
| diogenes-project | inactive | no | none |
| vergils-nemesis | inactive | no | none |

### 4.6 Documentation and stale details

- The hub gains a page describing the navigation standard (§4.1), which
  `vergil-tooling`'s docs for `vrg-validate` and the check link to.
- `vergil-tooling`'s docs describe the check and its audit mode.
- Two stale details on the hub are fixed in passing: the repository table
  lists the TypeScript fixtures beside the Python and Rust ones, and the
  vergil-claude-plugin site loses its old "Standard Tooling Plugin" title.
- Each site's fix task also removes any remaining "standard tooling" naming
  in that site's docs.

## 5. Delivery outline

1. The check, in `vergil-tooling`: gate and audit modes, the hub list, and
   the `vrg-repo-init` site template brought to the standard.
2. The fleet audit: an operational task that runs the unreleased check
   against every site and records the findings, then the human's decision on
   scope.
3. The hub restructure (§4.2), the standard's page, the stale details, and
   the hoshin skill's update of the home page's latest entries.
4. One fix task per member site, in each site's own repo, under a proxy
   epic for each other org.
5. A re-audit, then the `vergil-tooling` release and its rollout, with a
   validation that the released gate passes across the fleet.

## 6. Left to the builder

- How the crawl reads the built site (HTML parsing, the theme's navigation
  markup) and how it treats generated pages, redirects and the mike version
  selector.
- How the header link to the hub is added to each site (theme override,
  `extra` configuration or another MkDocs mechanism), as long as it is the
  same on every member site.
- The exact wording of the standard's page and of each site's home-page map.
- How the check's findings are formatted, and the name of its audit mode.
- The grouping of each member site's sections into at most 7 tabs.
