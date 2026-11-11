# Feedback for paad: pushback against impossible or unrealistic scenarios

A note for the paad author (`Ovid/paad`), who asked for feedback. It
proposes a guideline for `paad:pushback`, with a worked example from the
design review of epic vergil-project/.github#1019 (planning issue
vergil-project/.github#1017), and a related pattern seen in test suites.
Ready to send as written. The skill text quoted below is the pushback
`SKILL.md` as installed in November 2026.

## The gap

Pushback already insists that every finding be a claim you can defend:
"If the spec ships as written, **Y** happens, because **Z**", with Z
resting on a file and line, a command, a commit or a concrete step. That
rule cuts vague findings well. It does not cut a finding whose Y is
concrete and whose Z is a real mechanism, but whose failure cannot
actually happen, or is already caught by something in place. Those
findings pass the Y/Z test and still push the design the wrong way: each
proposes a new check, guard or mechanism, and the spec grows machinery
that defends against nothing.

The same blind spot runs the other way. When the spec itself proposes a
guard against an impossible failure, pushback reviews the guard's edge
cases rather than asking whether it is needed at all.

## Proposed guideline

Before a finding asks for a new check, guard or mechanism, answer two
questions and put the answers in the finding:

1. **Can the failure happen?** Name the sequence of real events that
   produces it. If the only path is something the tools guarantee
   cannot happen, the failure is impossible: drop the finding and count
   it as a discard.
2. **Is it already caught?** Name what is already in place: the tool's
   own guarantees, an existing check, a validation gate, CI. If one of
   them already catches the failure, loudly and before harm is done, the
   finding says so and proposes no machinery. It may still be worth one
   line if the existing catch is late or confusing.

A failure that can happen only when a human deliberately works against
the tooling (hand-editing a file the tooling wrote, for example) is not
impossible, but it needs a stated reason before it earns machinery: what
it costs when it happens, and whether it already fails loudly.

And the reverse check: when the spec proposes a guard, apply the same
two questions to the guard. A guard against an impossible or
already-caught failure is itself a finding, with deletion as the
recommended option.

This fits the existing text in three places:

- **"Every finding must be a claim you can defend."** Add a third
  condition beside Y and Z: the triggering path is real and nothing in
  place already stops it.
- **"Name what would change your mind."** The existing catch is often
  exactly that fact; naming it there would dissolve the finding before
  it is presented.
- **Common Mistakes.** A new row: "Proposing a guard against a failure
  that cannot happen, or that an existing check already catches → Name
  the triggering path and the existing catch; if either is missing,
  drop it. If the spec proposes such a guard, recommend deleting it."

## Worked example: the tree-equality check

Epic #1019 batches several small issues into one PR, keeping one commit
per issue. Each issue's later commits are fixups of its first commit,
folded with a non-interactive `rebase --autosquash` before the next
issue starts. The first draft of the spec (§5.2, step 4) said:

> The folded commit's tree must equal the tree that was validated and
> reviewed, so neither is invalidated. The tooling checks this, not the
> agent.

The check defends against an impossible failure. Folding an issue's
fixups into that issue's own first commit, at the tip of the branch,
reorders nothing and drops nothing, so git leaves the final tree
unchanged. And validation and PR CI run on the final branch anyway, so
even a changed tree would be caught by the existing gates.

Spec pushback round 1 did not ask whether the check was needed. It
raised a finding about the check's edge case instead (S1.2, serious,
high): a fix after the PR opens, to an earlier issue's commit, rewrites
every later commit in the stack, and the tree-equality check could not
handle that. That treats ordinary git behaviour, rewriting the commits
that follow a changed one, as a defect. Both were resolved by deleting
the check: rewriting later commits is normal, and validation and PR CI
cover the final branch.

Under the proposed guideline the reverse check would have flagged the
tree-equality check in round 1 as a guard against an impossible failure,
and S1.2 would not have been raised.

### Two related findings from the same review, both rejected

- **A pre-existing conflict case (S1.1, repeated as S2.1).** Finalize's
  documented conflict remedy merges `develop` into the PR, which would
  put a merge commit on a batch branch. Rejected: finalize's existing
  conflict guard already handles the case as designed, and the case
  exists independently of batching. Question 2 ("is it already
  caught?") answers it.
- **A human hand-editing a PR body (S2.4).** Finalize identifies a batch
  PR from the PR body, which a human could edit. Rejected: the only path
  is a human hand-editing a body that `vrg-submit-pr` wrote, and the
  guard adds complexity with no real failure behind it. Round 1 had
  already discarded the same candidate, noting that such an edit fails
  loudly on the commit-per-issue check. The deliberate-misuse paragraph
  above covers it.

The full round records are the comments on vergil-project/.github#1017.

## A related pattern: tests that only freeze history

The same reflex, defending against something that will not happen,
shows up in test suites. A rename or removal lands, and with it a
permanent unit test, written in regression-test style, asserting that
the old name, file or line never comes back. It does not guard against
a bug recurring: nothing in the code can bring the old thing back except
someone writing it again on purpose. These tests pile up as cruft. Each
one is read, kept passing and reasoned about long after anyone remembers
the rename. If such a check is worth having at all, it should carry an
expiry date rather than live forever in the suite.

The fleet's suites have concrete examples. The clearest is in
`vergil-tooling`
([`test_repo_config.py`, lines 329 to 331](https://github.com/vergil-project/vergil-tooling/blob/f961c8fbe259dab9350023c9ba54dfc0395790c2/tests/vergil_tooling/test_repo_config.py#L329-L331)):

```python
def test_previous_template_files_are_gone(self) -> None:
    assert not (_TEMPLATE_PATH.parent / "claude_md_consumer_previous.md").exists()
    assert not (_TEMPLATE_PATH.parent / "claude_settings_previous.json").exists()
```

Commit `0a391f319b` (vergil-project/vergil-tooling#3797) deleted those
two data files and added this test in the same change. It checks that
files are absent from the source tree, not any behaviour, so it can only
fail if someone recreates them by hand.

Two more of the same kind:

- [`test_update_deps_pr.py`, line 61](https://github.com/vergil-project/vergil-tooling/blob/f961c8fbe259dab9350023c9ba54dfc0395790c2/tests/vergil_tooling/test_update_deps_pr.py#L61)
  asserts `"#1379" not in body`. The dependency-update PR body used to
  end with a hard-coded `Ref #1379`. Commit `15ebef912b`
  (vergil-project/vergil-tooling#3265) removed it and added the
  assertion. The other asserts in that test check real behaviour; this
  one freezes the removal of one issue number.
- [`test_vrg_commit.py`, lines 218 to 234](https://github.com/vergil-project/vergil-tooling/blob/f961c8fbe259dab9350023c9ba54dfc0395790c2/tests/vergil_tooling/test_vrg_commit.py#L218-L234),
  `test_stale_vrg_co_author_env_var_is_ignored`, sets the retired
  `VRG_CO_AUTHOR` environment variable and checks that no trailer is
  stamped. No code in `src/` reads that variable any more, so the test
  exercises nothing that could regress. Its "no `Co-Authored-By`
  trailer" assertion does encode a live policy, but a test without the
  retired variable would cover that.

The distinction matters. Several nearby tests look the same but guard a
real failure: one keeps out a retired scanner that gave a false
all-clear for one package manager, and another keeps out retired CI
check names whose return would deadlock merges. Those earn their place.
The test is the same two questions as above: can the failure happen,
and does something already catch it?

Pushback reads the codebase, so it could apply this to plans as well:
when a plan's task adds a "make sure the old X is gone" test after a
rename, the finding is that the test guards nothing, with "delete it"
or "give it an expiry" as the options.
