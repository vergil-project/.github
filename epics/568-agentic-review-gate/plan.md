# Agentic review in `issue-implement` — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `issue-implement` runs a `paad:agentic-review` loop after
`vrg-validate` and before `report-ready`, records every round on the task
issue, files everything it does not fix, and pauses for the human when the
rules say so; `epic-retrospective` sweeps for untracked problems.

**Architecture:** almost all of the change is skill and doctrine text in
vergil-claude-plugin; the loop is carried out by the implementing agent
following that text. vergil-tooling gains one managed `.gitignore` line and
the `vrg-git` read-only additions. A re-spike validates the exact reviewer
prompt first; a deployment and a dogfood run close the epic out.

**Tech Stack:** Markdown skills and docs (vergil-claude-plugin); Python 3.12+
and pytest (vergil-tooling); `paad:agentic-review` 2.0.2; GitHub issues.

**Spec:** `epics/568-agentic-review-gate/spec.md` (vergil-project/.github#568;
planning issue vergil-project/vergil-claude-plugin#677)

**Shape:** an epic-level plan. Each numbered task is **one issue and one PR in
the repo named in its heading**, with paths relative to that repo's root.
`validation` and `deployment` tasks are operational: no code PR, closed on
`Outcome: SUCCESS`.

## Global Constraints

- **Start simple** (spec §1, §5). Build only what the spec's section 4 says.
  Anything from spec section 5 is out of scope; if a task seems to need one,
  stop and ask the human.
- **Binding rules** are umbrella spec §2, §3.1 and §3.3; they are not reopened.
- **Round cap 3**; blocking = in-scope Critical and Important; a Suggestion
  never causes a round on its own (spec §2).
- **Severity mapping** (doctrine §1): Critical → `critical`, Important →
  `high`, Suggestion → `medium`.
- **Trusted logins are configuration, never hard-coded** (spec 4.4). No
  skill, doctrine text, tool or test fixture outside Task 4's own tests names
  a real login. Skills and the doctrine name only the accessor command,
  `vrg-trusted-authors`.
- **Public-repo rule:** no security detail in any comment, commit message,
  intake issue, task, PR text or `report-ready` notes; counts only.
- **External prerequisites:**
  - child C's `docs/paad-review-doctrine.md` (epic vergil-project/.github#553)
    exists before Task 5 starts;
  - child A′ (vergil-project/.github#573) has shipped before Task 9.
- **Tooling:** validation `vrg-container-run -- vrg-validate`; commits
  `vrg-commit`; handoff `vrg-pr-workflow report-ready`. Releases are
  human-gated; the agent never cuts one.
- **No nits, no cracks:** any problem noticed and not fixed is filed with
  `vrg-intake-create`.

**Order:** Tasks 2, 3 and 4 have no dependencies within the epic and run in
parallel. Task 1 needs Tasks 2 and 3, released and installed. Task 5 needs
Tasks 1 and 4 and child C's doctrine document. Tasks 6 and 7 need Task 5.
Task 8 needs Tasks 2, 3 and 4. Task 9 needs Tasks 6, 7 and 8 and child A′.
Task 10 needs Task 9. Then the bookends: the documentation review
(vergil-project/vergil-tooling#3721), then the retrospective
(vergil-project/.github#570).

## Review Focus

1. **paad writes outside the task's worktree** (C's spec says paad resolves
   `paad/` from the session root). Expected: every file paad writes is under
   the worktree being reviewed. Measured in Task 1 on a three-level chain;
   checked every round by the output check written in Task 6.
2. **A plan-driven task with a one-line issue body.** Expected: Spec
   Compliance uses `requirements.md`, which carries the plan task's text, and
   does not bail or flag the real work as out-of-scope additions. Task 1 runs
   exactly this case.
3. **A repo with no local `main`** (vergil-tooling, vergil-vm). Expected: paad
   resolves the default branch without stopping. Pinned by Task 3's
   `symbolic-ref` tests and by Task 1's run in vergil-tooling.
4. **A security finding in a round.** Expected: nothing about it reaches a
   public record; the issue shows a count. Task 6's text and Task 10's
   check.
5. **Re-running `epic-implement` on a paused task with no ruling.** Expected:
   the run reports the pause and stops without a review round. Task 6's text
   and Task 10's check.

---

### Task 1: Re-spike the exact reviewer prompt (.github, validation)

Blocked by Tasks 2 and 3. Validates spec 4.2–4.5 before any skill text ships
(spec 4.11), under the conditions the gate will really run in.
**Precondition (human-attested):** a vergil-tooling release containing Tasks
2 and 3 is installed in the agent VM (`vrg-vm update`).

The targets are historical commits, whose own `.gitignore` predates Task 2's
managed line. For each target repo, add `paad/code-reviews/` to that repo's
local `.git/info/exclude` (untracked, local only) before the first run. This
has the same effect as the managed line will have.

**Targets:** two merged, plan-driven tasks, each in a detached throwaway
worktree at the merge commit, base = the commit's first parent (full SHA):

- vergil-claude-plugin `624f6ad` (#740, closes plugin#732 = epic #517 plan
  Task 7);
- vergil-tooling `22c3c03dda` (#3697, closes tooling#3681 = epic #517 plan
  Task 6). vergil-tooling has no local `main`, so this run also exercises
  Review Focus 3.

**Procedure** (for each target, twice):

- [ ] Write `requirements.md` in a `mktemp -d` directory, per spec 4.4: the
  issue's title and body, epic #517 plan Task 6 or 7 verbatim, and the
  spec #517 sections that task cites.
- [ ] Draft the reviewer prompt from spec 4.3 (this task's outcome comment
  records the final text, which Task 5 copies into the doctrine).
- [ ] From a depth-1 `general-purpose` subagent, dispatch a fresh depth-2
  reviewer with that prompt, mirroring the planning-time spike.
- [ ] Apply the spec 4.5 output checks, and record where every file paad wrote
  landed (worktree, main checkout, or elsewhere).
- [ ] Collect: read the report and file any real findings about the reviewed
  code in intake; then empty `paad/code-reviews/` and confirm `vrg-git status
  --porcelain` is empty.
- [ ] **Security findings:** A′ has not shipped, so leave any file under
  `paad/security/` in place and tell the human how many there are (counts
  only in the outcome comment). The human decides what to record.
- [ ] Remove the worktree only after the human has seen any security
  findings.

**Acceptance:** all four runs pass every output check; Spec Compliance's
intent sources name `requirements.md`; every paad file lands in that run's
worktree. The outcome comment carries the final prompt text and, per run,
the time and subagent count. **On failure**, record it and stop: the result
goes to the human before Task 5 starts.

---

### Task 2: Managed `.gitignore` ignores `paad/code-reviews/` (vergil-tooling, implementation)

No blockers.

**Files:**

- Modify: `src/vergil_tooling/data/gitignore/base`: add `paad/code-reviews/`
  after `paad/security/`.
- Test: `tests/vergil_tooling/test_gitignore.py`: add `"paad/code-reviews/"`
  after `"paad/security/"` in `_LEGACY_GITIGNORE_PATTERNS`, and extend its
  comment to name this epic.

- [ ] **Step 1: Write the failing tests:**
  - `test_base_ignores_paad_code_reviews`:
    `"paad/code-reviews/" in gitignore.load_base()`;
  - `test_compose_every_language_ignores_paad_code_reviews`: for every `lang`
    in `(*gitignore.FRAGMENT_LANGS, None)`,
    `"paad/code-reviews/" in gitignore.compose(lang)`.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: both new
  tests FAIL.
- [ ] **Step 3: Make the data-file edit,** then resync this repo's own fence
  with `vrg-gitignore-sync`.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
- [ ] **Step 5: Commit** (`vrg-commit --type feat --scope gitignore`) and
  run report-ready.

---

### Task 3: `vrg-git` read-only additions (vergil-tooling, implementation)

Existing issue vergil-project/vergil-tooling#3722; its body is the scope. No
blockers.

**Interfaces:**

- Produces: `vrg-git symbolic-ref refs/remotes/origin/HEAD` (read form) and
  `vrg-git -C <path> <subcommand> …`, with every existing policy check
  applied to the repo `-C` names.

- [ ] **Step 1: Write the failing tests** in the `vrg-git` test module:
  - `test_symbolic_ref_read_is_allowed`;
  - `test_symbolic_ref_write_and_delete_are_refused` (`symbolic-ref <name>
    <ref>`, `-d`, `--delete`);
  - `test_dash_c_runs_in_named_repo`;
  - `test_dash_c_applies_policy_to_named_repo`, parametrized over every
    policy check `vrg-git --help` lists (the main-worktree branch switch,
    `worktree add` outside `.worktrees/`, the issue-branch/directory
    lockstep, protected-branch force, the freeze, and the flag deny-list):
    each gives the same answer under `-C` as under `cd`.
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: FAIL.
- [ ] **Step 3: Implement:** parse a leading `-C <path>` before subcommand
  dispatch and resolve the policy context from that path; allow
  `symbolic-ref` only with one argument and no `-d`/`--delete`/`-m`. Update
  the `--help` text.
- [ ] **Step 4:** check every other read-only command that agentic-review's
  `SKILL.md` and `references/*.md` (paad 2.0.2) name against the allowlist,
  and add any missing read-only one with a test.
- [ ] **Step 5: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
  Commit (`vrg-commit --type feat --scope vrg-git`) and run report-ready.

---

### Task 4: Trusted-authors configuration (vergil-tooling, implementation)

No blockers.

**Files:**

- Create: `src/vergil_tooling/lib/trusted_authors.py`.
- Create: `src/vergil_tooling/bin/vrg_trusted_authors.py`, and its console
  entry point `vrg-trusted-authors` in `pyproject.toml`.
- Modify: `src/vergil_tooling/lib/identity.py`: parse an optional top-level
  `[trusted_authors]` stanza in `identities.toml`.
- Modify: `src/vergil_tooling/lib/vm_guest.py`: an
  `_inject_trusted_authors(transport, authors: TrustedAuthors | None)` step
  that writes `~/.config/vergil/trusted-authors.toml` (mode 600) into the VM,
  and **deletes** that file when `authors` is `None` (the host has no stanza),
  so the accessor fails loudly rather than trusting a stale list.
- Modify: `src/vergil_tooling/bin/vrg_vm.py`: call it on **every** path that
  writes vergil config into a VM: create, start, restart, rebuild, the cloud
  credentials stage, and `update` (`_update_over_transport`). The stanza is
  on `IdentityConfig`, not `Identity`, so pass it to those call sites
  explicitly.
- Test: `tests/vergil_tooling/test_trusted_authors.py`, and additions to
  `tests/vergil_tooling/test_identity.py` and `test_vm_guest.py`.

**Interfaces:**

- Produces:
  - `@dataclass(frozen=True) class TrustedAuthors: human: tuple[str, ...];
    agents: tuple[str, ...]`;
  - `parse_trusted_authors(raw: Mapping[str, object]) -> TrustedAuthors`:
    raises `ValueError` naming the problem when `human` is missing or empty,
    when any entry is not a non-empty string, or when a login is in both
    lists;
  - `load_trusted_authors() -> TrustedAuthors`: reads the VM file
    `~/.config/vergil/trusted-authors.toml` if it exists, else the
    `[trusted_authors]` stanza of `~/.config/vergil/identities.toml`; raises
    `TrustedAuthorsNotConfiguredError` naming both paths when neither has
    one;
  - `vrg-trusted-authors`: prints `human: <login>` and `agent: <login>`, one
    per line; `--json` prints `{"human": [...], "agents": [...]}`; exits 1
    with the error message when not configured. Never prints a default.

- [ ] **Step 1: Write the failing tests** (logins in tests are invented, such
  as `example-human` and `example-agent`):
  - `test_parse_trusted_authors_reads_both_lists`;
  - `test_parse_trusted_authors_requires_a_human`;
  - `test_parse_trusted_authors_rejects_login_in_both_lists`;
  - `test_load_prefers_vm_file_over_identities`;
  - `test_load_raises_when_not_configured`;
  - `test_cli_prints_logins` and `test_cli_exits_1_when_not_configured`;
  - `test_identity_config_parses_trusted_authors_stanza`;
  - `test_inject_trusted_authors_writes_vm_file`;
  - `test_inject_trusted_authors_removes_vm_file_when_host_has_none`;
  - `test_update_delivers_trusted_authors` and
    `test_start_delivers_trusted_authors` (in the `vrg-vm` tests).
- [ ] **Step 2: Run `vrg-container-run -- vrg-validate`.** Expected: FAIL.
- [ ] **Step 3: Implement** the interfaces above.
- [ ] **Step 4: Run `vrg-container-run -- vrg-validate`.** Expected: PASS.
- [ ] **Step 5:** document the stanza where `identities.toml` is documented,
  and say that it seeds vergil-project/.github#579's system-wide rule.
  Commit (`vrg-commit --type feat --scope trusted-authors`) and run
  report-ready.

---

### Task 5: Doctrine §3 and the paad policy (vergil-claude-plugin, implementation)

Blocked by Task 1. Also needs child C's `docs/paad-review-doctrine.md` on
`develop` (a text dependency: C's tasks are not filed yet; link the C task
with `Blocked-by:` once it exists).

**Files:**

- Modify: `docs/paad-review-doctrine.md` §3 (replace C's stub).
- Modify: `docs/paad-policy.md` §1, §2, §4 and the §7 register.

- [ ] **Doctrine §3 "Mode 2: agentic review in `issue-implement`"** with
  these subsections, each written from the named spec section, as
  instructions to the implementing agent:
  - §3.1 The loop: spec 4.2, including classification and the repair pause;
  - §3.2 The reviewer prompt: spec 4.3, with the **template text from Task
    1's outcome comment**, verbatim;
  - §3.3 The requirements file: spec 4.4. It tells the agent to get the
    trusted logins by running `vrg-trusted-authors` (Task 4), and to stop
    and report if the command fails. It names no login;
  - §3.4 Output checks: spec 4.5;
  - §3.5 Task-issue records: spec 4.6, with one example of each comment;
  - §3.6 Security findings: spec 4.7;
  - §3.7 Intake and the handoff: spec 4.8;
  - §3.8 Pauses: spec 4.9.
- [ ] **Doctrine §1:** confirm the agentic-review column of the severity
  table reads Critical → `critical`, Important → `high`, Suggestion →
  `medium`.
- [ ] **paad-policy:** replace the agentic-review report-files row in §7 with
  spec 4.12's four rows and its note on the refined rule; rewrite §2's
  agentic-review bullet to match. Then make §1, §4 and the §7 row on paad
  being unpinned ("…D and C may rely on the experimental `paad/config/` …
  D detects drift") say, in content: D adopts no config file and does no
  drift detection; if C builds its config fallback, C owns that file's
  version check (spec section 5).
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit
  (`vrg-commit --type docs --scope paad`) and run report-ready.

---

### Task 6: The gate in `issue-implement`, and `epic-implement` (vergil-claude-plugin, implementation)

Blocked by Task 5.

**Files:**

- Modify: `skills/issue-implement/SKILL.md`.
- Modify: `skills/epic-implement/SKILL.md`, step 4.
- Modify: `docs/site/docs/skills/index.md`, the `issue-implement` entry.

- [ ] **`issue-implement`:** add the section "The agentic-review gate"
  between "Implement" and "Record the PR metadata and hand off". It states the
  steps of spec 4.2 in order and links doctrine §3 for the detail. It must
  say:
  - the gate always runs, after `vrg-validate` is green and all work is
    committed;
  - a run that finds the task paused with no ruling from the human in the
    conversation reports the pause and stops, with no review;
  - the `report-ready` notes carry the handoff of doctrine §3.7, with
    security findings as counts only.

  Change "Record the PR metadata and hand off" to start "When the
  agentic-review gate has passed".
- [ ] **`epic-implement` step 4:** tasks paused at the agentic-review gate are
  listed among the items waiting on the human, each with its reason; when the
  human rules, pass the ruling to a new `issue-implement` run for that task.
- [ ] **Skills index:** the `issue-implement` entry mentions the gate.
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit
  (`vrg-commit --type feat --scope issue-implement`) and run report-ready.

---

### Task 7: The unaddressed-findings sweep in `epic-retrospective` (vergil-claude-plugin, implementation)

Blocked by Task 5.

**Files:**

- Modify: `skills/epic-retrospective/SKILL.md`: a step after the preflight
  and before "Authoring model".
- Modify: `docs/site/docs/skills/index.md`, the `epic-retrospective` entry.

- [ ] Add the sweep from spec 4.10: for each task of the epic, read its
  `agentic-review summary` comment and confirm every item not fixed has an
  issue; read the task's other comments, its PR review comments and the
  planning issue's design-review comments for problems never filed; for each,
  find the issue, file one with `vrg-intake-create` (security-class only via
  `vrg-security-record`), or have the human drop it with a reason. Record the
  result in retrospective §4. Tasks from before the gate shipped have no
  summary; say so.
- [ ] Run `vrg-container-run -- vrg-validate`; expect PASS. Commit
  (`vrg-commit --type feat --scope epic-retrospective`) and run report-ready.

---

### Task 8: Deploy the tooling and sync the fleet (vergil-tooling, deployment)

Blocked by Tasks 2, 3 and 4. **Preconditions (human-attested):**

- a vergil-tooling release containing Tasks 2, 3 and 4 is cut;
- the host's `~/.config/vergil/identities.toml` has its `[trusted_authors]`
  stanza (the human fills it in; the agent never writes it).

**Procedure:**

- [ ] In the agent VM, `vrg-vm update`; confirm `vrg-git symbolic-ref
  refs/remotes/origin/HEAD` works, `vrg-trusted-authors` prints the
  configured human and agent logins, and `paad/code-reviews/` is in the
  managed base (`vrg-gitignore-sync --check` in a clone).
- [ ] Fleet sync, as epic #500 plan Task 2 did for `paad/security/`:
  `vrg-fleet-sync --target gitignore --dry-run --repos <clones>`, then without
  `--dry-run`. List `<clones>` at run time with
  `grep -l '>>> vergil-managed' ~/dev/projects/*/{*,.github}/.gitignore`.
  The human submits and merges each sync PR.
- [ ] Record `Outcome: SUCCESS` with the released version and the list of
  synced repos, once every sync PR has merged.

---

### Task 9: Deploy the plugin (vergil-claude-plugin, deployment)

Blocked by Tasks 6, 7 and 8. **Preconditions (human-attested):**

- child A′ (vergil-project/.github#573) has shipped;
- a vergil-claude-plugin release containing Tasks 5–7 is published.

**Procedure:**

- [ ] In the agent VM, `vrg-vm update`, and confirm the installed
  `issue-implement` skill contains "The agentic-review gate" and the doctrine
  contains its §3.
- [ ] Record `Outcome: SUCCESS` with the released version.

---

### Task 10: Dogfood the gate (.github, validation)

Blocked by Task 9.

- [ ] Run `issue-implement` on the next real code task of any epic (name it
  in the outcome) and let the gate run.
- [ ] Check: the task issue has a findings and a response comment per round
  and a summary, in doctrine §3.5's form; the output checks ran; intake
  issues were filed per round and named in the summary; the `report-ready`
  notes list every finding not fixed; no security detail is in any public
  record; the worktree is clean at finalize.
- [ ] If the run paused, check that re-dispatching it without a ruling
  reports the pause and stops (Review Focus 5); otherwise record that pauses
  were not exercised.
- [ ] Record in the outcome what the loop actually did: rounds, findings by
  severity and bucket, time, and anything that surprised. This is the first
  data for tuning (spec section 5).

---

## Self-Review

- **Spec coverage:**
  - 4.1 → Tasks 5–7;
  - 4.2–4.9 → Task 5 (doctrine) and Task 6 (skill);
  - 4.10 → Task 7;
  - 4.4's trusted-authors configuration → Task 4;
  - 4.11 → Tasks 1, 2, 3, 4, 8, 9 and 10, plus the external prerequisites;
  - 4.12 → Task 5;
  - section 7 deliverables → Tasks 2–9, with docs through the documentation
    review (vergil-project/vergil-tooling#3721);
  - the spike (section 3) needs no task.
- **Type consistency:** the comment headers (`agentic-review round <n>:
  findings`, `… response`, `agentic-review summary: …`) and the section names
  ("The agentic-review gate", doctrine §3.1–3.8) are used the same way in
  Tasks 5, 6, 7 and 10.
- **Proportion:** most tasks are text changes whose content the spec fixes;
  the plan names where each part goes rather than restating it.
