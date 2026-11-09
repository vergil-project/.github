# Remove Java support: implementation plan

- **Epic:** vergil-project/.github#952
- **Planning issue:** vergil-project/.github#951 (review history is in its
  comments)
- **Status:** draft, in design review

> **For agentic workers:** each task below is filed as its own GitHub issue
> in the repo its PR lands in, and run with `issue-implement` (the deployment
> task with `issue-deploy`). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** remove Java as a supported language from vergil-containers,
vergil-tooling and vergil-actions, correct the org profile, delete the Java
images and retire the Maven Central credentials.

**Architecture:** deletion only, plus one addition: `java` joins the retired
primary languages, so a config naming it is rejected. Each task deletes code
and its references, and is proven by the existing gates, the one new
rejection test, and the leftover search in its last steps.

**Tech stack:** Python and shell (vergil-tooling, vergil-containers),
composite actions and reusable workflows (vergil-actions), Dockerfiles,
MkDocs sites.

**Spec:** `epics/952-remove-java-support/spec.md`.

## Global constraints

- Remaining primary languages: `python`, `go`, `rust`, `cpp`, `typescript`.
- Retirement reason, verbatim: `Java support was removed in epic vergil-project/.github#952`.
- The leftover search for a repo is, from its root:

  ```bash
  grep -rniE '\bjava\b|\bmaven\b|\bmvnw?\b|pom\.xml|\bjdk\b|central-username|CENTRAL_USERNAME|MAVEN_GPG' . \
    --exclude-dir=.git --exclude-dir=.worktrees --exclude-dir=build \
    --exclude-dir=specs --exclude-dir=plans --exclude-dir=superpowers \
    --exclude-dir=reports --exclude-dir=paad --exclude-dir=releases \
    --exclude=CHANGELOG.md
  ```

  Word-bounded, so `javascript` never matches. Paths named `java` are
  removed by name (each task's Files list).
- The review search runs beside it, with the same excludes, and is not a
  zero-hit gate: every hit is read by hand and either removed or confirmed as
  not Java (a `javascript` neighbour, a third-party fact). It catches what
  the word boundary hides, such as snake_case identifiers
  (`_JAVA_VERSION_RE`) and `temurin`:

  ```bash
  grep -rniE 'java|maven|mvn|jdk|temurin' . <same excludes> | grep -vi javascript
  ```

- History is never edited: `docs/specs/`, `docs/plans/`, `docs/superpowers/`,
  `docs/reports/`, `paad/`, `releases/`, `CHANGELOG.md`, `epics/`.
- GPG keys and secrets are never revoked, rotated or deleted. Package
  signing (`PACKAGE_SIGNING_KEY`, vergil-tooling `lib/package/`) is untouched.
  Only Maven-only forwarding of `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` is
  removed.
- No deprecation period, flag or shim.
- Validation is `vrg-container-run -- vrg-validate` only.
- A task touching `.github/workflows/**` (Tasks 1 and 4) cannot be pushed by
  the agent. report-ready records it and the human's `vrg-submit-pr` pushes.

## Review focus

1. **`javascript` caught by a sloppy search.** Expected: TypeScript's CodeQL
   mapping (`typescript → javascript-typescript`) and every `javascript` URL
   or comment stay. Tasks 2 and 4 use the word-bounded search only, and Task 4
   step 3 runs the existing codeql `resolve-language` tests.
2. **A `vergil.toml` still naming `java` validating as language-less.**
   Expected: it fails with the retirement reason. Task 2 step 1 pins it.
3. **Package signing broken by removing Maven's GPG inputs.** Expected: the
   apt/rpm signing steps in `cd-release.yml` and vergil-tooling
   `lib/package/` still run. Task 4 step 4 checks `PACKAGE_SIGNING_KEY` use is
   unchanged.
4. **A remaining generator, scaffold or audit that branches on `"java"` and
   is reached only at runtime.** Expected: no `src/` code tests for
   `"java"`. Task 2 step 5's search covers `src/` and packaged data.
5. **Broken links to the deleted Java pages.** Expected: the strict mkdocs
   build passes. Tasks 1 and 3 remove the nav entries and inbound links with
   the pages.

---

### Task 1: vergil-containers: remove the Java image, its adapter and osv-scanner

Repo: vergil-project/vergil-containers. Independent.

**Files:**

- Delete: `docker/java/` (`Dockerfile.template`)
- Delete: `docker/common/osv-scanner.dockerfile`
- Delete: `docker/versions/lifecycle/adapters/java.py` and its fixture
  `docker/versions/fixtures/endoflife-eclipse-temurin.json`
- Modify: `docker/build.sh:72-73` (the two `build java JDK_VERSION` lines),
  `docker/generate.sh:53` (`java` from `langs`)
- Modify: `docker/versions/lifecycle/adapters/__init__.py` (the `java`
  registration), `docker/versions/manifest.yml`, `matrix.json`,
  `window.json`, `testdata/manifest.yml` (the `java` entries)
- Modify: `docker/pins/pins.yml:80`, `docker/pins/CATALOG.md:13`,
  `docker/pins/resolve_latest.py:40,57` (the `osv-scanner` pin), and
  `docker/pins/test_resolve.py:9` (`osv-scanner` from the `TOOLS` set)
- Modify: `docker/pins/test_pins.py` (delete
  `test_osv_scanner_is_an_auto_managed_java_pin`; drop other Java cases),
  `docker/versions/test_adapters_report_only.py`, `test_lifecycle_core.py`,
  `test_manifest.py` (drop the Java cases)
- Modify: `.github/workflows/package-cache-warm.yml:65,73`,
  `.github/workflows/package-cleanup.yml:88,96,183,199` (`dev-java`,
  `prod-java`)
- Modify: `.trivyignore` (step 5, including the osv-scanner mention at :218)
- Delete: `docs/site/docs/languages/java.md`; Modify: `docs/site/mkdocs.yml`
  (its nav entry), `docs/site/docs/images/index.md`,
  `architecture/index.md`, `index.md`, `languages/lifecycle.md`,
  `operations/version-management.md:100` (osv-scanner), `README.md`

**Interfaces:** none. The deployment task (Task 6) depends on this task's
workflow change being on `develop`.

This task only deletes, so it has no red/green cycle: the existing tests,
the leftover and review searches and `vrg-validate` stand in for red.

- [ ] **Step 1: Delete the image, `osv-scanner.dockerfile` and the adapter**,
  and remove `java` from `build.sh`, `generate.sh` and the adapter registry.
- [ ] **Step 2: Remove the `java` entries** from the versions manifest,
  matrix, window and test data, and the `osv-scanner` pin from `pins.yml`,
  `CATALOG.md` and `resolve_latest.py`.
- [ ] **Step 3: Edit the tests.** Delete the osv-scanner pin test; remove the
  Java cases from the adapter, lifecycle and manifest tests.
- [ ] **Step 4: Remove `dev-java` and `prod-java`** from both package
  workflows.
- [ ] **Step 5: Trim `.trivyignore`.** Java appears mostly as carrier
  bullets inside CVE entries other images share (around :86-89, :147-149,
  :206-207). Delete every Java, pebble and osv-scanner carrier bullet; strip
  the temurin and pebble clause from every "Remove once" condition (around
  :22, :117, :151); and delete a whole CVE entry only when the Java image was
  its last carrier.
- [ ] **Step 6: Edit the docs.** Delete `languages/java.md` and its nav entry;
  remove Java from the listed pages and the README.
- [ ] **Step 7: Search.** The Global constraints search returns nothing, and
  `grep -rn 'osv-scanner' .` with the Global constraints excludes returns
  nothing. Then run the review search and resolve every hit.
- [ ] **Step 8: Validate.** `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 9: Commit** with `vrg-commit --type refactor --scope images
  --message "remove the Java image, its lifecycle adapter and osv-scanner"`.
  vergil-project/vergil-containers#639 is closed by Task 6, after this task
  merges.

### Task 2: vergil-tooling: retire `java` and remove its code

Repo: vergil-project/vergil-tooling. Independent.

**Files:**

- Modify: `src/vergil_tooling/lib/config.py:33` (drop `"java"` from the
  `primary-language` set) and `:55-57` (`_RETIRED_PRIMARY_LANGUAGES` gains
  `"java"` with the reason from Global constraints)
- Modify: `src/vergil_tooling/lib/languages.py` (delete the `"java"` entry at
  :452-491, `_MAVEN_LICENSES_ALLOWLIST` at :147, `JAVA_SBOM` and
  `_JAVA_OSV_SCAN` at :161-180)
- Modify: `src/vergil_tooling/lib/version.py:22` (`_JAVA_VERSION_RE`) and
  `:32,41,120,199`,
  `lib/container.py:107-108`, `lib/container_cache.py:51,80`,
  `lib/dependabot.py:31-37`, `lib/github_config.py:282`,
  `lib/gitignore.py:83`, `lib/repo_init.py:550` (the `java` secrets-map
  entry) and `:559` (the `java` entry of `_CODEQL_LANGUAGES`; `"javascript"`
  at `:562` stays), `lib/semgrep.py:15`
- Delete: `src/vergil_tooling/data/gitignore/java`
- Modify: `src/vergil_tooling/bin/vrg_generate_commands.py` (delete the Java
  generator, :276-360, and its entries at :4, :593, :597, :612, :628),
  `bin/vrg_semgrep_scan.py:24`
- Modify: `tests/vergil_tooling/test_config.py` (new test, step 1)
- Modify: `tests/vergil_tooling/test_languages.py`, `test_version.py`,
  `test_container.py`, `test_dependabot.py`, `test_github_config_lib.py`,
  `test_gitignore.py`, `test_repo_init.py`, `test_semgrep.py`,
  `test_versions_window.py`, `test_vrg_generate_commands.py`,
  `test_vrg_release_validate_inputs.py`, `fixtures/versions/window.json`

**Interfaces:** none.

- [ ] **Step 1 (red): Write the rejection test** in `test_config.py`, beside
  `test_config_rejects_retired_ruby_language`:

  ```python
  def test_config_rejects_retired_java_language(tmp_path: Path) -> None:
      """Java is retired: it fails loudly instead of reading as unset."""
      toml = _VALID_TOML.replace('primary-language = "python"', 'primary-language = "java"')
      (tmp_path / "vergil.toml").write_text(toml)
      with pytest.raises(
          ConfigError,
          match=re.escape(
              "primary-language 'java' is no longer supported"
              " (Java support was removed in epic vergil-project/.github#952)"
          ),
      ):
          read_config(tmp_path)
  ```

  Add `import re` to the file's imports. `vrg-container-run -- vrg-validate`
  fails on it.
- [ ] **Step 2 (green): Retire `java` in `config.py`** (both edits under
  Files). The new test passes.
- [ ] **Step 3: Remove Java from the language registry and every `lib/`
  module** under Files, and delete the gitignore fragment.
- [ ] **Step 4: Remove the Java generator** from `vrg_generate_commands.py`
  and `java` from `vrg_semgrep_scan.py`'s help.
- [ ] **Step 5: Edit the tests.** Delete tests that exist only for removed
  code; drop `java` from tests that list languages; drop the `java` entry
  from the window fixture. Then search: the Global constraints search over
  `src tests` returns only the `_RETIRED_PRIMARY_LANGUAGES` entry and the
  step 1 test, and `grep -rn '"java"' src` returns only the retired entry.
  Then run the review search over `src tests` and resolve every hit.
- [ ] **Step 5a (refactor): Tidy what the removal trimmed.** Re-read
  `_ENUMS` in `config.py`, `_CODEQL_LANGUAGES` in `repo_init.py` and
  `_CODEQL_SUPPORTED_LANGUAGES` in `github_config.py`, and the comments that
  keep the CodeQL lists aligned, so each still reads correctly without Java.
  Remove any helper left with no caller.
- [ ] **Step 6: Validate.** `vrg-container-run -- vrg-validate` passes with
  100% branch coverage.
- [ ] **Step 7: Commit** with `vrg-commit --type refactor --scope languages
  --message "retire java and remove its support from the tooling"`.

### Task 3: vergil-tooling: remove Java from the site docs

Repo: vergil-project/vergil-tooling. Independent of Task 2 (no shared file).

**Files:**

- Delete: `docs/site/docs/standards/development/java/` (`overview.md`,
  `naming-conventions.md`); Modify: `docs/site/mkdocs.yml:131-133`
- Modify: `docs/site/docs/guides/ci-architecture.md`,
  `consuming-repo-setup.md`, `credential-management.md`,
  `reference/cli-tools-overview.md`, `reference/config-audit.md`,
  `standards/development/runtime-version-support-policy.md`
- Modify: `docs/site/docs/reference/lint/repo-profile.md:35-36` (drop `java`
  from the supported list; the retired sentence names `ruby` and `java`)

**Interfaces:** none.

- [ ] **Step 1: Delete the Java standards section** and its nav entries,
  and remove every inbound link to it.
- [ ] **Step 2: Edit the guide and reference pages** so none lists Java as
  supported; `repo-profile.md` says "A retired language (`ruby`, `java`) is
  rejected outright."
- [ ] **Step 3: Search.** The Global constraints search over `docs/site`
  returns only the `repo-profile.md` retired sentence. Then run the review
  search over `docs/site` and resolve every hit.
- [ ] **Step 4: Validate.** `vrg-container-run -- vrg-validate` passes,
  including the strict mkdocs build.
- [ ] **Step 5: Commit** with `vrg-commit --type docs --scope site
  --message "remove Java from the supported languages"`.

### Task 4: vergil-actions: remove the Maven Central path and Java branches

Repo: vergil-project/vergil-actions. Independent.

**Files:**

- Modify: `.github/workflows/cd-release.yml` (:7 description; :80-87 the
  `CENTRAL_USERNAME`, `CENTRAL_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`
  secrets; :603 `"java"` in the publish-language list; :615-618 their
  forwarding). The package-signing steps (:344-381) are unchanged.
- Modify: `.github/workflows/ci-security.yml:7,67,69` (Java in descriptions
  and comments)
- Modify: `actions/cd/release/registry-publish/action.yml` (the
  `central-username`, `central-token`, `gpg-private-key`, `gpg-passphrase`
  inputs, the Maven credential step at :86-90 and the Maven deploy path),
  `publish.sh:28`, `tests/publish.test.sh` (Java cases)
- Modify: `actions/cd/release/validate-inputs/action.yml:8` and its Java
  case
- Modify: `actions/ci/security/codeql/tests/resolve-language.test.sh:51`
  (the `java passthrough` case), `actions/ci/security/semgrep/action.yml:11`
- Modify: `scripts/fleet-cicd-rename.sh:22,84`
- Modify: `docs/development/publish-workflow.md`, and the site pages
  `actions/cd-release-registry-publish.md`, `cd-release-validate-inputs.md`,
  `ci-security-codeql.md`, `ci-security-semgrep.md`,
  `ci-gates/security-scanning.md`, `configuration.md`, `getting-started.md`,
  `workflows/cd-release.md`, `workflows/ci-security.md`

**Interfaces:** none. No major version: the removed inputs' only caller is
`mq-rest-admin-java`.

This task only deletes, so it has no red/green cycle: the existing shell
tests, the leftover and review searches and `vrg-validate` stand in for red.
`resolve-language.test.sh`'s `typescript->javascript-typescript` case stays
as the regression guard for Review focus 1.

- [ ] **Step 1: Remove the Maven path from registry-publish**: the four
  inputs, the credential and deploy steps, the `publish.sh` mapping and the
  Java test cases.
- [ ] **Step 2: Remove Java from `cd-release.yml` and `ci-security.yml`**:
  the four secrets, their forwarding and `"java"` in the language list.
- [ ] **Step 3: Remove the Java cases** from validate-inputs, the codeql
  test and the semgrep description; `resolve-language.test.sh` still asserts
  `typescript->javascript-typescript`.
- [ ] **Step 4: Check signing is intact.** Compare content, not line
  numbers: `grep PACKAGE_SIGNING .github/workflows/cd-release.yml` (no `-n`)
  prints the same output as on `develop`, and the package-sign step block is
  unchanged.
- [ ] **Step 5: Edit the script and docs** under Files. Two lines state
  CodeQL facts, not vergil's: `ci-gates/security-scanning.md:87` (CodeQL's
  language list) and `actions/ci-security-codeql.md:19` (`java` as an example
  language). Reword both to name the languages vergil scans: `python`, `go`,
  `rust`, `cpp`, `typescript`.
- [ ] **Step 6: Search.** The Global constraints search returns nothing.
  Then run the review search and resolve every hit.
- [ ] **Step 7: Validate.** `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 8: Commit** with `vrg-commit --type refactor --scope release
  --message "remove the Maven Central publish path and Java branches"`.
  Hand off with report-ready; the push is the human's (Global constraints).

### Task 5: vergil-project/.github: correct the org profile and open planning items

Repo: vergil-project/.github. Independent.

**Files:**

- Modify: `profile/README.md:10-11`

**Interfaces:** none.

- [ ] **Step 1: Correct the profile's language list** to "five supported
  languages" naming Python, Go, Rust, C++ and TypeScript, with Python first,
  and no Ruby or Java.
- [ ] **Step 2: Validate.** `vrg-container-run -- vrg-validate` passes.
- [ ] **Step 3: Commit** with `vrg-commit --type docs --scope profile
  --message "list the five supported languages"`.
- [ ] **Step 4: Edit the bodies** of vergil-project/.github#384 and #410 to
  drop Java from their scope (`vrg-gh issue edit --body-file`), and record
  both edits in the PR notes.

### Task 6: deployment: delete the Java images and retire the Maven Central credentials

Repo: vergil-project/vergil-containers, `--kind deployment`. Blocked by
Task 1.

Human-attested preconditions, pasted into the task by the human, with no
secret value:

- `dev-java` and `prod-java` are deleted from GHCR, with the output of
  `gh api /orgs/vergil-project/packages/container/dev-java` and
  `.../prod-java`, each a 404;
- the Maven Central publishing token is revoked, and `CENTRAL_USERNAME` and
  `CENTRAL_TOKEN` are deleted wherever set in mq-rest-admin-project (org and
  repo level). GPG keys and secrets are untouched.

Agent steps:

- confirm both 404 outputs name both packages and the credential statement
  is present;
- close vergil-project/vergil-containers#639 ("language lifecycle: java add
  25") as not planned, with a comment citing Task 1's merged PR (the Java
  lifecycle adapter that files such issues is gone);
- record `Outcome: SUCCESS` with that evidence and the closure.

### Bookends

- Documentation review: vergil-project/vergil-tooling#4107 (blocked by
  Tasks 1–5). Sweeps the `docs` repo's `docs/site/docs/languages.md` and
  `index.md`, and `vergil-claude-plugin`'s
  `hooks/scripts/detect-deprecation-warnings.sh`, filing a same-repo sibling
  task for each. vergil-python's JVM example stays. It also runs the
  Global constraints leftover search over the whole vergil-tooling repo,
  which must return exactly the three allowed hits (spec §5), since Tasks 2
  and 3 each search only part of it.
- Retrospective: vergil-project/.github#954 (terminal).

## Delivery

| Task | Repo | Kind | Blocked by |
|---|---|---|---|
| 1 Java image, adapter, osv-scanner | vergil-containers | impl | — |
| 2 retire `java`, remove code | vergil-tooling | impl | — |
| 3 site docs | vergil-tooling | impl | — |
| 4 Maven path, Java branches | vergil-actions | impl | — |
| 5 org profile, planning items | .github | impl | — |
| 6 delete images, retire credentials | vergil-containers | deployment | 1 |
| Documentation review (#4107) | vergil-tooling | bookend | 1–5 |
| Retrospective (#954) | .github | bookend | all |

## Filing outside this epic

At task-filing time, file these in mq-rest-admin-project as plain issues,
not linked to this epic, and list their refs in the tasks-filed progress
comment:

- `mq-rest-admin-project/.github`: archive `mq-rest-admin-java` (a
  `deployment` task under mq-rest-admin-project/.github#31, human-attested,
  checked with `vrg-gh repo view mq-rest-admin-project/mq-rest-admin-java
  --json isArchived`, as mq-rest-admin-project/.github#43 did for Ruby).
- `mq-rest-admin-common`: remove the Java references (fragments, docs,
  README, CLAUDE.md), under its ad-hoc epic.
- `mq-rest-admin-dev-environment`: remove the Java references, under its
  ad-hoc epic.
- `mq-rest-admin-project/.github`: remove Java from the profile and
  community files, under its ad-hoc epic.
- Edit mq-rest-admin-project/.github#18's body to drop Java.

## Evolution during execution

- **2026-11-09, Task 5 (#955).** Merge review found a second stale
  language count at `profile/README.md:93`, which the plan's Files list
  missed; it was fixed in the same PR. The human also ruled to retitle
  #384 to drop Java and Ruby, and #384's body was edited to drop Ruby.
- **2026-11-09, Task 3 (vergil-tooling#4112).** `vrg-validate` runs no
  strict mkdocs build, contrary to the plan's assumption (Task 3 step 4,
  Review focus 5), so broken links to the deleted Java pages were checked
  by search instead.
- **2026-11-09, Task 4 (vergil-actions#1106).** `vrg-validate` does not
  run vergil-actions' shell tests; CI runs them.
- **2026-11-09, Task 6 (vergil-containers#768).** The Maven Central
  token had already expired (Sonatype showed none active), and
  `CENTRAL_USERNAME` and `CENTRAL_TOKEN` existed only at repo level on
  `mq-rest-admin-java`.
- **2026-11-09, Documentation review (vergil-tooling#4107).**
  vergil-tooling needed no change, so the review closed without a PR. It
  spawned vergil-project/docs#91 and
  vergil-project/vergil-claude-plugin#995; a human scope change widened
  #995 to also remove the retired Ruby test commands from the same hook
  regex.
- **2026-11-09, #960 (added).** A follow-up task added to correct two
  stale statements in the org profile found while checking the language
  list: the repository count in "What is it?" (the org has nine
  repositories, five of them the core) and the VERGIL-Containers link,
  which still pointed at `vergil-docker`.
