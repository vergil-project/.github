# Package index: retention-first build — design

- **Epic:** vergil-project/.github#1003
- **Planning issue:** vergil-project/.github#1002 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

`vrg-package index` should do work in proportion to what it publishes, not to
how many releases exist. Today the `build-index` job of every `publish-index`
run takes 20 to 26 minutes. The slow index publish is blocking vergil-tooling
releases (vergil-project/packages#46).

## 2. Starting point (verified 2026-11-10)

`build_site` (vergil-tooling `src/vergil_tooling/lib/package/index/site.py`)
runs, in order:

1. `collect.collect()`: for each configured product, `stable_tags()` (one
   `gh release list`), then for every stable tag a `gh release view` for its
   asset names. For every release with a `packages-manifest.json`, it downloads
   the manifest and every `.deb`/`.rpm`, matches them against the manifest and
   reads their metadata.
2. `collect.verify()`: `gh attestation verify` for every downloaded package,
   and an rpm signature check for every `.rpm`.
3. `retention.select()`: duplicate-bytes check across all collected artifacts;
   retire typo check across all collected names; the newest `keep` releases in
   each of the newest `lines` major.minor lines; the dependency closure over
   all collected artifacts; the retired-dependency check.

`packages.toml` sets `keep = 1`, `lines = 2`. So retention keeps one release in
each of two lines, but everything before it costs time for every release in
history.

Volume (data, 2026-11-10): vergil-tooling has 417 releases, packaged from
v2.1.231 on, with four packages of about 7.5 MB per release. vergil-python has 5
releases of 128 to 263 MB of packages each. **Estimate, not profiled:** about
420 release views, 2.7 GB of downloads and 290 attestation verifies per run.

`collect.collect`, `collect.verify` and `retention.select` are called only from
`site.py`. `collect.stable_tags`, `collect.release_packages` and
`published.parse_filename` are also used by `published.py`, the `vrg-release`
preflight check. That check is not changed by this epic.

The order comes from epic vergil-project/.github#356's spec (§4.2 data flow and
§7.2), which lists collect, then verify, then retention.

## 3. Goals

- **G1.** The index build downloads and verifies only the artifacts the index
  publishes: the retained releases and the dependency providers the closure
  adds.
- **G2.** GitHub API calls to choose them are a small, fixed number per
  product, about one per 100 releases.
- **G3.** For the same release history and `packages.toml`, the published index
  has the same packages as the full-scan algorithm, except as §5 states.
- **G4.** The `build-index` job takes **under 5 minutes** on the first live
  `publish-index` run after the fix is released.

## 4. Non-goals and anti-goals

- **Non-goals:** the Pages deployment that reports success but never goes live
  (vergil-project/packages#46); a pruning policy for old releases, artifacts and
  evidence; the full release scan in `published.py`.
- **Anti-goal:** weakening any trust check on what is published. Every
  published package is still attestation-verified pinned to the release
  workflow on `main`, every published `.rpm` is still signature-checked, and
  every failure is still a hard error. Nothing is skipped to save time.
- **Anti-goal:** a silent cap. If a listing could be truncated, the build fails;
  it never indexes from a partial history.

## 5. Design

### 5.1 Inventory

For each product, list every release with its asset names in a small number of
calls: `gh api --paginate "repos/<product>/releases?per_page=100"` (judgment:
the REST listing returns each release's `tag_name`, `draft`, `prerelease` and
`assets[].name`). The listing replaces `stable_tags()` and the per-tag
`release_view` calls for the index. Like every other `gh` call in the index,
it goes through `_invoke`, so it gets the shared transient-error retry.

A release is a **packaged stable release** when its tag matches `vX.Y.Z`, it is
not a draft or prerelease, and its assets include `packages-manifest.json`.
Only those count anywhere below. A stable release without the manifest is
pre-packaging history and is skipped silently: no per-release line, since there
are hundreds. One summary line per product gives the counts.

Each `.deb`/`.rpm` asset name of a packaged release gives its identity with
`published.parse_filename`. A name that does not parse is a hard error, as in
`published.py`.

### 5.2 Selection by name

On the inventory, with no downloads:

1. **Retire typo check:** every `[retention].retire` name must be carried by
   some packaged release in the inventory, anywhere in history. This keeps
   today's check and its meaning. `vergil-python3.14.8` still matches
   vergil-python v1.0.0.
2. **Per-line retention:** for each product, group packaged releases by
   major.minor, take the newest `lines` lines that have one, and in each the
   newest `keep`. This matches `retention._by_lines` today: it already counts
   only collected, packaged releases.

### 5.3 Download, verify, close

1. Download, manifest-match and read the metadata of the retained releases only,
   reusing `_release_artifacts` (manifest match and metadata checks unchanged).
2. **Dependency closure:** for each dependency name of a kept artifact, find the
   providers in the inventory: same format, same name, arch equal or `all`, from
   any packaged release of any product. Take the newest by `version_key`. If it
   is not already kept, download and check its release's artifact through the
   same path and repeat. A dependency no inventory entry provides is an OS
   package, as today. Retired names are excluded from retention, and a kept
   artifact that depends on one is still a hard error (`_check_retired`).
3. **Verify** every downloaded artifact with the existing `collect.verify`.
4. **Duplicate-bytes check** over every downloaded artifact. Identical duplicates
   collapse, as today.

### 5.4 Intended behaviour changes

- A duplicate-bytes clash, or a manifest mismatch, in a release the index does
  not publish no longer fails the index. Judgment: it cannot reach apt or dnf.
  The `vrg-release` preflight (#3311) still refuses a re-published identity
  before release.
- The per-release `no packages-manifest.json … skipped` lines become one summary
  line per product.
- `RELEASE_LIST_LIMIT` (1000) no longer applies to the index, because the
  paginated listing has no cap. Judgment: `gh api --paginate` follows `Link`
  headers to the last page, so a listing cut short surfaces as a `gh` error,
  which is already a hard failure. Output that does not parse as the expected
  list of releases is also a hard error.

### 5.5 Observability

`build_site` prints the elapsed time of each phase (inventory, download, verify,
index, sign) and `indexing <kept> of <inventory> packaged release(s)`, so a
future slowdown is visible in the log.

## 6. Delivery

| # | Task | Repo | Kind | Blocked by |
|---|---|---|---|---|
| 1 | Retention-first index build, with tests | vergil-tooling | implementation | — |
| 2 | Release vergil-tooling with the fix | vergil-tooling | deployment | 1 |
| 3 | Live `publish-index` run meets G3 and G4 | packages | validation | 2 |

Deployment (judgment, from the pins): packages calls
`vergil-actions/.github/workflows/publish-index.yml@v2.1` and pins
`[dependencies] vergil = "v2.1"`, and `build-index` installs vergil-tooling
from git at that pin. So the next vergil-tooling `v2.1.x` release is picked up
with no pin change. The release itself is a human action.

The validation compares the new index with the last full-scan index: the
noble `Packages` and the dnf `primary` package lists must name the same
packages and versions.

Bookends: the documentation task (this spec and the plan), the
documentation-review sweep, which also corrects the order in epic #356's spec,
and the retrospective.

## 7. Left to the builder

- How the inventory is shaped in code (a dataclass per release, or a mapping).
- How the existing `collect`/`retention` functions are split or reused, provided
  `published.py`'s public functions keep their behaviour.
- Test structure, using the existing `Run` injection and fixtures.
