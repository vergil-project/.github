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
  outcome in a response (spec §6). A flagged item's outcome or decision is
  one of `fixed`, `filed`, `deferred`, `accepted`, `declined` or
  `wont-fix`.
- Every item ID, flagged or not, matches `^[A-Za-z0-9]+(\.[A-Za-z0-9]+)*$`
  with at most 16 characters.
- All text the tools handle (item text and recommendations, requests,
  responses, drafts and `--response-text`) is printable ASCII, U+0020 to
  U+007E, plus `\n`, and never contains `MARKER_PREFIX` outside the marker
  line; anything else is refused, never converted (spec §7).
- Every GitHub call runs inside `github.target_org(<the issue's owner>)`,
  whatever the clone's `origin` is.
- No silent failures: every refusal and fatal check is a non-zero exit with
  a message naming the check and, where there is one, the comment ID.

## Review Focus

1. Text holding anything outside printable ASCII (a right-to-left override
   U+202E, a zero-width space U+200B, an em dash, a tab, an ANSI sequence):
   every tool refuses it, naming the character and its offset, and never
   converts it (Tasks 1 to 4).
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
- Create: `src/vergil_tooling/lib/rulings/plain_text.py`
- Test: `tests/vergil_tooling/test_rulings_identity.py`,
  `test_rulings_comments.py`, `test_rulings_marker.py`,
  `test_rulings_exchange.py`, `test_rulings_plain_text.py`

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
    IssueComment, agent: str) -> Marker | None`, `malformed_marker(comment:
    IssueComment, agent: str) -> bool`, `MARKER_PREFIX = "<!-- vrg-ruling"`
  - `check_plain_text(text: str, *, what: str) -> None` (raises
    `RulingError` naming `what`, the character's code point and its offset)
  - `class FatalCheck(RulingError)` with `check: str` and `comment_id: int |
    None`
  - `open_request(stream: list[IssueComment], agent: str) -> int | None`
  - `read_reply(stream: list[IssueComment], *, request_id: int, agent: str,
    human: str, all_ids: frozenset[int]) -> IssueComment | None`
    (`all_ids`: every comment ID on the issue, trusted or not)
  - `check_respond(stream: list[IssueComment], *, request_id: int,
    reply_id: int, agent: str, human: str, all_ids: frozenset[int]) -> int
    | None` (`None`: ready to respond; an ID: the response that already
    landed)

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

def test_fetch_comments_keeps_64_bit_ids(fake_graphql):
    # one node with fullDatabaseId "4300000001" (above 2**31), as a string
    assert fetch_comments("o/r", 7)[0].id == 4300000001

def test_marker_round_trip():
    m = Marker("response", 11, 12)
    c = IssueComment(1, "h-vergil-user", "text\n" + render_marker(m), False)
    assert parse_marker(c, "h-vergil-user") == m

def test_quoted_marker_is_plain_text():
    quoted = render_marker(Marker("request", None, None))
    mid = IssueComment(1, "h-vergil-user", f"findings\n{quoted}\nmore", False)
    fenced = IssueComment(2, "h-vergil-user", f"```\n{quoted}\n```", False)
    two = IssueComment(3, "h-vergil-user", f"{quoted}\n{quoted}", False)
    assert parse_marker(mid, "h-vergil-user") is None
    assert parse_marker(fenced, "h-vergil-user") is None
    assert parse_marker(two, "h-vergil-user") is None

@pytest.mark.parametrize("bad", ["a\tb", "a\rb", "a\x1b[2Kb", "a\u202eb",
                                 "a\u200bb", "a\u3164b", "a\u2014b", "caf\u00e9",
                                 "x\n<!-- VRG-RULING kind=request -->"])
def test_check_plain_text_refuses(bad):
    with pytest.raises(RulingError, match="text"):
        check_plain_text(bad, what="text")

def test_check_plain_text_allows_ascii_and_newline():
    check_plain_text("R1.1: fix it -- see `x`\nR1.2: file\n", what="text")

def test_malformed_marker_is_not_parsed():
    c = IssueComment(1, "h-vergil-user", "x\n<!-- vrg-ruling kind=bogus -->", False)
    assert parse_marker(c, "h-vergil-user") is None
    assert malformed_marker(c, "h-vergil-user") is True

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
    # request 1 by an untrusted author (in all_ids, not in the stream)
    #   -> FatalCheck("request-author", 1)
    # request 1 with a malformed marker -> FatalCheck("marker", 1)
    # request 1, reply 2, a second human comment 3 -> FatalCheck("turn", 3)
    # request 1, reply 2, an agent comment 3 with no response marker
    #   -> FatalCheck("turn", 3)
    # request 1, reply 2, response(request=1, reply=2) -> comment 2

def test_check_respond():
    # request 1, reply 2, then a second human comment 3 -> FatalCheck("turn", 3)
    # reply 2 absent -> FatalCheck("deleted", 2)
    # request 1, reply 2 and nothing after -> None
    # request 1, reply 2, response(request=1, reply=2) by agent -> its id
    # request 1, reply 2, response naming another reply -> FatalCheck("turn")

def test_open_request():
    # empty -> None; request 1 -> 1; request 1, reply 2 -> 1;
    # request 1, reply 2, response(request=1, reply=2) -> None
    # an earlier round's agent comment with a malformed marker, then
    #   request 5 -> 5 (plain text outside the exchange in hand)
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
nodes { fullDatabaseId author { login } body lastEditedAt } } } }` with
`github.graphql`, stripping `app/` from each login; `id` is
`int(fullDatabaseId)` (a `BigInt`, returned as a string, so it matches the
64-bit REST `id`; never `databaseId`, a 32-bit `Int` that current comment IDs
overflow); `edited` is `lastEditedAt is not None`. A node with no author (a deleted account) is
kept with author `""`, so it is never trusted. `parse_issue_ref` accepts
only `^([\w.-]+/[\w.-]+)#([1-9][0-9]*)$`.

- [ ] **Step 5: Implement `marker.py` and `plain_text.py`**

The marker is one line, `<!-- vrg-ruling kind=request -->` or `<!--
vrg-ruling kind=response request=<id> reply=<id> -->`, parsed with a
`fullmatch` on that exact form. A marker counts only when it is the body's
last line (after stripping trailing whitespace) and the only line in the body
that starts with `MARKER_PREFIX`. `parse_marker` returns `None` unless the
comment's author is `agent`, and `None` for any other comment: one with no
marker line, a marker-like line anywhere but the last line, or several
prefix lines. A prefix line in that position is plain text, never a fatal
check. `parse_marker` never raises: a last line that starts with
`MARKER_PREFIX`, in an agent comment with no other prefix line, but does not
match the exact form makes it return `None` and `malformed_marker` return
`True`. Only `exchange.py` turns that into a fatal check, and only for the
comments of the exchange in hand (spec §5.4).

`check_plain_text` refuses any character other than U+0020 to U+007E and
`\n`, and any line containing `MARKER_PREFIX` (case-insensitive), naming
`what`, the code point and the offset.

- [ ] **Step 6: Implement `exchange.py`**

```text
read_reply(stream, request_id, agent, human):
  find request_id in stream          -> absent and in all_ids:
                                          FatalCheck("request-author", request_id)
                                        absent: FatalCheck("deleted", request_id)
  malformed_marker(request)          -> FatalCheck("marker", request_id)
  request.author != agent or its marker kind != request
                                     -> FatalCheck("request-author", request_id)
  request.edited                     -> FatalCheck("edited", request_id)
  after = comments after the request
  none                               -> return None (no reply yet)
  first = after[0]
  first.author != human              -> FatalCheck("turn", first.id)
  first.edited                       -> FatalCheck("edited", first.id)
  rest = comments after first
  malformed_marker(rest[0])          -> FatalCheck("marker", rest[0].id)
  rest is empty, or rest[0] is by agent with a well-formed response marker
  naming (request_id, first.id) and is the only comment in rest
                                     -> return first
  otherwise                          -> FatalCheck("turn", rest[0].id)
check_respond: read_reply must return the comment reply_id (a different
  comment, or None, -> FatalCheck("deleted", reply_id)); then
  nothing after it                   -> None (ready to respond)
  first comment after it is by agent with a well-formed response marker
  naming (request_id, reply_id), and nothing after that
                                     -> that comment's id (already responded)
  anything else after it             -> FatalCheck("turn", that id)
open_request: the latest comment by agent with a request marker that has
  no response marker naming it afterwards, else None; malformed markers are
  ignored here (they belong to no exchange in hand)
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
- Modify: `src/vergil_tooling/lib/github.py` (`post_json_once`'s
  `timeout`)
- Modify: `pyproject.toml` (`[project.scripts]`: `vrg-ruling =
  "vergil_tooling.bin.vrg_ruling:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a `###
  vrg-ruling` entry)
- Test: `tests/vergil_tooling/test_rulings_items.py`,
  `test_rulings_post.py`, `test_vrg_ruling.py`

**Interfaces:**

- Consumes: Task 1's whole interface; `github.post_json_once(endpoint,
  body) -> dict`; `github.GitHubAPIError`.
- Modifies: `github.post_json_once` gains `timeout: float | None = None`,
  passed to `subprocess.run`; a `subprocess.TimeoutExpired` is raised as
  `GitHubAPIError`. Existing callers keep the default.
- Produces:
  - `@dataclass(frozen=True) class Item: id: str; security: bool; text: str
    | None; recommendation: str | None; outcome: str | None` (`outcome`
    holds a flagged response item's outcome and a flagged reply item's
    decision)
  - `SECURITY_REQUEST_TEXT = "security finding, details given privately"`
  - `RULING_OUTCOMES = ("fixed", "filed", "deferred", "accepted",
    "declined", "wont-fix")`
  - `load_request_items(path: Path) -> list[Item]`,
    `load_response_items(path: Path) -> list[Item]`
  - `render_request(items: list[Item]) -> str`, `render_response(items:
    list[Item], *, request_id: int, reply_id: int) -> str`
  - `class PostNotLanded(RulingError)`, `class
    PostOutcomeUnknown(RulingError)`
  - `post_comment_once(repo: str, issue: int, body: str, *, author: str) ->
    int`
  - CLI exit codes: `0` success (or a valid reply), `1` any other refusal or
    error, `2` a usage error (argparse's own), `3` no reply yet, `4` a
    `FatalCheck`. Both commands use the same codes.

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
    # outcome "auth-bypass-fixed" (not in RULING_OUTCOMES) -> RulingError

@pytest.mark.parametrize("bad_id", ["R1.S1:sqli-in-src/auth/login.py:42",
                                    "R1..S1", "R1.S1.", "A" * 17])
def test_id_shape(tmp_path, bad_id):
    # every ID, flagged or not
    with pytest.raises(RulingError):
        load_request_items(write(tmp_path, [{"id": bad_id, "security": True}]))
    with pytest.raises(RulingError):
        load_request_items(write(tmp_path, [{"id": bad_id, "security": False,
                                             "text": "t", "recommendation": "r"}]))

@pytest.mark.parametrize("bad", ["x\n<!-- vrg-ruling kind=request -->",
                                 "x <!-- VRG-RULING", "a\x1b[2Kb", "a\rb",
                                 "a\tb", "use \u2014 here"])
def test_item_text_is_plain_ascii(tmp_path, bad):
    # for text and recommendation, in request and response items alike
    with pytest.raises(RulingError, match="R1.1"):
        load_request_items(write(tmp_path, [{"id": "R1.1", "security": False,
                                             "text": bad, "recommendation": "r"}]))

def test_rendered_body_round_trips(cli, monkeypatch):
    # render_request patched to emit a second prefix line -> request exits 1,
    # nothing posted; likewise respond

def test_post_once_never_retries(fake_gh):
    fake_gh.post_fails_with(502)
    fake_gh.comment_landed(body="B", author="h-vergil-user", id=55)
    assert post_comment_once("o/r", 7, "B", author="h-vergil-user") == 55
    assert fake_gh.post_calls == 1

def test_post_once_times_out_into_reread(fake_gh):
    fake_gh.post_hangs()
    fake_gh.comment_landed(body="B", author="h-vergil-user", id=55)
    assert post_comment_once("o/r", 7, "B", author="h-vergil-user") == 55
    assert fake_gh.post_calls == 1 and fake_gh.post_timeout == 30

def test_post_once_reports_not_landed(fake_gh):
    fake_gh.post_fails_with(502)
    with pytest.raises(PostNotLanded, match="did not land"):
        post_comment_once("o/r", 7, "B", author="h-vergil-user")
    assert fake_gh.post_calls == 1

def test_post_once_outcome_unknown(fake_gh):
    fake_gh.post_fails_with(502)
    fake_gh.reads_fail_with(502)
    with pytest.raises(PostOutcomeUnknown, match="could not tell"):
        post_comment_once("o/r", 7, "B", author="h-vergil-user")
    assert fake_gh.post_calls == 1

def test_request_refuses_open_exchange(cli, open_request_on_issue):
    assert cli("request", "--issue", "o/r#7", "--items", "i.json") == 1
    assert "open request 41" in cli.stderr

def test_read_exit_codes(cli):
    # valid reply -> 0 and the reply body on stdout; none -> 3;
    # FatalCheck -> 4 with the check and comment id on stderr; a missing
    # --request -> 2 (usage), never 4

def test_respond_records_ids(cli):
    # body ends with render_marker(Marker("response", 41, 42))

def test_refuses_in_audit_mode(cli, audit_mode):
    assert cli("read", "--issue", "o/r#7", "--request", "41") == 1

def test_uses_issue_owner_org(cli, origin_is("other/repo"), target_org_spy):
    cli("read", "--issue", "o/r#7", "--request", "41")
    assert target_org_spy.orgs == {"o"}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, `items`, `post` and `vrg_ruling` do not exist.

- [ ] **Step 3: Implement `items.py`**

Items files are JSON arrays. Every item has `id`, matching
`^[A-Za-z0-9]+(\.[A-Za-z0-9]+)*$` with at most 16 characters, and
`security` (bool). A request item, when not flagged, has `text` and
`recommendation` (non-empty strings); a flagged one has nothing else. A
response item has either `text` (unflagged: how the agent read the reply
and what it did) or `outcome` (flagged, one of `RULING_OUTCOMES`). Every
`text` and `recommendation` passes `check_plain_text`. Any other key, a
flagged item with anything else, or any failed check raises `RulingError`
naming the item's ID.
`render_request` writes one line per item (`<id>: <text>
Recommendation: <recommendation>`, or `<id>: security finding, details
given privately`) and ends with `render_marker(Marker("request", None,
None))`; `render_response` likewise, ending with the response marker.

- [ ] **Step 4: Implement `post.py`**

Snapshot the IDs from `fetch_comments` first, then call
`github.post_json_once(f"repos/{repo}/issues/{issue}/comments", {"body":
body}, timeout=30)` once and return its `id`. On `GitHubAPIError`, fetch again, up to
three times two seconds apart: a new comment by `author` whose body equals
`body` is the landed post (return its ID); a successful last re-read
without it raises `PostNotLanded("comment did not land on <repo>#<n>;
nothing more was posted")`. If no re-read succeeded, it raises
`PostOutcomeUnknown("could not tell whether the comment landed on
<repo>#<n>; nothing more was posted; re-run to check")`, carrying the
re-read errors. The POST itself is never repeated.

- [ ] **Step 5: Implement `bin/vrg_ruling.py`**

Subcommands `request --issue --items`, `read --issue --request` and
`respond --issue --request --reply --items`. Each resolves
`agent_login(load_trusted_authors(), identity_mode.current_mode())` first,
then, inside `github.target_org(<the issue's owner>)` for everything that
follows, including `_after_respond`, reads the stream with
`trusted(fetch_comments(...))`. Before posting, `request` and `respond`
parse their own rendered body with `parse_marker` (as the agent's comment)
and refuse unless it gives back the intended marker. `request` refuses
when `open_request` is not `None`, naming it, then posts and prints the new
ID. `read` maps `read_reply`'s result to the exit codes. `respond` runs
`check_respond`: an ID means the response already landed, so it prints that
ID, posts nothing and exits `0`; otherwise it posts. Either way it then
deletes the draft ref (Task 3 adds the delete;
leave the call site as `_after_respond(repo, issue, request_id)`, a no-op
until then).

- [ ] **Step 6: Add the `### vrg-ruling` entry to the CLI Tools Overview**

One paragraph naming the subcommands, with no link: the pages it will link
do not exist until Task 5, and the docs build runs in strict mode in CI
(`ci-docs.yml`), which local `vrg-validate` does not run.

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
  - `fetch_draft(repo: str, issue: int, request_id: int) -> str | None`
    (REST, for the human tool)
  - `delete_draft_api(repo: str, issue: int, request_id: int) -> None`
  - `reply_command(repo: str, issue: int, request_id: int) -> str`
    returning `vrg-ruling-reply --issue <repo>#<issue> --request <id>`
  - in `items.py`: `load_reply_items(path: Path) -> list[Item]` and
    `render_reply(items: list[Item]) -> str`

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
    assert cli("draft", "--issue", "o/r#7", "--request", "41", "--items", "d.json") == 1

def test_reply_items_security_decision_only(tmp_path):
    # {"id": "R1.S1", "security": True, "decision": "fixed"} renders "R1.S1: fixed"
    # {"id": "R1.S1", "security": True, "decision": "fix the SQL in x"}
    #   -> RulingError naming R1.S1; any extra key on a flagged item -> RulingError
    # {"id": "R1.1", "security": False, "text": "file it"} renders "R1.1: file it"
    # text "fix\u00e9", "a\ufe0fb" or "a\u2028b" (not printable ASCII)
    #   -> RulingError naming R1.1; the marker prefix -> RulingError

def test_draft_refuses_in_audit_mode_or_unlisted_login(cli):
    # audit mode -> 1; agent login not on the agents list -> 1; nothing pushed

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

`draft --issue --request --items`: resolves `agent_login` first and runs
inside `github.target_org(<the issue's owner>)`, exactly as Task 2 Step 5's
subcommands do; refuses unless the repo of `origin`'s URL is the issue's
repo, refuses unless the request is still open (`read_reply` returns
`None`), loads the items with `load_reply_items` (the shape of response
items, with `decision` in place of `outcome`: a flagged item carries only
its ID and a decision from `RULING_OUTCOMES`, and every `text` passes
`check_plain_text`), checks the rendered reply with `check_plain_text`
too, pushes `render_reply(items)` (one `<id>: <text or decision>` line per
item), and prints `reply_command(...)`. The agent never pushes free text: the screening
happens before anything is published. Fill `_after_respond` with
`delete_draft_api(repo, issue, request_id)`, addressed by the issue's repo
whatever the clone's `origin` is.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope rulings --message "relay drafted replies through a reserved ref"`

### Task 4: `vrg-ruling-reply`, the human tool

**Files:**

- Create: `src/vergil_tooling/bin/vrg_ruling_reply.py`
- Modify: `pyproject.toml` (`vrg-ruling-reply =
  "vergil_tooling.bin.vrg_ruling_reply:main"`)
- Modify: `docs/site/docs/reference/cli-tools-overview.md` (a `###
  vrg-ruling-reply` entry)
- Test: `tests/vergil_tooling/test_vrg_ruling_reply.py`

**Interfaces:**

- Consumes: Tasks 1 to 3 (including `check_plain_text`);
  `identity_mode.is_agent()`, `confirm.confirm(prompt, *, assume_yes)`,
  `github.read_output`.
- Produces: the `vrg-ruling-reply` command only.

- [ ] **Step 1: Write the failing tests**

```python
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

def test_repushed_draft_is_not_posted(cli, tty, draft("R1.1: fix\n")):
    # the draft ref is pushed again while the prompt waits
    cli.answer("y", then=lambda gh: gh.ref("vergil/ruling-draft/7/41",
                                           blob_text="R1.1: file\n"))
    assert cli("--issue", "o/r#7", "--request", "41") == 0
    assert cli.posted_body == "R1.1: fix\n"
    assert cli.fetch_draft_calls == 1

def test_rechecks_after_confirmation(cli, tty, draft("R1.1: fix\n")):
    # a comment lands on the issue while the prompt waits
    cli.answer("y", then=lambda gh: gh.add_comment(author="h-vergil-user"))
    assert cli("--issue", "o/r#7", "--request", "41") == 1
    assert cli.posted_body is None

def test_declined_keeps_draft(cli, tty, draft("R1.1: fix\n")):
    cli.answer("n")
    assert cli("--issue", "o/r#7", "--request", "41") == 1
    assert fetch_draft("o/r", 7, 41) == "R1.1: fix\n"

@pytest.mark.parametrize("bad", ["a\u3164b", "caf\u00e9", "a\tb"])
def test_refuses_non_ascii_reply(cli, tty, bad):
    # from --response-text and from the draft alike; nothing is posted
    assert cli("--issue", "o/r#7", "--request", "41", "--response-text", bad) == 1
    assert cli.posted_body is None

def test_response_text_also_deletes_draft(cli, tty, draft("x\n")):
    cli.answer("y")
    assert cli("--issue", "o/r#7", "--request", "41", "--response-text", "deny") == 0
    assert fetch_draft("o/r", 7, 41) is None
```

- [ ] **Step 2: Run them to verify they fail**

Run: `vrg-container-run -- vrg-validate`
Expected: FAIL, `vrg_ruling_reply` does not exist.

- [ ] **Step 3: Implement `bin/vrg_ruling_reply.py`**

Arguments: `--issue`, `--request`, optional `--response-text`. In order:
refuse if `identity_mode.is_agent()` (as `vrg-submit-pr` does, before
parsing, so `--help` also redirects); refuse unless stdin and stdout are
TTYs; resolve `human_login`; refuse unless `github.read_output("api",
"user", "--jq", ".login")` equals it; refuse unless `open_request` on the
trusted stream is this request and `read_reply` returns `None`; read the
reply once (`--response-text`, else `fetch_draft`, where `None` is a
refusal); `check_plain_text(reply, what="reply")`; print the reply as it
is, then `bytes: <n>` and `sha256: <hex>`; `confirm("Post this reply as
<login>?", assume_yes=False)`; on yes, fetch the comments again and re-run
the `open_request` and `read_reply` checks, refusing without posting if
either has changed; then `post_comment_once(..., author=human)` with
exactly those bytes, then `delete_draft_api`; on no, exit 1 and keep the
draft. Every GitHub call runs inside `github.target_org(<the issue's
owner>)`.

- [ ] **Step 4: Add the `### vrg-ruling-reply` entry to the CLI Tools Overview**

One paragraph: a human-only command, run in the human's own shell, never
through `!` in an agent session. No link yet (Task 5 adds it).

- [ ] **Step 5: Run the tests to verify they pass**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS.

- [ ] **Step 6: Commit**

`vrg-commit --type feat --scope rulings --message "vrg-ruling-reply: post the human's reply as the human"`

### Task 5: the protocol guide and the `build/tmp/` convention

**Files:**

- Create: `docs/site/docs/guides/human-replies.md`
- Create: `docs/site/docs/reference/dev/ruling.md` and
  `docs/site/docs/reference/dev/ruling-reply.md` (one reference page per
  command: every subcommand, flag and exit code)
- Modify: `docs/site/mkdocs.yml` (nav entries for both pages)
- Modify: `docs/specs/worktree-convention.md` (rule 3: the `build/tmp/`
  subdirectory)
- Modify: `docs/site/docs/guides/git-workflow.md` (name `build/tmp/` beside
  `build/.vergil-state/` where it restates rule 3)
- Test: none locally. The nav and the links are checked only by the strict
  docs build in CI (`ci-docs.yml`), which `vrg-validate` does not run.

**Interfaces:**

- Consumes: the commands and exit codes from Tasks 2 to 4.
- Produces: the page the merge-review epic cites,
  `guides/human-replies.md`.

- [ ] **Step 1: Write `guides/human-replies.md` and the two reference pages**

Sections, from the spec: the round (spec §4), what each command does and
its exit codes, the checks (spec §5: acceptance, turn order, write-once,
deletions, the audit's ceiling), security items (spec §6, including that
the agent drafts the human's reply with security items by ID only and the
human reads it before confirming), the printable-ASCII rule (spec §7), where
the response states the agent's reading (spec §7), the draft relay (spec
§9), posting
exactly once and recovery by a new issue (spec §10), and the assumption
that agent sessions hold only App credentials (spec §3).

- [ ] **Step 2: Add the `build/tmp/` paragraph to the worktree convention's rule 3**

Within the existing `build/` exception: everything under `build/tmp/` is
scratch, and a file there can be deleted once nothing is using it. Add one
clause naming `build/tmp/` to `guides/git-workflow.md`, next to its mention
of `build/.vergil-state/`.

- [ ] **Step 3: Link the CLI Tools Overview entries**

Link the `### vrg-ruling` entry in
`docs/site/docs/reference/cli-tools-overview.md` to `reference/dev/ruling.md`,
the `### vrg-ruling-reply` entry to `reference/dev/ruling-reply.md`, and
both to `guides/human-replies.md`.

- [ ] **Step 4: Add the three pages to `mkdocs.yml`'s nav and run validation**

Run: `vrg-container-run -- vrg-validate`
Expected: PASS (markdownlint). The task is accepted only once the PR's CI
docs check is green.

- [ ] **Step 5: Commit**

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
  deleted after `read`, a trusted comment out of turn); a reply deleted
  before `read` gives exit 3 (no reply yet); a deleted untrusted
  comment is not fatal; and the logins GitHub reports on the posted
  comments match the checks (`h-vergil-user` without `app/`, and the human
  login), and the comment IDs `fetch_comments` reads equal the REST IDs
  `post_comment_once` returned.
- **Documentation review, vergil-project/vergil-tooling#3917,** then **the
  retrospective, vergil-project/.github#891.**

## Security advisories

Each is retargeted to its task once the tasks are filed:

| Advisory | Task |
| --- | --- |
| GHSA-9p3c-62q9-v2cg, GHSA-mv5w-h5vr-wjfm, GHSA-hw7h-g8vr-wj94, GHSA-w29q-p7r4-c8fw | Task 1 |
| GHSA-r5f4-5586-mqfp, GHSA-2fjg-qxrj-fc2v, GHSA-8459-fr73-66rm, GHSA-g8m7-qqg9-mvch, GHSA-gqxr-g9pv-3p2v, GHSA-gm44-f32j-78px, GHSA-r862-g48m-wffg | Task 2 |
| GHSA-mx99-fwcp-2vrh | Task 3 |
| GHSA-wrf8-xm8j-wvw4, GHSA-pjwf-v9g2-4h8f, GHSA-pm85-5hg9-h9w8, GHSA-vw78-jmq7-p52j | Task 4 |
| GHSA-xffj-678w-ggjp | Validation #3919 |
| GHSA-9q3r-h2xj-rgq3 | Task 5 (`guides/human-replies.md` states the assumption in spec §3); the human closes it |
| GHSA-w38c-m89h-5v9q | Task 5 (not applicable here); the human closes it |
