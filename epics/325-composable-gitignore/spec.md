# Spec — Composable `.gitignore` (base + language fragments) + fleet sync

Epic: vergil-project/.github#325

## 1. Problem

The baseline `.gitignore` is a single monolithic file
(`src/vergil_tooling/data/gitignore.baseline`, 55 patterns) that the standards
audit (`repo_config._check_gitignore`) requires every managed repo to carry as a
verbatim **superset**. Two structural failures follow:

1. **One size fits all is wrong.** Every repo is forced to carry every
   language's rules. `.github`, the `docs` repos, and every non-C++ repo all
   carry the 16 C++/Conan ignore lines they will never use.
2. **Propagation is a manual N-PR sweep.** A baseline change must be hand-applied
   to every repo. Epic vergil-project/.github#322 proved it: a 12-PR sweep across
   two orgs to add a few lines — worsened by the `vrg-finalize-pr` stale-check
   race (vergil-tooling#2914) surfaced along the way.

The superset audit is also **append-only**: it can never *remove* a now-unneeded
line, so the C++ rules just pushed fleet-wide can never be cleaned up.

## 2. Goals / Non-goals

**Goals:**

- A repo carries only the ignore rules for `base` + its own primary language.
- Baseline changes propagate across a repo collection from **one command**.
- Propagation can **add, remove, and reorder** lines (idempotent), not just add.
- The audit and the fix share one implementation, so they cannot disagree.
- Build the tooling to a clean seam so a generic fleet-sweep driver is a later
  extraction, not a rewrite.

**Non-goals (this epic):**

- Multi-language composition (reserved escape hatch; built only if forced).
- The fully-generic "apply an arbitrary change-script across a repo set" driver
  (seeded as the follow-on brainstorm, #328).
- Auto-fixing CI (CI audits and reports; humans hold the submit/merge gate).

## 3. Composition model

A repo's managed ignore set = **`base` + the one fragment named by its
`primary_language`** (from `vergil.toml`, resolved by the existing
`resolve_language`). Base is language-agnostic; each fragment is that language's
build/cache/dep output. Genuinely shared rules are promoted into `base` rather
than duplicated across fragments.

**No-fragment repos compose base-only.** When a repo's language has no fragment —
absent, empty, or a value not in the registry (e.g. `.github` declares
`language: shell`, `docs` declares none) — the composition is **`base` alone**.
This is the correct answer for the epic's motivating repos: `.github` gets `base`
and nothing else. One rule covers all three cases (absent / empty / unlisted).

## 4. File layout

Fragment sources live in vergil-tooling and are packaged as data:

```text
src/vergil_tooling/data/gitignore/
  base
  python   cpp   go   ruby   rust   java   typescript
```

One fragment per language in the `languages.py` registry. `rust` and `java`
start minimal (no language-specific lines in today's baseline) and are populated
as needed. There is no `shell` fragment; shell repos compose base-only (§3).

## 5. Content split (all 55 current patterns assigned)

Principle: **base = language-agnostic; fragment = language-specific.**

**base (24):**
`*.swp`, `*.swo`, `*~`, `*.bak`, `.idea/`, `.vscode/`, `.DS_Store`, `Thumbs.db`,
`.env`, `.env.*`, `*.log`, `.venv/`, `.worktrees/`, `.vergil/`, `.superpowers/`,
`.claude/scheduled_tasks.lock`, `.claude/settings.local.json`, `build/`, `dist/`,
`coverage.xml`, `junit.xml`, `licenses.json`, `docs/site/site/`, `node_modules/`

- `node_modules/` is in base deliberately: node-based tooling (markdownlint,
  etc.) appears fleet-wide, not only in TypeScript repos.
- `build/`, `dist/`, `coverage.xml`, `junit.xml`, `licenses.json` are
  cross-language build/packaging/validate-evidence artifacts.

**python (10):**
`__pycache__/`, `*.pyc`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`,
`*.egg-info/`, `.coverage`, `pip-audit.json`, `quality-ruff.json`,
`quality-mypy.xml`

**cpp (16):**
`build-sanitize/`, `*.o`, `*.obj`, `*.a`, `*.so`, `CMakePresets.json`,
`CMakeUserPresets.json`, `cmakedeps_macros.cmake`, `conan_toolchain.cmake`,
`conandeps_legacy.cmake`, `conanbuild*.sh`, `conanrun*.sh`, `conanbuildenv-*.sh`,
`conanrunenv-*.sh`, `deactivate_conanbuild*.sh`, `deactivate_conanrun*.sh`

**go (2):** `*.test`, `*.out` — **ruby (2):** `.bundle/`, `vendor/bundle/` —
**typescript (1):** `*.tsbuildinfo` — **rust/java:** empty for now.

A Python repo composes 24 + 10 = **34** lines (sheds the 21 cpp/go/ruby/ts lines
it carries today). A shell/no-language repo (`.github`) composes the 24 base
lines only (sheds all 31 language lines).

**Lossless-split invariant:** the union of `base` + **all** fragments must equal
the 55 current monolith patterns exactly — no pattern dropped, none invented.
This is enforced by a test (§11) and is what makes deleting the monolith safe
(§9, §10).

## 6. Managed-block format

One fenced region at a fixed, predictable location (top of file; repo-local
content follows). Exact marker lines:

```text
# >>> vergil-managed: base + <lang> (managed by vrg-gitignore-sync; do not edit) >>>
<base lines, in base order>
<fragment lines, in fragment order>   # omitted entirely for base-only repos
# <<< vergil-managed <<<
```

- The BEGIN/END lines are the parser's anchors. The `base + <lang>` descriptor
  (or `base` alone for no-fragment repos) is regenerated deterministically from
  `primary_language`, so the whole block is byte-for-byte reproducible (the audit
  compares exactly).
- Repo-local entries live outside the fence and are never touched.
- **Fence presence is the bootstrap/update signal** (§9).

## 7. Components — the two-layer seam

### 7.1 `lib/gitignore.py` (pure logic, no git/PR knowledge)

- `load_base() -> list[str]`, `load_fragment(lang) -> list[str]`
- `compose(lang) -> list[str]` — `base` then the language fragment (base-only
  when no fragment); de-duplicated, order-stable
- `MANAGED_BEGIN_RE`, `MANAGED_END`, `render_block(lang) -> str`
- `parse(text) -> Parsed(repo_local: list[str], fence: str | None)`
- `managed_vocabulary() -> set[str]` — union of `base` + **all** fragments (the
  complete language set). **No** separate legacy-monolith constant: because the
  split is lossless (§5), this union reconstitutes every pattern the monolith
  ever held, so historical foreign-language lines (e.g. cpp lines in a python
  repo) are recognized and filtered.
- `sync(text, lang) -> str` — bootstrap or update per fence presence (§9);
  idempotent
- `check(text, lang) -> Compliance` — fence present, equals `render_block(lang)`
  byte-for-byte, and no managed-vocabulary lines stray outside the fence

### 7.2 `vrg-gitignore-sync` — the applicator CLI (one repo)

- Resolves `primary_language` from the repo's `vergil.toml` (base-only when
  absent/empty/unlisted).
- `--check`: exit non-zero with a precise report when non-compliant (this *is*
  the audit).
- `--write`: apply `sync`, write the file, log what was removed and why
  (e.g. "dropped 16 lines matching the `cpp` fragment; this repo is `python`").
  Idempotent — a compliant repo is a no-op that reports "already in sync."
- Knows nothing about branches, commits, issues, or PRs.

### 7.3 `repo-init` integration

`repo_init.render_gitignore()` composes through `lib/gitignore.py` — it emits the
fenced `base + <language>` block (base-only when no fragment), so **init, sync,
and audit are one code path**. A newly-inited or adopted repo is born compliant
with the new audit. (Without this, `repo-init` would keep stamping the monolith
and produce files that fail the very audit this epic ships — commit `f8f8973b4`
made repo-init regenerate managed files, so it is a live second writer of
`.gitignore`.)

### 7.4 Fleet driver — the generic per-repo work-chain

A driver that, given a repo list and an applicator + sweep metadata, runs per
repo: ensure the tracking issue (ad-hoc or under a given epic) → worktree/branch
→ run the applicator → **if it changed anything** commit + `report-ready`; **else
skip cleanly** → collect results into a summary. Dry-run supported. Humans still
submit/merge.

Built gitignore-shaped now but factored to the seam (applicator = file change;
driver = git/PR/issue work-chain + iteration), so the generic
arbitrary-change-script driver (#328) is an extraction, not a rewrite.

## 8. Audit integration

`repo_config._check_gitignore` becomes a thin caller of `gitignore.check()` — the
same code path as `vrg-gitignore-sync --check` — so audit and fix can never
diverge. It runs in the common/standards check exactly where the superset check
runs today. Violations are reported as `DiffItem`s naming the exact drift
(missing fence, block mismatch, stray managed line).

## 9. Migration (bootstrap vs update)

The tool distinguishes the two cases by **fence-marker presence**:

- **Bootstrap (no fence).** The repo still carries the loose monolith. Write the
  `base + <this language>` fence, then build the repo-local section from the
  existing lines that match **none** of base/any-fragment
  (`managed_vocabulary()`). Because the vocabulary spans *all* languages, the
  historical cpp/java/ts lines in a python repo match their fragments and are
  filtered out → a minimal `.gitignore`. Genuinely repo-local lines (in no
  baseline file) flow through into the repo-local section.
- **Update (fence present).** The repo is already converted. Rewrite the fence to
  the current `render_block(lang)`; trust the already-clean repo-local section.

Idempotent: re-running on a converted repo is a no-op. Safety: every change ships
as a reviewed per-repo PR, so each removal is a visible diff line; a genuinely-
polyglot repo that wanted, say, `*.o` sees it removed at review and is handled by
promote-to-base / the multi-language escape hatch.

**Monolith deletion is safe** because filtering against `base + all-fragments` is
provably equivalent to filtering against the old monolith once the lossless-split
invariant (§5, §11) is green. No frozen legacy constant is retained — the
partitioned base+fragments *are* the legacy vocabulary.

## 10. Rollout (release first, always)

The audit swap must not strand un-migrated repos with red CI, and consuming
repos run the audit from the **pinned, released** vergil-tooling tag, not develop.
So the sweep always runs against a released tool. Sequence:

1. **Merge + release** the compose lib + `vrg-gitignore-sync` + `repo-init`
   integration + fleet driver, with the audit made **transitional**: a repo
   passes if it is *either* the legacy monolith superset *or* correctly fenced.
   No repo goes red on release.
2. **Deployment gate (operational task).** Verify the released tag carries the
   transitional audit and the tool composes the intended fences — the sweep is
   **blocked-by** this task. (`impl → deploy → sweep`.)
3. **Sweep** all repos in both orgs via the fleet driver, run with the released
   tool (dogfoods the driver; replaces the manual sweep). Each repo gets a
   migration PR that *removes* its foreign-language lines.
4. **Tighten** the audit to fenced-only once every repo carries the fence, and —
   with the lossless-split invariant test green — **delete** the legacy monolith
   and its acceptance path.

## 11. Testing

- **Unit (`lib/gitignore.py`):** `compose` per language + base-only; `parse`
  (fence / no fence / malformed); `render_block` determinism; `check` (compliant,
  missing fence, block mismatch, stray managed line); `sync` bootstrap
  (foreign-language-line removal from a python repo, repo-local preservation) and
  update (fence rewrite, repo-local untouched); idempotency.
- **Lossless-split invariant:** `base ∪ all-fragments == the 55 monolith
  patterns` (no pattern dropped or invented). Gates monolith deletion (§10.4).
- **`repo-init`:** a freshly-inited repo is born compliant (fenced,
  base-only when no fragment).
- **Applicator CLI:** `--check` exit codes; `--write` idempotency + removal
  logging.
- **Fleet driver:** dry-run; skip-when-compliant; per-repo result collection;
  failure isolation (one repo failing does not abort the sweep).
- **Audit:** transitional acceptance (both forms), then fenced-only.
- Replace the old monolith drift-guard test (`baseline ⊆ flagship`) with the
  fenced-form equivalent for vergil-tooling's own `.gitignore`.

## 12. Risks & backward compatibility

- **Release/version skew:** closed by "release first, always" + the deployment
  gate (§10) — the sweep runs with the same released tool CI enforces, so the
  fence it writes matches the fence the audit expects byte-for-byte.
- **Transition window:** the transitional audit (§10.1) means no red-CI window
  and no strict tool-vs-audit ship ordering.
- **Wrongly-classified repo-local line:** a repo-local line coincidentally
  matching a baseline pattern is removed on bootstrap; caught by per-repo PR
  review. Acceptable and rare.
- **`build/` in base:** ubiquitous build-dir name; harmless where absent.
- **Fence relocation churn:** bootstrap may move content; one-time, reviewed.

## 13. Follow-on

The generic fleet-sweep driver (#328): extract the driver's per-repo work-chain
into a mechanism that applies an arbitrary change-script across a repo
collection — the reusable engine for standards enforcement across the fleet.
This epic deliberately builds to that seam.
