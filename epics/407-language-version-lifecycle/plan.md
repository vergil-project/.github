# Language version lifecycle: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace every hand-kept language version list with a rolling window
that is derived from upstream metadata. A scheduled check opens the add and drop
PRs for Python and Go. Consumers follow the window by default.

**Architecture:**

- **`vergil-containers`** gains a version manifest (`docker/versions/`) as the
  single source of truth.
  - A generator writes every derived list from it, plus a published
    `window.json`.
  - A lifecycle check in the existing `ops.yml` runs per-language adapters
    against upstream. It reports on every language, opens add/drop PRs for
    Python and Go, and opens issues for everything else.
- **`vergil-tooling`** gains the single resolver (`vrg-ci-resolve`), which reads
  the published window. `vrg-validate` and `vergil-actions`' `derive.sh` both
  call it.
- **`vergil-python`** publishes one runtime package per `x.y`. (As built,
  2026-10-23: it publishes one series at a time, now 3.15, with 3.14 frozen at
  `3.14.8+20261013`; see spec §7 and `vergil-project/vergil-python#36`.)
- A per-repo sweep removes the hand-kept `[ci].versions` lists.

**Tech Stack:**

- **`vergil-containers`:** Python 3 scripts that use PyYAML and the standard
  library, run in the `prod-base` image, in the style of `docker/pins/`. Tests
  use `pytest` with injected fetchers and never touch the network.
- **`vergil-tooling`:** Python 3.12+ with `pytest` and 100% branch coverage.
- **`vergil-actions`:** bash with `*.test.sh` tests.
- **External data:** the endoflife.date API, `gcc.gnu.org`, the GitHub releases
  API for `llvm/llvm-project`, and GitHub App tokens (the `bump-tools.yml`
  pattern).

**Spec:** `epics/407-language-version-lifecycle/spec.md`, in the same
directory. Read it before starting any task. Section references (§N) point
into it.

## Global Constraints

- Validation is exactly `vrg-container-run -- vrg-validate`. For the TDD loop
  only, run focused tests with `vrg-container-run -- uv run pytest <path> -v`
  in `vergil-tooling`. In `vergil-containers`, the `docker/versions/` tests
  run the same way the pins job runs (`uv run --with pytest --with pyyaml
  python -m pytest -q`). Wire them into `ci.yml` so `vrg-validate` and CI both
  execute them.
- `vergil-tooling` enforces **100% branch coverage** on every interpreter it
  tests.
- Use git and GitHub only through `vrg-git` and `vrg-gh`. Commit with
  `vrg-commit --type <t> --scope <s> --message <m>`.
- **One PR per task, in the repo named in the task heading** (the placement
  law). No PR `Closes` an issue in another repo.
- **Fail loud.** No fetch, parse or lookup failure may become "in sync", "no
  change" or a silent fallback (§6.2, §8.3). The published window is **never**
  silently replaced by a cached copy. It is overridable only by setting
  `VRG_VERSION_WINDOW=<path>` explicitly.
- **Shared code has no language branches** (principle 7, §8.3).
  - In `vergil-containers`, language specifics live only in
    `docker/versions/lifecycle/adapters/<lang>.py`.
  - In `vergil-tooling`, they live only in
    `lib/versions/extensions/<lang>.py`.
  - The language-name literals allowed in shared code are the registry keys,
    and nothing else.
- Versions are compared at `x.y`, or at the major for whole-number languages,
  by semantic-version order (principle 8). Use the existing `_version_sort_key`
  semantics. Family prefixes (`clang-`, `gcc-`, `node-`) are ignored for
  ordering.
- **No backfill** (principle 12). A target version older than a family's
  current oldest manifest version is never proposed as an add. This is enforced
  once, in `lifecycle/core.py`.
- File formats are YAML for hand-edited data (the manifest) and JSON for
  generated machine output (`window.json`, resolver output). Never XML.
- The published window URL is
  `https://raw.githubusercontent.com/vergil-project/vergil-containers/<ref>/docker/versions/window.json`,
  where `<ref>` is the consumer's `[dependencies].vergil` (for example `v2.1`).
- Error messages from the resolver name the offending version, the window, and
  the fix (for example: `remove it from [ci].versions or update the pin`).
- Site docs (`docs/site/…`) are edited only where a task names them. The
  documentation-review bookend (`vergil-containers#610`) sweeps the rest.

## Review Focus

These are conditions the spec implies but that no task's ordinary tests would
exercise, most likely to bite first. Each has a pinned test in the task named.

1. **The endoflife.date response changes shape or fields go missing.** Examples:
   `eol: false` versus a date, `lts` as a date string, or an absent
   `releaseDate`. The adapter must fail loud for that language and never emit
   an empty target, which would read as "drop everything". Tests:
   - Task 3: `test_endoflife_rejects_missing_cycle_field`,
     `test_endoflife_parses_eol_false_and_date`.
   - Task 3: `test_empty_target_is_an_error_not_a_drop_all`.
2. **An upstream version listed before its GA date.** Examples are a future
   `releaseDate` or a pre-release cycle. It must not be proposed as an add
   (principle 3). Test: Task 3 `test_future_release_date_is_not_ga`.
3. **A repo that is offline, or has a `[dependencies].vergil` ref with no
   `window.json`** (an old tag from before this epic). Resolution must fail
   loud, naming the URL and the `VRG_VERSION_WINDOW` override. It must not
   fall back to `_DEFAULT_VERSIONS`. Test: Task 8
   `test_load_window_missing_at_ref_fails_loud`.
4. **A packaged repo whose newest window version has no runtime,** and an
   older one does. Resolution must pick the older one. When **no** window
   version has a runtime, resolution must fail loud. Tests: Task 9
   `test_python_packaged_falls_back_to_newest_with_runtime`,
   `test_python_packaged_no_runtime_fails_loud`.
5. **Two scheduled runs before a bot PR is merged.** The second run must
   refresh the same PR and branch, not open a duplicate. Likewise, a
   failing-check issue that already exists is refreshed, not duplicated, and
   is closed on recovery. Tests: Task 6 `test_rerun_refreshes_existing_pr`,
   Task 5 `test_failure_issue_refreshed_then_closed`.

---

### Task 1: Version manifest, generator and drift gate [`vergil-containers`]

The manifest becomes the single source of truth, and every derived list is
generated from it. There is no behaviour change: the generated output must
reproduce today's matrix exactly.

**Files:**

- Create: `docker/versions/manifest.yml` (seeded with the exact current
  publish matrix: python 3.12/3.13/3.14, ruby 3.2/3.3/3.4, java 17/21,
  go 1.26, rust 1.92/1.93, cpp-clang 20/19, cpp-gcc 14/13, ts-node 24/22)
- Create: `docker/versions/manifest.py`, `docker/versions/generate.py`
- Create: `docker/versions/test_manifest.py`, `docker/versions/test_generate.py`
- Create (generated): `docker/versions/window.json`,
  `docker/versions/matrix.json`
- Modify:
  - `.github/workflows/cd-docker-publish.yml`: a `matrix` setup job emits
    `matrix.json`, and `build-scan-push` uses `fromJSON`, replacing the literal
    `include:` list at lines 56-108;
  - `docker/build.sh`: generated block between `# BEGIN generated: versions`
    and `# END generated: versions`;
  - `docker/pins/report_exposure.py:163-172`: `_MATRIX` is read from the
    manifest;
  - `.github/workflows/package-cleanup.yml`: generated package lists, which
    also adds the missing `cpp-clang`, `cpp-gcc` and `ts-node` packages;
  - every `docker/*/Dockerfile.template` `ARG <BUILD_ARG>=` default (the
    newest version per family);
  - the version tables in `README.md`, `CLAUDE.md`,
    `docs/site/docs/index.md` and `docs/site/docs/images/index.md`, as
    generated blocks between `<!-- BEGIN generated: versions -->` markers;
  - `.github/workflows/ci.yml`: the pins job gains `Versions are up to date`
    (`python3 docker/versions/generate.py --check`) and `Versions tooling
    tests`.

**Interfaces:**

- Produces, in `manifest.py`:

  ```python
  @dataclass(frozen=True)
  class ManifestVersion:
      version: str                          # "3.14", "20", "1.26"
      ga: date | None
      eol: date | None
      in_policy: bool
      runtime_published: bool | None        # None for languages with no runtime package

  @dataclass(frozen=True)
  class Family:
      key: str            # image suffix: "python", "cpp-clang", "ts-node"
      language: str       # vergil primary-language: "python", "cpp", "typescript"
      token_prefix: str   # "", "clang-", "gcc-", "node-"
      build_arg: str      # "PYTHON_VERSION", "CLANG_VERSION", "NODE_MAJOR"
      automation: Literal["pr", "report"]
      versions: tuple[ManifestVersion, ...]   # newest first

  @dataclass(frozen=True)
  class Manifest:
      families: tuple[Family, ...]

  def load_manifest(path: Path) -> Manifest: ...   # raises ManifestError (loud) on any schema violation
  def dump_manifest(m: Manifest) -> str: ...       # stable YAML, round-trips load_manifest
  ```

- Produces, in `generate.py`: `render_all(m: Manifest, root: Path) -> dict[Path, str]`
  (the full new content of every generated file) and `main(argv) -> int`.
  `--check` exits 1 and lists each file that is out of date.
- `window.json` schema (version 1):

  ```json
  {"schema": 1,
   "languages": {"python": {"families": [
     {"key": "python", "token_prefix": "",
      "versions": [{"version": "3.14", "eol": "2030-10-31",
                    "in_policy": true, "runtime_published": true}]}]}}}
  ```

  Versions are newest first. Every family in the manifest appears. Ruby is
  still present until Task 7.

- [ ] **Step 1: Write the manifest tests.**
  - `test_load_manifest_seed_matches_current_matrix`: asserts the loaded
    families and versions equal the literal list in
    `cd-docker-publish.yml:56-108` at the start of the task.
  - `test_load_manifest_rejects_unknown_key`, `test_rejects_duplicate_version`,
    `test_rejects_versions_not_newest_first`, `test_rejects_bad_automation`:
    each raises `ManifestError` naming the family and field.
  - `test_dump_roundtrip`: `load(dump(m)) == m`.
- [ ] **Step 2: Run** `uv run --with pytest --with pyyaml python -m pytest -q docker/versions`. Expected: FAIL (module missing).
- [ ] **Step 3: Implement `manifest.py`** and the seed `manifest.yml`, with
  `in_policy: true` everywhere for now. Task 3's report computes policy; the
  seed only records what's built. Set `runtime_published: true` for python
  3.14, `false` for 3.12/3.13 (only 3.14.8 is published today), and `null`
  for other languages.
- [ ] **Step 4: Write the generator tests.**
  - `test_render_matrix_json_equals_current_include_list`: the generated
    `matrix.json` equals today's `include:` entries, with the same
    language/version/build-arg triples.
  - `test_render_build_sh_block`, `test_render_arg_defaults_newest`,
    `test_render_package_cleanup_includes_all_families`, and
    `test_render_docs_tables`.
  - `test_window_json_schema`: asserts the exact shape above for python and
    cpp, including `token_prefix: "clang-"`.
  - `test_check_mode_reports_drift`: after one generated file is mutated,
    `main(["--check"]) == 1` and the output names that file.
- [ ] **Step 5: Implement `generate.py`.** Replace the literal lists in the
  listed files with generated blocks, run `generate.py` (write mode), and
  commit the output.
- [ ] **Step 6: Run the tests,** then `vrg-container-run -- vrg-validate`.
  Expected: PASS. Also run `python3 docker/versions/generate.py --check`,
  which should exit 0.
- [ ] **Step 7: Commit.** `vrg-commit --type feat --scope versions --message "generate every version list from one manifest"`.

### Task 2: Per-language support policies [`vergil-containers`]

**Files:**

- Create: `docs/site/docs/languages/lifecycle.md` (the shared rules:
  principles 1-13, written for readers) and
  `docs/site/docs/languages/<key>.md` for `python`, `go`, `rust`, `java`,
  `cpp-clang`, `cpp-gcc` and `ts-node`
- Modify: the docs-site nav (`docs/site/mkdocs.yml`)

Each policy page has exactly these headings:

1. **Upstream source of truth:** an official URL, plus the endoflife.date
   product where one exists.
2. **Support rule:** the window.
3. **What counts as GA.**
4. **Adapter quirks.**
5. **Deployed runtime package:** yes or no.
6. **Automation:** `pr` or `report`.
7. **Current out-of-policy versions.**

Use the spec §5 table and §5.1 as the content. Confirm each upstream fact
against the official page and cite it with its URL (data versus judgment, as in
spec §11):

| Family | Window rule (exact) |
|---|---|
| python | every minor version that is GA and not EOL, subject to no-backfill |
| go | the two newest releases |
| rust | newest and newest−1 |
| java | LTS releases that are not EOL (no-backfill keeps 11 out) |
| ts-node | LTS (even-numbered) majors that are not EOL (As built, 2026-10-23: the adapter reads the LTS status itself, not the major's parity, since from Node.js 27 every major becomes LTS; see `vergil-project/vergil-containers` PR #627.) |
| cpp-clang | the newest LLVM major and newest−1 that `apt.llvm.org` publishes for the Debian base |
| cpp-gcc | release lines gcc.gnu.org lists as maintained, limited to Debian base packages |

The Clang and GCC rules are judgment calls by the policy author. Mark them as
such on the page.

- [ ] **Step 1:** Write the eight pages. Each cites its official source URL,
  with the date it was checked.
- [ ] **Step 2:** Run `vrg-container-run -- vrg-validate`, which covers
  markdownlint and the docs build. Expected: PASS.
- [ ] **Step 3: Commit.** `docs(languages): add the per-language support policies`.

### Task 3: Lifecycle core, upstream sources, and the Python and Go adapters [`vergil-containers`]

**Files:**

- Create:
  - `docker/versions/lifecycle/__init__.py`, `core.py`, `sources.py` and
    `report.py`;
  - `docker/versions/lifecycle/adapters/__init__.py` (the registry), `python.py`
    and `go.py`;
  - `docker/versions/lifecycle/check.py`, the CLI entry point;
  - `docker/versions/test_lifecycle_core.py`, `test_sources.py` and
    `test_adapters_python_go.py`;
  - fixtures: `docker/versions/fixtures/endoflife-{python,go}.json`, which
    are copies of the 2026-10-17 responses.

**Interfaces:**

- Consumes: `load_manifest`, `Family` and `ManifestVersion` (Task 1).
- Produces, in `core.py`:

  ```python
  @dataclass(frozen=True)
  class UpstreamRelease:
      version: str; ga: date | None; eol: date | None; lts: bool

  @dataclass(frozen=True)
  class TargetVersion:
      version: str; ga: date | None; eol: date | None; installable: bool

  class Adapter(Protocol):
      key: str                                    # manifest family key
      def fetch(self) -> list[UpstreamRelease]: ...        # raises SourceError
      def target(self, releases: list[UpstreamRelease], today: date) -> list[TargetVersion]: ...
      def extra_changes(self, change: "Change") -> list[str]: ...   # manual follow-ups for the PR body

  ChangeKind = Literal["add", "drop", "eol_soon", "blocked", "out_of_policy"]

  @dataclass(frozen=True)
  class Change:
      family: str; kind: ChangeKind; version: str; detail: str

  @dataclass(frozen=True)
  class FamilyResult:
      family: str; changes: tuple[Change, ...]; error: str | None

  def classify(family: Family, target: list[TargetVersion], today: date,
               lookahead_days: int) -> list[Change]: ...
  def run(manifest: Manifest, adapters: Mapping[str, Adapter], today: date,
          lookahead_days: int = 90) -> list[FamilyResult]: ...
  ```

- `classify` rules, all shared:
  - **No backfill:** first drop any target older than the family's oldest
    manifest version.
  - `add`: a target version that is GA and `installable`, and not in the
    manifest.
  - `blocked`: a target version that is GA but not `installable`.
  - `drop`: a manifest version absent from the target, or past `eol` on
    `today`.
  - `eol_soon`: an `eol` within `lookahead_days`.
  - `out_of_policy`: a `drop` on a family whose `automation == "report"`.
    These are reported and not removed (spec §6.1).
- **An empty target is a `SourceError`**, never a "drop everything".
- `run` catches an adapter's `SourceError` or `ValueError` and records it in
  `FamilyResult.error`. It keeps going with the other families and never
  swallows the error.
- In `sources.py`:
  - `endoflife(product: str, fetch: Callable[[str], bytes]) -> list[UpstreamRelease]`
    parses `https://endoflife.date/api/<product>.json`.
  - `fetch_url(url: str) -> bytes` is the only real network call, in the
    `resolve_latest.py` style.
- Generic window helpers in `core.py`, which adapters compose:
  - `ga_not_eol(rs, today)`;
  - `newest_n(rs, n)`;
  - `lts_only(rs)`.
- Adapters:
  - `python.py`: `endoflife("python")` plus `ga_not_eol`. `installable`
    checks that the `python:<v>-slim` tag exists, through an injected
    `tag_exists(image, tag) -> bool` backed by the Docker Hub tags API.
  - `go.py`: `endoflife("go")` plus `newest_n(2)`, with `golang:<v>` tag
    existence. Its `extra_changes` returns a follow-up line for each
    `docker/go/Dockerfile.template` `case go<v>*)` branch and each `pins.yml`
    constraint that names a dropped version.
- `check.py`: `main(argv) -> int` with `--report <path>` (Markdown output) and
  `--json <path>` (the list of `FamilyResult`). It exits 1 when any
  `FamilyResult.error` is set, after writing both outputs.

- [ ] **Step 1: Write the source tests.**
  - `test_endoflife_parses_eol_false_and_date`: `eol: false` gives `None`, a
    date string gives a `date`.
  - `test_endoflife_rejects_missing_cycle_field`: raises `SourceError` naming
    the product.
  - `test_endoflife_http_error_raises`.
- [ ] **Step 2: Write the core tests,** using a hand-built `Family` and targets.
  - `test_add_when_ga_target_missing` (python 3.15 GA on 2026-10-09 →
    `add 3.15`).
  - `test_no_backfill_skips_older_than_oldest`: a target containing 3.11 with
    manifest oldest 3.12 gives no `add 3.11`.
  - `test_drop_past_eol`.
  - `test_future_release_date_is_not_ga`: a 3.16 release on 2027-10-01, with
    today 2026-10-21, gives no add.
  - `test_blocked_when_not_installable`.
  - `test_report_family_drop_is_out_of_policy`.
  - `test_eol_soon_within_lookahead`.
  - `test_empty_target_is_an_error_not_a_drop_all`.
  - `test_run_records_adapter_error_and_continues`: two families, the first
    raises; its result has `error` set, and the second has changes.
- [ ] **Step 3: Write the adapter tests,** from the fixtures with today =
  2026-10-21.
  - Python: the target is {3.15, 3.14, 3.13, 3.12, 3.11}, and against the
    seed manifest `classify` gives exactly `[add 3.15]`.
  - Go: the target is {1.27, 1.26}, giving exactly `[add 1.27]`.
  - `test_go_extra_changes_names_case_branch`: dropping 1.26 lists the
    `case go1.26*)` follow-up.
- [ ] **Step 4: Run.** Expected: FAIL.
- [ ] **Step 5: Implement** `core.py`, `sources.py`, the two adapters, the
  registry (`ADAPTERS: dict[str, Adapter]` keyed by family) and `check.py`.
  Add a grep-based test `test_core_has_no_language_literals`. It asserts that
  `core.py`, `sources.py` and `report.py` contain none of
  `python|go|rust|java|clang|gcc|node|ruby` as string literals.
- [ ] **Step 6:** Run the tests and `vrg-container-run -- vrg-validate`.
  Expected: PASS.
- [ ] **Step 7: Commit.** `feat(versions): add the lifecycle check core with the Python and Go adapters`.

### Task 4: Report-only adapters (Rust, Java, Node, Clang, GCC) [`vergil-containers`]

**Files:**

- Create: `docker/versions/lifecycle/adapters/{rust,java,node,clang,gcc}.py`,
  `docker/versions/test_adapters_report_only.py`, and fixtures
  (`endoflife-{rust,eclipse-temurin,nodejs}.json`, `gcc-releases.html`,
  `llvm-releases.json`, `apt-llvm-trixie-index.html`)
- Modify: `docker/versions/lifecycle/sources.py`, adding
  `gcc_releases(fetch)` (parses `https://gcc.gnu.org/releases.html` and the
  per-series "no longer supported" marker) and `llvm_releases(fetch)`
  (GitHub releases API, final releases only)
- Modify: `docker/versions/lifecycle/adapters/__init__.py` (registry)

**Interfaces:** Each class implements `Adapter` (Task 3), applying the window
rules listed in Task 2.

- `installable` for clang checks the `apt.llvm.org` trixie index for
  `llvm-toolchain-trixie-<n>`.
- `installable` for gcc checks the Debian trixie package list for
  `gcc-<n>`. A newer upstream that isn't installable surfaces as `blocked`.

- [ ] **Step 1: Write the tests,** with today = 2026-10-21.
  - Against the seed manifest, each family produces exactly these changes:
    - **rust:** `add 1.99`, `add 1.98`, and `out_of_policy` 1.92 and 1.93.
    - **java:** `add 25`, and 17/21 in policy.
    - **ts-node:** `add 26` only once LTS (fixture with today = 2026-10-29),
      and none before.
    - **cpp-clang:** `out_of_policy` 19/20 and `add`s for the installable
      newest two. If trixie lacks 23, that's `blocked 23`.
    - **cpp-gcc:** `out_of_policy 13` (or `drop 13` once Task 7 is in) and
      `blocked 15/16`.
  - `test_gcc_parser_rejects_page_without_series_table`: raises
    `SourceError`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement** the sources and the five adapters.
- [ ] **Step 4:** Run the tests and `vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(versions): add the report-only language adapters`.

### Task 5: Scheduled lifecycle job: report, issues and failure issues [`vergil-containers`]

**Files:**

- Modify: `.github/workflows/ops.yml`, adding a job `language-lifecycle`.
  - It has **no** `continue-on-error`.
  - It runs in `prod-base`, mints an App token with
    `actions/create-github-app-token@v3`, exactly as in `bump-tools.yml:43-53`,
    and runs `python3 docker/versions/lifecycle/check.py --report … --json …`
    and then `python3 docker/versions/lifecycle/publish.py`.
  - It uploads the report as a job summary and an artifact.
- Create: `docker/versions/lifecycle/publish.py`,
  `docker/versions/test_publish.py`

**Interfaces:**

- Consumes: the `FamilyResult` JSON (Task 3).
- Produces, in `publish.py`:
  - `sync_issues(results, gh: GhClient) -> None`, where `GhClient` is a
    protocol with `find_open_issue(title) -> int | None`,
    `create_issue(title, body, labels) -> int`,
    `update_issue(number, body) -> None` and
    `close_issue(number, comment) -> None`.
  - The real client wraps `gh`. Tests inject a fake.
- Exact issue titles:
  - `language lifecycle: <family> <kind> <version>`, one per `Change` on a
    `report` family, with the label `language-lifecycle`;
  - `lifecycle check failing for <family>`, one per family with `error`,
    whose body is the error text.
- A failure issue is closed, with the comment `recovered in run <run-url>`, on
  the next run where that family has no error.
- An issue for a change that no longer appears is closed with
  `no longer reported`.

- [ ] **Step 1: Write the tests.**
  - `test_report_family_change_opens_issue`.
  - `test_existing_issue_is_updated_not_duplicated`.
  - `test_failure_issue_refreshed_then_closed`: run 1 with an error creates
    the issue, run 2 with an error updates the same number, and run 3 clean
    closes it.
  - `test_stale_change_issue_closed`.
  - `test_pr_family_changes_open_no_issue`: python and go are left to Task 6.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement** `publish.py` and the `ops.yml` job.
  - The job's final step exits non-zero when `check.py` returned 1. Use
    `set -o pipefail`, and **don't** mask the exit code (see the memory note
    "don't mask exit codes").
- [ ] **Step 4:** Run the tests and `vrg-validate`, which includes
  actionlint. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(ops): run the language lifecycle check nightly with issues for gaps and failures`.

### Task 6: Add/drop PR automation for `pr` families [`vergil-containers`]

**Files:**

- Create: `docker/versions/lifecycle/apply.py`, `docker/versions/test_apply.py`
- Modify:
  - `docker/versions/lifecycle/publish.py`: adds `sync_prs`;
  - `docker/versions/manifest.yml`: sets `automation: pr` for `python` and
    `go` (all other families stay `report`);
  - `.github/workflows/ops.yml`: the job gets `contents: write` and
    `pull-requests: write`, through the App token.

**Interfaces:**

- Consumes: `Change`, `Adapter.extra_changes` (Task 3), `dump_manifest`,
  `render_all` (Task 1), `GhClient` (Task 5), extended with
  `find_open_pr(head) -> int | None`, `create_pr(head, title, body) -> int`
  and `update_pr(number, body) -> None`.
- Produces:
  - `apply_change(m: Manifest, change: Change, target: TargetVersion) -> Manifest`.
    An `add` inserts a `ManifestVersion` with `in_policy=True`, the `ga` and
    `eol` dates, and `runtime_published=False` for families with a runtime.
    A `drop` removes the version.
  - Branch: exactly `lifecycle/<family>-<kind>-<version>`, for example
    `lifecycle/python-add-3.15`.
  - PR title: exactly `feat(versions): add <family> <version>` or
    `feat(versions): drop <family> <version> (EOL <date>)`.
  - PR body: the change, the upstream evidence URL, the regenerated-file
    list, and an `extra_changes` checklist (`- [ ] …`).
- **One PR per change.** A rerun force-pushes the regenerated branch and
  updates the body, and never opens a second PR.

- [ ] **Step 1: Write the tests.**
  - `test_apply_add_python_315_inserts_newest_first`.
  - `test_apply_drop_removes_version`.
  - `test_apply_then_render_regenerates_all_files`: the manifest and every
    Task 1 output changed consistently, and `--check` is clean.
  - `test_rerun_refreshes_existing_pr`.
  - `test_drop_pr_body_lists_go_case_followup`.
  - `test_report_family_never_gets_pr`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement** `apply.py`, `sync_prs` and the workflow
  permissions. Push with the App token, so that the PR triggers CI
  (`bump-tools.yml:15-16`).
- [ ] **Step 4:** Run the tests and `vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(versions): open add and drop PRs for Python and Go`.

### Task 7: Drop Ruby and GCC 13 [`vergil-containers`]

**Files:**

- Modify: `docker/versions/manifest.yml` (remove the `ruby` family and
  `cpp-gcc` 13), then regenerate.
- Delete: `docker/ruby/`.
- Modify:
  - remove the Ruby `.trivyignore` entries (lines 18-23 area, and any others
    that name `dev-ruby`/`prod-ruby`);
  - `package-cleanup.yml` is regenerated, and the existing `ruby` packages
    are left in GHCR (spec §8.2);
  - `docs/site/docs/images/index.md` prose that describes Ruby.

- [ ] **Step 1: Write the tests.**
  - `test_window_has_no_ruby`.
  - `test_cpp_gcc_window_is_14_only`.
- [ ] **Step 2:** Edit the manifest, regenerate, delete `docker/ruby/`, and
  remove the Ruby trivy ignores.
- [ ] **Step 3:** Run the tests and `vrg-validate`. Expected: PASS, with
  `grep -ri ruby docker/ .github/` showing only CHANGELOG history.
- [ ] **Step 4: Commit.** `feat(versions)!: drop Ruby and GCC 13 from the image matrix`.

### Task D1: Deployment: release `vergil-containers` so that `window.json` is published at `v2.1` [`vergil-containers`, `--kind deployment`]

**Blocked by:** Tasks 1, 7.

**Precondition (human-attested):** a `vergil-containers` release that contains
Tasks 1 and 7 has been cut and promoted to `v2.1`.

**Procedure:**

1. `curl -fsSL https://raw.githubusercontent.com/vergil-project/vergil-containers/v2.1/docker/versions/window.json`
   returns schema 1, with no `ruby` and `cpp-gcc` = [14].

**Acceptance:**

- The fetch succeeds.
- The nightly `cd-docker-publish` matrix (now driven by `fromJSON`) built the
  same image set minus Ruby and GCC 13.

### Task 8: Window loading and the single resolver [`vergil-tooling`]

**Files:**

- Create:
  - `src/vergil_tooling/lib/versions/__init__.py`, `window.py`, `resolve.py`;
  - `src/vergil_tooling/lib/versions/extensions/__init__.py` (the registry,
    empty except for the protocol);
  - `src/vergil_tooling/bin/vrg_ci_resolve.py`;
  - tests: `tests/vergil_tooling/test_versions_window.py`,
    `test_versions_resolve.py`, `test_vrg_ci_resolve.py`.
- Modify:
  - `pyproject.toml`: adds `vrg-ci-resolve = "vergil_tooling.bin.vrg_ci_resolve:main"`.
  - `lib/config.py`: `[ci].versions` becomes optional, and
    `REPOSITORY_TYPE_DEFINITIONS` is added (below). `primary_ci_version()` is
    removed, and its callers use `resolve()`.
  - `lib/container.py`: `_DEFAULT_VERSIONS` is removed. `default_image()`
    resolves through `resolve()`.
  - `lib/repo_init.py:1152-1172`: `_default_ci_versions` is removed, so new
    repos get no `[ci].versions`.
  - The `vrg-validate` config check calls `resolve()`, so any
    `ResolutionError` fails validation.

**Interfaces:**

- Produces, in `window.py`:

  ```python
  @dataclass(frozen=True)
  class WindowVersion:
      version: str; eol: date | None; in_policy: bool; runtime_published: bool | None
  @dataclass(frozen=True)
  class WindowFamily:
      key: str; token_prefix: str; versions: tuple[WindowVersion, ...]   # newest first
  @dataclass(frozen=True)
  class Window:
      languages: Mapping[str, tuple[WindowFamily, ...]]
  WINDOW_URL = "https://raw.githubusercontent.com/vergil-project/vergil-containers/{ref}/docker/versions/window.json"
  def load_window(vergil_ref: str, *, fetch: Callable[[str], bytes] = ..., env: Mapping[str, str] = os.environ) -> Window
  ```

  `VRG_VERSION_WINDOW` (a path) wins when it's set. Otherwise the window is
  fetched, and any error raises `WindowError` naming the URL and the override.

- Produces, in `resolve.py`:

  ```python
  ResolutionKind = Literal["library", "app"]
  @dataclass(frozen=True)
  class Resolution:
      language: str; kind: ResolutionKind
      versions: tuple[str, ...]          # newest first; tokens as written in vergil.toml (e.g. "clang-20")
      primary: str
      container_tags: Mapping[str, str]  # version token -> image tag ("clang-20" -> "20", "3.14" -> "3.14")
      warnings: tuple[str, ...]
  class ResolutionError(Exception): ...
  def resolve(cfg: VergilConfig, window: Window, *, repo_root: Path, strict_app_pins: bool) -> Resolution | None
  ```

  It returns `None` when `primary-language` is absent or `none`. Those repos
  are untouched (spec §6.3).

- Resolution rules (spec §6.3, exact):

  | Condition | Result |
  |---|---|
  | kind | `library` iff `repository-type == "library"`, else `app` |
  | no pin, library | every window version across the language's families |
  | no pin, app | the newest version (single-family languages only) |
  | no pin, family language (cpp, typescript) | `ResolutionError("<language> must pin [ci].versions until its lifecycle turn")` |
  | pin | every entry must be in the window, otherwise `ResolutionError` naming the version, the window list and the fix |
  | app pin with more than one entry | a warning when `strict_app_pins=False`, a `ResolutionError` when `True` |
  | `[ci].primary-version` | if set, it must be a resolved version (otherwise an error). If unset, the primary is the newest resolved version |

- Extensions: `Extension = Callable[[ExtensionContext, tuple[str, ...]], tuple[str, ...]]`
  narrows the candidates. The registry is `EXTENSIONS: dict[str, Extension]`.
  Task 9 adds `"python"`.
- `strict_app_pins` comes from a module constant
  `STRICT_APP_PINS = False`. Task 14 flips it.

  (As built, 2026-10-23: Task 14 flipped it, then
  `vergil-project/vergil-tooling#3442` removed the switch. `STRICT_APP_PINS`,
  the `strict_app_pins` parameter and `Resolution.warnings` no longer exist.
  An app multi-pin is always a `ResolutionError`, and `resolve()` takes only
  `cfg`, `window` and `repo_root`. `Resolution` also gained
  `container_suffixes`, and the CLI's `json` output a `container-suffixes`
  object.)
- `REPOSITORY_TYPE_DEFINITIONS: Mapping[str, str]` holds one sentence per
  type.
  - `library`: provides code that runs in other repos' environments, whether
    imported or installed as a tool there (see Task 8 note).
  - The other types are written to match current use.
- `vrg-ci-resolve [--format json|github-output]` prints, as `json`, an object
  with:
  - `versions`: a JSON array;
  - `primary-version`;
  - `primary-container-tag`;
  - `container-tags`: an object.

  `github-output` prints the first three as `key=value` lines, with the same
  names and meanings as `derive.sh` emits today. A repo without a language
  prints `versions=["latest"]`, `primary-version=latest` and
  `primary-container-tag=latest`, which preserves today's non-language
  behaviour.

> **Note on `vergil-tooling` itself.** CI installs `vergil-tooling` with a bare
> `uv tool install` inside each consumer's image
> (`vergil-actions/actions/shared/setup/vergil/action.yml:88`). So it runs on
> whatever window version a consumer is building, and must support the whole
> window. The `library` definition above covers this case, and Task 15 sets
> `vergil-tooling`'s `repository-type` to `library` in its sweep (spec §2.3,
> §6.3).

- [ ] **Step 1: Write the window tests.**
  - `test_load_window_parses_schema_1`.
  - `test_load_window_env_override_wins`.
  - `test_load_window_missing_at_ref_fails_loud`: a 404 raises a `WindowError`
    whose message contains the URL and `VRG_VERSION_WINDOW`.
  - `test_load_window_rejects_unknown_schema`.
- [ ] **Step 2: Write the resolver tests,** one per row of the table above.
  Window fixture: python [3.15, 3.14, 3.13, 3.12], go [1.27, 1.26], cpp
  families clang [20, 19] and gcc [14].
  - `test_library_no_pin_gets_whole_window` == ("3.15", "3.14", "3.13", "3.12").
  - `test_app_no_pin_gets_newest` == ("3.15",).
  - `test_pin_outside_window_is_fatal`: with `["1.25", "1.26"]`, the message
    contains `1.25`, `1.27, 1.26` and `remove it from [ci].versions`.
  - `test_cpp_unpinned_is_fatal`.
  - `test_cpp_pin_tags_strip_prefix`: `clang-20` maps to `20`.
  - `test_app_multi_pin_warns_when_not_strict`.
  - `test_app_multi_pin_fatal_when_strict`.
  - `test_primary_override_must_be_resolved`.
  - `test_no_language_returns_none`.
  - `test_language_absent_from_window_is_fatal`: `primary-language = "ruby"`
    raises `ResolutionError("ruby is not in the supported window …")`.
- [ ] **Step 3: Write the CLI tests.**
  - `test_cli_github_output_matches_derive_contract`: the keys are exactly
    `versions`, `primary-version` and `primary-container-tag`.
  - `test_cli_no_language_emits_latest`.
  - `test_cli_error_exits_1_with_message`.
- [ ] **Step 4: Run.** Expected: FAIL.
- [ ] **Step 5: Implement** the code, remove `_DEFAULT_VERSIONS`,
  `primary_ci_version` and `_default_ci_versions`, and update their callers
  and tests.
- [ ] **Step 6:** Run `vrg-container-run -- vrg-validate`. Expected: PASS at
  100% branch coverage. Set `VRG_VERSION_WINDOW` in the test environment
  through the fixtures; there must be no network access in tests.
- [ ] **Step 7: Commit.** `feat(versions): resolve CI versions from the published window with one resolver`.

### Task 9: Python extension, toolchain selection and floors [`vergil-tooling`]

**Files:**

- Create: `src/vergil_tooling/lib/versions/extensions/python.py` and
  `src/vergil_tooling/lib/versions/floors.py`, with tests
  `tests/vergil_tooling/test_versions_python_ext.py` and `test_versions_floors.py`.
- Modify:
  - `lib/versions/extensions/__init__.py`: registers `"python"`.
  - The `vrg-validate` config check: runs `check_floors` and
    `check_toolchain_selection`, and prints warnings. Toolchain-selection
    findings are errors.
  - The container run (`lib/container.py`, the env passed into
    `vrg-container-run`): exports `UV_PYTHON=<primary>` for Python repos.
  - `lib/config.py:776-790`: `[package.python].runtime` becomes optional.
    When it's absent, it is derived (below). An explicit value is an
    exact-patch pin whose `x.y` must be in the window.

**Interfaces:**

- Consumes: `Resolution`, `Window`, `EXTENSIONS` (Task 8).
- Produces:
  - `python_extension(ctx, candidates) -> candidates`: for a repo with
    `[package].builder == "python"`, it keeps only candidates whose window
    entry has `runtime_published` true. It keeps the newest **one** such
    version for apps, or all of them for libraries. If none is left, it
    raises `ResolutionError("no published vergil-python runtime for any of <candidates>")`.
  - `derived_runtime(resolution: Resolution) -> str` returns the `x.y` that
    the package builder installs against (`/opt/vergil/python/<x.y>`, Task 13).
  - `check_floors(repo_root, resolution, window) -> list[str]` returns
    warnings:
    - `requires-python` floor below the window's oldest: `requires-python floor 3.11 is below the oldest supported version 3.12`.
    - an upper bound excluding the newest: `requires-python upper bound <3.15 excludes 3.15`.
    - the same checks for the `go` directive in `go.mod`.
    - A floor above the oldest version produces no output.
  - `check_toolchain_selection(repo_root, language) -> list[str]` returns
    errors:
    - a `.python-version` file;
    - `[tool.uv] python-preference` in `pyproject.toml`, or `python-preference`
      in `uv.toml`.

    For Go there's nothing to check. `rust-toolchain.toml` is out of scope
    (spec §6.5).

- (As built, 2026-10-23, after the §6.5 amendment on
  `vergil-project/.github#407`:)
  - `python-preference` is **not** an error; the check flags only
    `.python-version` (`vergil-project/vergil-tooling#3410`). uv's
    `python-downloads` is not flagged either.
  - The container env is `UV_PYTHON=<primary>` **plus**
    `UV_PYTHON_PREFERENCE=only-system`, in `vrg-container-run`,
    `vrg-container-test` and the cached dev-image build
    (`vergil-project/vergil-tooling#3410`, #3421). `vrg-container-docs` sets only
    `UV_PYTHON_PREFERENCE` (#3414). Every Python CI leg sets both, and the docs
    jobs only the source (`vergil-project/vergil-actions#994`, #1002).
  - `python_extension` narrows only a packaged **app** to the newest version
    with a runtime. A packaged **library** keeps every candidate in CI, and the
    rule picks only its deployed runtime through `derived_runtime`
    (`vergil-project/vergil-tooling#3368`, PR #3401). Either kind fails loud if
    no candidate has a published runtime.

- [ ] **Step 1: Write the tests.**
  - `test_python_packaged_falls_back_to_newest_with_runtime`: the window has
    3.15 (no runtime) and 3.14 (runtime), giving an app result of ("3.14",).
  - `test_python_packaged_no_runtime_fails_loud`.
  - `test_python_unpackaged_ignores_runtime`.
  - `test_floor_below_oldest_warns`.
  - `test_upper_bound_excluding_newest_warns`: using the lab's
    `>=3.14,<3.15`.
  - `test_floor_above_oldest_is_silent`.
  - `test_go_directive_floor_warns`.
  - `test_python_version_file_is_error`.
  - `test_python_preference_is_error`.
  - `test_container_env_sets_uv_python`.
  - `test_runtime_derived_from_resolution`.
  - `test_explicit_runtime_outside_window_fatal`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement** the code.
- [ ] **Step 4:** Run `vrg-validate`. Expected: PASS at 100% coverage.
- [ ] **Step 5: Commit.** `feat(versions): add the Python resolver extension, toolchain-selection and floor checks`.

### Task 10: Finish the Go 1.25 retirement and the Ruby removal in `vergil-tooling` docs [`vergil-tooling`]

**Files:**

- Modify:
  - `docs/site/docs/guides/ci-architecture.md:463` (it still says `dev-go`
    1.25/1.26);
  - every docs page that lists Ruby as a supported language, found with
    `grep -rn -i "ruby\|1\.25" docs/site/docs`;
  - the tests that use Ruby as a supported-language fixture, where the
    fixture asserts support (not historical ones).

- [ ] **Step 1:** Edit the docs. Every page that lists versions links to the
  generated table in `vergil-containers` instead of repeating the numbers.
- [ ] **Step 2:** Run `vrg-validate`. Expected: PASS.
- [ ] **Step 3: Commit.** `docs: finish the Go 1.25 retirement and drop Ruby from the supported languages`.

### Task D2: Deployment: release `vergil-tooling` with the resolver [`vergil-tooling`, `--kind deployment`]

**Blocked by:** D1, D4, Tasks 8, 9, 10.

**Precondition (human-attested):** a `vergil-tooling` release that contains
Tasks 8-10 has been cut and promoted to `v2.1`.

**Procedure:**

1. In a consumer checkout (the lab), run `vrg-ci-resolve --format json`.
   Expected:
   - `versions: ["3.14"]`: the lab's existing pin is honoured;
   - a warning for the `<3.15` upper bound;
   - an error for `.python-version` and `python-preference`, which is
     expected until Task 15's sweep.

**Acceptance:** The output matches the expected result above, and
`vrg-validate` in `vergil-tooling` itself is green.

### Task 11: `derive.sh` calls the resolver, and CI asserts the running interpreter [`vergil-actions`]

**Files:**

- Modify:
  - `actions/shared/setup/vergil/derive.sh`: replace the inline Python
    resolution with `vrg-ci-resolve --format github-output >> "$GITHUB_OUTPUT"`.
    Any non-zero exit fails the step, with the resolver's message surfaced as
    `::error::`.
  - `actions/shared/setup/vergil/derive.test.sh`: the fixtures set
    `VRG_VERSION_WINDOW` to a fixture `window.json`.
  - `actions/ci/matrix/resolve.sh`: consumes `container-tags` from the
    resolver's output instead of re-splitting family prefixes. The
    family-routing logic is removed.
  - `actions/shared/setup/vergil/action.yml`: the `Derive` step gains `env:
    VRG_VERSION_WINDOW` only when the caller sets it. The input stays
    optional, for tests.
- Create: `actions/shared/assert-toolchain/action.yml` and `assert.sh`, with
  `assert.test.sh`. Given `language` and `expected` (`x.y`):
  - **python:** run `uv run python -c 'import sys; print(f"{sys.version_info[0]}.{sys.version_info[1]}")'`
    and compare.
  - **go:** compare `go env GOVERSION` trimmed to `x.y`.
  - **Mismatch:** `::error::CI resolved <expected> but the job runs <actual>`,
    exit 1.
  - **Other languages:** no-op, with a notice line.
- Modify: remove the Ruby language branches from the reusable workflows and
  actions, found with `grep -rn -i ruby .github/workflows actions`. Callers
  with `language: ruby` then fail at resolution (Task 8).
- Modify: the reusable test workflows that run per-version legs. They call
  `assert-toolchain` right after setup, with `UV_PYTHON` set to the leg's
  version for Python.

**Interfaces:** Consumes `vrg-ci-resolve` (Task 8). Produces the same three
action outputs as today.

- [ ] **Step 1: Write the tests** (bash).
  - `derive_uses_resolver_outputs`.
  - `derive_fails_on_resolver_error`.
  - `matrix_uses_container_tags`.
  - `assert_python_mismatch_fails`: a fake `uv` prints 3.14 while 3.15 is
    expected.
  - `assert_python_match_passes`.
  - `assert_go_match_passes`.
- [ ] **Step 2: Run** `bash actions/shared/setup/vergil/derive.test.sh` and
  `bash actions/shared/assert-toolchain/assert.test.sh`. Expected: FAIL.
- [ ] **Step 3: Implement** the code.
- [ ] **Step 4:** Run `vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(setup): derive CI versions through vrg-ci-resolve and assert the running toolchain`.

### Task D3: Deployment: release `vergil-actions` [`vergil-actions`, `--kind deployment`]

**Blocked by:** D2, Task 11.

**Precondition (human-attested):** a `vergil-actions` release that contains
Task 11 has been cut and promoted to `v2.1`.

**Acceptance:** A PR CI run in `vergil-tooling` shows:

- the `Derive` step output coming from `vrg-ci-resolve`;
- the `assert-toolchain` step passing on every leg.

### Task 12: Validation: the first automated adds, Python 3.15 and Go 1.27 [`vergil-containers`, `--kind validation`]

**Blocked by:** D3, T6, T13b.

**Precondition:** D3 is closed (consumers resolve from the window), and the
nightly `language-lifecycle` job is green.

**Procedure:**

1. Confirm that the nightly opened exactly two PRs:
   `feat(versions): add python 3.15` and `feat(versions): add go 1.27`.
2. Confirm the PR CI built and scanned `dev-python:3.15` and `dev-go:1.27`.
3. The human merges both, and a `vergil-containers` release is cut and
   promoted (human-gated).
4. The `vergil-python` 3.15 runtime is released (human-gated, Task 13). The
   next nightly sets `runtime_published: true` for 3.15, as a lifecycle PR.
5. Confirm, by running `vrg-ci-resolve`:
   - an unpinned Python library resolves to 3.15–3.12;
   - `vergil-tooling` (packaged) resolves to 3.14 until step 4 lands, and to
     3.15 after it.
6. Confirm the report shows Rust, Java, Clang and Node gaps as issues, and
   that no PR was opened for them.

**Acceptance:** Every step above is observed. Record the PR and issue URLs.

Go 1.26 does not reach EOL during the epic, because that happens when 1.28
ships, around February 2027. The drop path is proven by Task 6's
`test_apply_drop_removes_version` and `test_drop_pr_body_lists_go_case_followup`,
and by Task 3's `test_drop_past_eol`. Record this in the results comment, as
spec §8.1 item 8 requires.

### Task 13: Per-`x.y` runtime packages, and the runtime-published signal [`vergil-python`]

**Files:**

- Modify:
  - `vergil.toml` `[package]`: `name = "vergil-python<x.y>"` (for example
    `vergil-python3.14`), `version = "<x.y.z>+<pbs_tag>"`, and smoke path
    `/opt/vergil/python/<x.y>/bin/python<x.y>`.
  - `runtime.toml`: adds `series = "<x.y>"`, and `cpython` floats within the
    series.
  - `packaging/build.sh`: install root `/opt/vergil/python/<x.y>`. It fails
    if `series` and `cpython` disagree.
  - `scripts/dev/update-deps.sh`, the existing hook from `vergil-python#12`:
    bumps `cpython` to the newest patch **within** `series`, plus the PBS tag
    and checksums.
  - `docs/runtime-package-pattern.md`: the per-series model.
- Process: the package is built from the newest window `x.y`. Moving
  `series` to a new `x.y` is one PR, made after the containers add.
  Previously published series stay in the apt repo, so they remain
  co-installable (spec §7). (As built, 2026-10-23: a single series is
  published at a time. The series moved to 3.15 by a manual PR
  (`vergil-project/vergil-python#34`), and 3.14 is frozen at
  `3.14.8+20261013`, kept in the apt repo only while retention keeps it.
  Nothing drops a series at EOL, and multi-series publishing is deferred to
  `vergil-project/vergil-python#36`.)
- Create, in `vergil-containers` (same PR author, separate task row in the
  filing map: **Task 13b**): `docker/versions/lifecycle/runtime.py`.
  - `runtime_published(series: str, fetch) -> bool` reads the vergil apt
    index for `vergil-python<series>`.
  - `check.py` adds a `runtime` change kind for `pr` families with a runtime,
    and `apply.py` sets `runtime_published` from it.
  - Tests: `test_runtime_flag_flips_when_package_appears` and
    `test_runtime_index_unreachable_is_error`.

**Interfaces:** Produces the install path `/opt/vergil/python/<x.y>`, which
Task 9's `derived_runtime` and `vergil-tooling`'s `python_builder.py` use.
The `python_builder.py` path change (`/opt/vergil/python/<x.y>`, and the
package dependency `vergil-python<x.y>`) lands in Task 9's PR. **Task D4 must
release this task first,** so `vergil-tooling` never depends on a package that
doesn't exist yet.

- [ ] **Step 1: Write the tests,** in `vergil-python`'s test suite under
  `tests/`.
  - `test_build_rejects_series_mismatch`.
  - `test_update_hook_stays_within_series`: with 3.14.8 available and 3.15.0
    available, the hook picks 3.14.x only.
- [ ] **Step 2: Run** `vrg-container-run -- vrg-validate`. Expected: FAIL.
- [ ] **Step 3: Implement** the code.
- [ ] **Step 4:** Run `vrg-validate`. Expected: PASS.
- [ ] **Step 5: Commit.** `feat(runtime)!: publish one runtime package per Python series`.

### Task D4: Deployment: release the per-series `vergil-python3.14` [`vergil-python`, `--kind deployment`]

**Blocked by:** Task 13.

**Precondition (human-attested):** a `vergil-python` release that contains
Task 13 has been cut and published to the vergil apt repo.

**Acceptance:** The apt index lists `vergil-python3.14` with a `3.14.x+<pbs>`
version. Installing it on a scratch VM gives
`/opt/vergil/python/3.14/bin/python3.14`. The smoke command from `vergil.toml`
passes.

### Task 14: Make an app's multi-version pin fatal [`vergil-tooling`]

**Blocked by:** every Task 15 sweep row.

**Files:** `src/vergil_tooling/lib/versions/resolve.py` (`STRICT_APP_PINS = True`), and its tests.

- [ ] **Step 1:** Flip the test `test_app_multi_pin_default_is_fatal`.
- [ ] **Step 2:** Flip the constant, then run `vrg-validate`. Expected: PASS.
- [ ] **Step 3: Commit.** `feat(versions)!: an app pins exactly one version`.

### Task 15: Fleet sweep, one row per repo [each repo listed]

**Blocked by:** D3.

There is one task per repo (the placement law), and each one is a single PR.
Each PR does the following:

- removes `[ci].versions`, unless the repo needs a pin. A kept pin is
  recorded with a linked issue titled `Remove the <language> version pin`;
- removes `.python-version` and `[tool.uv] python-preference`; (As built,
  2026-10-23: the sweep keeps `python-preference`, which the §6.5 amendment
  allows, and removes only `.python-version`.)
- removes any `requires-python` upper bound, keeping the floor;
- runs `vrg-container-run -- vrg-validate` and `vrg-ci-resolve`, and pastes
  the resolved versions into the PR body.

| Row | Repo | `repository-type` → resolves as | Expected change |
|---|---|---|---|
| 15a | `vergil-project/vergil-tooling` | `tooling` → set to **`library`** (see the Task 8 note) | remove the list; whole window |
| 15b | `logical-minds-foundry/mq-resiliency-lab-for-linux` | `tooling` → app | remove the list, `.python-version`, `python-preference` (confirm `UV_PYTHON` covers local runs, spec §6.5 item 4) and the `<3.15` bound |
| 15c | `logical-minds-foundry/mq-resiliency-observability` | `library` | remove the list |
| 15d | `logical-minds-foundry/mq-gateway-replay-lab` | `tooling` → app | remove the list |
| 15e | `mnemosys-project/melete` | `application` → app | remove the list |
| 15f | `diogenes-project/diogenes` | `library` | remove the list |

(As built, 2026-10-23, per the scope decision on `vergil-project/.github#407`
of 2026-10-22:)

- **15a** is done (`vergil-project/vergil-tooling#3371`).
- **15b** kept a pin. Lab#1681 removed `.python-version` and the `<3.15`
  bound, kept `python-preference = "only-managed"` and added
  `python-downloads = "manual"`. It kept `[ci].versions = ["3.14"]`, recorded by
  `logical-minds-foundry/mq-resiliency-lab-for-linux#1685`.
- **15c–15f** moved out of this epic, to their repos' or orgs' ad-hoc epics,
  on hold: `logical-minds-foundry/mq-resiliency-observability#49`,
  `logical-minds-foundry/mq-gateway-replay-lab#314`,
  `mnemosys-project/melete#246` and `diogenes-project/diogenes#248`.
- `mq-protocol-gateway`'s `clang-20` pin is recorded by
  `logical-minds-foundry/mq-protocol-gateway#25`.

Not swept:

- `mq-protocol-gateway` (C++ must pin `clang-20` until its turn).
- The on-hold mq-rest-admin ports.
- `wphillipmoore/the-infrastructure-mindset` (`primary-language = "none"`).
  Its `["3.12"]` is outside resolution. Record it in the documentation review
  if it's stale.

### Task 16: Archive `mq-rest-admin-ruby` [`mq-rest-admin-project/.github`]

Archiving a repo is a human action (`gh repo archive` is outside the agent's
allowlist), so this is an operational task: `--kind deployment`.

- **Precondition (human-attested):** `mq-rest-admin-project/mq-rest-admin-ruby`
  is archived.
- **Acceptance:** `vrg-gh repo view mq-rest-admin-project/mq-rest-admin-ruby --json isArchived` returns `true`.

---

## Bootstrap order

This is the ordering spec §6.3 requires. Each arrow is a `Blocked-by`.

```text
T1 ─┬─▶ T7 ─▶ D1 ─▶ T8,T9,T10 ─▶ D2 ─▶ T11 ─▶ D3 ─┬─▶ T15a–f ─▶ T14
    │                                              └─▶ T12
    ├─▶ T2
    └─▶ T3 ─▶ T4 ─▶ T5 ─▶ T6 ─▶ T13b ─▶ T12
T13 (vergil-python) ─▶ D4 ─▶ D2   (T9's builder path needs the released package)
T13 ─▶ T13b ─▶ T12
T16: independent
```

Why this order is deadlock-free:

1. **The window is published before anything reads it.** D1 comes before T8.
2. **`vergil-tooling`'s own CI stays on its existing list** through T8-T11.
   Its list is honoured as a pin, so its own build never needs the resolver
   it is adding.
3. **CI switches to the resolver only once the resolver is released.** D2
   comes before T11.
4. **PR automation can run before D3, but nothing merges a version add until
   T12, after D3.** So no consumer sees 3.15 before resolution, the packaged
   rule and the CI assertion are live.

## Task filing map (epic-create step 9)

| Task | Repo | Kind | Blocked by |
|---|---|---|---|
| T1 Manifest, generator and drift gate | vergil-project/vergil-containers | task | — |
| T2 Per-language support policies | vergil-project/vergil-containers | task | T1 |
| T3 Lifecycle core and the Python and Go adapters | vergil-project/vergil-containers | task | T1 |
| T4 Report-only adapters | vergil-project/vergil-containers | task | T3 |
| T5 Scheduled job: report, issues, failure issues | vergil-project/vergil-containers | task | T4 |
| T6 Add/drop PR automation (Python and Go) | vergil-project/vergil-containers | task | T5 |
| T7 Drop Ruby and GCC 13 | vergil-project/vergil-containers | task | T1 |
| D1 Release `vergil-containers` (`window.json` at v2.1) | vergil-project/vergil-containers | deployment | T1, T7 |
| T8 Window loading and the single resolver | vergil-project/vergil-tooling | task | D1 |
| T9 Python extension, toolchain selection and floors | vergil-project/vergil-tooling | task | T8 |
| T10 Go 1.25 and Ruby docs cleanup | vergil-project/vergil-tooling | task | — |
| D2 Release `vergil-tooling` | vergil-project/vergil-tooling | deployment | D1, D4, T8, T9, T10 |
| T11 `derive.sh` via the resolver, plus the toolchain assertion | vergil-project/vergil-actions | task | D2 |
| D3 Release `vergil-actions` | vergil-project/vergil-actions | deployment | D2, T11 |
| T13 Per-`x.y` runtime packages | vergil-project/vergil-python | task | — |
| D4 Release the per-series `vergil-python3.14` | vergil-project/vergil-python | deployment | T13 |
| T13b The runtime-published signal | vergil-project/vergil-containers | task | T6, T13 |
| T12 First automated adds (Python 3.15, Go 1.27) | vergil-project/vergil-containers | validation | D3, T6, T13b |
| T15a–f Fleet sweep (one per repo) | each repo in the Task 15 table | task | D3 |
| T14 An app's multi-version pin is fatal | vergil-project/vergil-tooling | task | T15a–f |
| T16 Archive `mq-rest-admin-ruby` | mq-rest-admin-project/.github | deployment | — |

Bookends already filed:

- `vergil-project/.github#408` (docs, closed by this PR);
- `vergil-project/vergil-containers#610` (documentation review, blocked by
  every row above);
- `vergil-project/.github#409` and `#410` (follow-on brainstorms);
- `vergil-project/.github#411` (retrospective, terminal).

Follow-ons filed during the epic (2026-10-23):

- `vergil-project/.github#424` (brainstorm: advisory pre-release toolchain
  support, promoted to required CI at GA);
- `vergil-project/.github#425` (brainstorm: version resolution for apps whose
  runtime lives on a host).

**Cross-org rows.** T15b–f and T16 are in other orgs. Cross-org linking is out
of scope (epic-create Notes), so each of those rows is filed under its own
org's ad-hoc epic, with a `Ref vergil-project/.github#407` line, and tracked
here by URL.
