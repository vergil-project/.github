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

- **G1.** The index build downloads and verifies only the releases it
  publishes.
- **G2.** Choosing them takes a small, fixed number of GitHub API calls per
  product, about one per 100 releases.
- **G3.** The `build-index` job takes **under 5 minutes** on the first live
  `publish-index` run after the fix is released.

## 4. Non-goals and anti-goals

- **Non-goals:** the Pages deployment that reports success but never goes live
  (vergil-project/packages#46); a pruning policy for old releases, artifacts and
  evidence; the full release scan in `published.py`.
- **Anti-goal:** weakening the trust checks on what is published. Every
  published package is still attestation-verified pinned to the release
  workflow on `main`, and every published `.rpm` is still signature-checked.

## 5. Design

### 5.1 Inventory

For each product, list its releases with their asset names in one paginated
call: `gh api --paginate --slurp "repos/<product>/releases?per_page=100"`.
`--paginate` alone prints each page as its own JSON array, back to back, which
one `json.loads` cannot parse. `--slurp` wraps the pages in one outer array,
which the code flattens. The REST listing gives each release's `tag_name`,
`draft`, `prerelease` and `assets[].name`. It replaces `stable_tags()` and the
per-tag `gh release view` calls for the index, and goes through `_invoke` like
every other `gh` call. Its test uses a listing of more than one page.

A release is a **packaged stable release** when its tag matches `vX.Y.Z`, it is
not a draft or prerelease, and its assets include `packages-manifest.json`.
Other releases are ignored, with no log line.

### 5.2 Selection

For each product, group its packaged stable releases by major.minor, take the
newest `lines` lines, and in each the newest `keep` releases. This is today's
`retention._by_lines`, run on the inventory instead of on downloaded artifacts.

### 5.3 Download, verify, index

1. Download, manifest-match and read the metadata of the selected releases
   only, with the existing `_release_artifacts`.
2. Drop every artifact whose package name is in `[retention].retire`.
3. Verify what remains with the existing `collect.verify`.
4. Write, sign and size-check the index from it, as today.

### 5.4 What is removed

These parts of `retention.py` only made sense when the index held every
release, and are deleted with their tests:

- **The dependency closure** (`_close`). Every provider it could add is already
  selected: the vergil-tooling package depends on the vergil-python runtime,
  and the selected vergil-python release ships it.
- **The duplicate-bytes check** (`_check_conflicts`). With one release per line,
  two selected releases cannot share a package version, and the `vrg-release`
  preflight (#3311) refuses a re-published version before release.
- **The retire checks** (`_check_retire_matches`, `_check_retired`). `retire`
  stays as a plain filter.

`collect.collect`'s full scan is replaced. `collect.stable_tags`,
`collect.release_packages` and `published.py` stay as they are for the
`vrg-release` preflight.

## 6. Delivery

| # | Task | Repo | Kind | Blocked by |
|---|---|---|---|---|
| 1 | Retention-first index build, with tests | vergil-tooling | implementation | — |
| 2 | Release vergil-tooling with the fix | vergil-tooling | deployment | 1 |
| 3 | Live `publish-index` run meets G3 | packages | validation | 2 |

Deployment: packages calls
`vergil-actions/.github/workflows/publish-index.yml@v2.1` and pins
`[dependencies] vergil = "v2.1"`, and `build-index` installs vergil-tooling
from git at that pin. So the next vergil-tooling `v2.1.x` release is picked up
with no pin change. The release itself is a human action.

The validation checks the `publish-index` run that the release triggers: its
`build-index` job takes under 5 minutes, and the new vergil-tooling version is
in the published noble `Packages` file.

Bookends: the documentation task (this spec and the plan), the
documentation-review sweep, which also corrects the order in epic #356's spec,
and the retrospective.

## 7. Left to the builder

- How the inventory is shaped in code.
- How `collect` and `retention` are split or reused, provided `published.py`'s
  public functions keep their behaviour.
- Test structure, using the existing `Run` injection and fixtures.
