# Language version lifecycle: a rolling, upstream-derived support window

Epic: `vergil-project/.github#407`. Documentation task: `vergil-project/.github#408`.
Folds in: `vergil-project/.github#383` (language and toolchain lifecycle
monitoring) and `vergil-project/.github#403` (retire Go 1.25 and other
end-of-life versions). Related but **not** folded in:
`vergil-project/.github#384` (per-language library dependency updaters) and
`vergil-project/.github#406` (choosing a language per problem).

## 1. Summary

Vergil supports several languages, and it chooses and maintains each language's
supported versions **by hand**. An audit on 2026-10-17 showed most of that matrix
had fallen behind upstream, and the Go 1.25 retirement had to be done reactively
when its CVEs blocked the image gate.

This epic replaces hand maintenance with a **rolling support window derived from
upstream metadata**:

- Each kept language has a **written support policy based on its upstream
  community's own rules**.
- **One version manifest** in `vergil-containers` records the window. Every
  other version list is generated from it.
- A **scheduled check** compares the manifest with upstream. When a version goes
  GA or reaches end of life (EOL), it **opens the PR** that adds or drops it.
  Shared machinery calls one **adapter per language** for that language's quirks.
- **Consumers follow the window by default.**
  - `[ci].versions` becomes an optional **pin**.
  - The existing `repository-type` field (redefined in this epic) marks whether a repo provides libraries:
    - **Library:** tests across the whole window.
    - **App:** builds on the newest version.
  - A pin naming a version that has been dropped is a **fatal error**.
- The deployed Python runtime (`vergil-python`) follows the window, one
  series at a time (§7).

**Python and Go are built together** in this epic. Two languages at once keep the
generic machinery honest. Ruby is dropped. The other kept languages get written
policies and a ranked queue for full support.

## 2. The problem in detail

### 2.1 How far behind the matrix is (data, 2026-10-17)

The matrix is defined in `vergil-containers/.github/workflows/cd-docker-publish.yml`.

| Language | Matrix | Newest GA upstream | Trailing-edge status |
|---|---|---|---|
| Python | 3.12, 3.13, 3.14 | 3.15 (GA 2026-10-09) | 3.12 EOL 2028-10-31 |
| Ruby | 3.2, 3.3, 3.4 | 4.0 (2025-12-25) | **3.2 EOL 2026-03-31, past** |
| Java (Temurin) | 17, 21 | 25 LTS (2025-09-22) | 17 EOL 2027-10-31 |
| Go | 1.26 | 1.27 (2026-08-19) | 1.26 supported until 1.28 ships |
| Rust | 1.92, 1.93 | 1.99 (2026-10-01) | upstream supports only the newest stable |
| C++ Clang | 19, 20 | LLVM 23.1 (2026-08-25) | upstream patches only the newest major |
| C++ GCC | 13, 14 | 16.2 | **GCC 13 is no longer supported** (13.5 was the last release) |
| TS/Node | 22, 24 | 26 (LTS 2026-10-28) | 22 EOL 2027-04-30 |

Sources:

- endoflife.date API, `https://endoflife.date/api/<product>.json` for `python`,
  `ruby`, `eclipse-temurin`, `go`, `rust` and `nodejs`.
- `https://gcc.gnu.org/releases.html` and `https://gcc.gnu.org/gcc-13/`.
- GitHub releases for `llvm/llvm-project`.

### 2.2 Every version list is maintained by hand (data, recon 2026-10-17)

- **`vergil-containers` has about 30 hand-maintained places** that record a
  version or version list:
  - the publish matrix, `docker/build.sh` and the Dockerfile `ARG` defaults;
  - `docker/pins/report_exposure.py` `_MATRIX` and the `package-cleanup.yml`
    lists;
  - `pins.yml`, the Go tool `case` block and `.trivyignore` comments;
  - the README, `CLAUDE.md` and docs-site tables.

  These lists have drifted apart. For example, ts-node and C++ are missing from
  several of them.
- **`vergil-tooling`** hard-codes defaults:
  - `lib/container.py` `_DEFAULT_VERSIONS`.
  - `lib/repo_init.py` `_default_ci_versions`, which seeds tags that don't exist:
    go `1.23` and rust `stable`.
- **Nothing validates `[ci].versions`** against what is published.
  `lib/config.py:925` only checks for a non-empty list of strings. As a result
  `mq-rest-admin-go` still lists 1.25 and passes validation. A retired version
  fails only at image-pull time.
- **The two "primary version" rules disagree.** Local tooling uses `versions[0]`
  (`config.py:1077`). CI and `vergil-actions` use the maximum.
- **`vergil-actions` already derives everything.** `derive.sh` and `resolve.sh`
  map `[ci].versions` to image tags with no hand-kept list. It is the model the
  rest of the system should follow.
- **The Go 1.25 drop was only partly done.** It is still listed in
  `vergil-tooling/docs/site/docs/guides/ci-architecture.md:463`, all through
  `mq-rest-admin-go`, and in the GHCR tags.
- **The only upstream version check** in the fleet is `vergil-vm`'s Node LTS
  probe in `scripts/dev/update-deps.sh`. `bump-tools.yml` only bumps binary
  tools in `docker/common/`.

### 2.3 Where Python actually comes from at run time (data, recon 2026-10-21)

Three separate interpreter pins are in play, and none of them comes from the CI
matrix:

| Runs | Interpreter | Source |
|---|---|---|
| `vrg-*` tooling in the VM | `/opt/vergil/python/3.14.8` | the `vergil-python` deb (`runtime.toml`); the tooling venv is built with `UV_PYTHON_DOWNLOADS=never` |
| `mqlab` (from its checkout) | a uv-managed 3.14 with the patch level floating | `.python-version` = `3.14`; uv `python-preference = "only-managed"` |
| Lab guest components | `/opt/vergil/cpython-3.14` 3.14.8 | a python-build-standalone tarball pinned in the lab's `lab/versions.yaml`, separate from `vergil-python` |

Two more findings:

- **The two 3.14.8 pins have already drifted apart** on their build tag.
- **Nothing imports `vergil-tooling`, but it runs everywhere.** Nothing under
  `~/dev/projects` depends on or imports it, and it always deploys on 3.14.8.
  However, CI installs it with a bare `uv tool install` inside each consumer's
  image (`vergil-actions/actions/shared/setup/vergil/action.yml:88`), so it runs
  on every version a consumer builds. Its 3.12–3.14 test matrix is therefore
  needed.

## 3. Principles

1. **Follow each community.** Vergil's support policy for a language is based on,
   and compatible with, that upstream's own policy. Vergil never supports a
   version its community has stopped supporting. Vergil's own rules fill gaps
   only where an upstream has no rule.
2. **GA is the trigger, and adoption is immediate.** Once a version is GA and
   supported upstream, Vergil is behind until it supports that version. The goal
   is the leading edge, not the bleeding edge.
3. **No pre-release builds by default.** Building pre-releases with non-fatal
   errors was tried before, and it mostly produced noise. A pre-release can be
   adopted early only as a deliberate, one-off decision driven by a real need.
4. **EOL has no exceptions.** At upstream EOL a version is dropped. It gets no
   further updates and no new Vergil releases include it. Dropping at
   security-fix-only status is allowed as a judgment call. Negotiating
   trailing-edge pain with real users is out of scope: Vergil has one user.
5. **Moving forward is the default, and opting out is explicit.** A new version
   reaches every consumer that isn't pinned on its next build.
   - If that breaks a repo, the fix is to fix forward, or pin and file an issue.
   - A pin is visible technical debt.
   - This trade-off is deliberate for a single-user ecosystem. At scale, opt-in
     would be the better choice.
6. **Derive, don't hand-maintain.** Every version list is generated from one
   manifest, and the manifest is computed from upstream metadata. As little as
   possible depends on a human or an AI reading websites.
7. **One shared machine, with an adapter per language.** The shape is the same for
   every language: read upstream, apply the policy, compute the target window,
   then change it. Each language's quirks live in that language's adapter, never
   in the shared code.
8. **Work at `x.y` and ignore the patch level.** A pin of 3.14 is satisfied by any
   3.14 patch release. Version order is semantic-version order.
9. **No special handling for major-version jumps.** A Python 4 (if one ever ships)
   is adopted like any other version. Anything that breaks is pinned and dealt
   with as usual.
10. **Vergil does not track who uses which version.** Finding every consumer of a
    version is like finding every symlink to a file: it doesn't scale. The
    consumer side enforces EOL instead (§6.3).
11. **Leading edge over trailing edge.** Change is speeding up across the
    industry, at different rates in different communities. The only way to keep
    support current is to stay on the leading edge, and the price is deliberately
    giving up the trailing edge.
12. **No backfill.** The window grows only at the top, through new GA releases.
    A version older than the window's current oldest is never added, even if
    upstream still supports it (for example Python 3.11 or Java 11). The rule is
    enforced once, in the shared machinery.
13. **Vergil owns which toolchain runs, and a language's own files declare only
    a floor.** Every ecosystem has two kinds of version setting (§6.5):
    - "which toolchain to use", which duplicates Vergil and is removed;
    - "minimum supported", which the ecosystem needs and keeps, as a floor with
      no upper bound.

## 4. Language portfolio

| Language | Status | Why |
|---|---|---|
| Python | **kept, in active use** | Vergil's own tooling language |
| Go | **kept, built in this epic** | Strategic, and simple. It is the second language that tests the design |
| Rust | kept, **next in the queue** | Most strategic for performance-sensitive work |
| Java | kept | One of the most widely used languages, and common in the MQ world |
| C++ (Clang, GCC) | kept | Needed for specific reasons |
| TypeScript/Node | kept | Planned for a Nemesis tool, which is on hold |
| Ruby | **dropped** | Niche, with no expected use. Its one consumer was never published |

**The rule for a dropped language:** remove it from the matrix and archive the
repos that depend on it. Bringing it back later is a new, cheap addition.

A kept-but-unused language stays only while tooling keeps its upkeep cheap. If
one becomes a real burden on other work, it can be dropped under the same rule.

Perl is a possible future addition, only if a real need arises. It is out of
scope here.

## 5. Per-language support policies

Each kept language gets a written policy in `vergil-containers`, all in the same
format:

- the upstream source of truth for releases and EOL;
- the support rule that determines the window;
- what counts as GA;
- the adapter's quirks;
- whether a deployed runtime package is needed.

The table below is the **starting point** for those policies. The cadence and
support facts come from upstream policies as summarised by endoflife.date, and
the builder must confirm each one against the official page. The proposed
windows are judgment that applies principle 1.

| Language | Upstream support fact | Proposed Vergil window | Adapter quirks |
|---|---|---|---|
| Python | Each minor version gets about 5 years, with a fixed EOL date (PEP 602) | every minor version that is GA and not EOL | Official `python:<v>-slim` base. Needs a `vergil-python` runtime (§7) |
| Go | The two newest releases are supported | the two newest | `golang:<v>` base. `GOTOOLCHAIN=local` makes some tool pins depend on the Go version (`case` block, `pins.yml`) |
| Rust | Only the newest stable release is supported; a new one every 6 weeks | newest and newest−1 (a decision, §5.1) | `rust:<v>-slim` base. Fast cadence, so auto-PRs matter most here |
| Java (Temurin) | LTS releases have fixed EOL dates; non-LTS releases last 6 months | LTS releases that are not EOL (decision for the Java policy) | `eclipse-temurin:<v>-jdk` base |
| Node | Majors become LTS, with fixed EOL dates. The adapter reads the LTS status itself, not the parity of the major, because from Node.js 27 every major becomes LTS | LTS majors that are not EOL | NodeSource `setup_<n>.x` on trixie. The separate `vergil-vm` Node pin must line up |
| C++ Clang | Only the newest LLVM major gets patches | set in the Clang policy | Limited to majors that `apt.llvm.org` publishes for the Debian base |
| C++ GCC | Several release lines are maintained in parallel | set in the GCC policy | Limited to Debian base packages. A newer GCC usually means a newer Debian base. Report "a newer upstream version exists but the base can't install it" separately from "behind" |

### 5.1 Decided

- **Rust:** newest and newest−1. That is one version wider than upstream, to
  absorb the 6-week cadence.
- **Go:** the two newest releases.
- **Python:** every minor version that is GA and not EOL, subject to the
  no-backfill rule (principle 12). Today that is 3.12–3.15. 3.11 stays out
  because of that rule, not because of a special case.

The other policies are written in this epic but set to **report-only**; their PR
automation is enabled later (§9).

## 6. Design

### 6.1 The version manifest (in `vergil-containers`)

A single data file is the source of truth for the window. Each entry holds:

- the language, or language family (`cpp-clang`, `cpp-gcc`, `ts-node`);
- the version, at `x.y` or major level;
- the build argument and upstream base image;
- the upstream GA date and EOL date (if the upstream publishes dates);
- the state;
- **for each Python version, whether a `vergil-python` runtime is published**
  (`runtime_published`). The nightly lifecycle job sets this field by reading
  the vergil apt index for `vergil-python<x.y>`; `vergil-python`'s release only
  publishes the package. The field travels in the published window (item 6
  below), so resolution reads one source and never queries package
  repositories.

> **Amended after implementation** (vergil-project/vergil-containers#621, PR
> vergil-project/vergil-containers#638). The original text said
> `vergil-python`'s release updates the field. As built, the lifecycle job
> reads the apt index: a newly published package is a `runtime` change that
> opens a PR setting the flag to `true`, and a recorded runtime that disappears
> from the index is a `runtime_missing` issue. An added version always starts
> as `false`.

**The manifest records what is built, and labels each version in or out of
policy.**

- For a language whose automation is on, the two always converge: the PRs
  remove every out-of-policy version.
- For a report-only language, an out-of-policy version stays in the manifest
  and stays buildable. Examples today: Rust 1.92/1.93 and Clang 19/20, which
  upstream no longer patches.
  - The report flags it on every run.
  - A tracked issue records it.
- This is an explicit, **temporary exception to principle 1** for languages not
  yet automated. It ends when each language gets its turn.
- GCC 13 is past upstream support. It is dropped in this epic, because GCC 14
  remains and the removal is as small as Ruby's.

Generated from it:

1. the `cd-docker-publish.yml` matrix (for example via a setup job and `fromJSON`);
2. the `docker/build.sh` lists and smoke-test tag lists;
3. `report_exposure._MATRIX` and the `package-cleanup.yml` package lists;
4. the Dockerfile `ARG` defaults (the newest version);
5. every version table in the docs (README, `CLAUDE.md`, the docs site);
6. a **published window**: a machine-readable artifact that consumers read at
   build time (§6.3).

A validation gate fails if any generated file is out of date with the manifest,
following the same pattern as `CATALOG.md` from `pins.yml`.

**Not generated: Go and scanner leftovers.** Three of the hand-kept places in
§2.2 stay hand-kept: the Go tool `case` block in
`docker/go/Dockerfile.template`, the version constraints in `pins.yml`, and the
version notes in `.trivyignore` comments. The Go adapter lists the `case`
branch and each `pins.yml` constraint that names a dropped version as manual
checklist items in the drop PR's body (§6.2). Nothing lists the `.trivyignore`
comments.

**Left to the builder:** the file format (JSON or YAML; never XML), the exact
schema, and whether the manifest is only the computed window or also records
per-language policy settings.

> **Decided by the builder** (vergil-project/vergil-containers#613). The
> manifest is `docker/versions/manifest.yml`, in YAML, with a top-level
> `schema: 1`. It records policy settings as well as the window: `automation`
> (`pr` or `report`) is set per family, and `in_policy` per version entry. The
> generated machine outputs are `docker/versions/matrix.json` (the publish
> matrix) and `docker/versions/window.json` (the published window).

### 6.2 The lifecycle check and the adapters

**Scheduling.** The check runs from `vergil-containers`' existing `ops.yml`
scheduled workflow. Daily or weekly is the builder's call.

> **Decided by the builder** (vergil-project/vergil-containers#617). The check
> is the `language-lifecycle` job in `ops.yml`, which runs daily on that
> workflow's `37 6 * * *` schedule. The look-ahead is 90 days. The report is
> written to the job summary and uploaded as the `language-lifecycle-report`
> artifact.

**Each run:**

1. For each language, the **adapter** reads upstream metadata, applies the
   policy, and returns the **target window**: the versions to support, with
   their GA and EOL dates.
2. The shared code **diffs the target against the manifest** and classifies each
   language:
   - in sync;
   - a version to add (GA shipped);
   - a version to drop (EOL reached);
   - EOL coming within a look-ahead period;
   - blocked (for example, the base can't install it).
3. It writes a **matrix report** covering every language, every run.
4. **For a language whose automation is turned on,** it opens (or refreshes) one
   **add/drop PR** per change. The PR updates the manifest and regenerates every
   derived file.
   - `vergil-containers`' PR CI builds no images. An add PR's new images are
     first built, scanned and published by the CD run on `develop` after the
     PR merges. Building the images a PR changes in PR CI is deferred to
     `vergil-project/.github#213`.
   - The automation never enables auto-merge. A human reviews and merges every
     lifecycle PR.
   - A change that no PR can carry (`blocked`, `eol_soon` and
     `runtime_missing`) opens or refreshes a tracked issue, as for a
     report-only language.
5. **For a language that is report-only,** it opens or refreshes one tracked issue
   per change.

> **Amended after implementation** (vergil-project/vergil-containers#618 and
> #622, vergil-project/vergil-containers#637 / PR #670). The original text said
> the PR's CI builds and scans the new images; it never did, and the human
> deferred PR-CI image builds to `vergil-project/.github#213` during #622. The
> temporary hold that marked add PRs "do not merge while #622 is open" was
> removed by #637 once #622 passed.

PRs and issues are the only signals, and no new notification channel is
introduced. Getting failing scheduled jobs in front of the human is a separate
concern, owned by other work.

**A failing check reports itself through the same channel.** The job must
**not** use `continue-on-error`, unlike the existing `pin-exposure` job in
`ops.yml`, which is documented as "never a gate". When one language's adapter
fails:

- the run still finishes the other languages;
- it opens or refreshes **one tracked issue** for the failing language, titled
  `lifecycle check failing for <family>` (for example
  `lifecycle check failing for go`), with the error text in the body;
- the next successful run closes that issue.

That way, no PR and no open issue reliably means "in sync".

**The adapter contract.** The shared code knows nothing about any one language.
An adapter supplies:

- where the release and EOL data come from (endoflife.date covers Python, Go,
  Rust, Temurin and Node; GCC and LLVM need their own sources);
- the window rule;
- how to check that the base image or package can actually install a version;
- any changes beyond the matrix itself (for example, Go's per-version tool
  holds). These are listed in the PR body as manual checklist items; they are
  not generated (§6.1).

**Fail loud.** If an adapter can't fetch or parse upstream data, the run reports
an error for that language. It never treats "no data" as "in sync".

**The test.** Python and Go are built together. The test is that **adding Go
needs no change to the shared code**. Anything Go forces into the shared code is
a design bug to fix in this epic.

**Left to the builder:**

- the implementation language and location of the check (it lives in
  `vergil-containers`);
- how a PR is refreshed or replaced;
- the look-ahead period;
- the report format and where it is stored.

### 6.3 Consumer resolution (in `vergil-tooling` and `vergil-actions`)

**Library or app: reuse `[project].repository-type` and define it properly.**
Whether a repo **provides libraries** (its code runs in other repos'
environments) is expressed through the existing `repository-type` field, not a
new attribute.

- **The field was never well defined.** Today
  `vergil-tooling/src/vergil_tooling/lib/config.py:29` allows `library`,
  `application`, `infrastructure`, `tooling` and `documentation`, but nothing
  except `repo_init` reads it.
- **This epic writes a definition for every value** and gives it a
  behavioural effect:
  - `library` means the repo provides code that runs in other repos'
    environments, whether it is imported there or installed there as a tool.
  - Every other value means the repo is an app, for version resolution.
- **A repo with no language matrix is unaffected,** whatever its type. For
  example, `vergil-claude-plugin` and `vergil-actions` are `library` but have no
  language, so there is nothing to resolve.
- **The epic re-checks each repo's current value against the new definitions.**
  For example, `vergil-tooling` (today `tooling`) becomes `library`. Nothing
  imports it, but CI installs it in every consumer's image, so it must support
  the whole window (§2.3).
- *Direction from the human, with the caveat that the field needs this
  clean-up to carry the meaning.*

**Resolving the versions a repo builds:**

| `[ci].versions` | `repository-type` | Resolved versions |
|---|---|---|
| absent | `library` | the whole current window for the language |
| absent | any other value | the newest version in the window |
| present (a pin) | `library` | the pinned list |
| present (a pin) | any other value | the single pinned version (an app pin must have exactly one entry) |

Rules:

- **Any pinned version not in the current window is a fatal configuration
  error.** The message names the version and the fix ("remove it or update the
  pin"). It is checked by `vrg-validate` and by the CI setup step. This is the
  allowlist that would have caught 1.25.
- **The "primary" version is the newest resolved version, everywhere.** This
  removes the current `versions[0]` versus maximum mismatch.
- **Consumers read the published window at build time** (§6.1, item 6), not a
  copy built into `vergil-tooling`. Otherwise they would lag a tooling release
  behind.
- **The defaults in `vergil-tooling`** (`_DEFAULT_VERSIONS`, the `repo_init`
  seeds) come from the published window. New repos are created with no
  `[ci].versions`.
- **The change ships inside Vergil 2.1** as an incremental change, not a 2.2.
- **Existing `[ci].versions` lists are removed** *(the human's decision)*:
  - The epic sweeps every Vergil-owned repo in every org the human owns. Each
    repo gets a small PR that deletes its list, so the repo follows the window.
  - A repo keeps a list only if it genuinely needs a pin. That pin is recorded
    with an issue. For example, `mq-resiliency-lab-for-linux` keeps `["3.14"]`
    (`logical-minds-foundry/mq-resiliency-lab-for-linux#1685`), and
    `mq-protocol-gateway` keeps `["clang-20"]`
    (`logical-minds-foundry/mq-protocol-gateway#25`).
  - The on-hold mq-rest-admin ports are excluded (§8.2).
  - The sweep covers only the active repos. The on-hold repos
    (`mq-resiliency-observability`, `mq-gateway-replay-lab`, `melete` and
    `diogenes`) were moved to their own repos' or orgs' ad-hoc epics, and are
    swept when work on them resumes.
- **An app with a list of more than one version is a fatal configuration
  error.** The message tells the repo to pin a single version, or to set
  `repository-type = "library"` if it really provides code that runs in other
  repos' environments.

> **Amended after implementation** (vergil-project/vergil-tooling#3372 / PR
> #3445, vergil-project/vergil-tooling#3442; scope decision on
> `vergil-project/.github#407`, 2026-10-22). The original text gave a
> transition rule: a warning during the sweep, fatal once it was done. The
> rule went fatal in #3372 at the end of the sweep, and #3442 removed the
> transition switch and the resolver's warnings channel, so only the fatal rule
> remains. The sweep was also cut down to the active repos.

**Further resolution rules (builder-check calls, marked as my calls):**

- **Report-only languages** (Rust, Java, C++, Node) also get the shared
  resolution and the fatal check on a pin outside the window. Their window is
  what the manifest records as built, out-of-policy versions included (§6.1).
- **C++ and TypeScript** keep their family-prefixed tokens (`clang-20`,
  `gcc-14`, `node-24`). Until each language's turn, consumers must pin them, and
  the fatal check still applies. The unpinned default for family-keyed languages
  is decided in that language's turn. *(My call.)*
- **The existing `[ci].primary-version` override stays,** as an override in the
  same spirit as a pin. It must name a resolved version, and validation fails if
  it doesn't. Without it, the primary version is the newest resolved version.
  *(My call.)*
- **The on-hold mq-rest-admin ports** may go red when the fatal check ships.
  That is accepted: they aren't being built, and their pins are fixed when the
  ports are revisited. *(The human, during the brainstorm.)*

**One resolver.** Resolution is implemented **once**, as a `vergil-tooling`
command (for example `vrg-ci-resolve`). It outputs as JSON:

- the resolved versions;
- the primary version;
- the image tags;
- the image suffixes (the image family for each resolved version, for example
  `cpp-clang` for `clang-20`).

`vrg-validate` uses that command. So does `vergil-actions`' `derive.sh`, which
calls it rather than reimplementing it. CI already installs `vergil-tooling` in
the same setup action (`vergil-actions/actions/shared/setup/vergil/action.yml:69-87`).
The bash and Python copies of resolution are removed.

**Bootstrap (chicken and egg).** `vergil-tooling` uses its own tooling. When the
window gains 3.15, its own next build resolves to 3.15 as well. The rollout order
must let the change that publishes a new version, and the tooling that reads it,
build under the new window without a release deadlock. The plan must state that
order explicitly.

**Fetching the window.** The window is fetched at the consumer's pinned Vergil
ref (the 2.1 line), and **it is never cached**. A failed fetch is a loud error
that names the URL and the explicit override, a local file path. Offline work
uses that override deliberately. A cached copy would go stale exactly when the
window changes, and it could hide a drop. *(The human's call, at alignment.)*

**Left to the builder:** the publishing mechanism and the name of the override.

> **Decided by the builder** (vergil-project/vergil-tooling#3367 and #3392).
> The window is read from
> `https://raw.githubusercontent.com/vergil-project/vergil-containers/<ref>/docker/versions/window.json`,
> where `<ref>` is the consumer's `[dependencies].vergil` (for example `v2.1`).
> The override is `VRG_VERSION_WINDOW`, set to a local file path; when it is
> set it always wins, and `vrg-container-run` mounts that file into the
> container.

**Open problem: one release can read two windows.** "Fetched at the ref, never
cached" means every reader fetches the moving `v2.1` window on its own. The
parallel package-build cells of one release each derive the runtime from their
own fetch, so a window change mid-release splits the release:
`vergil-tooling` 2.1.258 shipped its amd64 packages on the 3.15 runtime and
its arm64 packages on 3.14, because it was released minutes after the 3.15
runtime was marked published. The fix is undecided and tracked in
`vergil-project/vergil-tooling#3478`. This wording is updated once that issue
decides it.

### 6.4 Language floors

Some consumer files declare a language floor of their own:

- Python: `requires-python`, the `target-version` in ruff and mypy.
- Go: the `go` directive in `go.mod`.
- Rust: `rust-version`.
- Java: the `release` in `pom.xml`.

These go stale when the bottom of the window moves. In this epic:

- **Python floors:** `vrg-validate` checks the repo's own floors against the
  window. Each repo finds its own staleness, so Vergil still doesn't track who
  uses what (principle 10). Floor fixes are each repo's own work and are not
  automated PRs from the lifecycle check. *(My call.)* The check has two
  outcomes:
  - **Warning:** a floor below the window's oldest version.
  - **Warning:** an upper bound that excludes the newest version, such as the
    lab's `requires-python = ">=3.14,<3.15"`. That bound would block an app from
    moving to 3.15.
- **A floor above the window's oldest version is allowed** as a deliberate
  restriction, for example a library that needs 3.14 features. It needs no
  special check. If the repo forgets the matching pin, its build on the older
  versions fails, and that failure is the signal *(the human's call)*.
- **Go floors:** the `go` directive in `go.mod`, handled the same way.

Automating floors for other languages is left to each language's turn.

### 6.5 Toolchain selection belongs to Vergil

A repo's own files can choose the interpreter or toolchain behind Vergil's
back, and a CI run can then pass on a version it never ran. For example, the
lab keeps uv `python-preference = "only-managed"` so that its VM venv is never
seeded from the host interpreter. The CI setup runs `uv sync`, so without a
guard that setting would make uv download its own managed interpreter inside
a `python:3.15` image instead of using the image's. Vergil's
`UV_PYTHON_PREFERENCE=only-system` (requirement 1) is that guard.

| Language | "Which toolchain" settings (remove; Vergil owns this) | "Minimum supported" settings (keep, as a floor with no upper bound; §6.4) |
|---|---|---|
| Python | `.python-version` | `requires-python` |
| Rust | `rust-toolchain.toml` | `rust-version` in `Cargo.toml` |
| Go | — | `go` directive in `go.mod` |
| Java | — | `release` in `pom.xml` |

Requirements:

1. **Vergil hands the resolved version to the language tool directly, and pins
   the source.** For uv, that is `UV_PYTHON` (the version) plus
   `UV_PYTHON_PREFERENCE=only-system` (use the image's interpreter, never a
   managed download). Both are set by `vrg-container-run` and
   `vrg-container-test`, by the cached dev-image build, and on every Python CI
   leg. The docs jobs and `vrg-container-docs` set only
   `UV_PYTHON_PREFERENCE=only-system`, since they resolve no version.
2. **`vrg-validate` flags any toolchain-selection setting** as a pin that
   conflicts with the window. For Python that is only `.python-version`.
3. **The CI setup step asserts that the interpreter or toolchain actually
   running the job is the resolved `x.y`,** and fails loudly if it isn't. This
   backstop catches any selection mechanism the list above misses.
4. **The fleet sweep (§6.3) removes these settings** along with the
   `[ci].versions` lists. The plan was to check whether the lab still needs
   `python-preference` to keep the VM's system Python out of local runs, and
   whether `UV_PYTHON` covers that case. It does not: the lab builds its venv
   outside `vrg-container-run`, so it keeps `python-preference = "only-managed"`
   with `python-downloads = "manual"`.

uv's `python-preference` and `python-downloads` choose where an interpreter
comes from, not which version, so repos may keep them.

> **Amended after implementation** (decision on `vergil-project/.github#407`,
> 2026-10-22; vergil-project/vergil-tooling#3410, #3421 and #3414;
> vergil-project/vergil-actions#994 and #1002;
> `logical-minds-foundry/mq-resiliency-lab-for-linux#1681`). The original text
> listed `python-preference` as a "which toolchain" setting to remove, and
> relied on `UV_PYTHON` alone. Measured on the lab VM, `UV_PYTHON` picks the
> version but not the source, so the human chose to allow `python-preference`
> and have Vergil pin the source itself.

Only Python and Go are enforced in this epic. Rust's `rust-toolchain.toml` is
handled in Rust's turn.

## 7. Deployed runtime alignment (Python)

Containers provide development, test and CI. **Deployed code runs on whatever
interpreter it's installed with.** For Python that is `vergil-python`, plus the
lab's separate guest-runtime pin (§2.3).

- **Packaging is opt-in per repo,** through a `[package]` block in
  `vergil.toml`. Today the only packaged Python repo is `vergil-tooling`, a
  **library** (§6.3) whose runtime is derived, with no
  `[package.python].runtime` set. `vergil-python` and `packages` package
  runtimes and infrastructure. Every other repo is unpackaged and simply follows
  the window.
- **Two things must be ready before a packaged app can move to a new version.**
  The containers must provide the version (the development and CI side), and
  `vergil-python` must publish a runtime for it (the deployment side). These are
  separate operations, released separately. The gap between them can be a few
  days, for example while python-build-standalone catches up after a CPython
  release.
- **The resolution rule for a packaged Python repo** *(the human's design)*:
  - Resolve to the **newest version in the window that also has a published
    `vergil-python` runtime**.
  - If the newest version has no runtime yet, fall back to the next newest, and
    so on.
  - For a packaged **app**, this rule picks the version its CI builds. For a
    packaged **library**, it picks only the deployed runtime: the library's CI
    keeps the whole window, since its code runs on every window version in
    other repos (§2.3). Either kind fails loud if no window version has a
    published runtime.

  Nothing breaks during the gap. The packaged repo deploys on 3.14 until the
  3.15 runtime is published, then moves on its next build. Unpackaged repos are
  unaffected and move as soon as the containers have the version. The rule is
  Python-specific, so it lives in the Python adapter or resolver extension,
  never in the shared code (§8.3).
- **`[package.python].runtime` stops being a hand-set exact patch.** It is
  optional. When it is absent, the runtime is derived from the resolved `x.y`
  and the patch `vergil-python` publishes for it. An explicit value remains
  possible as a pin: it must be an exact `x.y.z`, and its `x.y` obeys the fatal
  check on versions outside the window.
- **`vergil-python` publishes one series at a time.** The package is
  `vergil-python<x.y>`, installed at `/opt/vergil/python/<x.y>`, with the patch
  level floating inside the series (principle 8). Its update-deps hook
  refreshes the patch and build within the series, and never moves the series.
  - Today the series is 3.15. The 3.14 package is frozen at `3.14.8+20261013`:
    it is no longer rebuilt, and it stays in the apt repo only while the
    repo's retention keeps it.
  - Moving to a new series is a manual, reviewed PR
    (`vergil-project/vergil-python#34`), made after the containers add the
    version.
  - Nothing drops a series at EOL.
  - Publishing several co-installable series at once is deferred to
    `vergil-project/vergil-python#36`, for when a second consumer needs an
    older series.
- **Compiled languages (Go, Rust) need no runtime package.** For each new
  interpreted language, the need for one is assessed when the language is added.
- **The lab's guest runtime pin** (`lab/versions.yaml`) is a consumer of the same
  rule. Moving it onto `vergil-python`, or onto the window, is recorded as a
  follow-on for the lab repo. It is outside this epic's repos.

> **Amended after implementation** (vergil-project/vergil-tooling#3368 / PR
> #3401, #3400, #3455; vergil-project/vergil-python#25, #34 and #36). The
> original text called `vergil-tooling` a packaged app with a hand-set
> `runtime = "3.14.8"`, and had `vergil-python` publish one co-installable
> runtime per window `x.y`, added and dropped by the lifecycle check. As built,
> `vergil-tooling` is a packaged library with a derived runtime, the
> newest-with-a-runtime rule governs only a library's deployed runtime, and
> `vergil-python` publishes a single series that moves by hand.

## 8. Scope

### 8.1 In scope

1. Per-language policy documents for every kept language.
2. The version manifest and every generated version list (§6.1).
3. The lifecycle check, the matrix report, and the adapters:
   - **Python and Go** with PR automation;
   - the rest report-only.
4. Consumer resolution: the optional pin, `repository-type` defined and given its library/app meaning, the fatal
   check on a dropped version, the unified primary version, defaults read
   from the published window, and the single resolver (§6.3).
5. Python and Go floors (§6.4), and Vergil-owned toolchain selection, including
   the CI assertion of the running version (§6.5).
6. `vergil-python` alignment (§7). This covers:
   - one runtime package per series, `vergil-python<x.y>`, publishing one
     series at a time;
   - the packaged-repo resolution rule;
   - the derived `[package.python].runtime`.
7. The fleet sweep that removes existing `[ci].versions` lists and
   toolchain-selection settings (§6.3, §6.5).
8. **Catching up, as the first real runs of the automation:**
   - add Python 3.15;
   - add Go 1.27;
   - drop Go 1.26 when its EOL arrives during the epic (if it doesn't, the drop
     path is proven with a test fixture and recorded).
9. **Dropping Ruby and GCC 13:**
   - remove Ruby from the matrix, the generated lists, the `vergil-tooling`
     defaults and the docs, and archive `mq-rest-admin-ruby` (archived);
   - remove Ruby support from the tooling itself, so
     `primary-language = "ruby"` is a configuration error
     (`vergil-project/vergil-tooling#3377`);
   - remove the GCC 13 image, which is past upstream support (§6.1).
10. **Finishing the Go 1.25 retirement** in the Vergil repos (the stale
   `ci-architecture.md:463` and the like).
11. Fixing the drift the recon found, where the manifest doesn't already
    replace it:
    - the `repo_init` seeds that point at tags that don't exist;
    - the `package-cleanup` gaps.

### 8.2 Non-goals

- **The mq-rest-admin language ports** (other than archiving the Ruby one). They
  are on hold and will be revisited separately.
- **Per-language library dependency updaters**, which stay in
  `vergil-project/.github#384`.
- **Catching up Rust, Java, C++ and Node.** This epic gives them written policies
  and the report only.
- **Deleting old GHCR tags.** Dropped versions stop being rebuilt, and existing
  tags are left in place.
- **Scheduled rebuilds of repos whose code hasn't changed** (drift detection).
  That's a related idea, kept separate.
- **A way to defer a drop past EOL.** This was considered and rejected.

### 8.3 Anti-goals: the epic fails if any of these happen, even if it technically works

- The shared machinery contains Python- or Go-specific branches.
- An adapter treats an upstream fetch failure as "nothing to do".
- A version list survives anywhere as a hand-maintained copy. (The Go tool
  `case` block, the `pins.yml` constraints and the `.trivyignore` notes in §6.1
  are per-version settings, not version lists. They stay hand-kept, and only
  the first two appear as drop-PR checklist items.)
- A dropped version can still be pinned without a hard failure.
- An app's CI resolves to a Python version that `vergil-python` can't deploy.
- A CI job passes for a version it did not actually run.
- A broken lifecycle check is indistinguishable from "in sync".
- Version resolution exists in more than one implementation.

## 9. Rollout (constraint on the plan's ordering)

1. **Policies and the manifest come first,** with the generated lists and the
   validation gate. No behaviour change yet.
2. **The report next, for every language.** This makes the gap visible
   immediately.
3. **Python and Go adapters together, with PR automation.** Their first real PRs
   are Python 3.15 (with the `vergil-python` 3.15 build) and Go 1.27.
4. **Consumer resolution second-to-last,** in the bootstrap order §6.3 requires,
   once the published window exists.
5. **The Ruby drop and the cleanups** can run in parallel with the stages above,
   wherever they don't conflict.

Node 26 becomes LTS on 2026-10-28, while the epic is underway. It is recorded in
the report and handled when Node's turn comes. It is not implemented here.

## 10. Closing bookends and follow-ons

- `vergil-project/vergil-containers#610`: documentation review, a multi-repo
  sweep.
- `vergil-project/.github#409`: brainstorm a per-language sample-repo testbed for
  integration test releases.
- `vergil-project/.github#410`: rank the next languages for full lifecycle
  support, with Rust first.
- `vergil-project/.github#411`: retrospective (terminal).
- `vergil-project/.github#424`: brainstorm advisory pre-release toolchain
  support, promoted to required CI at GA.
- `vergil-project/.github#425`: brainstorm version resolution for apps whose
  runtime lives on a host (for example the lab's VM venv), to retire pins like
  the lab's `["3.14"]`.
- `vergil-project/.github#213`: the container-build epic, which now holds
  building and scanning a PR's changed images in PR CI (§6.2).
- `vergil-project/vergil-python#36`: publish several co-installable runtime
  series at once (§7).
- **Follow-on recorded, not seeded here:** move the lab guest runtime pin onto
  `vergil-python` or the window (in `mq-resiliency-lab-for-linux`).

## 11. Sources

**Data, fetched 2026-10-17:**

- `https://endoflife.date/api/{python,ruby,eclipse-temurin,go,rust,nodejs}.json`
  (a community aggregator; confirm against the official pages):
  - `https://peps.python.org/pep-0602/`
  - `https://www.ruby-lang.org/en/downloads/branches/`
  - `https://adoptium.net/support/`
  - `https://go.dev/doc/devel/release`
  - `https://nodejs.org/en/about/previous-releases`
- `https://gcc.gnu.org/releases.html` and `https://gcc.gnu.org/gcc-13/` ("This
  release series is no longer supported").
- `https://api.github.com/repos/llvm/llvm-project/releases`.

**Recon of local checkouts on 2026-10-17 and 2026-10-21.** The file and line
references in §2 are from those checkouts.

**Judgment, not sourced:** the proposed windows in §5 (other than those decided
in §5.1), the "no language cliff" reading in principle 9, and every item marked
"left to the builder".
