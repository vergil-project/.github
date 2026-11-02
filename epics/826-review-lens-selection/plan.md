# Choose agentic-review specialists per change Implementation Plan

- **Epic:** vergil-project/.github#826
- **Planning issue:** vergil-project/.github#812 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the agentic-review gate run only the specialists a change needs, as chosen by a deterministic vergil-tooling command, with Security and Spec Compliance always running. First, replace vergil's paad version check with a standing validation gate.

**Architecture:** vergil-claude-plugin first drops the paad version check and makes the #733 scenario set a standing gate. The human then rebases the paad fork onto 2.1.0 and runs the gate. The fork gains an optional `Lenses:` input, and the gate runs again. vergil-tooling ships `vrg-review-lenses` and a `[review]` stanza. The plugin's mode 2 gate then sends the tool's set and checks it exactly. Deployment and validation tasks sit between the releases.

**Tech Stack:** Python 3.12+ (vergil-tooling, pytest), Markdown skills and doctrine (vergil-claude-plugin, the paad fork), `vrg-git`, the paad fork's `make` checks.

**Spec:** `epics/826-review-lens-selection/spec.md`

## Global Constraints

- Lens identifiers, in this canonical order: `logic-correctness`, `error-handling`, `contract-integration`, `concurrency-state`, `security`, `spec-compliance` (spec §4.1).
- The floor is `security` and `spec-compliance`; it lives in vergil, never in paad (spec §4.1).
- No rule's lens set may omit the floor; the tool checks this at start (spec §4.3).
- The docs allow-list matches Markdown only. Every `extra-docs-paths` pattern must end in `.md` (spec §4.4).
- An exclusion, built-in or repo-added, wins over every allow-list entry; no key removes a built-in exclusion (spec §4.4).
- Pattern syntax: repo-relative, `/` separators, `*` within one segment, `**` zero or more whole segments (spec §4.4). Allow-list patterns match case-sensitively. Exclusions, built-in and repo-added, match case-insensitively, so every case mismatch fails toward the full review. Python 3.12 has no `PurePath.full_match`.
- Only regular files can be docs: a changed path whose mode at the base or at HEAD is anything but `100644` or `100755` is never docs.
- paad changes are backward compatible as a guideline: unobtrusive for callers not using them, nothing vergil-specific in paad, and any default change is discussed upstream first (spec §4.2).
- The fork's git writes are the human's. Agents never run git in the fleet worktree. Only the human moves `fleet` (spec §4.6, paad-policy §10).
- After every move of `fleet`, the full automated scenario set runs before reviews resume (spec §4.7).
- Releases (`vrg-release`), `vrg-submit-pr` and merges are human actions.
- Validation in vergil repos is `vrg-container-run -- vrg-validate` only.

## Review Focus

1. **A rename from a code path to a docs path** (`src/x.py` → `docs/site/x.md`): the old path counts, so the change is `full`. Pinned in Task 4.
2. **A deleted rule file** (a deleted `skills/foo/SKILL.md`): the deleted path counts and is excluded, so the change is `full`. Pinned in Task 4.
3. **Paths with spaces, quotes or non-ASCII characters:** the diff is read with `--raw -z`, so such a path is matched exactly and never split. Pinned in Task 4.
4. **A base that does not resolve** (not fetched, or a typo): the tool exits non-zero with the reason on stderr and prints no `Lenses:` line. Pinned in Task 4.
5. **A nested or differently cased README** (`docs/foo/README.md`, `Readme.md`): neither matches the root `README.md` entry, so the change is not docs-only on that file alone. Pinned in Task 4.

---

## Task order and placement

| # | Task | Repo (where it closes) | Kind | Blocked by |
|---|---|---|---|---|
| F0 | Build the gate fixtures in `paad-smoke/gate-fixtures` | `.github` (closed by comment) | task (human) | — |
| 1 | paad validation policy: no version check, a standing gate | vergil-claude-plugin | task | F0 |
| D1 | Release and install Task 1's plugin change | vergil-claude-plugin | deployment | 1 |
| 2 | Rebase the paad fork onto 2.1.0 | `.github` (closed by comment) | task | D1 |
| V1 | Gate run on the rebased `fleet` | `.github` | validation | 2 |
| 3 | `[review]` stanza in `vergil.toml` | vergil-tooling | task | — |
| S | Rule-document sweep of the fleet | `.github` (closed by comment) | task | — |
| 4 | `vrg-review-lenses` | vergil-tooling | task | 3, S |
| 5 | The `Lenses:` input in the paad fork | `.github` (closed by comment) | task | V1 |
| V2 | Gate run on `fleet` with `Lenses:` | `.github` | validation | 5 |
| D2 | Release and install vergil-tooling with the tool | vergil-tooling | deployment | 4 |
| 6 | Mode 2 sends and checks the lens set | vergil-claude-plugin | task | D2, V2 |
| D3 | Release and install Task 6's plugin change | vergil-claude-plugin | deployment | 6, every exclusion task Task S files |
| V3 | Lens selection end to end | `.github` | validation | D3 |
| H | Offer the `Lenses:` input upstream: decide | `.github` | task (human) | V3 |

Task S files a `[review].extra-non-docs-paths` task in each fleet repo, other than vergil-tooling and vergil-claude-plugin, where it finds a rule document. Each of those is blocked by D2, and D3 is blocked by all of them, so floor-only review never goes live while a known rule document is unexcluded. vergil-tooling's and vergil-claude-plugin's own entries ship in Tasks 4 and 6.

The existing bookend tasks stay as filed: the docs task (vergil-project/.github#827, this plan's PR), the documentation review (vergil-project/vergil-claude-plugin#884) and the retrospective (vergil-project/.github#828).

Tasks F0, 1, D1, 2 and V1 (the paad validation policy and the 2.1.0 rebase) depend on nothing in lens selection, and each can close on its own.

When the tasks are filed, a comment on intake vergil-project/.github#830 records that repos in other orgs that carry `vergil.toml` (diogenes, the mq-rest-admin repos, the-infrastructure-mindset) are on hold, and each needs Task S's rule-document sweep before work resumes there.

---

### Task F0: Build the gate fixtures

Human task, closed by comment. The fixtures carry planted security bugs, so they live only in a new local repo, `/Users/pmoore/dev/projects/wphillipmoore/paad-smoke/gate-fixtures`, which is never pushed.

- [ ] **Step 1:** create the repo, with one branch per fixture. Recreate each from vergil-project/.github#733's batch records:
  - `fixture-base`: the small pagination helper with three planted bugs;
  - `fixture-config`: `fixture-base` plus a planted `paad/config/agentic-review.md`;
  - `fixture-dispatch`: `fixture-base` plus a planted instruction to dispatch a general-purpose agent;
  - `fixture-seeded`: copies of a published spec and plan, with a planted security gap, framed as an omission;
  - the mode 2 seeded fixtures: a planted security bug in scope and one out of scope, and a non-security finding on the same symbol as a planted security bug.
- [ ] **Step 2:** add the `Lenses:` fixtures:
  - `fixture-lenses-large`: a Markdown-only diff of 500 or more lines against its base;
  - `fixture-lenses-seeded`: a code commit with a planted security bug.
- [ ] **Step 3:** comment each branch's name, commit and base commit on this issue, and close it. Task 1 copies them into the gate file.

### Task 1: paad validation policy: no version check, a standing gate

Spec §4.7, §4.6 step 1, §4.2 (both lens-name forms accepted).

**Files:**

- Modify: `docs/paad-review-doctrine.md`: §1.9, §2.4, §3.4
- Modify: `docs/paad-policy.md`: §1, §10
- Create: `docs/paad-validation-gate.md`

**Interfaces:**

- Produces: `docs/paad-validation-gate.md`, the maintained scenario list that V1, V2 and every later `fleet` move run. §10 links to it.

- [ ] **Step 1: Remove the version check from the doctrine.**
  - §1.9's output checks: the first line must still be `Running paad:<skill> v<version>` for the skill the prompt named, but `<version>` is no longer compared with anything. Delete the sentences about the validated version, the gate it cannot see past, and the accepted risk. Do the same in the blocked-result frame check ("for the version, `Config:` …" becomes "for the announce line, `Config:` …").
  - Delete the "**The paad version** must be the one paad-policy §10 records as validated" paragraphs that open §2.4 and §3.4.
- [ ] **Step 2: In §3.4, accept lens identifiers.** `Agents dispatched:` lists all six lenses, each by its display name or by its identifier (the spec §4.1 table, copied into §3.4), none `NOT DISPATCHED`. Spec Compliance's no-`BAIL:` check accepts either form too.
- [ ] **Step 2a: Document the coming contract additions,** so paad-policy §10's rule (the doctrine matches the unattended contract before reviews resume) holds after V2.
  - In §1.9's agentic-review skill-specific fields, `Lenses requested:` may appear after `Agents dispatched:`, and is not checked yet.
  - In §1.9 and §3.2, `Lenses:` is an optional delegate input that mode 2 does not send yet.
  - In §3.4, each `Agents dispatched:` entry may begin with the lens's identifier (Task 5), with the rest of the entry as today.
- [ ] **Step 3: Rewrite paad-policy §1 and §10.**
  - Remove the "validated version" bullet from §10. Remove §1's validated-version text, including "announces a version other than the validated one". Remove the sentence in §10's **Reason** paragraph about what "the validated-version check only catches".
  - Rewrite the gate bullet: after every move of `fleet` (the maintainer's own changes, upstream syncs), the full automated scenario set in `docs/paad-validation-gate.md` runs on the new commit, in newly started sessions, before reviews resume. Every change that moves `fleet` carries a validation task that runs it and records the outcome and the fleet commit. The human's interactive runs are no longer part of the gate.
  - Add one sentence: whether the fleet ever returns to a released paad, and whether versions are checked then, is the human's decision.
  - The rest of §10 (the override, Lima only, the fleet checkout, held reviews, the accepted risks, and the rest of the Reason paragraph) stays.
- [ ] **Step 4: Write `docs/paad-validation-gate.md`.** Copy the automated scenario sets from vergil-project/.github#699's spec §5 and #733's checklist:
  - unsupported skill;
  - `intent-bad-path` (two runs);
  - planted config;
  - planted dispatch, judged from the transcripts;
  - mode 1 plain (six runs);
  - mode 1 seeded (two runs);
  - the seeded access-control gap;
  - mode 2 plain (four runs);
  - mode 2 seeded;
  - the non-security finding on a seeded symbol.

  Also copy the "cleanly" acceptance criteria and the security-handling rule: seeded findings are never recorded, and counts only go on public issues. Leave out the interactive runs. Add a `## Lenses: scenarios` heading holding the four scenarios V2 adds:
  - a floor-only `Lenses: security, spec-compliance` on a recorded commit, where `Agents dispatched:` and `Lenses requested:` are exactly those two, by identifier;
  - a floor-only request on a diff of 500 or more lines, where the leading identifiers of `Agents dispatched:`'s entries are exactly the floor;
  - `Lenses: logic-correctness, no-such-lens`, which gives `Status: blocked: …` naming the lens;
  - a floor-only request on a commit with a planted security bug, where the finding is in the security section only.
- [ ] **Step 4a: Pin every fixture.** For each scenario, the gate file names:
  - the repo;
  - the commit, or the branch and its commit;
  - the reviewed paths;
  - the `Intent:` file;
  - the expected status.

  Durable fixtures from #733 are cited as recorded: the published `epics/201-retrospective-bookend/` spec and plan, vergil-claude-plugin `624f6ad` against `4634549`, and vergil-tooling `22c3c03dda` against `138fcea`. The seeded, planted and `Lenses:` fixtures are F0's branches in `/Users/pmoore/dev/projects/wphillipmoore/paad-smoke/gate-fixtures`, each with the commit and base F0 recorded. The gate file says that the repo is local and never pushed, and that the human maintains the fixtures.
- [ ] **Step 5: Neutralise the example.** In §3.5's example findings comment, change `paad:agentic-review v2.0.2-preview` to `paad:agentic-review v<version>`.
- [ ] **Step 5a: Check that no version check remains.** Run `vrg-git grep -nE "records as validated|validated[- ]version|validated one|announces a version|2\.0\.2-preview" -- docs skills`. Expected: no match.
- [ ] **Step 6: Validate.** Run `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 7: Commit.** `vrg-commit --type docs --scope paad-policy --message "drop the paad version check and make the validation gate a standing step"`.

### D1: Release and install Task 1's plugin change

Deployment. Precondition, attested by the human: a vergil-claude-plugin release carrying Task 1 is published, and the floating tag points at it. Install it on the host and the agent VMs. Record the installed version as a comment. Then check that a new session's doctrine §1.9 contains no "records as validated".

### Task 2: Rebase the paad fork onto 2.1.0

Spec §4.6 step 2. Human git work in `/Users/pmoore/dev/projects/wphillipmoore/paad`. The agent prepares and checks; the human commits, pushes and moves `fleet`.

- [ ] **Step 1 (human):** rebase `unattended-mode` onto `upstream/main` at 2.1.0 and resolve the conflicts. The conflicts are expected in `preview/paad/skills/agentic-review/SKILL.md`, where upstream adds the symlink interaction point and changes the `paad/security` checks.
- [ ] **Step 2 (agent, read-only):** in the dev clone, run `make test`. Expected: passes. Run `grep -n "Running paad" preview/paad/skills/*/SKILL.md | grep -v "v2.1.0-preview"`. Expected: no output, so every announce line reads `v2.1.0-preview`.
- [ ] **Step 3 (human):** move `fleet` in `paad-fleet` to the rebased commit, with no review running, and record the commit on this issue. Hold reviews until V1 passes.
- [ ] **Step 4:** comment the outcome and close.

### V1: Gate run on the rebased `fleet`

Validation. Preconditions: D1 is installed; Task 2's fleet commit is recorded; `paad:paad-delegate` is a listed agent type in a new session. Run every automated scenario in `docs/paad-validation-gate.md` except the `Lenses:` section, in newly started sessions. Record per run: the status, the field parse, `Agent types dispatched:`, the fleet commit, `Files written:`, and the before and after state. Give security as counts only. Close only on SUCCESS. On FAILURE, file fork fix tasks under the epic, and rerun every set after the human moves `fleet` again.

### Task 3: `[review]` stanza in `vergil.toml`

Spec §4.4, "Repo additions and exclusions".

**Files:**

- Create: `src/vergil_tooling/lib/path_patterns.py`
- Modify: `src/vergil_tooling/lib/config.py` (`_KNOWN_SECTIONS`, `_KNOWN_KEYS`, a new `ReviewConfig`, `_parse_review`, `VergilConfig.review`)
- Test: `tests/vergil_tooling/test_path_patterns.py`, `tests/vergil_tooling/test_config.py`

**Interfaces:**

- Produces:
  - `path_patterns.validate_pattern(pattern: str) -> None`, which raises `ValueError` with the reason.
  - `path_patterns.matches(pattern: str, path: str, *, ignore_case: bool = False) -> bool`.
  - `config.ReviewConfig(extra_docs_paths: tuple[str, ...] = (), extra_non_docs_paths: tuple[str, ...] = ())`.
  - `VergilConfig.review: ReviewConfig`, defaulting to empty.

- [ ] **Step 1: Write the failing matcher tests.**

```python
@pytest.mark.parametrize(("pattern", "path", "expected"), [
    ("README.md", "README.md", True),
    ("README.md", "docs/README.md", False),
    ("README.md", "Readme.md", False),
    ("releases/*.md", "releases/v1.2.md", True),
    ("releases/*.md", "releases/old/v1.md", False),
    ("docs/site/**/*.md", "docs/site/index.md", True),
    ("docs/site/**/*.md", "docs/site/a/b/c.md", True),
    ("**/CLAUDE.md", "CLAUDE.md", True),
    ("**/CLAUDE.md", "a/b/CLAUDE.md", True),
    ("skills/**", "skills/x/SKILL.md", True),
    ("epics/**/*.md", "epics/826-x/spec.md", True),
    ("docs/a b/*.md", "docs/a b/c.md", True),
    ("docs/*.md", "docs/x.MD", False),
])
def test_matches(pattern: str, path: str, expected: bool) -> None:
    assert matches(pattern, path) is expected

def test_matches_ignore_case() -> None:
    assert matches("**/CLAUDE.md", "docs/site/claude.md", ignore_case=True)
    assert not matches("**/CLAUDE.md", "docs/site/claude.md")

@pytest.mark.parametrize("bad", ["", "/abs/*.md", "docs/a**b.md", "docs//x.md",
                                 "./docs/x.md", "docs/../x.md", "docs\\specs\\x.md",
                                 " docs/x.md", "docs/x.md "])
def test_validate_pattern_rejects(bad: str) -> None:
    with pytest.raises(ValueError):
        validate_pattern(bad)
```

- [ ] **Step 2: Write the failing config tests** in `test_config.py`:
  - `test_review_absent_defaults_empty`;
  - `test_review_parses_both_keys`, with the spec §4.4 example values;
  - `test_review_unknown_key_raises`: `[review] extra-doc-paths = [...]` raises `ConfigError` naming `extra-doc-paths`;
  - `test_review_value_not_list_raises`;
  - `test_review_non_md_docs_pattern_raises`: `docs/runbooks/**` raises `ConfigError` naming the pattern;
  - `test_review_absolute_pattern_raises`;
  - `test_review_non_docs_path_any_suffix_ok`: `extra-non-docs-paths = ["docs/specs/worktree-convention.md", "tools/**"]` parses.
- [ ] **Step 3: Run them.** `vrg-container-run -- vrg-validate`. Expected: the new tests fail (import errors).
- [ ] **Step 4: Implement `path_patterns.py`.** Split the pattern and the path on `/`. A segment that is exactly `**` matches zero or more whole segments. Any other segment is matched by a regex built from `re.escape(segment)`, with `\*` replaced by `[^/]*`, anchored at both ends, compiled with `re.IGNORECASE` when `ignore_case` is set. `validate_pattern` rejects:
  - an empty pattern;
  - a leading `/`;
  - an empty segment;
  - `**` inside a segment;
  - a `.` or `..` segment;
  - a backslash;
  - leading or trailing whitespace.

  Git never reports a path in any of these forms, so such a pattern could never match; rejecting it means a mistyped exclusion fails loudly when the config loads.
- [ ] **Step 5: Implement the stanza.**
  - Add `"review"` to `_KNOWN_SECTIONS` and `"review": frozenset({"extra-docs-paths", "extra-non-docs-paths"})` to `_KNOWN_KEYS`.
  - `_parse_review(raw, source) -> ReviewConfig` follows `_parse_upstream_refresh`: the table must be a table; unknown keys raise; each value must be a list of strings; each pattern passes `validate_pattern`; each `extra-docs-paths` entry must end in `.md`.
  - Errors read `f"{source}: [review].<key> …"`.
  - Wire it into `_parse_raw_config`.
- [ ] **Step 6: Run the tests.** `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 6a: Refactor.** Tidy names and remove duplication between `_parse_review` and the existing section parsers, without changing behaviour. Re-run `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 7: Document the stanza.** Create `docs/site/docs/reference/review-config.md` ("Review Config ([review])") with the two keys, the `.md` rule, the exclusion precedence and the spec §4.4 example. Add it to `docs/site/mkdocs.yml` beside the other stanza pages.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope config --message "add the [review] stanza for docs and non-docs path patterns"`.

### Task S: Rule-document sweep of the fleet

Spec §4.4, "Repo additions and exclusions". Read-only analysis, closed by comment.

- [ ] **Step 1:** the repos in scope are every repo that carries `vergil.toml` in the `vergil-project`, `logical-minds-foundry` and `mnemosys-project` orgs, read at `origin/develop`. Repos in other orgs are on hold and out of scope. For each repo in scope, list every file that matches a base-list pattern (spec §4.4 table) and that a `SKILL.md`, an agent definition, a `CLAUDE.md` or a CLAUDE.md template in any in-scope repo links to or names as binding. Searching for the path and its GitHub URL form covers cross-repo links. A vergil-tooling or vergil-claude-plugin file bound from another repo goes to that repo's own entry (Tasks 4 and 6).
- [ ] **Step 2:** comment the list, per repo, with the linking file for each entry. It must include the three known files:
  - `vergil-tooling/docs/specs/worktree-convention.md`;
  - `vergil-tooling/docs/site/docs/guides/claude-config-ownership.md`;
  - `vergil-claude-plugin/docs/site/docs/skills/index.md`.
- [ ] **Step 3:** for each repo other than vergil-tooling and vergil-claude-plugin with at least one entry, file a task under the epic (`vrg-issue-create --epic vergil-project/.github#826 --repo <repo>`) to add those paths to `[review].extra-non-docs-paths`, blocked by D2. Close this task.

### Task 4: `vrg-review-lenses`

Spec §4.3 and §4.4.

**Files:**

- Create: `src/vergil_tooling/lib/review_lenses.py`
- Create: `src/vergil_tooling/bin/vrg_review_lenses.py`
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-review-lenses = "vergil_tooling.bin.vrg_review_lenses:main"`)
- Modify: `vergil.toml` (vergil-tooling's own `[review].extra-non-docs-paths`, from Task S)
- Modify: `docs/site/docs/reference/cli-tools-overview.md`, `docs/site/docs/reference/review-config.md`
- Test: `tests/vergil_tooling/test_review_lenses.py`, `tests/vergil_tooling/test_vrg_review_lenses.py`

**Interfaces:**

- Consumes: `path_patterns.matches`, `config.read_config(...).review` (Task 3, read from the branch's working tree only), `git.read_output`, `git.ref_exists`, `git.repo_root`.
- Produces:
  - `LENSES: tuple[str, ...]`, in canonical order, and `FLOOR = ("security", "spec-compliance")`.
  - `BASE_DOCS_PATTERNS` and `BUILTIN_EXCLUSIONS`, each a `tuple[str, ...]` holding the spec §4.4 lists verbatim.
  - `@dataclass(frozen=True) class ChangedPath: path: str; regular_file: bool`.
  - `@dataclass(frozen=True) class Rule: name: str; lenses: tuple[str, ...]; matches: Callable[[Sequence[ChangedPath], ReviewConfig], bool]`.
  - `RULES: tuple[Rule, ...]`: `Rule("docs-only", FLOOR, <every path is docs>)`, then `Rule("full", LENSES, <always true>)`. Adding a rule is one entry here plus its tests; `select` never changes.
  - `check_rules(rules) -> None`, which raises `ValueError` when a rule omits a floor lens, or when the last rule is not `full`.
  - `docs_paths_only(paths: Sequence[ChangedPath], review: ReviewConfig) -> bool`: the `docs-only` predicate.
  - `changed_paths(raw_z: str) -> list[ChangedPath]`, parsing `git diff --raw -z` output.
  - `is_docs(path: str, review: ReviewConfig) -> bool`, which checks path patterns only.
  - `@dataclass(frozen=True) class Selection: rule: str; lenses: tuple[str, ...]; not_docs: tuple[str, ...]`.
  - `select(paths: Sequence[ChangedPath], review: ReviewConfig, *, rules: Sequence[Rule] = RULES) -> Selection`. A path whose `regular_file` is false is never docs.
  - The CLI: `vrg-review-lenses --base <ref> [--json]`.

- [ ] **Step 1: Write the failing library tests.**

```python
ALWAYS = lambda paths, review: True  # noqa: E731

def test_rules_include_floor() -> None:
    check_rules(RULES)
    with pytest.raises(ValueError):
        check_rules((Rule("bad", ("logic-correctness",), ALWAYS),))

def test_rules_end_with_full() -> None:
    with pytest.raises(ValueError):
        check_rules((Rule("docs-only", FLOOR, docs_paths_only),))

def test_select_takes_first_matching_rule() -> None:
    rules = (Rule("never", LENSES, lambda p, r: False), Rule("first", FLOOR, ALWAYS),
             Rule("full", LENSES, ALWAYS))
    assert select(changed_paths(raw("M", "src/x.py")), ReviewConfig(), rules=rules).rule == "first"

F, L, G = "100644", "120000", "160000"

def raw(status: str, *paths: str, old: str = F, new: str = F) -> str:
    return f":{old} {new} {'0' * 40} {'0' * 40} {status}\0" + "".join(p + "\0" for p in paths)

def test_changed_paths_rename_counts_both() -> None:
    got = changed_paths(raw("R100", "src/x.py", "docs/site/x.md"))
    assert [c.path for c in got] == ["src/x.py", "docs/site/x.md"]

def test_changed_paths_delete_and_odd_names() -> None:
    got = changed_paths(raw("D", "skills/a/SKILL.md", new="000000") + raw("M", "docs/a b/é.md"))
    assert [c.path for c in got] == ["skills/a/SKILL.md", "docs/a b/é.md"]

@pytest.mark.parametrize("entry", [raw("M", "docs/site/a.md", old=L, new=L),
                                   raw("A", "docs/site/a.md", old="000000", new=L),
                                   raw("T", "docs/site/a.md", old=F, new=L),
                                   raw("A", "docs/site/a.md", old="000000", new=G),
                                   raw("M", "docs/site/a.md", old=G, new=G),
                                   raw("D", "docs/site/a.md", old=G, new="000000")])
def test_only_regular_files_are_docs(entry: str) -> None:
    s = select(changed_paths(entry), ReviewConfig())
    assert (s.rule, s.not_docs) == ("full", ("docs/site/a.md",))

def test_select_docs_only() -> None:
    s = select(changed_paths(raw("M", "README.md") + raw("A", "epics/826-x/spec.md")), ReviewConfig())
    assert (s.rule, s.lenses, s.not_docs) == ("docs-only", FLOOR, ())

def test_select_rename_from_code_is_full() -> None:
    s = select(changed_paths(raw("R100", "src/x.py", "docs/site/x.md")), ReviewConfig())
    assert (s.rule, s.lenses, s.not_docs) == ("full", LENSES, ("src/x.py",))

@pytest.mark.parametrize("path", ["skills/a/SKILL.md", "docs/site/CLAUDE.md", "vergil.toml",
                                  ".github/PULL_REQUEST_TEMPLATE.md", "paad/config/x.md",
                                  "docs/foo/README.md", "Readme.md", "docs/paad-policy.md",
                                  "docs/site/mkdocs.yml", "docs/site/claude.md",
                                  "epics/x/Agents.md", "docs/specs/skill.md"])
def test_not_docs(path: str) -> None:
    assert not is_docs(path, ReviewConfig())

def test_repo_addition_and_exclusion() -> None:
    review = ReviewConfig(extra_docs_paths=("notes/**/*.md",),
                          extra_non_docs_paths=("docs/specs/worktree-convention.md",))
    assert is_docs("notes/a/b.md", review)
    assert not is_docs("docs/specs/worktree-convention.md", review)
    assert not is_docs("notes/skills/SKILL.md", review)
```

- [ ] **Step 2: Write the failing CLI tests.** Use a temporary git repo fixture, following the existing `test_vrg_*` tests that build one. Assert:
  - `test_cli_docs_only_output`: the stdout lines are exactly `Lenses: security, spec-compliance` and `Rule: docs-only`, with exit 0.
  - `test_cli_full_output_truncates`: with five non-docs paths, the third line is `Not docs: <p1>, <p2>, <p3> (+2 more)`, with paths in diff order.
  - `test_cli_json`: the output is `{"lenses": [...], "rule": "full", "not_docs": [all five]}`.
  - `test_cli_bad_base`: `--base origin/nope` exits non-zero, writes the reason on stderr, and prints no `Lenses:` on stdout.
  - `test_cli_empty_diff`: exits non-zero with `empty diff` on stderr.
  - `test_cli_invalid_stanza`: `[review] extra-docs-paths = ["docs/**"]` exits non-zero with the config error on stderr.
  - `test_cli_control_character_path`: a changed path containing a newline or other control character makes the tool exit non-zero, with `ERROR:` on stderr and nothing on stdout.
  - `test_cli_json_single_values`: the `--json` object has exactly one `lenses` and one `rule` key, and `lenses` is a list drawn from `LENSES`.
- [ ] **Step 3: Run them.** `vrg-container-run -- vrg-validate`. Expected: the new tests fail.
- [ ] **Step 4: Implement `review_lenses.py`.**
  - `is_docs` is false if any built-in or repo exclusion matches with `ignore_case=True`. Otherwise it is true if any base or repo docs pattern matches case-sensitively.
  - `select` walks `rules` in order and returns the first whose `matches` is true; it holds no rule names. `docs_paths_only` is true when every path is a `regular_file` and `is_docs`. `not_docs` lists the non-docs paths in input order.
  - `changed_paths` parses `--raw -z` records: `:<old mode> <new mode> <old sha> <new sha> <status>`, then one path, or two for `R` and `C`. `regular_file` is true only when every mode the record carries is `100644` or `100755`, ignoring `000000` (the missing side of an add or delete). A symlink, a submodule, a type change or any other mode makes it false. Both paths of a rename or copy carry that flag.
- [ ] **Step 5: Implement the CLI.**
  - Call `check_rules(RULES)` at start.
  - Check the base with `git.ref_exists(base)`.
  - Read `git.read_output("diff", "--raw", "-z", "--no-abbrev", f"{base}...HEAD")` and `read_config(git.repo_root()).review`.
  - Refuse any changed path that contains a control character (`\x00`–`\x1f` or `\x7f`): exit 1 with `ERROR:` naming the count of such paths. No legitimate fleet file has one.
  - `--json` is the form the gate reads; the text form is for people.
  - Print as in Step 2, with the first three `Not docs:` paths in text mode.
  - On any error (`ConfigError`, `FileNotFoundError`, an unresolved base, an empty diff, a git failure), print `ERROR: <reason>` on stderr and return 1. Never print a lens set.
- [ ] **Step 6: Run the tests.** `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 6a: Refactor.** Tidy the library and CLI without changing behaviour; keep `select` free of rule names. Re-run `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 7: Add vergil-tooling's own stanza and the docs.** Add `[review] extra-non-docs-paths` with Task S's vergil-tooling entries. Add a `vrg-review-lenses` entry to `cli-tools-overview.md`, and a "The lens-selection tool" section to `review-config.md` with the base list, the exclusions, the rule table and the output format. The section also states the accepted gap: a rule document created later in an allowed path is docs-only until a repo exclusion names it, which lasts until vergil-project/.github#840.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope review-lenses --message "add vrg-review-lenses, which picks agentic-review lenses from the changed files"`.

### Task 5: The `Lenses:` input in the paad fork

Spec §4.2. The agent edits files in the dev clone on branch `lenses-input`, which the human creates from the rebased `unattended-mode`. The human commits, pushes and merges into `fleet`.

**Files:**

- Modify: `preview/paad/skills/agentic-review/SKILL.md` (`## Unattended mode`, the `## Common Mistakes` row, any "all six every run" text)
- Modify: `preview/paad/skills/agentic-review/references/report-template.md` (`Agents dispatched:`)
- Modify: `preview/paad/skills/paad-help/SKILL.md` (agentic-review's entry, which today describes only `Intent:`)
- Modify: `scripts/check_unattended.py` (agentic-review's `SKILL_RULES` entry)

- [ ] **Step 0: Pin the rules in the fork's lint first.** Add the `Lenses:` rules to agentic-review's entry in `scripts/check_unattended.py`'s `SKILL_RULES`, as verbatim lines:
  - the blocking conditions (an unknown lens, no lens, a repeated line);
  - each `Agents dispatched:` entry begins with the lens identifier.

  Run `make test`. Expected: FAIL, naming the missing lines. A later upstream sync that drops one then fails `make test`.
- [ ] **Step 1: Add the `Lenses:` input** to `## Unattended mode`, beside the `Intent:` paragraph. Spell out:
  - the line's syntax;
  - without the line, all six run;
  - with it, exactly the named lenses run in Phase 2;
  - the Verifier always runs, and the large-diff rule doubles each named lens;
  - an unknown lens, an empty value or a repeated line gives `Status: blocked: <reason>` naming the problem;
  - a lens named twice is dispatched once;
  - Phase 1 is unchanged.
- [ ] **Step 2: Define the result fields.**
  - In every unattended result, each `Agents dispatched:` entry begins with the lens's §4.1 identifier, one entry per dispatched lens. The rest of the entry (a `BAIL:` reason, the large-diff partitioning) stays as paad writes it today.
  - A new skill-specific field, `Lenses requested:`, comes right after `Agents dispatched:`. It gives the line's value, or `all (no Lenses: input)`.
  - A lens not requested appears in neither field nor as `NOT DISPATCHED`.
  - In the unattended interaction table, the large-diff row's "two instances of each specialist, 12 dispatches" becomes "two instances of each dispatched lens".
- [ ] **Step 3: Reconcile the interactive text.** In the Common Mistakes row, the report template and any body text saying all six are dispatched every run, add that under the delegate a `Lenses:` input takes precedence. Leave interactive behaviour unchanged.
- [ ] **Step 3a: Update paad-help.** Its agentic-review entry describes the `Lenses:` input and the `Lenses requested:` field, beside `Intent:`.
- [ ] **Step 4: Run the fork's checks.** `make test`. Expected: passes, including Step 0's lines.
- [ ] **Step 4a: Refactor.** Tidy the new unattended-section text and the `SKILL_RULES` lines without changing behaviour. Re-run `make test`. Expected: passes.
- [ ] **Step 5 (human):** commit, push, merge `lenses-input` into `fleet`, move `paad-fleet`, and record the commit. Hold reviews until V2 passes. Comment and close.

### V2: Gate run on `fleet` with `Lenses:`

Validation. Precondition: Task 5's fleet commit is recorded. Run the full automated set in `docs/paad-validation-gate.md`, including its `Lenses:` scenarios. Each `Lenses:` scenario's prompt is mode 2's delegate prompt (doctrine §3.2) with a `Lenses: <value>` line after `Intent:`. Also check that every plain mode 2 run names lenses by identifier and gives `Lenses requested: all (no Lenses: input)`, and that today's §3.4 check, which accepts both forms after Task 1, passes on them. Record and close as in V1.

### D2: Release and install vergil-tooling with the tool

Deployment. Precondition, attested by the human: a vergil-tooling release carrying Tasks 3 and 4 is published. Install it on the host and the agent VMs. On a VM, in a scratch worktree of the `.github` repo, commit a one-line `README.md` edit on a throwaway branch and run `vrg-review-lenses --base origin/develop`. Expected stdout, exactly: `Lenses: security, spec-compliance` then `Rule: docs-only`, with exit 0. Record the output, then delete the branch and the worktree.

### Task 6: Mode 2 sends and checks the lens set

Spec §4.5.

**Files:**

- Modify: `docs/paad-review-doctrine.md`: §1.9, §3.1, §3.2, §3.4, §3.5, §3.8
- Modify: `skills/issue-implement/SKILL.md` (`## The agentic-review gate`)
- Modify: `docs/paad-policy.md` (a register entry for `Lenses:`)
- Modify: `vergil.toml` (vergil-claude-plugin's own `[review].extra-non-docs-paths`, from Task S)

- [ ] **Step 1: §3.4 pre-dispatch.** After `vrg-git fetch origin`, run `vrg-review-lenses --base origin/<base> --json` in the worktree. Parse the JSON object: it must have exactly one `lenses` list, every entry a §4.1 identifier, and exactly one `rule` string. Anything else pauses the run. The prompt's `Lenses:` value is the list joined with `, `. A non-zero exit pauses the run with the tool's stderr. Add that cause to §3.8's list of pause causes before dispatch.
- [ ] **Step 2: §3.2 and §1.9.**
  - Add `Lenses: {{LENSES}}` after the template's `Intent:` line. The placeholder row reads "the tool's `Lenses:` value, verbatim".
  - Update §3.2's provenance note: the prompt was validated in #733, and its `Lenses:` line in V2.
  - In §1.9, `Lenses:` becomes an input mode 2 always sends, and `Lenses requested:` a required agentic-review field, replacing Task 1's "not yet" wording.
- [ ] **Step 3: §3.4 output checks.**
  - `Lenses requested:` equals the tool's value exactly.
  - Take the leading identifier of each `Agents dispatched:` entry (Task 5); that set equals the tool's set exactly. An entry that does not begin with a §4.1 identifier fails the check.
  - Any difference, including extra lenses, fails the check and pauses the run.
  - Task 1's display-name acceptance is removed.
  - The Spec Compliance and intent-source checks stay.
- [ ] **Step 4: §3.5 and §3.1.** The findings comment carries `Lenses: <identifiers> (rule: <rule>)`. When the rule is not `docs-only`, it also carries the first three `not_docs` paths and the count of the rest, inside a fenced code block, under the same rule §3.5 applies to text copied from the result. §3.1's "including on docs-only and trivial changes" becomes: the gate always runs, with the tool's lens set, never below the floor.
- [ ] **Step 5: `issue-implement`.** Add the tool run and the `Lenses:` line to the gate procedure's steps, citing §3.4 and §3.2. Repeat no rules.
- [ ] **Step 6: The register entry and the stanza.** Add a paad-policy register entry: vergil uses the fork's `Lenses:` input, not yet upstream; the floor lives in `vrg-review-lenses` and §3.4. Add Task S's plugin entries to `vergil.toml`, including `docs/site/docs/skills/index.md`.
- [ ] **Step 7: Validate.** `vrg-container-run -- vrg-validate`. Expected: passes.
- [ ] **Step 8: Commit.** `vrg-commit --type feat --scope paad-doctrine --message "run the agentic-review lenses vrg-review-lenses picks, with the floor checked exactly"`.

### D3: Release and install Task 6's plugin change

Deployment. Precondition, attested by the human: a vergil-claude-plugin release carrying Task 6 is published, and the floating tag points at it. Install it on the host and the agent VMs, and record the version.

### V3: Lens selection end to end

Validation. When it runs, V3 files two small, genuine tasks under the epic (`vrg-issue-create --epic vergil-project/.github#826`), and runs `issue-implement` on each:

- a docs-only one, such as a wording fix in a `.github` README, that changes only base-list Markdown;
- a code one, such as a small real fix in vergil-tooling.

Check for each round:

- the tool's output;
- the delegate prompt's `Lenses:` line;
- `Lenses requested:` and `Agents dispatched:`, which must be exactly the floor for the first branch and all six for the second;
- the findings comment's `Lenses:` line, and its fenced non-docs paths when the rule is `full`.

Also check the recompute, in two parts. A clean round finishes the run and freezes the branch, so the check does not rely on a second live round.

- **The tool:** on a throwaway branch with a docs-only commit, `vrg-review-lenses --base origin/develop --json` gives `"rule": "docs-only"`. After a second commit that touches a code file, it gives `"rule": "full"`, all six lenses, and the code file in `not_docs`. Delete the branch afterwards.
- **The gate:** for every V3 task whose run took more than one round, each round's findings comment carries that round's own `Lenses:` line, so the tool ran before each dispatch (§3.4).

Close only on SUCCESS. On FAILURE, record the evidence, file fix tasks under the epic, and run V3 again, with new tasks, once they ship.

### H: Offer the `Lenses:` input upstream: decide

Human task, after V3. Decide whether and how to offer the `Lenses:` input and the identifier change to paad's upstream author, and record the decision. This epic changes no paad default.
