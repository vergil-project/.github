# Package index: retention-first build — implementation plan

- **Epic:** vergil-project/.github#1003
- **Planning issue:** vergil-project/.github#1002 (review history is in its comments)
- **Status:** draft, in design review

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `vrg-package index` downloads and verifies only the releases it publishes, so `build-index` takes minutes, not 20 to 26.

**Architecture:** One paginated release listing per product gives the packaged stable tags. Per-line retention picks tags from that list. Only those releases are downloaded, retired names are dropped, the rest is verified and indexed. The dependency closure, the duplicate-bytes check and the retire checks are deleted.

**Tech Stack:** Python 3, `gh` CLI (through the existing `Run` injection), pytest.

**Spec:** `epics/1003-index-retention-first/spec.md`

## Global Constraints

- Every published package is attestation-verified pinned to the release workflow on `main`, and every published `.rpm` is signature-checked (`collect.verify`, unchanged).
- The release listing is `gh api --paginate --slurp "repos/<product>/releases?per_page=100"`, through `_invoke`.
- A packaged stable release: tag matches `vX.Y.Z`, not a draft, not a prerelease, assets include `packages-manifest.json`.
- `collect.stable_tags`, `collect.release_packages` and `published.py` keep their current behaviour (the `vrg-release` preflight uses them).
- Validation is `vrg-container-run -- vrg-validate` only.

## Review Focus

- A product with more than 100 releases: the listing spans pages, and every page's releases are read. Pinned by `test_packaged_tags_reads_every_page` (Task 1).
- A line whose newest release is unpackaged: the newest *packaged* release in that line is selected. Pinned by `test_collect_selects_newest_packaged_release_in_a_line` (Task 1).
- Lines compared numerically (`v2.10` newer than `v2.9`). Pinned by `test_select_tags_lines_compare_numerically` (Task 1).
- Retired names dropped before verification, so a retired package is never verified or indexed. Pinned by `test_build_site_runs_every_step_in_order` (Task 1).
- A release outside the selection is never viewed or downloaded. Pinned by `test_collect_downloads_only_selected_releases` (Task 1).

---

### Task 1: Retention-first index build

**Files:**
- Modify: `src/vergil_tooling/lib/package/index/collect.py` (add `packaged_tags`; `collect` downloads selected tags only; remove the no-manifest branch of `_release_artifacts` and `collect`; module docstring)
- Modify: `src/vergil_tooling/lib/package/index/retention.py` (replace `select` with `select_tags` and `drop_retired`; delete `_check_conflicts`, `_close`, `_check_retire_matches`, `_check_retired`; keep `version_key`, which `apt.py` uses; module docstring)
- Modify: `src/vergil_tooling/lib/package/index/site.py:155-160` (new call order; module docstring)
- Modify: `docs/site/docs/reference/package-config.md` (the `[retention]` section: selection, download, retire filter, verify; no closure, no retire failures)
- Test: `tests/vergil_tooling/test_package_index_collect.py`, `tests/vergil_tooling/test_package_index_retention.py`, `tests/vergil_tooling/test_package_index_site.py`

**Interfaces:**
- Produces:
  - `collect.packaged_tags(product: str, gh: Run = local_run) -> list[str]`: the packaged stable release tags of `product`, from the paginated listing (`--slurp` output is a list of pages; flatten it).
  - `retention.select_tags(tags: list[str], keep: int, lines: int) -> list[str]`: the newest `lines` major.minor lines among `tags`, the newest `keep` in each, newest first. A non-`vX.Y.Z` tag raises `PackageError`.
  - `retention.drop_retired(artifacts: list[Artifact], retire: frozenset[str]) -> list[Artifact]`: `artifacts` without any whose `name` is in `retire`, in input order.
  - `collect.collect(cfg: IndexConfig, workdir: Path, gh: Run = local_run) -> list[Artifact]`: unchanged signature; for each product, `_release_artifacts` for each tag in `select_tags(packaged_tags(product), cfg.keep, cfg.lines)` only.
  - `site.build_site` order: `collect.collect` → `retention.drop_retired` → `collect.verify` → apt/rpm write → sign → size guard.

- [ ] **Step 1: Write the failing tests**

In `test_package_index_collect.py`, teach `_FakeGh` to answer `gh api --paginate --slurp repos/<product>/releases?per_page=100` from `releases`, returning pages of `page_size` releases (a new keyword, default 100), each release as `{"tag_name", "draft", "prerelease", "assets": [{"name"}]}`. Then add:

```python
def test_packaged_tags_reads_every_page() -> None:
    rel = [(f"v1.0.{i}", ["packages-manifest.json", f"p_1.0.{i}-1_amd64.deb"]) for i in range(5)]
    gh = _fake_gh({"o/p": rel}, page_size=2)
    assert collect.packaged_tags("o/p", gh) == [f"v1.0.{i}" for i in range(5)]

def test_packaged_tags_skips_unpackaged_draft_prerelease_and_unstable() -> None:
    gh = _fake_gh(
        {"o/p": [("v1.0.0", []), ("v1.0.1", [MANIFEST]), ("v1.0.2", [MANIFEST]),
                 ("v1.0.3", [MANIFEST]), ("develop-1", [MANIFEST])]},
        drafts=("v1.0.2",), prereleases=("v1.0.3",),
    )
    assert collect.packaged_tags("o/p", gh) == ["v1.0.1"]

def test_collect_downloads_only_selected_releases(tmp_path: Path) -> None:
    # keep=1, lines=1 over v2.0.1, v2.1.0, v2.1.1 (all packaged)
    # assert every "gh release view"/"gh release download" call names v2.1.1 only

def test_collect_selects_newest_packaged_release_in_a_line(tmp_path: Path) -> None:
    # keep=1, lines=1: v2.1.0 packaged, v2.1.1 without a manifest
    # assert v2.1.0's packages are collected and v2.1.1 is never viewed or downloaded
```

Change these existing collect tests:

- `test_release_without_manifest_is_ignored`: becomes a `packaged_tags` test that a release without the manifest is not returned (no log line is asserted).
- The three retry tests (`test_transient_gh_release_blip_is_retried_then_succeeds`, `test_persistent_transient_gh_release_failure_is_fatal_after_budget`, `test_answer_type_gh_release_failure_is_not_retried`): the `"gh release list"` fault-injection prefix becomes `"gh api"`, so the transient-error retry is tested on the new listing call.

In `test_package_index_retention.py`, replace the `select` tests with:

```python
def test_select_tags_keep_per_line_newest_lines() -> None:
    tags = ["v1.0.0", "v2.0.0", "v2.0.1", "v2.1.0", "v2.1.1", "v2.1.2"]
    assert retention.select_tags(tags, keep=2, lines=2) == ["v2.1.2", "v2.1.1", "v2.0.1", "v2.0.0"]

def test_select_tags_lines_compare_numerically() -> None:
    assert retention.select_tags(["v2.9.0", "v2.10.0"], keep=1, lines=1) == ["v2.10.0"]

def test_select_tags_counts_only_given_tags() -> None:
    assert retention.select_tags(["v2.0.5", "v2.1.0"], keep=1, lines=2) == ["v2.1.0", "v2.0.5"]

def test_select_tags_rejects_a_non_stable_tag() -> None:
    with pytest.raises(PackageError, match="develop-1"):
        retention.select_tags(["develop-1"], keep=1, lines=1)

def test_drop_retired_removes_every_fmt_and_arch() -> None:
    # artifacts named "rt" (deb amd64, rpm arm64) and "app"; retire {"rt"} -> only "app" remains
```

Delete every test of `select`, the closure, the duplicate-bytes check and the retire checks: every test in the file except `test_full_version` and the new ones above.

In `test_package_index_site.py`, replace `fake_select` with `fake_drop_retired(arts, retire)` recording `("drop_retired", sorted(retire))`, and set the expected event order in `test_build_site_runs_every_step_in_order` to collect, drop_retired, verify, then the existing apt, rpm, sign and size-guard events.

Existing collect tests that call `collect()` over several releases and expect all of them set `keep` and `lines` in `_cfg` high enough that every release is selected.

- [ ] **Step 2: Run validation to see the new tests fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL on the new tests (`packaged_tags`, `select_tags`, `drop_retired` not defined; site event order).

- [ ] **Step 3: Implement**

`collect.packaged_tags`: `_json` on the listing, flatten the pages, keep packaged stable releases in listing order. `collect.collect`: per product, `retention.select_tags(packaged_tags(product, gh), cfg.keep, cfg.lines)`, then `_release_artifacts` for each. `retention.select_tags`: today's `_by_lines` logic on tags. `retention.drop_retired`: a list filter. `site.build_site`: the new order above. Delete the removed functions and any import they leave unused. Only packaged tags reach `_release_artifacts` now, so it returns `list[Artifact]` and its no-manifest branch, and `collect`'s `is not None` check, are removed. Rewrite the three module docstrings and the `[retention]` section of `package-config.md` to describe selection, download, the retire filter and verification.

- [ ] **Step 4: Run validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
vrg-git add src/vergil_tooling/lib/package/index tests/vergil_tooling docs/site/docs/reference/package-config.md
vrg-commit --type fix --scope package-index --message "select index releases before downloading and verifying them"
```

### Task 2: Release vergil-tooling with the fix (deployment)

Blocked by Task 1. The release is human-gated: the human runs the vergil-tooling release once Task 1 merges. The agent does not cut it.

- [ ] **Precondition:** the human has cut a vergil-tooling `v2.1.x` release after Task 1 merged (`vrg-gh release list --repo vergil-project/vergil-tooling --limit 1`). If not, comment "blocked: preconditions not met" and stop.
- [ ] **Procedure:** packages calls `publish-index.yml@v2.1` and pins `vergil = "v2.1"`, so the release is picked up with no pin change. Find the run it triggered: in `vrg-gh run list --repo vergil-project/packages --workflow publish-index.yml --json databaseId,event,displayTitle,createdAt`, the `repository_dispatch` run whose `displayTitle` names `vergil-project/vergil-tooling` and whose `createdAt` is after the release's `publishedAt`.
- [ ] **Acceptance:** that run exists. Record its run ID in the result comment for Task 3.

### Task 3: Live `publish-index` run meets G3 (validation)

Blocked by Task 2.

- [ ] **Precondition:** the run whose ID Task 2 recorded has finished.
- [ ] **Check 1:** its `build-index` job took under 5 minutes: `vrg-gh run view <run-id> --repo vergil-project/packages --json jobs`, comparing `startedAt` and `completedAt` of the `build-index` job.
- [ ] **Check 2:** the new vergil-tooling version appears in `https://vergil-project.github.io/packages/deb/dists/noble/main/binary-amd64/Packages`. If it does not because the deployment did not go live, that is vergil-project/packages#46: record FAILURE with that reference.
- [ ] **Outcome:** SUCCESS when both checks pass.
