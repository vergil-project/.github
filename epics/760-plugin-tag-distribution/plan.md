# Plugin distribution via the floating `v2.1` tag Implementation Plan

- **Epic:** vergil-project/.github#760
- **Planning issue:** vergil-project/.github#759 (review history is in its
  comments)
- **Status:** in design review

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the `vergil-marketplace` ref from `main` to the floating `v2.1` tag, with the tag protected as the deploy ref, a hardened `vrg-promote` as the rollback tool, and a fully mechanized switch of the whole fleet.

**Architecture:** The vergil-tooling changes come first: the rolling-tag rulesets, the `vrg-promote` hardening and a shared marketplace-reconcile module used by the VM refresh and a new host command. A deployment applies the rulesets, a second deployment switches the fleet, and a validation runs a real rollback round trip. The docs change in the plugin repo and vergil-tooling.

**Tech Stack:** Python 3.14 (vergil-tooling, pytest), the GitHub rulesets API through `vrg-github-repo-config`, the Claude Code plugin CLI (`claude plugin marketplace …`), Markdown docs.

**Spec:** `epics/760-plugin-tag-distribution/spec.md`

## Global Constraints

- The marketplace source is `{"source": "github", "repo": "vergil-project/vergil-claude-plugin", "ref": "v2.1"}` (spec §2).
- Rolling tags `vX.Y` are protected by a tag ruleset that blocks update and deletion, bypassed only by the repository Admin role (`RepositoryRole`, `actor_id` 5) (spec §2).
- No branch may match `refs/heads/v*`; the branch-name ruleset has no bypass actor (spec §2).
- `vrg-promote` refuses an agent identity with the `vrg-release` check, failing closed on any agent signal (spec §2).
- `vrg-promote` moves a rolling tag only to an existing `vX.Y.Z` tag whose commit is on `origin/main` (spec §2).
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
| 4 | Plugin distribution docs | vergil-claude-plugin | task | — |
| 5 | Consumer setup and rollback runbook | vergil-tooling | task | 2, 3 |
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

def test_rolling_tag_branch_ruleset() -> None:
    r = desired_rolling_tag_branch_ruleset()
    assert r.name == "Rolling tag branch names"
    assert r.target == "branch"
    assert r.enforcement == "active"
    assert r.ref_include == ["refs/heads/v*"]
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
Expected: FAIL in the three tests above (wrong `ref_include`, undefined function, missing ruleset).

- [ ] **Step 3: Implement**

Add `"refs/tags/v*.*"` to `desired_tag_protection_ruleset().ref_include`. Add `desired_rolling_tag_branch_ruleset()` next to it, shaped like `desired_branch_protection_ruleset()`, with the values in the test. Append it in `compute_desired_state` right after the tag ruleset; app mode nulls its bypass actors like the others.

The classic-branch-protection helpers derive branch names from a branch ruleset's includes (`_ruleset_branches` via `_branch_from_ref`), and would query `branches/v*/protection`. Make `_branch_from_ref` return nothing for an include containing a glob character (`*`, `?`, `[`), with the test `test_branch_from_ref_skips_glob_includes` (`"refs/heads/v*"` gives no branch, `"refs/heads/main"` gives `"main"`).

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
- Test: `tests/vergil_tooling/test_promote.py`, `tests/vergil_tooling/test_vrg_promote.py`

**Interfaces:**
- Produces, in `lib/promote.py`:
  - `class PromoteError(ValueError)`
  - `@dataclass(frozen=True) class Promotion: origin_repo: str; rolling_tag: str; release_tag: str; old_commit: str; new_commit: str`. `old_commit` is `""` when the rolling tag does not exist yet.
  - `plan_promotion(version: str) -> Promotion`: first refuses with `PromoteError` when `identity_mode.is_agent() or identity_mode.any_agent_signal()`, before any git call, so every caller (the CLI and `vrg-release`'s promote stage) fails closed. It then validates the version, resolves both tags on origin, and raises `PromoteError` when the release tag is absent on origin or its commit is not an ancestor of `origin/main`.
  - `describe(p: Promotion) -> str`: the lines the CLI prints for the dry run and before confirmation: `origin: <repo>`, `<rolling> <old[:12] or (new)> -> <release> <new[:12]>`.
  - `promote(version: str, *, dry_run: bool = False) -> None`: unchanged signature (the `vrg-release` orchestrator calls it). It calls `plan_promotion`, so the target check applies to releases too, and it never prompts.
- The CLI (`bin/vrg_promote.py`) owns the identity refusal, the dry-run print and the confirmation prompt.

- [ ] **Step 1: Write the failing library tests** (`test_promote.py`, patching `subprocess.run` as the existing tests do)

```python
def test_plan_promotion_refuses_tag_not_on_main() -> None:
    # ls-remote gives a v2.1.50 commit; `merge-base --is-ancestor` exits 1
    with pytest.raises(PromoteError, match="not on origin/main"):
        plan_promotion("2.1.50")

def test_plan_promotion_refuses_missing_release_tag() -> None:
    # ls-remote for refs/tags/v2.1.99 returns nothing
    with pytest.raises(PromoteError, match="v2.1.99 does not exist on origin"):
        plan_promotion("2.1.99")

def test_plan_promotion_reports_repo_and_commits() -> None:
    p = plan_promotion("2.1.50")
    assert p == Promotion("vergil-project/vergil-claude-plugin", "v2.1", "v2.1.50", OLD, NEW)

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
| `test_promote_runs_tag_and_push` | Assert exactly one `git tag -f v2.1 v2.1.50` and one `git push origin v2.1 --force`, after the planning calls |
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
    # isatty -> True, input -> "y"; assert `git push origin v2.1 --force` was called

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

`plan_promotion` uses `_peeled_commit` for both tags and `git merge-base --is-ancestor <new> origin/main`, after `git fetch --tags --force origin main`, so that `origin/main` and the tags are current. A non-zero fetch raises `PromoteError`. `promote` becomes: `p = plan_promotion(version)`; if `p.old_commit == p.new_commit`, print the existing "already promoted" line and return; if `dry_run`, print `describe(p)` and return; otherwise tag and push as today.

- [ ] **Step 5: Implement the CLI**

In `main()`, before anything else:

```python
if identity_mode.is_agent() or identity_mode.any_agent_signal():
    print("vrg-promote: moving a rolling tag is a human maintainer action. "
          "Agents cannot run vrg-promote.", file=sys.stderr)
    return 1
```

Then `p = plan_promotion(version)` (a `PromoteError` prints `Error: …` and returns 1). If `p.old_commit == p.new_commit`, print `<rolling> already points at <release> — already promoted.` and return 0, before the dry-run, TTY and prompt branches. Otherwise print `describe(p)`. With `--dry-run`, return 0. Without a TTY (`not sys.stdin.isatty()`), print `vrg-promote: refusing to move <rolling> without a terminal to confirm on.` and return 1. Otherwise ask `Move <rolling> on <repo>? [y/N] `, and on anything but `y`/`yes` return 1. Then call `promote(version)`.

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

**Precondition self-check:** `claude --version` runs on the host, and an isolated `CLAUDE_CONFIG_DIR` under the session scratchpad can be created.

**Procedure** (in a fresh isolated `CLAUDE_CONFIG_DIR`, with `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1`):

1. Register the marketplace at `main` and install `vergil@vergil-marketplace`, exactly as a current machine is set up.
2. **Rewrite rung (spec §6, rung 1):** edit `settings.json` to `"ref": "v2.1"`, edit the registered source's `ref` in `plugins/known_marketplaces.json` to `v2.1`, then run `claude plugin marketplace update vergil-marketplace` and `claude plugin update vergil@vergil-marketplace`. Record whether `claude plugin marketplace list --json` shows `ref: v2.1`, whether the checkout HEAD is the `v2.1` commit, and whether the installed version matches the release `v2.1` points at.
3. **Only if step 2 fails, remove rung (spec §6, rung 2):** from the same starting state, run `claude plugin marketplace remove vergil-marketplace`, `claude plugin marketplace add vergil-project/vergil-claude-plugin@v2.1` and `claude plugin install vergil@vergil-marketplace`, and record the same three facts. Also record what `remove` deleted under `plugins/data/`.
4. **VM-shaped case:** repeat from step 1 with the new `settings.json` copied in first (as `vrg-vm` does), then run only what `vm_guest.update_plugins` runs today. Record that it does not switch, so that Task 3 is needed.

**Acceptance:** a recorded rung that ends with `list --json` showing `v2.1`, the checkout on the `v2.1` commit and the installed version matching; the Claude Code version; the exact commands; and, verbatim, the raw `claude plugin marketplace list --json` output and the `vergil-marketplace` entry of `plugins/known_marketplaces.json`, both before and after the switch. The result comment is the input to Task 3, whose fixtures are these raw outputs.

### Task 3: Marketplace reconcile for VMs and host

**Files:**
- Create: `src/vergil_tooling/lib/claude_marketplace.py`
- Modify: `src/vergil_tooling/lib/vm_guest.py` (`update_plugins`)
- Modify: `src/vergil_tooling/bin/vrg_host.py` (new `plugins` subcommand)
- Modify: `src/vergil_tooling/bin/vrg_vm.py` (`_update_over_transport`)
- Test: `tests/vergil_tooling/test_claude_marketplace.py`, `tests/vergil_tooling/test_vm_guest.py`, `tests/vergil_tooling/test_vrg_host.py`, `tests/vergil_tooling/test_vrg_vm.py`

**Interfaces:**
- Consumes: the rung and the exact commands recorded by V1.
- Produces, in `lib/claude_marketplace.py`:
  - `Run = Callable[..., subprocess.CompletedProcess[str]]`, the shape of `Transport.run(*argv)`.
  - `@dataclass(frozen=True) class Declared: name: str; repo: str; ref: str | None`
  - `declared_marketplaces(settings: dict[str, Any]) -> list[Declared]`, from `extraKnownMarketplaces` github sources.
  - `registered_marketplaces(run: Run, prefix: str) -> dict[str, str | None]`: name to ref, from `claude plugin marketplace list --json`. It raises `MarketplaceError` on output that is not a JSON list of objects with a `name` (Review Focus 5).
  - `reconcile(run: Run, settings: dict[str, Any], prefix: str) -> list[str]`: for each declared marketplace, adds it when it is not registered (Review Focus 4), does nothing when the registered ref equals the declared ref, and otherwise applies V1's proven rung. A missing registered ref counts as different (Review Focus 3). Returns the names it switched. `prefix` is the shell prefix each command runs under (`_PLUGIN_PATH_EXPORT` in a VM, `""` on the host).
  - `class MarketplaceError(RuntimeError)`
  - `shell(prefix: str, command: str) -> list[str]`: returns `["bash", "-c", f"{prefix} && {command}"]` when `prefix` is non-empty and `["bash", "-c", command]` when it is empty. Every Claude CLI call in this module goes through it.
  - `update_installed_plugins(run: Run, settings: dict[str, Any], prefix: str) -> list[str]`: the per-install `claude plugin update` (each entry at its own scope, project entries from their `projectPath`) and the install of enabled-but-missing ids, moved here from `vm_guest` (`_update_plugin_entry` and its caller). Returns failure labels. `vm_guest.update_plugins` and `vrg-host plugins` both call it.
  - `local_run(*argv: str) -> subprocess.CompletedProcess[str]`: the host's `Run`, `subprocess.run(argv, check=True, capture_output=True, text=True)`.
- `vm_guest.update_plugins` replaces its add-only loop for github sources with `reconcile(transport.run, settings, _PLUGIN_PATH_EXPORT)`, keeps the official-marketplace derivation and `marketplace update`, and adds a failure for any `MarketplaceError`.
- `_update_over_transport` (`bin/vrg_vm.py`) calls `copy_claude_config(transport, Path.home() / ".claude")` before `update_plugins`, so `vrg-vm update` delivers the host's current `settings.json` to a running VM before reconciling. Today only start, rebuild and cloud sessions copy it. Stopped VMs are skipped by `vrg-vm update --all` and switch on their next start, which copies the config.
- `vrg-host plugins` reads `~/.claude/settings.json` and runs `reconcile(local_run, settings, "")`, then `claude plugin marketplace update` and `update_installed_plugins(local_run, settings, "")`. It refuses under an agent identity, and exits 1 and names each failure if any step fails.

- [ ] **Step 1: Write the failing tests** (`test_claude_marketplace.py`, a fake `run` that records argv and returns canned `list --json` output)

```python
def test_reconcile_switches_mismatched_ref() -> None:
    # registered {"vergil-marketplace": "main"}, declared ref "v2.1"
    assert reconcile(fake, SETTINGS_V21, "") == ["vergil-marketplace"]
    # fake.calls contains V1's proven commands, in V1's order

def test_reconcile_leaves_matching_ref_alone() -> None:
    # registered "v2.1" -> returns [], and no add, remove or rewrite call

def test_reconcile_treats_missing_registered_ref_as_mismatch() -> None:  # RF3
    # registered {"vergil-marketplace": None} -> switched

def test_reconcile_adds_unregistered_marketplace() -> None:  # RF4
    # registered {} -> one `marketplace add vergil-project/vergil-claude-plugin@v2.1`, no remove

def test_registered_marketplaces_rejects_malformed_json() -> None:  # RF5
    with pytest.raises(MarketplaceError):
        registered_marketplaces(fake_returning('{"not": "a list"}'), "")

def test_shell_without_prefix_has_no_leading_and() -> None:
    assert shell("", "claude plugin marketplace update") == [
        "bash", "-c", "claude plugin marketplace update"]
    assert shell("export PATH=x", "claude plugin list --json") == [
        "bash", "-c", "export PATH=x && claude plugin list --json"]
```

The `list --json` and `known_marketplaces.json` fixtures are V1's recorded raw outputs, copied verbatim into the test module.

In `test_vrg_vm.py`, add `test_update_over_transport_copies_config_before_plugins`: the recorded calls show `copy_claude_config` before `update_plugins`. In `test_vm_guest.py`, add `test_update_plugins_switches_marketplace_ref`: the guest settings declare `v2.1`, the fake transport's `list --json` reports `main`, and the recorded calls include the switch before `marketplace update`. In `test_vrg_host.py`, add `test_plugins_refuses_agent` and `test_plugins_runs_reconcile_then_update`.

- [ ] **Step 2: Run to verify the tests fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL (the module does not exist).

- [ ] **Step 3: Implement `lib/claude_marketplace.py`** with the interfaces above and V1's commands. If V1 proved the rewrite rung, the rewrite edits `known_marketplaces.json` through `run` (`python3 -c` or `jq` in the guest), and checks that the entry has the shape V1 recorded (`source.source == "github"`, `source.repo`, optional `source.ref`). It raises `MarketplaceError` on any other shape, so that a Claude Code format change fails loudly.

- [ ] **Step 4: Wire it into `vm_guest.update_plugins` and `_update_over_transport`, and add `vrg-host plugins`.** The existing `test_vm_guest.py` tests of `_update_plugin_entry` move to `test_claude_marketplace.py` with the function, unchanged in intent.

- [ ] **Step 5: Refactor**

With the tests green, remove what is left of the old add-only loop and any now-unused helper in `vm_guest.py`, without changing behaviour.

- [ ] **Step 6: Run validation to verify it passes**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 7: Document `vrg-host plugins`**

Add it to `docs/site/docs/reference/cli-tools-overview.md` (`### vrg-host`), and update the `vrg-host` module docstring and the subcommand metavar (`{status,upgrade,pin,unpin}` gains `plugins`).

- [ ] **Step 8: Commit**

`vrg-commit --type feat --scope plugins --message "reconcile the registered marketplace source with the declared one on host and VMs"`

### Task D1: Apply the rolling-tag rulesets fleet-wide (deployment)

Operational: no PR. Run with `issue-deploy`.

**Precondition (human-attested):** a vergil-tooling release that contains Task 1 is cut and installed on the host. The agent never cuts it.

**Timing:** run D1 right after that release. From the release on, the nightly `ops.yml` audit expects the new rulesets, so it reports drift on every repo until D1 applies them. Those drift issues are expected, and D1 closes each one once its repo is applied.

**Procedure:** the human runs `vrg-github-repo-config apply --repo <owner/repo>` for each repo in the fleet (the rulesets are org-wide desired state, so every repo with rolling tags gets them). Then `vrg-github-repo-config audit --repo vergil-project/vergil-claude-plugin` is run, and `vrg-git ls-remote --heads https://github.com/vergil-project/vergil-claude-plugin 'v*'` confirms that no branch matches.

**Verification of the access rules:**
1. The human runs `vrg-github-repo-config audit` with their own (non-App) credentials on vergil-claude-plugin, vergil-tooling and vergil-actions, so that bypass actors are compared too.
2. The agent, under its App identity, force-pushes `v2.1` to the commit it already points at in vergil-claude-plugin, and records that GitHub refuses the push.
3. The human force-pushes `v2.1` to the commit it already points at, from their own shell (`git push --force origin v2.1` after `git fetch --tags --force origin`), and records that GitHub accepts it. `vrg-promote` is not used here, because a same-target run pushes nothing.

**Acceptance:** all three audits report no drift, including both rulesets and their bypass actors; the agent's push is refused and the human's is accepted; and no `refs/heads/v*` branch exists. This is the spec §2 order condition; D2 may not start before it holds.

### Task 4: Plugin distribution docs (vergil-claude-plugin)

**Files:**
- Modify: `README.md` ("Distribution model", "Install"), `CLAUDE.md` (the distribution paragraph near line 120), `docs/site/docs/index.md` (the settings example)

- [ ] **Step 1:** Change each `"ref": "main"` example to `"ref": "v2.1"` and `vergil-marketplace@main` to `vergil-marketplace@v2.1`.
- [ ] **Step 2:** Rewrite the "Distribution model" section. Consumers follow the floating `v2.1` tag; a release moves it forward; a rollback is `vrg-promote` to an earlier `v2.1.Z`, linking the vergil-tooling rollback runbook (`docs/site/docs/guides/releasing.md`, rollback section); the decision record is `epics/760-plugin-tag-distribution/spec.md`, which supersedes epic #45's tag decision.
- [ ] **Step 3:** Remove the "external orgs" wording from `README.md` and `CLAUDE.md`.
- [ ] **Step 4:** `vrg-container-run -- vrg-validate`. Expected: PASS (markdown lint and docs build).
- [ ] **Step 5:** `vrg-commit --type docs --scope distribution --message "consumers follow the floating v2.1 tag"`

### Task 5: Consumer setup and rollback runbook (vergil-tooling docs)

**Files:**
- Modify: `docs/site/docs/guides/consuming-repo-setup.md` (the `enabledPlugins` row and the marketplace definition)
- Modify: `docs/site/docs/guides/releasing.md` (new "Rolling back a release" section)

- [ ] **Step 1:** In `consuming-repo-setup.md`, change the marketplace definition to `ref: "v2.1"`, and add a "Switching an existing machine" paragraph: `vrg-host plugins` on the host and `vrg-vm update --all` for the VMs, which reconcile the registered source (Task 3). If V1 proved the remove rung, list what it deletes.
- [ ] **Step 2:** In `releasing.md`, add "Rolling back a release" with spec §5 verbatim in substance: the four exact steps (`cd` into the repo's checkout, `vrg-git fetch --tags --force origin`, `vrg-promote --dry-run <X.Y.Z>`, `vrg-promote <X.Y.Z>`); picking up the move on each machine (`vrg-host plugins`, `vrg-vm update --all`, then a restart of Claude Code); and that the rollback lasts until the next release, held by hand with `vrg-release --no-promote`, linking vergil-project/.github#763.
- [ ] **Step 3:** `vrg-container-run -- vrg-validate`. Expected: PASS.
- [ ] **Step 4:** `vrg-commit --type docs --scope release --message "document rolling back a release and switching the marketplace ref"`

### Task D2: Switch the fleet to `v2.1` (deployment)

Operational: no PR. Run with `issue-deploy`.

**Precondition (human-attested):** D1 accepted; a vergil-tooling release containing Tasks 2 and 3 is cut and installed on the host and in every VM (`vrg-vm update --all` installs it).

**Procedure:**
1. Edit the host `~/.claude/settings.json`: `extraKnownMarketplaces.vergil-marketplace.source.ref` from `"main"` to `"v2.1"`.
2. Run `vrg-host plugins`. Expected: it reports `vergil-marketplace` switched, and `claude plugin marketplace list --json` shows `ref: v2.1`.
3. Run `vrg-vm update --all`. Expected: each VM reports the marketplace switched, with no failure.
4. Sweep the local repos: `grep -l extraKnownMarketplaces /Users/pmoore/dev/projects/vergil-project/*/.claude/settings.json`. Expected: no output. Any hit is removed through that repo's normal worktree and PR flow, in favour of the template.

**Acceptance:** the host and every running VM show `ref: v2.1` in `claude plugin marketplace list --json` and the installed version equals the release `v2.1` points at; the sweep is clean. A VM that was stopped is listed in the result comment and switches on its next `vrg-vm start`.

### Task V2: Rollback round trip (validation)

Operational: no PR. Run with `issue-validate`. The tag moves are human acts; the agent observes.

**Precondition self-check:** D2 accepted; Task 5's runbook is published.

**Procedure:**
1. The human picks an earlier release `v2.1.<B>` and runs the runbook's four steps against vergil-claude-plugin.
2. Update the host (`vrg-host plugins`) and the VMs (`vrg-vm update --all`). On the host and every VM, record the installed version (`claude plugin list --json`), and in a freshly started session, the version the session loaded. Confirm that `~/.config/vergil/plugin-dirs.toml` lists no vergil plugin tree on any VM, so that the session loads the installed plugin.
3. The human runs `vrg-promote <latest v2.1.Z>` by the same steps.
4. Repeat step 2 for the forward version.

**Acceptance:** every machine shows `<B>` after step 2 and the latest release after step 4, by both observations. Where the fleet is left afterwards is the human's decision, recorded in the result comment.
