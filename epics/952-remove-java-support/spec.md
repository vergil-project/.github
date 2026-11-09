# Remove Java support — design

- **Epic:** vergil-project/.github#952
- **Planning issue:** vergil-project/.github#951 (review history is in its comments)
- **Status:** draft, in design review

## 1. Intent

Remove Java as a supported language from the vergil fleet, the way Ruby was
removed under epic #407, and delete the published Java container images.

Java has **zero users**. Its only consumer,
`mq-rest-admin-project/mq-rest-admin-java`, is being archived. Every piece of
Java support is dead code that still costs maintenance: a language registry
entry, version handling, two container images and their lifecycle adapter, a
Maven Central publishing path, security-scan branches and a standards section
in the docs. Removing it reduces the fleet's complexity.

After this epic, the fleet's primary languages are `python`, `go`, `rust`,
`cpp` and `typescript`.

## 2. Goals

1. No current vergil code, workflow, image, test or doc supports or describes
   Java as a supported language.
2. A repo whose `vergil.toml` declares `primary-language = "java"` is rejected
   loudly with a clear reason, exactly as `ruby` is today. It is never
   silently accepted or treated as a language-less project.
3. The `dev-java` and `prod-java` GHCR packages are deleted.
4. Everything only Java needed goes with it, including shared infrastructure
   whose sole user is Java (§4.1, `osv-scanner`).

## 3. Non-goals and anti-goals

- **Non-goal: the mq-rest-admin-project side.** Archiving `mq-rest-admin-java`
  and removing Java references from `mq-rest-admin-common`,
  `mq-rest-admin-dev-environment` and the org profile happen in that org, on
  its own schedule (§6). This epic neither performs nor waits on them.
- **Non-goal: historical records.** Past specs, plans, superpowers docs,
  paad reports, changelogs and release notes keep their Java mentions. They
  record what was true then.
- **Non-goal: Maven Central.** Published Java artifacts cannot be deleted and
  stay as they are.
- **Non-goal: bringing Java back.** Git history keeps it.
- **Anti-goal: a soft deprecation.** No warning period, no "deprecated" flag,
  no compatibility shim. Java has zero users, so the removal is complete and
  immediate.
- **Anti-goal: collateral removal.** Unrelated code that happens to match a
  Java-ish search term stays. In particular, the GPG signing in vergil-tooling's
  `lib/package/` and in vergil-actions' package-signing steps signs apt and rpm
  package indexes, not Maven artifacts, and is untouched.
- **Anti-goal: touching GPG keys or secrets.** The GPG key signs packages
  and other artifacts beyond Maven. No GPG key, and no GitHub secret holding
  one or its passphrase, is revoked, rotated or deleted in any org. Only
  Maven-specific code paths that forward them to Maven are removed (§4.4).

## 4. What is removed

The inventory below was taken against each repo's `develop`. It is a guide,
not a contract: each task's acceptance is the case-insensitive grep in §5, so
anything the inventory missed is still in scope.

### 4.1 vergil-containers

- The Java image: `docker/java/` and the `java` entries in `docker/build.sh`
  and `docker/generate.sh`.
- The Java lifecycle adapter (`docker/versions/lifecycle/adapters/java.py`)
  and its registration, plus the `java` entries in `docker/versions/manifest.yml`,
  `matrix.json`, `window.json` and their test data.
- `osv-scanner`: `docker/common/osv-scanner.dockerfile`, its pin in
  `docker/pins/pins.yml` and `CATALOG.md`, and its entries in
  `docker/pins/resolve_latest.py`. Only the Java image includes it.
- `dev-java` and `prod-java` in `package-cache-warm.yml` and
  `package-cleanup.yml`.
- Java-only `.trivyignore` entries.
- Tests that exist only for the removed pieces; tests that list languages
  drop `java`.
- The `docs/site` Java language page, its nav entry and the Java mentions in
  the images, architecture, index and lifecycle pages, plus the README.

The lifecycle bot's open Java issue (vergil-project/vergil-containers#639,
"java add 25") is closed as not planned, since the adapter that files such
issues is gone.

### 4.2 vergil-tooling code

- `java` is added to `_RETIRED_PRIMARY_LANGUAGES` in `lib/config.py` with the
  reason `Java support was removed in epic vergil-project/.github#952`, and
  removed from the `primary-language` choice set.
- The `java` entry in `lib/languages.py`, with its Maven checks, the Java SBOM
  and `_JAVA_OSV_SCAN` audit, and the `MAVEN_GPG_PASSPHRASE` publish variable.
- Java in `lib/version.py` (`pom.xml` version file, detection and bumping),
  `lib/container.py` (`pom.xml`/`mvnw` detection), `lib/container_cache.py`,
  `lib/dependabot.py` (the `maven` ecosystem), `lib/github_config.py`,
  `lib/gitignore.py` (the Java fragment, and the packaged fragment data file
  if one exists), `lib/repo_init.py` (Java scaffolding, the
  `CENTRAL_USERNAME`, `CENTRAL_TOKEN`, `GPG_PRIVATE_KEY`, `GPG_PASSPHRASE`
  secrets map, and the `java` entry in `_CODEQL_LANGUAGES`) and
  `lib/semgrep.py` (`p/java`).
- The Java generator in `bin/vrg_generate_commands.py`, and `java` from the
  language help in `bin/vrg_semgrep_scan.py`. The generator's planned move to
  the mq-rest-admin project (vergil-project/.github#908) is unaffected: it
  moves with one fewer language.
- Tests that exist only for the removed code are deleted; tests that list
  languages drop `java`. The `tests/.../fixtures/versions/window.json` fixture
  drops its `java` entry.
- One new test: a `vergil.toml` with `primary-language = "java"` fails
  validation with the retirement reason.

### 4.3 vergil-tooling docs

- The `docs/site/docs/standards/development/java/` section and its
  `mkdocs.yml` nav entries.
- The Java mentions in the current site guides and references:
  `ci-architecture`, `consuming-repo-setup`, `credential-management`,
  `cli-tools-overview`, `config-audit`, `lint/repo-profile` and
  `runtime-version-support-policy`.
- One mention is added, not removed: `lint/repo-profile.md`'s retired-language
  sentence, which names `ruby` today, names `java` beside it.

The docs are a separate task from the code, as with Ruby, so each PR stays
reviewable.

### 4.4 vergil-actions

- `cd-release.yml`: `java` from the language list and the publish-language
  condition, the `CENTRAL_USERNAME`, `CENTRAL_TOKEN`, `GPG_PRIVATE_KEY` and
  `GPG_PASSPHRASE` secrets and their forwarding to registry-publish. The
  package-signing GPG steps stay (§3).
- `ci-security.yml`: the Java mentions in descriptions and comments.
- `registry-publish`: the `central-username`, `central-token`,
  `gpg-private-key` and `gpg-passphrase` inputs, the Maven credential and
  deploy steps, the `MAVEN_GPG_PASSPHRASE` mapping in `publish.sh`, and the
  Java test cases.
- `validate-inputs`: `java` from the accepted languages.
- `codeql` and `semgrep`: the Java cases and their tests.
- `scripts/fleet-cicd-rename.sh`: the `mq-rest-admin-java` entry and the
  `java-versions:` rewrite.
- The site docs and `docs/development/publish-workflow.md` Java mentions.

Removing the registry-publish inputs is not treated as a breaking change and
needs no major version: the only caller passing them is `mq-rest-admin-java`.

This task edits `.github/workflows/**`, so GitHub refuses the agent's push.
That is expected: report-ready records the handoff and the human's
`vrg-submit-pr` pushes the branch.

### 4.5 Remaining mentions

The `docs` repo's language page and index, and the Java pattern in
`vergil-claude-plugin`'s `hooks/scripts/detect-deprecation-warnings.sh`, are
small enough to need no task of their own. `vergil-python`'s
`docs/runtime-package-pattern.md` names the JVM only as a general example of
a runtime the packaging pattern fits, not as a vergil language, so it stays. The documentation-review bookend
(vergil-project/vergil-tooling#4107) sweeps them, filing a same-repo sibling
task in each repo that needs a change.

### 4.6 vergil-project/.github

- `profile/README.md`, the org's public profile, lists the supported
  languages as "seven … Python first, then Ruby, Rust, Go, Java, C++, and
  TypeScript". It is corrected to the five that remain (§1), with the count.
- Java is removed from the scope of vergil-project/.github#384 (restart the
  language updaters) and vergil-project/.github#410 (rank the next
  languages) by an edit to each body.

The profile edit lands as this task's PR; the two body edits need none.

## 5. Acceptance

- In each of vergil-containers, vergil-tooling and vergil-actions, a
  case-insensitive, word-bounded grep for `java`, `maven`, `mvn`, `pom.xml`,
  `jdk`, `central-username`, `CENTRAL_USERNAME` and `MAVEN_GPG` over current
  files returns no hits. Word-bounded means `\bjava\b`: `javascript` (the
  `typescript → javascript-typescript` CodeQL mapping, URLs) is not a Java
  hit and stays. Historical paths are excluded: `docs/specs/`, `docs/plans/`,
  `docs/superpowers/`, `docs/reports/`, `paad/`, `releases/` and
  `CHANGELOG.md`. The only allowed hits are vergil-tooling's
  `_RETIRED_PRIMARY_LANGUAGES` entry, its test, and the retired-language
  sentence in `docs/site/docs/reference/lint/repo-profile.md`.
- In vergil-containers, a grep for `osv-scanner` returns no hits outside
  those historical paths.
- vergil-project/.github's `profile/README.md` names exactly the five
  remaining languages.
- `primary-language = "java"` fails validation with the retirement reason.
- `vrg-container-run -- vrg-validate` passes in each changed repo, with
  vergil-tooling at 100% branch coverage.
- `gh api /orgs/vergil-project/packages/container/dev-java` and
  `.../prod-java` both return 404, run by the human (the user identity's
  `vrg-gh` denies `gh api`) and pasted into the deployment task as evidence.

## 6. The mq-rest-admin-project side

Cross-org links are out of scope, so this side is recorded here and filed in
that org as plain issues under its own epics. They are filed when this
epic's implementation tasks are filed, and the plan's task-filing step lists
each one as a filing it must make. Once
filed, they are not tracked by this epic and are not gated on. That project
is idle, so the issues wait for its next round of attention:

- **Archive `mq-rest-admin-java`**, under the repo's ad-hoc epic
  (mq-rest-admin-project/.github#31). A human archives the repo; the task
  confirms it, like mq-rest-admin-project/.github#43 did for Ruby. Its open
  issues close with the archive.
- **Remove the Java references** from `mq-rest-admin-common` (fragments,
  docs, README, CLAUDE.md), `mq-rest-admin-dev-environment` and the org
  `.github` profile and community files, each under that repo's ad-hoc epic.
- **Rescope mq-rest-admin-project/.github#18** (verified TLS across the
  language ports) to drop Java.

Until the archive happens, `mq-rest-admin-java`'s CI breaks once the Java
image and actions are gone. With zero users, that is accepted.

## 7. Delivery shape

Five implementation tasks, independent of each other: vergil-containers
(§4.1), vergil-tooling code (§4.2), vergil-tooling docs (§4.3),
vergil-actions (§4.4) and vergil-project/.github (§4.6). They can land in any
order, since nothing in the fleet resolves a Java window or image anymore.

One **deployment** task deletes the GHCR packages and retires the Maven
Central credentials. Its human-attested steps are:

- delete the `dev-java` and `prod-java` GHCR packages;
- revoke the Maven Central publishing token, and delete the
  `CENTRAL_USERNAME` and `CENTRAL_TOKEN` secrets wherever they are set in
  mq-rest-admin-project (org or repo level).

The attestation names what was done and records no secret value. GPG keys
and secrets are not touched (§3).

The task is blocked by the vergil-containers task, so the workflows stop
warming and cleaning the packages before they are deleted. Deleting a GHCR
package and revoking a credential are permanent and outside the agent's
allowlist, so both are human-attested preconditions: the human pastes the
two 404 responses (§5) and a statement that the token is revoked and the
two secrets deleted into the task. The agent checks that all of it is
present and names both packages, and records the outcome.

The documentation-review and retrospective bookends close the epic.

## 8. Left to the builder

- Wording of the docs edits that remove Java from lists and tables.
- Whether a test that mixes Java with other languages is trimmed or rewritten.
- Exact `.trivyignore` entries: keep any entry still matched by another image.
