# Authenticated human replies on task issues Implementation Plan

- **Epic:** vergil-project/.github#889
- **Planning issue:** vergil-project/.github#879 (review history is in its
  comments)
- **Status:** draft

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Two vergil-tooling commands, `vrg-ruling` (agent) and
`vrg-ruling-reply` (human), that carry a review decision from the human to
the agent as an authenticated, write-once comment on the task issue.

**Architecture:** A small library package, `vergil_tooling.lib.rulings`,
holds everything both commands share: identity, reading an issue's
comments, the hidden marker, the turn checks, posting exactly once and the
draft relay ref. The two commands are thin CLIs over it. The draft travels
as a reserved git ref on GitHub, built the way
`lib/pr_workflow/github_transport.py` builds the report-ready relay.

**Tech Stack:** Python 3, argparse, the `gh` CLI through
`vergil_tooling.lib.github`, git plumbing through `vergil_tooling.lib.git`,
pytest.

**Spec:** `epics/889-human-auth/spec.md` (in vergil-project/.github).

## Global Constraints

- Every task lands in vergil-project/vergil-tooling. Validate only with
  `vrg-container-run -- vrg-validate`; commit with `vrg-commit`.
- Identity comes only from `load_trusted_authors()`
  (`lib/trusted_authors.py`); strip a leading `app/` from every comment
  author before comparing. No new configuration.
- The `human` entry must hold exactly one login; zero or several is a
  refusal with a clear message (spec §3).
- The agent tool runs only in `IdentityMode.USER`; the expected agent login
  is `<the human login>-vergil-user` and must be on the `agents` list
  (spec §3).
- A comment with any edit (`lastEditedAt` not null) among the request and
  the reply is fatal; so is a missing request or reply, and any trusted
  comment out of turn (spec §5).
- Markers are recognised only in comments by the derived agent login
  (spec §7).
- Every comment is posted exactly once with no retry (spec §10).
- A security item carries only its ID and fixed text in a request
  (`security finding, details given privately`) and only its ID and an
  outcome in a response (spec §6).
- No silent failures: every refusal and fatal check is a non-zero exit with
  a message naming the check and, where there is one, the comment ID.

## Review Focus

1. A draft holding Unicode format characters (a right-to-left override
   U+202E, a zero-width space U+200B) can hide text on screen as well as an
   ANSI sequence can; the human tool must refuse it too (Task 4).
2. An issue with more than 100 comments: every comment must be read, or the
   turn checks see a truncated stream (Task 1).
3. A POST that landed but returned a 502 or timed out: the tool reports the
   landed comment's ID and never posts a second time (Task 2).
4. A comment by the human that contains a marker, or that comes first on an
   issue: it is never treated as a request or response (Task 1).
5. `--issue` naming a repo other than the clone's `origin`: `draft` must
   refuse rather than push the ref to the wrong repo (Task 3).

---

### Task 1: the shared library (identity, comments, marker, turn checks)

**Files:**

- Create: `src/vergil_tooling/lib/rulings/__init__.py`
- Create: `src/vergil_tooling/lib/rulings/identity.py`
- Create: `src/vergil_tooling/lib/rulings/comments.py`
- Create: `src/vergil_tooling/lib/rulings/marker.py`
- Create: `src/vergil_tooling/lib/rulings/exchange.py`
- Test: `tests/vergil_tooling/test_rulings_identity.py`,
  `test_rulings_comments.py`, `test_rulings_marker.py`,
  `test_rulings_exchange.py`

**Interfaces:**

- Consumes: `load_trusted_authors() -> TrustedAuthors`,
  `identity_mode.current_mode() -> IdentityMode`, `github.graphql(query,
  **variables) -> dict`.
- Produces:
  - `class RulingError(Exception)` in `__init__.py`: every refusal.
  - `human_login(authors: TrustedAuthors) -> str`
  - `agent_login(authors: TrustedAuthors, mode: IdentityMode) -> str`
  - `@dataclass(frozen=True) class IssueComment: id: int; author: str;
    body: str; edited: bool`
  - `parse_issue_ref(ref: str) -> tuple[str, int]` (`owner/repo#N` only)
  - `fetch_comments(repo: str, issue: int) -> list[IssueComment]` (oldest
    first, every page)
  - `trusted(comments: list[IssueComment], authors: TrustedAuthors) ->
    list[IssueComment]`
  - `@dataclass(frozen=True) class Marker: kind: Literal["request",
    "response"]; request_id: int | None; reply_id: int | None`
  - `render_marker(marker: Marker) -> str`, `parse_marker(comment:
    IssueComment, agent: str) -> Marker | None`, `MARKER_PREFIX =
    "<!-- vrg-ruling"`
  - `class FatalCheck(RulingError)` with `check: str` and `comment_id: int |
    None`
  - `open_request(stream: list[IssueComment], agent: str) -> int | None`
  - `read_reply(stream: list[IssueComment], *, request_id: int, agent: str,
    human: str) -> IssueComment | None`
  - `check_respond(stream: list[IssueComment], *, request_id: int,
    reply_id: int, agent: str, human: str) -> None`

Here `stream` is always the output of `trusted()`.

- [ ] **Step 1: Write the failing tests**

```python
def test_human_login_requires_exactly_one():
    assert human_login(TrustedAuthors(human=("h",), agents=())) == "h"
    with pytest.raises(RulingError, match="exactly one"):
        human_login(TrustedAuthors(human=(), agents=()))
    with pytest.raises(RulingError, match="exactly one"):
        human_login(TrustedAuthors(human=("a", "b"), agents=()))

def test_agent_login_user_mode_only():
    a = TrustedAuthors(human=("h",), agents=("h-vergil-user",))
    assert agent_login(a, IdentityMode.USER) == "h-vergil-user"
    with pytest.raises(RulingError, match="audit"):
        agent_login(a, IdentityMode.AUDIT)

def test_agent_login_must_be_listed():
    with pytest.raises(RulingError, match="h-vergil-user"):
        agent_login(TrustedAuthors(human=("h",), agents=()), IdentityMode.USER)

def test_fetch_comments_reads_every_page_and_strips_app(fake_graphql):
    # two pages of 100 and 1; one author "app/h-vergil-user"
    comments = fetch_comments("o/r", 7)
    assert len(comments) == 101
    assert comments[0].author == "h-vergil-user"
    assert comments[-1].edited is True   # lastEditedAt set on the last node

def test_marker_round_trip():
    m = Marker("response", 11, 12)
    c = IssueComment(1, "h-vergil-user", "text\n" + render_marker(m), False)
    assert parse_marker(c, "h-vergil-user") == m

def test_marker_ignored_unless_by_agent():
    body = render_marker(Marker("request", None, None))
    assert parse_marker(IssueComment(1, "h", body, False), "h-vergil-user") is None

def test_read_reply_outcomes():
    # request(1, agent) -> None; + reply(2, human) -> comment 2
    # edited request -> FatalCheck(check="edited", comment_id=1)
    # edited reply -> FatalCheck("edited", 2)
    # agent comment(2) before reply -> FatalCheck("turn", 2)
    # request id 1 absent -> FatalCheck("deleted", 1)
    # request 1 posted by the human -> FatalCheck("request-author", 1)

def test_check_respond():
    # request 1, reply 2, then a second human comment 3 -> FatalCheck("turn", 3)
    # reply 2 absent -> FatalCheck("deleted", 2)
    # request 1, reply 2 and nothing after -> passes

def test_open_request():
    # empty -> None; request 1 -> 1; request 1, reply 2 -> 1;
    # request 1, reply 2, response(request=1, reply=2) -> None
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, the `vergil_tooling.lib.rulings` modules do not exist.

- [ ] **Step 3: Implement `identity.py`**

`human_login` returns `authors.human[0]` when the tuple holds exactly one
login, else raises `RulingError("trusted authors must list exactly one human
login; found <n>")`. `agent_login` raises in any mode but `USER`, naming the
mode, then returns `f"{human_login(authors)}-vergil-user"` if it is in
`authors.agents`, else raises naming the missing login.

- [ ] **Step 4: Implement `comments.py`**

`fetch_comments` pages through `repository(owner, name) { issue(number) {
comments(first: 100, after: $cursor) { pageInfo { hasNextPage endCursor }
nodes { databaseId author { login } body lastEditedAt } } } }` with
`github.graphql`, stripping `app/` from each login; `edited` is
`lastEditedAt is not None`. A node with no author (a deleted account) is
kept with author `""`, so it is never trusted. `parse_issue_ref` accepts
only `^([\w.-]+/[\w.-]+)#([1-9][0-9]*)$`.

- [ ] **Step 5: Implement `marker.py`**

The marker is one line, `<!-- vrg-ruling kind=request -->` or `<!--
vrg-ruling kind=response request=<id> reply=<id> -->`, parsed with a
`fullmatch` on that exact form. `parse_marker` returns `None` unless the
comment's author is `agent`, and returns `None` for a comment with no marker
line. A malformed marker line in an agent comment raises
`FatalCheck("marker", comment.id)`.

- [ ] **Step 6: Implement `exchange.py`**

```text
read_reply(stream, request_id, agent, human):
  find request_id in stream          -> absent: FatalCheck("deleted", request_id)
  request.author != agent or its marker kind != request
                                     -> FatalCheck("request-author", request_id)
  request.edited                     -> FatalCheck("edited", request_id)
  after = comments after the request
  none                               -> return None (no reply yet)
  first = after[0]
  first.author != human              -> FatalCheck("turn", first.id)
  first.edited                       -> FatalCheck("edited", first.id)
  return first
check_respond: read_reply must return the comment reply_id (a different
  comment, or None, -> FatalCheck("deleted", reply_id)); any comment after
  it -> FatalCheck("turn", that id)
open_request: the latest comment by agent with a request marker that has
  no response marker naming it afterwards, else None
```

- [ ] **Step 7: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 8: Commit**

`vrg-commit --type feat --scope rulings --message "shared library for authenticated human replies"`

### Task 2: `vrg-ruling request`, `read` and `respond`

**Files:**

- Create: `src/vergil_tooling/lib/rulings/items.py`
- Create: `src/vergil_tooling/lib/rulings/post.py`
- Create: `src/vergil_tooling/bin/vrg_ruling.py`
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-ruling =
  "vergil_tooling.bin.vrg_ruling:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a `###
  vrg-ruling` entry)
- Test: `tests/vergil_tooling/test_rulings_items.py`,
  `test_rulings_post.py`, `test_vrg_ruling.py`

**Interfaces:**

- Consumes: Task 1's whole interface; `github.post_json_once(endpoint,
  body) -> dict`; `github.GitHubAPIError`.
- Produces:
  - `@dataclass(frozen=True) class Item: id: str; security: bool; text: str
    | None; recommendation: str | None`
  - `SECURITY_REQUEST_TEXT = "security finding, details given privately"`
  - `load_request_items(path: Path) -> list[Item]`,
    `load_response_items(path: Path) -> list[Item]`
  - `render_request(items: list[Item]) -> str`, `render_response(items:
    list[Item], *, request_id: int, reply_id: int) -> str`
  - `class PostNotLanded(RulingError)`
  - `post_comment_once(repo: str, issue: int, body: str, *, author: str) ->
    int`
  - CLI exit codes: `0` success (or a valid reply), `3` no reply yet, `2` a
    `FatalCheck`, `1` any other refusal or error.

- [ ] **Step 1: Write the failing tests**

```python
def test_request_items_security_fixed_text_only(tmp_path):
    ok = [{"id": "R1.S1", "security": True}]
    assert render_request(load_request_items(write(tmp_path, ok))).count(
        SECURITY_REQUEST_TEXT) == 1
    bad = [{"id": "R1.S1", "security": True, "text": "SQL injection in x"}]
    with pytest.raises(RulingError, match="R1.S1"):
        load_request_items(write(tmp_path, bad))

def test_request_items_flag_not_id_shape(tmp_path):
    # "S2.3" unflagged with text and recommendation is accepted as ordinary
    ...

def test_response_items_security_outcome_only(tmp_path):
    # {"id": "R1.S1", "security": True, "outcome": "fixed"} accepted
    # outcome "fixed in src/x.py" -> RulingError naming R1.S1
    # any extra key on a flagged item -> RulingError

def test_post_once_never_retries(fake_gh):
    fake_gh.post_fails_with(502)
    fake_gh.comment_landed(body="B", author="h-vergil-user", id=55)
    assert post_comment_once("o/r", 7, "B", author="h-vergil-user") == 55
    assert fake_gh.post_calls == 1

def test_post_once_reports_not_landed(fake_gh):
    fake_gh.post_fails_with(502)
    with pytest.raises(PostNotLanded, match="did not land"):
        post_comment_once("o/r", 7, "B", author="h-vergil-user")
    assert fake_gh.post_calls == 1

def test_request_refuses_open_exchange(cli, open_request_on_issue):
    assert cli("request", "--issue", "o/r#7", "--items", "i.json") == 1
    assert "open request 41" in cli.stderr

def test_read_exit_codes(cli):
    # valid reply -> 0 and the reply body on stdout; none -> 3;
    # FatalCheck -> 2 with the check and comment id on stderr

def test_respond_records_ids(cli):
    # body ends with render_marker(Marker("response", 41, 42))

def test_refuses_in_audit_mode(cli, audit_mode):
    assert cli("read", "--issue", "o/r#7", "--request", "41") == 1
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, `items`, `post` and `vrg_ruling` do not exist.

- [ ] **Step 3: Implement `items.py`**

Items files are JSON arrays. A request item has `id` (non-empty, no
whitespace), `security` (bool) and, when not flagged, `text` and
`recommendation` (non-empty strings); a flagged item has `id` and `security`
only. A response item has `id`, `security`, and either `text` (unflagged) or
`outcome` (flagged, matching `^[a-z][a-z-]{0,29}$`). Any other key, or a
flagged item with anything else, raises `RulingError` naming the item's ID.
`render_request` writes one line per item (`<id>: <text>
Recommendation: <recommendation>`, or `<id>: security finding, details
given privately`) and ends with `render_marker(Marker("request", None,
None))`; `render_response` likewise, ending with the response marker.

- [ ] **Step 4: Implement `post.py`**

Snapshot the IDs from `fetch_comments` first, then call
`github.post_json_once(f"repos/{repo}/issues/{issue}/comments", {"body":
body})` once and return its `id`. On `GitHubAPIError`, fetch again: a new
comment by `author` whose body equals `body` is the landed post (return its
ID); none raises `PostNotLanded("comment did not land on <repo>#<n>; nothing
more was posted")`.

- [ ] **Step 5: Implement `bin/vrg_ruling.py`**

Subcommands `request --issue --items`, `read --issue --request` and
`respond --issue --request --reply --items`. Each resolves
`agent_login(load_trusted_authors(), identity_mode.current_mode())` first,
then reads the stream with `trusted(fetch_comments(...))`. `request` refuses
when `open_request` is not `None`, naming it, then posts and prints the new
ID. `read` maps `read_reply`'s result to the exit codes. `respond` runs
`check_respond`, posts, then deletes the draft ref (Task 3 adds the delete;
leave the call site as `_after_respond(repo, issue, request_id)`, a no-op
until then).

- [ ] **Step 6: Add the `### vrg-ruling` entry to the CLI Tools Overview**

One paragraph naming the subcommands and linking the protocol guide
(Task 5's `guides/human-replies.md`).

- [ ] **Step 7: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS, including `test_cli_tools_overview.py` and
`test_help_coverage.py`.

- [ ] **Step 8: Commit**

`vrg-commit --type feat --scope rulings --message "vrg-ruling request, read and respond"`

### Task 3: the draft relay ref and `vrg-ruling draft`

**Files:**

- Create: `src/vergil_tooling/lib/rulings/draft_ref.py`
- Modify: `src/vergil_tooling/bin/vrg_ruling.py` (the `draft` subcommand and
  `_after_respond`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (`draft` in the
  entry)
- Test: `tests/vergil_tooling/test_rulings_draft_ref.py`, additions to
  `test_vrg_ruling.py`

**Interfaces:**

- Consumes: Task 1 and Task 2; `git.read_output`, `git.read_output_stdin`,
  `git.run`; `github.read_json`, `github.delete_if_exists`.
- Produces:
  - `draft_ref(issue: int, request_id: int) -> str` returning
    `refs/vergil/ruling-draft/<issue>/<request_id>`
  - `push_draft(issue: int, request_id: int, text: str, *, remote: str =
    "origin") -> None`
  - `delete_draft(issue: int, request_id: int, *, remote: str = "origin")
    -> None`
  - `fetch_draft(repo: str, issue: int, request_id: int) -> str | None`
    (REST, for the human tool)
  - `delete_draft_api(repo: str, issue: int, request_id: int) -> None`
  - `reply_command(repo: str, issue: int, request_id: int) -> str`
    returning `vrg-ruling-reply --issue <repo>#<issue> --request <id>`

- [ ] **Step 1: Write the failing tests**

```python
def test_push_draft_is_plumbing_only(git_repo_with_remote):
    push_draft(7, 41, "fix R1.1\n")
    assert git_repo_with_remote.head_unchanged()
    assert git_repo_with_remote.status_clean()
    assert git_repo_with_remote.remote_file(draft_ref(7, 41), "reply.md") == "fix R1.1\n"

def test_fetch_draft_via_rest(fake_gh):
    fake_gh.ref("vergil/ruling-draft/7/41", blob_text="fix R1.1\n")
    assert fetch_draft("o/r", 7, 41) == "fix R1.1\n"
    assert fetch_draft("o/r", 7, 99) is None

def test_draft_refuses_other_repo(cli, origin_is("o/other")):
    assert cli("draft", "--issue", "o/r#7", "--request", "41", "--reply-file", "r.md") == 1

def test_draft_refuses_answered_request(cli):
    # read_reply returns a reply -> 1, nothing pushed

def test_draft_prints_reply_command(cli):
    assert "vrg-ruling-reply --issue o/r#7 --request 41" in cli.stdout

def test_respond_deletes_leftover_draft(cli):
    # after respond, the ref is gone; absent ref is not an error
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, `draft_ref` does not exist.

- [ ] **Step 3: Implement `draft_ref.py`**

`push_draft` follows `GitHubTransport.write` exactly (`hash-object -w
--stdin`, `mktree` with one `100644 blob <sha>\treply.md` entry,
`commit-tree -m "vergil: ruling draft"`, `push --force <remote>
<commit>:<ref>` with stdout redirected to stderr) and never touches HEAD,
the index or the working tree. `fetch_draft` reads
`repos/<repo>/git/ref/vergil/ruling-draft/<issue>/<id>` (a 404 is `None`),
then the commit, its tree, the `reply.md` entry's blob, and decodes the
blob's base64 content as UTF-8 (invalid UTF-8 raises `RulingError`).
`delete_draft_api` calls `github.delete_if_exists` on
`repos/<repo>/git/refs/vergil/ruling-draft/<issue>/<id>`.

- [ ] **Step 4: Implement the `draft` subcommand**

`draft --issue --request --reply-file`: refuses unless the repo of
`origin`'s URL is the issue's repo, refuses unless the request is still
open (`read_reply` returns `None`), pushes the file's text, and prints
`reply_command(...)`. Fill `_after_respond` with `delete_draft`.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope rulings --message "relay drafted replies through a reserved ref"`

### Task 4: `vrg-ruling-reply`, the human tool

**Files:**

- Create: `src/vergil_tooling/lib/rulings/plain_text.py`
- Create: `src/vergil_tooling/bin/vrg_ruling_reply.py`
- Modify: `pyproject.toml` (`vrg-ruling-reply =
  "vergil_tooling.bin.vrg_ruling_reply:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a `###
  vrg-ruling-reply` entry)
- Test: `tests/vergil_tooling/test_rulings_plain_text.py`,
  `test_vrg_ruling_reply.py`

**Interfaces:**

- Consumes: Tasks 1 to 3; `identity_mode.is_agent()`,
  `confirm.confirm(prompt, *, assume_yes)`, `github.read_output`.
- Produces:
  - `check_plain_text(text: str) -> None` (raises `RulingError`)
  - `render_visible(text: str) -> str`

- [ ] **Step 1: Write the failing tests**

```python
@pytest.mark.parametrize("bad", ["a\rb", "a\x1b[2Kb", "a‮b", "a​b",
                                  "x <!-- vrg-ruling kind=request -->"])
def test_check_plain_text_refuses(bad):
    with pytest.raises(RulingError):
        check_plain_text(bad)

def test_check_plain_text_allows_newline_and_tab():
    check_plain_text("R1.1: fix\n\tR1.2: file\n")

def test_render_visible_shows_tabs():
    assert render_visible("a\tb") == "a→b"

def test_refuses_in_agent_mode(cli, agent_mode):
    assert cli("--issue", "o/r#7", "--request", "41") == 1

def test_refuses_wrong_login(cli, gh_user("someone")):
    assert cli("--issue", "o/r#7", "--request", "41") == 1

def test_refuses_without_tty(cli, no_tty):
    assert cli("--issue", "o/r#7", "--request", "41") == 1

def test_refuses_when_request_not_latest(cli, request_answered):
    assert cli("--issue", "o/r#7", "--request", "41") == 1

def test_posts_exactly_what_was_shown(cli, tty, draft("R1.1: fix\n")):
    cli.answer("y")
    assert cli("--issue", "o/r#7", "--request", "41") == 0
    assert cli.posted_body == "R1.1: fix\n"
    assert "bytes: 10" in cli.stdout and "sha256:" in cli.stdout
    assert fetch_draft("o/r", 7, 41) is None

def test_declined_keeps_draft(cli, tty, draft("R1.1: fix\n")):
    cli.answer("n")
    assert cli("--issue", "o/r#7", "--request", "41") == 1
    assert fetch_draft("o/r", 7, 41) == "R1.1: fix\n"

def test_response_text_also_deletes_draft(cli, tty, draft("x\n")):
    cli.answer("y")
    assert cli("--issue", "o/r#7", "--request", "41", "--response-text", "deny") == 0
    assert fetch_draft("o/r", 7, 41) is None
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, `plain_text` and `vrg_ruling_reply` do not exist.

- [ ] **Step 3: Implement `plain_text.py`**

`check_plain_text` refuses any character whose `unicodedata.category`
starts with `C` (`Cc`, `Cf`, `Cs`, `Co`, `Cn`) other than `\n` and `\t`,
naming its code point and offset, and refuses any line containing
`MARKER_PREFIX` (case-insensitive). `render_visible` replaces each tab with
`→`; nothing else needs showing once the check has passed.

- [ ] **Step 4: Implement `bin/vrg_ruling_reply.py`**

Arguments: `--issue`, `--request`, optional `--response-text`. In order:
refuse if `identity_mode.is_agent()` (as `vrg-submit-pr` does, before
parsing, so `--help` also redirects); refuse unless stdin and stdout are
TTYs; resolve `human_login`; refuse unless `github.read_output("api",
"user", "--jq", ".login")` equals it; refuse unless `open_request` on the
trusted stream is this request and `read_reply` returns `None`; read the
reply once (`--response-text`, else `fetch_draft`, where `None` is a
refusal); `check_plain_text`; print `render_visible(reply)`, `bytes: <n>`
and `sha256: <hex>`; `confirm("Post this reply as <login>?",
assume_yes=False)`; on yes, `post_comment_once(..., author=human)` with
exactly those bytes, then `delete_draft_api`; on no, exit 1 and keep the
draft.

- [ ] **Step 5: Add the `### vrg-ruling-reply` entry to the CLI Tools Overview**

One paragraph: a human-only command, run in the human's own shell, never
through `!` in an agent session.

- [ ] **Step 6: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 7: Commit**

`vrg-commit --type feat --scope rulings --message "vrg-ruling-reply: post the human's reply as the human"`

### Task 5: the protocol guide and the `build/tmp/` convention

**Files:**

- Create: `docs/site/docs/guides/human-replies.md`
- Create: `docs/site/docs/reference/dev/ruling.md` (the reference page for
  both commands: every subcommand, flag and exit code)
- Modify: `docs/site/mkdocs.yml` (nav entries for both pages)
- Modify: `docs/specs/worktree-convention.md` (rule 3: the `build/tmp/`
  subdirectory)
- Test: `tests/vergil_tooling/test_docs.py` (whatever it already checks for
  nav coverage; no new test)

**Interfaces:**

- Consumes: the commands and exit codes from Tasks 2 to 4.
- Produces: the page the merge-review epic cites,
  `guides/human-replies.md`.

- [ ] **Step 1: Write `guides/human-replies.md`**

Sections, from the spec: the round (spec §4), what each command does and
its exit codes, the checks (spec §5: acceptance, turn order, write-once,
deletions, the audit's ceiling), security items (spec §6, including that
the agent drafts the human's reply with security items by ID only and the
human reads it before confirming), the draft relay (spec §9), posting
exactly once and recovery by a new issue (spec §10), and the assumption
that agent sessions hold only App credentials (spec §3).

- [ ] **Step 2: Add the `build/tmp/` paragraph to the worktree convention's rule 3**

Within the existing `build/` exception: everything under `build/tmp/` is
scratch, and a file there can be deleted once nothing is using it.

- [ ] **Step 3: Add the guide to `mkdocs.yml`'s nav and run validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (markdownlint and the docs checks).

- [ ] **Step 4: Commit**

`vrg-commit --type docs --scope rulings --message "human replies protocol guide and the build/tmp convention"`

## Operational and closing tasks

These already exist under the epic; the plan fixes their dependencies and
content.

- **Deployment, vergil-project/vergil-tooling#3918:** blocked by Tasks 1 to
  5. Releasing vergil-tooling is a human-gated precondition; the task then
  confirms `vrg-ruling --help` and `vrg-ruling-reply --help` run in an agent
  VM and on the host.
- **Validation, vergil-project/vergil-tooling#3919:** blocked by #3918. On a
  throwaway issue, with a canned finding posted by hand in place of a
  review, it runs spec §12's cases: a full round from a Lima VM and from a
  cloud VM; a draft pushed again after it was shown is not what is posted;
  each fatal case (edited request, edited reply, deleted request, reply
  deleted after `read`, a trusted comment out of turn); a deleted untrusted
  comment is not fatal; and the logins GitHub reports on the posted
  comments match the checks (`h-vergil-user` without `app/`, and the human
  login).
- **Documentation review, vergil-project/vergil-tooling#3917,** then **the
  retrospective, vergil-project/.github#891.**

## Security advisories

Each is retargeted to its task once the tasks are filed:

| Advisory | Task |
| --- | --- |
| GHSA-9p3c-62q9-v2cg, GHSA-mv5w-h5vr-wjfm | Task 1 |
| GHSA-r5f4-5586-mqfp, GHSA-2fjg-qxrj-fc2v, GHSA-8459-fr73-66rm, GHSA-g8m7-qqg9-mvch | Task 2 |
| GHSA-wrf8-xm8j-wvw4, GHSA-pjwf-v9g2-4h8f | Task 4 |
| GHSA-xffj-678w-ggjp | Validation #3919 |
| GHSA-9q3r-h2xj-rgq3 | the docs task (the stated assumption in spec §3); the human closes it |
| GHSA-w38c-m89h-5v9q | the docs task (not applicable here); the human closes it |
