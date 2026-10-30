# Plugin distribution via the floating `v2.1` tag Implementation Plan

- **Epic:** vergil-project/.github#760
- **Planning issue:** vergil-project/.github#759 (review history is in its
  comments)
- **Status:** approved

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the `vergil-marketplace` ref from `main` to the floating `v2.1` tag, with the tag protected as the deploy ref, a hardened `vrg-promote` as the rollback tool, and a fully mechanized switch of the whole fleet.

**Architecture:** The vergil-tooling changes come first: the rolling-tag rulesets, the `vrg-promote` hardening and a shared marketplace-reconcile module used by the VM refresh and a new host command. A deployment applies the rulesets, a second deployment switches the fleet, and a validation runs a real rollback round trip. The docs change in the plugin repo and vergil-tooling.

**Tech Stack:** Python 3.14 (vergil-tooling, pytest), the GitHub rulesets API through `vrg-github-repo-config`, the Claude Code plugin CLI (`claude plugin marketplace …`), Markdown docs.

**Spec:** `epics/760-plugin-tag-distribution/spec.md`

## Global Constraints

- The marketplace source is `{"source": "github", "repo": "vergil-project/vergil-claude-plugin", "ref": "v2.1"}` (spec §2).
- Rolling tags `vX.Y` are protected by a tag ruleset that blocks update and deletion, bypassed only by the repository Admin role (`RepositoryRole`, `actor_id` 5) (spec §2).
- No branch may carry a rolling-tag-shaped name: the branch-name ruleset includes `refs/heads/v[0-9]*` where GitHub's ref matching supports character classes, otherwise `refs/heads/v*`. Task 1 picks the literal (`BRANCH_PATTERN`). The ruleset has no bypass actor (spec §2).
- `vrg-promote` refuses an agent identity with the `vrg-release` check, failing closed on any agent signal (spec §2).
- `vrg-promote` moves a rolling tag only to an existing `vX.Y.Z` tag whose commit is on `origin/main` and that has a published GitHub Release (spec §2).
- Every human-only command this epic touches (`vrg-promote`, `vrg-release`, `vrg-host plugins`) refuses with the fail-closed check, `identity_mode.is_agent() or identity_mode.any_agent_signal()` (spec §2, §6).
- Both rulesets are live on vergil-claude-plugin, and its audit reports them clean, before any machine switches (spec §2, "Order").
- No machine, host or VM, is switched by hand (spec §6).
- `claude plugin marketplace remove` is the last rung of the switch, used only if the rewrite rung fails (spec §6).
- Moving a rolling tag is a human act; agents never run `vrg-promote`, `vrg-release`, `vrg-github-repo-config apply` or `vrg-submit-pr`.
- vergil-tooling validation is `vrg-container-run -- vrg-validate` only.

## Review Focus

1. **`vrg-promote` with no terminal** (run from a script, CI or a pipe): it must refuse, not proceed. Pinned in Task 2.
2. **`vrg-promote` given the rolling tag's current target:** it must report "already promoted" and push nothing, as today. Pinned in Task 2.
3. **A registered marketplace with no `ref`** (registered from the default branch): the reconcile treats a missing ref as different from `v2.1` and switches it. Pinned in Task 3.
4. **A marketplace declared in settings but not registered at all** (a fresh box): the reconcile adds it and does not try to rewrite or remove anything. Pinned in Task 3.
5. **A malformed `claude plugin marketplace list --json`** (not a list, or an entry without `name`): the reconcile raises; it never skips silently. Pinned in Task 3.

---

## Task order and placement

| # | Task | Repo (where it closes) | Kind | Blocked by |
|---|---|---|---|---|
| 1 | Rolling-tag rulesets | vergil-tooling | task | — |
| 2 | Harden `vrg-promote` | vergil-tooling | task | — |
| V1 | Prove the switch mechanism | vergil-tooling | validation | — |
| 3 | Marketplace reconcile for VMs and host | vergil-tooling | task | V1 |
| D1 | Apply the rolling-tag rulesets fleet-wide | vergil-tooling | deployment | 1 |
| 4 | Plugin distribution docs | vergil-claude-plugin | task | 5, D1 |
| 5 | Consumer setup and rollback runbook | vergil-tooling | task | 2, 3, D1 |
| D2 | Switch the fleet to `v2.1` | vergil-tooling | deployment | D1, 2, 3 |
| V2 | Rollback round trip | vergil-tooling | validation | D2, 5 |

The `.github` change (the one-line note in epic #45's spec) ships in the documentation task's PR, vergil-project/.github#761, with this plan.

---

### Task 1: Rolling-tag rulesets

**Files:**

- Modify: `src/vergil_tooling/lib/github_config.py` (`desired_tag_protection_ruleset`, `compute_desired_state`)
- Test: `tests/vergil_tooling/test_github_config_lib.py`

**Interfaces:**

- Produces: `desired_tag_protection_ruleset() -> DesiredRuleset` with `ref_include == ["refs/tags/v*.*.*", "refs/tags/v*.*"]`; `desired_rolling_tag_branch_ruleset() -> DesiredRuleset`; `compute_desired_state` appends the new ruleset after the tag ruleset.

- [ ] **Step 1: Write the failing tests**

```python
def test_tag_protection_ruleset() -> None:  # existing test, updated
    r = desired_tag_protection_ruleset()
    assert r.ref_include == ["refs/tags/v*.*.*", "refs/tags/v*.*"]
    # existing assertions on bypass (RepositoryRole 5) and rules unchanged

def test_branch_from_ref_skips_glob_includes() -> None:
    assert _branch_from_ref("refs/heads/v*") is None
    assert _branch_from_ref("refs/heads/main") == "main"

def test_rolling_tag_branch_ruleset() -> None:
    r = desired_rolling_tag_branch_ruleset()
    assert r.name == "Rolling tag branch names"
    assert r.target == "branch"
    assert r.enforcement == "active"
    assert r.ref_include == [BRANCH_PATTERN]  # see the pattern note below
    assert r.bypass_actors == []
    assert [rule["type"] for rule in r.rules] == ["creation"]

def test_compute_desired_state_has_four_rulesets() -> None:  # replaces ..._has_three_rulesets
    state = compute_desired_state(_vergil_config(), visibility="public", is_org=True)
    assert len(state.rulesets) == 4
    assert [r.name for r in state.rulesets][:3] == [
        "Branch protection", "Tag protection", "Rolling tag branch names",
    ]
```

The existing `test_compute_desired_state_has_three_rulesets` (`test_github_config_lib.py`, about line 1098) is renamed and rewritten as above. No other existing test changes.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL in the four tests above: `test_branch_from_ref_skips_glob_includes` (today it returns `"v*"`), `test_rolling_tag_branch_ruleset` (undefined function), `test_tag_protection_ruleset` (no `refs/tags/v*.*`) and `test_compute_desired_state_has_four_rulesets` (three rulesets).

- [ ] **Step 3: Implement**

**Pattern note.** The branch pattern should block only rolling-tag-shaped names, so that branches such as `vm-…` stay possible. Use `refs/heads/v[0-9]*` if GitHub's ruleset ref matching supports character classes: check the rulesets documentation (fnmatch syntax) before writing the test, and cite it in the docstring. If it does not, use `refs/heads/v*` and say in the docstring why the pattern is broad. `BRANCH_PATTERN` in the test is the chosen literal.

Add `"refs/tags/v*.*"` to `desired_tag_protection_ruleset().ref_include`. Add `desired_rolling_tag_branch_ruleset()` next to it, shaped like `desired_branch_protection_ruleset()`, with the values in the test. Append it in `compute_desired_state` right after the tag ruleset; app mode nulls its bypass actors like the others.

The classic-branch-protection helpers derive branch names from a branch ruleset's includes (`_ruleset_branches` via `_branch_from_ref`), and would query `branches/v*/protection`. Make `_branch_from_ref` return `None` for an include containing a glob character (`*`, `?`, `[`), which Step 1's `test_branch_from_ref_skips_glob_includes` pins.

- [ ] **Step 4: Refactor**

With the tests green, check that the new ruleset reuses the shared `DesiredRuleset` construction rather than duplicating it, and tidy without changing behaviour.

- [ ] **Step 5: Run validation to verify it passes**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 6: Update the config-audit reference doc**

In `docs/site/docs/reference/config-audit.md`, list the new ruleset and the widened tag pattern next to the existing rulesets, and note that the App-token audit does not compare bypass actors.

- [ ] **Step 7: Commit**

`vrg-commit --type feat --scope repo-config --message "protect rolling vX.Y tags and forbid branches named like them"`

### Task 2: Harden `vrg-promote`

**Files:**

- Modify: `src/vergil_tooling/lib/promote.py`
- Modify: `src/vergil_tooling/bin/vrg_promote.py`
- Modify: `src/vergil_tooling/bin/vrg_release.py` (the first-stage identity guard)
- Test: `tests/vergil_tooling/test_promote.py`, `tests/vergil_tooling/test_vrg_promote.py`, `tests/vergil_tooling/test_vrg_release.py` (new `test_release_refuses_mixed_signals`: `VRG_IDENTITY_MODE=human` with `any_agent_signal()` patched to `True` exits 1 before any git call)

**Interfaces:**

- Produces, in `lib/promote.py`:
  - `class PromoteError(ValueError)`
  - `@dataclass(frozen=True) class Promotion: origin_repo: str; rolling_tag: str; release_tag: str; old_ref: str; old_commit: str; new_commit: str`. `old_ref` is the rolling tag's **raw** value on origin, the `ls-remote` line for `refs/tags/<rolling>` without `^{}`: for an annotated tag that is the tag object's id, not the commit. `old_commit` is its **peeled** commit (the `^{}` line, or `old_ref` itself for a lightweight tag). Both are `""` when the rolling tag does not exist yet. `old_ref` is used only for the lease; the no-op check, the confirmation comparison and `describe` use the peeled commits.
  - `plan_promotion(version: str) -> Promotion`: first refuses with `PromoteError` when `identity_mode.is_agent() or identity_mode.any_agent_signal()`, before any git call, so every caller (the CLI and `vrg-release`'s promote stage) fails closed. It then validates the version, resolves both tags on origin, and raises `PromoteError` when the release tag is absent on origin, when its commit is not an ancestor of `origin/main`, or when no published GitHub Release of that tag exists. The Release is read with `github.read_output` under `retry.retry_known_resource`, the convention `release/confirm.py` uses for the same read (#3137), so a transient 404 does not fail a release's promote stage. A missing Release or a draft refuses. The `origin/main` ancestry check is the primary guard, because it limits the target to merged work; the Release check is an additional layer.
  - `describe(p: Promotion) -> str`: the lines the CLI prints for the dry run and before confirmation: `origin: <repo>`, `<rolling> <old[:12] or (new)> -> <release> <new[:12]>`.
  - `promote_confirmed(confirmed: Promotion) -> None`: re-resolves with `plan_promotion(confirmed.release_tag)`, raises `PromoteError("… changed since it was confirmed …")` if the fresh `Promotion` differs from `confirmed`, and otherwise tags and pushes with `git push origin <rolling> --force-with-lease=refs/tags/<rolling>:<old_ref>`. When the rolling tag does not exist yet, `old_ref` is `""` and the lease is `--force-with-lease=refs/tags/<rolling>:` ("must not exist"). The lease is a compare-and-swap that origin applies atomically, so the tag that is overwritten is exactly the one confirmed, or the push is refused. There is no plain `--force` path.
  - `promote(version: str, *, dry_run: bool = False) -> None`: unchanged signature (the `vrg-release` orchestrator calls it). It calls `plan_promotion`, so the target check applies to releases too, then hands the result to `promote_confirmed`, and it never prompts.
- The identity refusal is enforced in both places, as defence in depth: `plan_promotion` in the library (so every caller fails closed) and `bin/vrg_promote.py`'s `main()` (so the CLI refuses before any output). The CLI alone owns the dry-run print and the confirmation prompt.
- `bin/vrg_release.py`'s first-stage guard uses `identity_mode.is_agent() or identity_mode.any_agent_signal()`, before any side effect.

- [ ] **Step 1: Write the failing library tests** (`test_promote.py`, patching `subprocess.run` as the existing tests do)

```python
def test_plan_promotion_refuses_tag_not_on_main() -> None:
    # ls-remote gives a v2.1.50 commit; `merge-base --is-ancestor` exits 1
    with pytest.raises(PromoteError, match="not on origin/main"):
        plan_promotion("2.1.50")

def test_plan_promotion_refuses_tag_without_published_release() -> None:
    # tag exists and is on origin/main, but `gh release view` exits 1
    with pytest.raises(PromoteError, match="no published GitHub Release"):
        plan_promotion("2.1.50")

def test_plan_promotion_refuses_missing_release_tag() -> None:
    # ls-remote for refs/tags/v2.1.99 returns nothing
    with pytest.raises(PromoteError, match="v2.1.99 does not exist on origin"):
        plan_promotion("2.1.99")

def test_describe_existing_rolling_tag() -> None:
    p = Promotion("o/r", "v2.1", "v2.1.50", "t" * 40, "a" * 40, "b" * 40)
    assert describe(p) == "origin: o/r\nv2.1 aaaaaaaaaaaa -> v2.1.50 bbbbbbbbbbbb"

def test_describe_new_rolling_tag() -> None:
    p = Promotion("o/r", "v2.2", "v2.2.0", "", "", "b" * 40)
    assert describe(p) == "origin: o/r\nv2.2 (new) -> v2.2.0 bbbbbbbbbbbb"

def test_promote_confirmed_refuses_changed_resolution() -> None:
    # the confirmed Promotion has old "a"*40; a fresh plan_promotion resolves old "c"*40
    with pytest.raises(PromoteError, match="changed since it was confirmed"):
        promote_confirmed(confirmed)
    # no `git push` call

def test_promote_confirmed_leases_on_raw_annotated_tag_object() -> None:
    # ls-remote for refs/tags/v2.1 returns two lines, as for the live annotated tag:
    #   "t"*40 refs/tags/v2.1      (tag object)
    #   "a"*40 refs/tags/v2.1^{}   (peeled commit)
    p = plan_promotion("2.1.50")
    assert (p.old_ref, p.old_commit) == ("t" * 40, "a" * 40)
    promote_confirmed(p)
    # the push argv is ["git", "push", "origin", "v2.1",
    #                   "--force-with-lease=refs/tags/v2.1:" + "t"*40]

def test_promote_confirmed_lightweight_tag_leases_on_commit() -> None:
    # ls-remote returns one line "a"*40 refs/tags/v2.1 -> old_ref == old_commit == "a"*40

def test_promote_confirmed_new_rolling_tag_uses_empty_lease() -> None:
    # no refs/tags/v2.2 on origin -> the push argv ends with "--force-with-lease=refs/tags/v2.2:"

def test_plan_promotion_reports_repo_and_commits() -> None:
    p = plan_promotion("2.1.50")
    assert p == Promotion("vergil-project/vergil-claude-plugin", "v2.1", "v2.1.50", OLD_REF, OLD, NEW)

def test_promote_already_promoted_pushes_nothing(capsys) -> None:  # Review Focus 2
    with patch("vergil_tooling.lib.promote.subprocess.run",
               side_effect=_git_responses(rolling=NEW, release=NEW)) as mock_run:
        promote("2.1.50")
    argvs = [c.args[0] for c in mock_run.call_args_list]
    assert not any(a[:2] in (["git", "tag"], ["git", "push"]) for a in argvs)
    assert "already promoted" in capsys.readouterr().out

def test_plan_promotion_refuses_agent_signal() -> None:
    # patch vergil_tooling.lib.promote.identity_mode.any_agent_signal -> True
    with pytest.raises(PromoteError, match="human maintainer action"):
        plan_promotion("2.1.50")
    # no git call was made
```

The origin repo comes from `git remote get-url origin`, normalized to `owner/repo` (strip `https://github.com/`, `git@github.com:` and `.git`).

**Existing `test_promote.py` tests.** `_already_promoted` is removed, because `plan_promotion` replaces it. The existing tests change as follows. Each patches `subprocess.run` with a side-effect helper, `_git_responses(...)`, that answers `remote get-url`, `fetch`, `ls-remote` and `merge-base` by argv. Each asserts on the `tag` and `push` calls, never on a total call count.

| Test | Change |
|---|---|
| `test_promote_runs_tag_and_push` | Assert exactly one `git tag -f v2.1 v2.1.50` and one `git push origin v2.1 --force-with-lease=refs/tags/v2.1:<OLD_REF>`, after the planning calls |
| `test_promote_dry_run_does_not_execute` | Assert no `tag` or `push` call (planning calls are now expected) |
| `test_promote_strips_v_prefix` | Unchanged intent; uses `_git_responses` |
| `test_promote_raises_on_tag_failure`, `test_promote_prints_stderr_on_failure`, `test_promote_prints_stderr_on_push_failure`, `test_promote_push_failure_no_stderr` | Unchanged intent; the failure is injected on the `tag` or `push` call only |
| `test_promote_invalid_version_raises` | Expects `PromoteError` instead of `ValueError` |
| `test_promote_skips_when_already_promoted` | Drives the no-op through `plan_promotion` (equal commits) instead of patching `_already_promoted` |
| `test_already_promoted_true_when_commits_match`, `..._false_when_release_tag_missing`, `..._false_when_rolling_differs` | Deleted with `_already_promoted`; their cases move to `plan_promotion` tests (`test_plan_promotion_reports_repo_and_commits` with equal and differing commits, `test_plan_promotion_refuses_missing_release_tag`) |
| `test_peeled_commit_*` | Unchanged |

`PromoteError` subclasses `ValueError`, so callers that catch `ValueError` today keep working.

- [ ] **Step 2: Write the failing CLI tests** (`test_vrg_promote.py`)

```python
def test_refuses_agent_identity() -> None:
    # patch vergil_tooling.bin.vrg_promote.identity_mode.any_agent_signal -> True
    assert main() == 1
    # stderr names vrg-promote as a human action; no git call was made

def test_dry_run_prints_repo_and_commits(capsys) -> None:
    # argv: ["vrg-promote", "--dry-run", "2.1.50"]
    out = capsys.readouterr().out
    assert "origin: vergil-project/vergil-claude-plugin" in out
    assert "v2.1" in out and "v2.1.50" in out
    # no `git tag` or `git push` call

def test_refuses_without_tty() -> None:  # Review Focus 1
    # sys.stdin.isatty() -> False, real run
    assert main() == 1
    # no `git push` call

def test_declined_confirmation_pushes_nothing() -> None:
    # isatty -> True, input -> "n"
    assert main() == 1

def test_confirmed_run_promotes() -> None:
    # isatty -> True, input -> "y"; assert
    # `git push origin v2.1 --force-with-lease=refs/tags/v2.1:<OLD_REF>` was called

def test_already_promoted_exits_without_prompt(capsys) -> None:
    # plan_promotion returns old_commit == new_commit; input is patched to fail if called
    assert main() == 0
    assert "already promoted" in capsys.readouterr().out
```

Update the existing CLI tests (`test_promote_explicit_version`, `test_promote_from_version_file`) to answer the prompt with `"y"` and patch `isatty`.

- [ ] **Step 3: Run to verify the tests fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL in the new tests and in the rewritten existing tests above.

- [ ] **Step 4: Implement the library**

`plan_promotion` uses `_peeled_commit` for both tags and `git merge-base --is-ancestor <new> origin/main`, after `git fetch --tags --force origin main`, so that `origin/main` and the tags are current. A non-zero fetch raises `PromoteError`. `promote` becomes: `p = plan_promotion(version)`; if `p.old_commit == p.new_commit`, print the existing "already promoted" line and return; if `dry_run`, print `describe(p)` and return; otherwise call `promote_confirmed(p)`.

- [ ] **Step 5: Implement the CLI**

In `main()`, before anything else:

```python
if identity_mode.is_agent() or identity_mode.any_agent_signal():
    print("vrg-promote: moving a rolling tag is a human maintainer action. "
          "Agents cannot run vrg-promote.", file=sys.stderr)
    return 1
```

Then `p = plan_promotion(version)` (a `PromoteError` prints `Error: …` and returns 1). If `p.old_commit == p.new_commit`, print `<rolling> already points at <release> — already promoted.` and return 0, before the dry-run, TTY and prompt branches. Otherwise print `describe(p)`. With `--dry-run`, return 0. Without a TTY (`not sys.stdin.isatty()`), print `vrg-promote: refusing to move <rolling> without a terminal to confirm on.` and return 1. Otherwise ask `Move <rolling> on <repo>? [y/N]`, and on anything but `y`/`yes` return 1. Then call `promote_confirmed(p)` with the `Promotion` the human confirmed, never `promote(version)`, so what runs is what was shown.

- [ ] **Step 6: Refactor**

With the tests green, fold the duplicated git-call plumbing in `lib/promote.py` (the tag and push `subprocess.run` blocks) into one helper, without changing behaviour.

- [ ] **Step 7: Run validation to verify it passes**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS, including the `vrg-release` orchestrator tests that call `promote()`.

- [ ] **Step 8: Update the CLI reference**

In `docs/site/docs/reference/cli-tools-overview.md` (`### vrg-promote`), state that it is human-only, the target rule, the dry-run output and the confirmation.

- [ ] **Step 9: Commit**

`vrg-commit --type feat --scope promote --message "refuse agents, check the target and confirm before moving a rolling tag"`

### Task V1: Prove the switch mechanism (validation)

Operational: no PR. Run with `issue-validate`.

**Precondition self-check:** `claude --version` runs where the validation runs (the agent's VM), its version is recorded, and an isolated `CLAUDE_CONFIG_DIR` under the session scratchpad can be created.

**Procedure** (in a fresh isolated `CLAUDE_CONFIG_DIR`, with `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`):

1. Register the marketplace at `main` and install `vergil@vergil-marketplace`, exactly as a current machine is set up.
2. **Rewrite rung (spec §6, rung 1):** `main` and `v2.1` may point at the same commit, which would let a switch that does nothing pass. So the switch target in this step is `v2.1.50`, a ref whose commit (`fc17882`) differs from `main`'s. Edit `settings.json` to `"ref": "v2.1.50"`, edit the registered source's `ref` in `plugins/known_marketplaces.json` to `v2.1.50`, then run `claude plugin marketplace update vergil-marketplace` and `claude plugin update vergil@vergil-marketplace`. Record whether `claude plugin marketplace list --json` shows `ref: v2.1.50`, whether the checkout HEAD is `fc17882`, and whether the installed version is `2.1.50`. Then switch the same way to `v2.1` and record that all three move to `v2.1`'s values. If `main` and `v2.1` have diverged by the time V1 runs, use any release whose commit differs from both.
3. **Only if step 2 fails, remove rung (spec §6, rung 2):** from the same starting state, run `claude plugin marketplace remove vergil-marketplace`, `claude plugin marketplace add vergil-project/vergil-claude-plugin@v2.1` and `claude plugin install vergil@vergil-marketplace`, and record the same three facts. Also record what `remove` deleted under `plugins/data/`.
4. **VM-shaped case:** repeat from step 1 with the new `settings.json` copied in first (as `vrg-vm` does), then run only what `vm_guest.update_plugins` runs today. Record that it does not switch, so that Task 3 is needed.

**Acceptance:** a recorded rung that ends with `list --json` showing `v2.1`, the checkout on the `v2.1` commit and the installed version matching; the Claude Code version; the exact commands; and, verbatim, the raw `claude plugin marketplace list --json` output and the `vergil-marketplace` entry of `plugins/known_marketplaces.json`, both before and after the switch. The post-switch `list --json` output must carry the registered ref, as a `ref` field equal to the declared ref; if it does not, V1 records where the registered ref can be read instead, and Task 3 reads it from there. The result comment is the input to Task 3, whose fixtures are these raw outputs.

### Task 3: Marketplace reconcile for VMs and host

**Files:**

- Create: `src/vergil_tooling/lib/claude_marketplace.py`
- Modify: `src/vergil_tooling/lib/vm_guest.py` (`update_plugins`)
- Modify: `src/vergil_tooling/bin/vrg_host.py` (new `plugins` subcommand)
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`_update_over_transport`)
- Modify: `src/vergil_tooling/lib/vm_memory.py` (`project_memory`, `lock_projection` locked set)
- Test: `tests/vergil_tooling/test_claude_marketplace.py`, `tests/vergil_tooling/test_vm_guest.py`, `tests/vergil_tooling/test_vrg_host.py`, `tests/vergil_tooling/test_vrg_vm.py`, `tests/vergil_tooling/test_vm_memory.py`

**Interfaces:**

- Consumes: the rung and the exact commands recorded by V1.
- Produces, in `lib/claude_marketplace.py`:
  - `class Exec(Protocol)`: `run(*argv: str) -> subprocess.CompletedProcess[str]`; `pipe(command: str, data: str) -> None` (runs `command` through `bash -c` with `data` on stdin); and `python: str` (the interpreter the fixed rewrite program runs under). Every function below takes an `Exec`, never a bare callable, so a data channel separate from command text always exists.
  - `TransportExec(transport: Transport)`: the guest `Exec`. `run` and `pipe` delegate to `transport.run` and `transport.pipe`, and `python` is `"python3"`.
  - `LocalExec()`: the host `Exec`. `run` is `subprocess.run(argv, check=True, capture_output=True, text=True)`, `pipe` is `subprocess.run(["bash", "-c", command], input=data, check=True, text=True)`, and `python` is `sys.executable`, never a bare `python3`, which on macOS may be the Command Line Tools stub.
  - `REWRITE_PROGRAM: str`: one fixed Python program, a module constant with no interpolation. It reads `{"path", "name", "repo", "ref"}` as JSON from stdin, checks the registration entry's shape, rewrites its `source`, and exits non-zero with a message on an unrecognized shape. It is run as `ex.pipe(f"{shlex.quote(ex.python)} -c {shlex.quote(REWRITE_PROGRAM)}", json.dumps(values))`, so that declared values travel only on stdin.
  - `@dataclass(frozen=True) class Declared: name: str; repo: str; ref: str | None`
  - `declared_marketplaces(settings: dict[str, Any]) -> list[Declared]`, from `extraKnownMarketplaces` github sources.
  - `registered_marketplaces(ex: Exec, prefix: str) -> dict[str, tuple[str | None, str | None]]`: name to `(repo, ref)`, from `claude plugin marketplace list --json`. `reconcile` switches a marketplace when either its repo or its ref differs from the declared one. It raises `MarketplaceError` on output that is not a JSON list of objects with a `name` (Review Focus 5).
  - `reconcile(ex: Exec, settings: dict[str, Any], prefix: str) -> list[str]`: for each declared marketplace, adds it when it is not registered (Review Focus 4), does nothing when the registered ref equals the declared ref, and otherwise applies V1's proven rung. A missing registered ref counts as different (Review Focus 3). Returns the names it switched. `prefix` is the shell prefix each command runs under (`_PLUGIN_PATH_EXPORT` in a VM, `""` on the host).
  - `class MarketplaceError(RuntimeError)`
  - `shell(prefix: str, command: str) -> list[str]`: returns `["bash", "-c", f"{prefix} && {command}"]` when `prefix` is non-empty and `["bash", "-c", command]` when it is empty. Every Claude CLI call in this module goes through it.
  - `update_installed_plugins(ex: Exec, settings: dict[str, Any], prefix: str) -> list[str]`: the per-install `claude plugin update` (each entry at its own scope, project entries from their `projectPath`) and the install of enabled-but-missing ids, moved here from `vm_guest` (`_update_plugin_entry` and its caller). Returns failure labels. `vm_guest.update_plugins` and `vrg-host plugins` both call it.
- `vm_guest.update_plugins` replaces its add-only loop for github sources with `reconcile(TransportExec(transport), settings, _PLUGIN_PATH_EXPORT)`, keeps the official-marketplace derivation and `marketplace update`, and adds a failure for any `MarketplaceError`.
- `copy_claude_config` (`lib/vm_guest.py`) becomes the sole owner of the guest `~/.claude/CLAUDE.md` file mode. The guest copy is a read-only copy of the host's on every transport. After writing `CLAUDE.md` it runs `chmod a-w ~/.claude/CLAUDE.md`, on Lima as well as cloud, so every path that copies it leaves it read-only: Lima start, rebuild and session; cloud create and session, including the session-listing path that skips memory projection; and update. `settings.json` stays writable. `vm_memory.project_memory` keeps locking only the memory subtree it owns, and stops writing and locking `CLAUDE.md`, because `copy_claude_config` now does both on every path that reaches it.
- `_update_over_transport` (`bin/vrg_vm.py`) calls `copy_claude_config(transport, Path.home() / ".claude")` before `update_plugins`, so `vrg-vm update` delivers the host's current `settings.json` to a running VM before reconciling. Today only start, rebuild and cloud sessions copy it. Stopped VMs are skipped by `vrg-vm update --all` and switch on their next start, which copies the config.
- `_cmd_update_all` (`bin/vrg_vm.py`): the Lima loop defers a `RuntimeError` from one VM the way the off-platform loop already does, so one VM's plugin failure (including `MarketplaceError`) never stops the rest from updating. The run still exits non-zero and names each failed VM. Test in `test_vrg_vm.py`, `test_update_all_continues_after_lima_plugin_failure`: two Lima VMs, the first one's `update_plugins` raises `RuntimeError`, the second is still updated, and the exit code is non-zero with the first VM named.
- `_read_guest_settings` no longer returns `{}`. A missing, unreadable or invalid guest `settings.json` raises `RuntimeError` naming the file, instead of reconciling nothing and succeeding. A missing file is not "nothing declared": the host file is required, and `vrg-vm` copies it in before reconciling, so its absence means something is broken.
- **Only if V1 proved the remove rung:** before `remove`, `reconcile` records every install of the marketplace's plugins from `claude plugin list --json` (id, scope, `projectPath`). After the re-add, `update_installed_plugins` reinstalls each at its own scope, from its `projectPath` for project scope, with a test that a project-scope install comes back at project scope.
- `vrg-host plugins` reads `~/.claude/settings.json` and runs `reconcile(LocalExec(), settings, "")`, then `claude plugin marketplace update` and `update_installed_plugins(LocalExec(), settings, "")`. It refuses under an agent identity with the fail-closed check (`identity_mode.is_agent() or identity_mode.any_agent_signal()`), exiting 1 before any `claude` call. If any step fails, it names each failure and exits 2, following `vrg-host`'s convention (0 OK, 1 refusal or warning, 2 failure).

- [ ] **Step 1: Write the failing tests** (`test_claude_marketplace.py`, a fake `run` that records argv and returns canned `list --json` output)

```python
def test_reconcile_switches_mismatched_ref() -> None:
    # registered {"vergil-marketplace": "main"}, declared ref "v2.1"
    assert reconcile(fake, SETTINGS_V21, "") == ["vergil-marketplace"]
    # fake.calls contains V1's proven commands, in V1's order

def test_reconcile_switches_mismatched_repo() -> None:
    # registered {"vergil-marketplace": ("other/repo", "v2.1")}, declared repo
    # "vergil-project/vergil-claude-plugin" -> switched


def test_reconcile_is_idempotent_on_switched_state() -> None:
    # fake returns V1's recorded post-switch `list --json` output verbatim
    assert reconcile(fake_post_switch, SETTINGS_V21, "") == []
    # no add, remove or rewrite call was recorded

def test_reconcile_leaves_matching_ref_alone() -> None:
    # registered "v2.1" -> returns [], and no add, remove or rewrite call

def test_reconcile_treats_missing_registered_ref_as_mismatch() -> None:  # RF3
    # registered {"vergil-marketplace": None} -> switched

def test_reconcile_adds_unregistered_marketplace() -> None:  # RF4
    # registered {} -> one `marketplace add vergil-project/vergil-claude-plugin@v2.1`, no remove

def test_registered_marketplaces_rejects_malformed_json() -> None:  # RF5
    with pytest.raises(MarketplaceError):
        registered_marketplaces(fake_returning('{"not": "a list"}'), "")

def test_reconcile_quotes_declared_values() -> None:
    # declared ref is "v2.1'; touch x"
    reconcile(fake, settings_with_ref("v2.1'; touch x"), "")
    # every recorded bash -c string contains shlex.quote("v2.1'; touch x"),
    # never the raw value; the known_marketplaces.json rewrite received the
    # value on stdin (fake.piped), not in its program text

def test_rewrite_rejects_unknown_registration_shape() -> None:
    # known_marketplaces.json entry has source.source == "git"
    with pytest.raises(MarketplaceError, match="unrecognized"):
        reconcile(fake_with_entry({"source": {"source": "git", "url": "x"}}), SETTINGS_V21, "")

def test_declared_marketplaces_skips_non_github_sources() -> None:
    settings = {"extraKnownMarketplaces": {
        "a": {"source": {"source": "github", "repo": "o/a", "ref": "v2.1"}},
        "b": {"source": {"source": "url", "url": "https://x/m.json"}}}}
    assert declared_marketplaces(settings) == [Declared("a", "o/a", "v2.1")]

def test_shell_without_prefix_has_no_leading_and() -> None:
    assert shell("", "claude plugin marketplace update") == [
        "bash", "-c", "claude plugin marketplace update"]
    assert shell("export PATH=x", "claude plugin list --json") == [
        "bash", "-c", "export PATH=x && claude plugin list --json"]
```

The `list --json` and `known_marketplaces.json` fixtures are V1's recorded raw outputs, copied verbatim into the test module.

In `test_vrg_vm.py`, add `test_update_over_transport_copies_config_before_plugins`: the recorded calls show `copy_claude_config` before `update_plugins`. In `test_vm_guest.py`, add `test_update_plugins_fails_on_missing_unreadable_or_invalid_guest_settings`, parametrized over the three cases: the guest `cat ~/.claude/settings.json` exits non-zero because the file is missing, exits non-zero because it cannot be read, or prints invalid JSON. Each raises `RuntimeError` matching `~/.claude/settings.json`, before any reconcile call. In `test_vm_guest.py`, also add `test_copy_claude_config_locks_claude_md` (the piped commands write `CLAUDE.md` and then run `chmod a-w ~/.claude/CLAUDE.md`, and never `chmod a-w` `settings.json`). In `test_vm_memory.py`, update the `project_memory` tests: it no longer writes `CLAUDE.md`, and the `lock_projection` locked set no longer names it.

Before implementing, confirm by grep that nothing inside a guest writes `~/.claude/CLAUDE.md` (`grep -rn 'CLAUDE.md' src/vergil_tooling` and the plugin's hooks). If something does, stop and record it in the task issue instead of locking. In `test_vm_guest.py`, add `test_update_plugins_switches_marketplace_ref`: the guest settings declare `v2.1`, the fake transport's `list --json` reports `main`, and the recorded calls include the switch before `marketplace update`. In `test_vrg_host.py`, add `test_plugins_refuses_agent` (agent identity, exit 1, no `claude` call), `test_plugins_refuses_mixed_signals` (`VRG_IDENTITY_MODE=human` with `any_agent_signal()` patched to `True`, exit 1, no `claude` call), `test_plugins_runs_reconcile_then_update` (calls in the order reconcile, `marketplace update`, per-install update) and `test_plugins_exits_2_naming_each_failure` (two failing installs, exit 2, stderr names both).

**Quoting rule:** every declared value (marketplace name, repo, ref) reaches a `bash -c` string only through `shlex.quote`, as `vm_guest` already does. The `known_marketplaces.json` rewrite is `REWRITE_PROGRAM`, run through `Exec.pipe` with its values as JSON on stdin, never spliced into program text. The test fake implements `Exec` (recording `run` argv and `pipe` command and data separately), so that `test_reconcile_quotes_declared_values` can assert the value appears in the piped data and never in a command string.

- [ ] **Step 2: Run to verify the tests fail**

Run: `vrg-container-run -- vrg-validate`
Expected, per file:

- `test_claude_marketplace.py`: every test errors on `ImportError` (`lib/claude_marketplace.py` does not exist).
- `test_vm_guest.py`: the marketplace-switch test fails because `update_plugins` only re-adds, which is a no-op; the guest-settings test fails because `_read_guest_settings` returns `{}`; the `CLAUDE.md` lock test fails because no `chmod a-w` is piped.
- `test_vm_memory.py`: the updated tests fail because `project_memory` still writes and locks `CLAUDE.md`.
- `test_vrg_vm.py`: the update-ordering test fails because `_update_over_transport` never calls `copy_claude_config`; the update-all test fails because the Lima loop lets `RuntimeError` escape.
- `test_vrg_host.py`: every test fails, because `vrg-host` has no `plugins` subcommand.

- [ ] **Step 3: Implement `lib/claude_marketplace.py`** with the interfaces above and V1's commands. If V1 proved the rewrite rung, the rewrite edits `known_marketplaces.json` with `REWRITE_PROGRAM` through `Exec.pipe`, and checks that the entry has the shape V1 recorded (`source.source == "github"`, `source.repo`, optional `source.ref`). It raises `MarketplaceError` on any other shape, so that a Claude Code format change fails loudly.

- [ ] **Step 4: Wire it into `vm_guest.update_plugins` and `_update_over_transport`, and add `vrg-host plugins`.** The existing `test_vm_guest.py` tests of `_update_plugin_entry` move to `test_claude_marketplace.py` with the function, unchanged in intent.

- [ ] **Step 5: Refactor**

With the tests green, remove what is left of the old add-only loop and any now-unused helper in `vm_guest.py`, without changing behaviour.

- [ ] **Step 6: Run validation to verify it passes**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 7: Document `vrg-host plugins`**

Add it to `docs/site/docs/reference/cli-tools-overview.md` (`### vrg-host`), and update the `vrg-host` module docstring and the subcommand metavar (`{status,upgrade,pin,unpin}` gains `plugins`). In `docs/site/docs/guides/agent-vm-claude-share-set.md` (the `plugins/` and `update_plugins` description, about lines 54–66), replace the add-only registration with the reconcile, the update-time `settings.json` copy and the read-only guest `CLAUDE.md`.

- [ ] **Step 8: Commit**

`vrg-commit --type feat --scope plugins --message "reconcile the registered marketplace source with the declared one on host and VMs"`

### Task D1: Apply the rolling-tag rulesets fleet-wide (deployment)

Operational: no PR. Run with `issue-deploy`.

**Precondition (human-attested):** a vergil-tooling release that contains Task 1 is cut and installed on the host. The agent never cuts it.

**Timing:** run D1 right after that release. From the release on, the nightly `ops.yml` audit expects the new rulesets, so it reports drift on every repo until D1 applies them. Those drift issues are expected, and D1 closes each one once its repo is applied.

**Procedure:** the human runs `vrg-github-repo-config apply --repo <owner/repo>` for each repo in the fleet (the rulesets are org-wide desired state, so every repo with rolling tags gets them). Then `vrg-github-repo-config audit --repo vergil-project/vergil-claude-plugin` is run, and `vrg-git ls-remote --heads https://github.com/vergil-project/vergil-claude-plugin 'v*'` confirms that no branch matches.

**Verification of the access rules:**

1. The human runs `vrg-github-repo-config audit` with their own (non-App) credentials on vergil-claude-plugin, vergil-tooling and vergil-actions, so that bypass actors are compared too.
2. The live `v2.1` is never touched. A same-commit push of it sends no ref update, so GitHub would evaluate no rule. The test uses a throwaway rolling-style tag that the widened `refs/tags/v*.*` include covers: the human creates `v0.0` in vergil-claude-plugin at an old release commit `<A>` (creating a tag is not blocked).
3. **Non-Admin refused.** The human, holding an agent App installation token, runs `GH_TOKEN=<installation token> gh api -X PATCH repos/vergil-project/vergil-claude-plugin/git/refs/tags/v0.0 -f sha=<B> -F force=true`, with `<B>` another release commit. GitHub must refuse it with a repository rule violation. The human runs this step because the agent's own wrappers deny both `push --force` and `gh api`, so an agent attempt would be refused by the wrapper, not by GitHub.
4. **Admin bypass accepted.** The human moves `v0.0` to `<B>` from their own shell (`git tag -f v0.0 <B> && git push --force origin v0.0`). GitHub must accept it.
5. The human deletes `v0.0` (`git push origin :refs/tags/v0.0`), which the Admin bypass also allows, and confirms it is gone with `vrg-git ls-remote --tags https://github.com/vergil-project/vergil-claude-plugin v0.0`.
6. **Branch-name ruleset refuses everyone.** The human runs `GH_TOKEN=<installation token> gh api -X POST repos/vergil-project/vergil-claude-plugin/git/refs -f ref=refs/heads/v0.0 -f sha=<A>` with the agent App installation token, then the same call with their own Admin credentials. GitHub must refuse both with a repository rule violation (the ruleset has no bypass). `vrg-git ls-remote --heads https://github.com/vergil-project/vergil-claude-plugin v0.0` then prints nothing.

**Acceptance:** all three audits report no drift, including both rulesets and their bypass actors; the App-token move is refused, with GitHub's rule-violation message recorded verbatim; the Admin move and delete succeed; both branch-creation attempts are refused, with GitHub's messages recorded verbatim, and no `v0.0` branch exists; the live branch-name ruleset's include, read with the human's credentials, equals Task 1's `BRANCH_PATTERN` literal exactly; `v2.1` is unchanged; and no `refs/heads/v*` branch exists. This is the spec §2 order condition; D2 may not start before it holds.

### Task 4: Plugin distribution docs (vergil-claude-plugin)

**Files:**

- Modify: `README.md` ("Distribution model", "Install"), `CLAUDE.md` (the distribution paragraph near line 120), `docs/site/docs/index.md` (the settings example)

- [ ] **Step 1:** Change each `"ref": "main"` example to `"ref": "v2.1"` and `vergil-marketplace@main` to `vergil-marketplace@v2.1`.
- [ ] **Step 2:** Rewrite the "Distribution model" section. Consumers follow the floating `v2.1` tag; a release moves it forward; a rollback is `vrg-promote` to an earlier `v2.1.Z`, linking the vergil-tooling rollback runbook (`docs/site/docs/guides/releasing.md`, rollback section); the decision record is `epics/760-plugin-tag-distribution/spec.md`, which supersedes epic #45's tag decision. Link Task 5's "Moving to a new minor line" paragraph.
- [ ] **Step 3:** Rewrite these passages, which describe `main` as what consumers run, so that each describes the floating `v2.1` tag and rollback and links `epics/760-plugin-tag-distribution/spec.md`. Do not just string-replace them, because that would make them false:
  - `README.md`, the paragraph after the settings example (about lines 88–91, "`"ref": "main"` pins the marketplace to the **release** branch…"): it now says `v2.1` follows the latest release and moves back on a rollback.
  - `CLAUDE.md`, the "Plugin distribution model" paragraph ("releasing to `main` is what makes a skill/hook live", and its epic #45 link): it now says a release makes a change live by moving `v2.1`, and that a rollback moves it back. It links this epic's spec, as superseding #45's tag decision.
  - `docs/site/docs/index.md`, about line 34 ("ships through a single released channel on `main`"): it now says the single channel is the floating `v2.1` tag.
- [ ] **Step 3a:** Remove the "external orgs" wording from `README.md` and `CLAUDE.md`.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Expected: PASS (markdown lint and docs build).
- [ ] **Step 5:** `vrg-commit --type docs --scope distribution --message "consumers follow the floating v2.1 tag"`

### Task 5: Consumer setup and rollback runbook (vergil-tooling docs)

**Files:**

- Modify: `docs/site/docs/guides/consuming-repo-setup.md` (the `enabledPlugins` row and the marketplace definition)
- Modify: `docs/site/docs/guides/releasing.md` (new "Rolling back a release" section)

- [ ] **Step 1:** In `consuming-repo-setup.md`, change the marketplace definition to `ref: "v2.1"`, and add a "Switching an existing machine" paragraph: `vrg-host plugins` on the host and `vrg-vm update --all` for the VMs, which reconcile the registered source (Task 3). If V1 proved the remove rung, list what it deletes. Add a "Moving to a new minor line" paragraph: when the plugin's first `X.Y+1.0` release creates a new rolling tag, `v2.1` stops moving, and the fleet stays on the last `v2.1.Z` until the human changes the host `ref` to the new tag and runs the same two commands. Moving is a deliberate act, never automatic (spec §8).
- [ ] **Step 2:** In `releasing.md`, add "Rolling back a release" with spec §5 verbatim in substance: the four exact steps (`cd` into the repo's checkout, `vrg-git fetch --tags --force origin`, `vrg-promote --dry-run <X.Y.Z>`, `vrg-promote <X.Y.Z>`); picking up the move on each machine (`vrg-host plugins`, `vrg-vm update --all`, then a restart of Claude Code); and that the rollback lasts until the next release, held by hand with `vrg-release --no-promote`, linking vergil-project/.github#763.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 4:** `vrg-commit --type docs --scope release --message "document rolling back a release and switching the marketplace ref"`

### Task D2: Switch the fleet to `v2.1` (deployment)

Operational: no PR. Run with `issue-deploy`.

**Precondition (human-attested):** D1 accepted; a vergil-tooling release containing Tasks 2 and 3 is cut and installed on the host and in every VM (`vrg-vm update --all` installs it).

**Who runs what:** steps 1 to 3 act on the host, which the agent cannot reach and `vrg-host plugins` refuses an agent, so **the human runs them** and pastes their output into the task. The agent runs `issue-deploy`, records that output, and runs the checks it can make itself inside its own VM.

**Procedure:**

1. The human edits the host `~/.claude/settings.json`: `extraKnownMarketplaces.vergil-marketplace.source.ref` from `"main"` to `"v2.1"`.
2. The human runs `vrg-host plugins`. Expected: it reports `vergil-marketplace` switched, and `claude plugin marketplace list --json` shows `ref: v2.1`.
3. The human runs `vrg-vm update --all`. Expected: each VM reports the marketplace switched, with no failure. The agent then confirms in its own VM that `claude plugin marketplace list --json` shows `ref: v2.1`.
4. The agent sweeps every fleet repo's `develop`, `.github` included, by listing the repos explicitly (a glob skips dot-directories) and reading each one's `origin/develop` rather than its working copy: `cd <repo> && vrg-git fetch -q origin develop && vrg-git show origin/develop:.claude/settings.json | grep -c extraKnownMarketplaces`. Expected: `0` for every repo. Any hit is removed through that repo's normal worktree and PR flow, in favour of the template.

**Acceptance:** the host and every running VM show `ref: v2.1` in `claude plugin marketplace list --json` and the installed version equals the release `v2.1` points at; the sweep is clean. A VM that was stopped is listed in the result comment and switches on its next `vrg-vm start`.

### Task V2: Rollback round trip (validation)

Operational: no PR. Run with `issue-validate`. The tag moves, the release and the host-side updates (`vrg-host plugins`, `vrg-vm update --all`) are human acts, because the agent cannot reach the host and `vrg-host plugins` refuses an agent. The human runs them and pastes their output; the agent records it and makes the observations it can make inside its own VM.

**Precondition self-check:** D2 accepted; Task 5's runbook is published.

**Procedure:**

1. The human picks an earlier release `v2.1.<B>` and runs the runbook's four steps against vergil-claude-plugin.
2. Update the host (`vrg-host plugins`) and the VMs (`vrg-vm update --all`). On the host and every VM, record the installed version (`claude plugin list --json`), and in a freshly started session, the version the session loaded. The human confirms that the host's `~/.config/vergil/plugin-dirs.toml` (host-local; `vrg-vm session` reads it and adds `--plugin-dir` for Lima sessions) lists no vergil plugin tree, and that the host session itself was not started with `--plugin-dir`, so that every session loads the installed plugin.
3. The human runs `vrg-promote <latest v2.1.Z>` by the same steps.
4. Repeat step 2 for the forward version.
5. **End-to-end release:** a trivial test change goes through the normal plugin development flow (an issue, a PR against vergil-claude-plugin, merge, `vrg-release`). That release's promote stage moves `v2.1` forward on its own. Repeat step 2 for the new version.

**Acceptance:** every machine shows `<B>` after step 2, the latest release after step 4, and the new test release after step 5, by both observations. Where the fleet is left afterwards is the human's decision, recorded in the result comment.
