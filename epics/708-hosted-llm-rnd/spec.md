# Hosted LLM API R&D: provider-neutral experiments for agentic automation

- **Epic:** vergil-project/.github#708
- **Planning issue:** vergil-project/.github#705 (review history is in its
  comments; the pricing research of 2026-10-29 is in its body)
- **Status:** draft
- **Repos:** vergil-tooling (the model layer, the experiments and the site
  docs), .github (the findings report)
- **Related, out of scope:** vergil-project/.github#57 (autonomous
  queue-processing agents), vergil-project/.github#470 (unattended merge),
  vergil-project/.github#579 (issue content as automation input), the Forgejo
  migration (vergil-project/vergil-tooling#1521, #1653)

## 1. Why

All of the fleet's AI work today happens in interactive Claude Code sessions
driven by the human. The long-term goal is automated implementation after
planning: work is designed and broken into tasks with expensive models, then
picked up and carried out without a human starting each session.

The earlier plan was to run open-weight models on a leased Mac Studio, so that
automated work would cost nothing per token. That hardware is no longer
available, and buying it is not affordable for at least a year, perhaps ever.
Several existing specs still assume a local Ollama will arrive (listed in the
planning issue).

Meanwhile the hosted API market has changed. Ollama Cloud moved to per-token
billing on 2026-08-31 and offers open-weight models at low rates. Claude API
prices have also fallen: Haiku 5.5 costs $0.10 per million input tokens and
$0.50 per million output tokens. A monthly budget of about $400–500, the money
once set aside for the Mac Studio, may now buy a useful amount of automated
work. Nobody has measured that yet.

This epic is R&D. Its product is understanding: what hosted models can and
can't do for this fleet, at what cost, given which inputs. That understanding
feeds a later, larger design of the queue-processing engine (bookend
vergil-project/.github#710). In the human's words, this epic studies the
engine before anyone designs the plane.

## 2. Goals

1. **A provider-neutral model layer** that sends a request to a named model
   profile, such as `cheap`, `mid` or `design`, and returns the result plus a
   usage record. Which provider and model sit behind a profile is
   configuration. Changing one setting moves a job between Ollama Cloud and
   the Anthropic API.
2. **A ladder of experiments**, smallest first, that tests ways of using the
   API (section 5.4). Each experiment records:
   - the inputs the model needed
   - which models handled the task and which failed
   - the real token counts and cost
   - the time taken
   - what broke
3. **Headless Claude Code on a non-Anthropic model.** Find out whether
   Claude Code can run headless in a fleet container, pointed at an Ollama
   Cloud model or at Haiku, and carry out a small, bounded task with the
   fleet's existing skills and guardrails.
4. **A findings report** in `epics/708-hosted-llm-rnd/findings.md`. It keeps
   data apart from judgment and covers:
   - what each pattern can and can't do
   - real cost and token profiles per job type
   - operational limits (concurrency, queueing, rate limits, outages, terms)
   - a recommended model-routing policy (which kind of work goes to which
     tier)
   - which subscription plan, if any, the data supports
   - what all of this means for the engine design
5. **Specs brought up to date** so they no longer assume local inference
   hardware (the documentation-review bookend).

## 3. Non-goals

- Building the queue-processing engine, its trigger model or its queue. That
  design belongs to bookend #710.
- Turning on any unattended automation in the live fleet. Experiments run on
  demand, started by a human or an agent in a session.
- Unattended merge or PR submission. PR submission and merge stay human
  actions.
- The Forgejo migration.
- Buying or leasing hardware, and running models locally.
- Choosing a long-term subscription plan before the data exists.
- Diogenes and other projects outside the fleet. The findings may inform them
  later.

## 4. Anti-goals

The epic fails, even if everything technically works, if:

- **Code becomes tied to one provider.** Any job that hard-codes a provider or
  model name outside configuration is a defect. Provider neutrality is a hard
  requirement.
- **Failures are hidden.** A provider error, a malformed structured output or
  a budget stop must surface as an error. There must be no silent fallback to
  another model, no swallowed exception and no default value standing in for
  a failed call.
- **Spending is unbounded.** No experiment runs without a spend cap set
  beforehand, and no run can overshoot its cap by more than one request.
- **Findings come from impressions.** Any claim in the report about cost or
  capability must trace to recorded runs. Judgment is labelled as judgment.
- **Private or secret content is sent to a third-party model.** Experiments
  use public fleet repositories and issues only, and credentials never appear
  in prompts, logs or committed records.
- **The ladder is skipped.** Starting at `issue-implement` because it is the
  goal, before the lower rungs show what inputs a model needs, defeats the
  point of the epic.

## 5. Design

### 5.1 Two layers

Using an LLM API for agentic work needs two things:

- **A model layer** that talks to providers.
- **An agent runtime** that gives a model tools: files, commands,
  worktrees and the `vrg-*` wrappers.

Rungs 1 and 2 of the ladder use the model layer directly, with our own code
as the runtime. Rungs 3 and 4 use Claude Code as the runtime, pointed at a
chosen model. Provider neutrality must hold in both cases: the library's
profiles for rungs 1–2, and the endpoint and model settings for rungs 3–4.

### 5.2 The model layer

A Python library in `vergil-tooling`, because that is where the fleet's
Python tooling lives.

- **Profiles.** A profile names a provider, a model ID and settings.
  Profiles are defined in configuration, not code. A job asks for a profile;
  an experiment can override it to sweep several models.
- **Providers.** The first two are:
  - the Anthropic Messages API
  - Ollama Cloud, through its native API or its OpenAI- or
    Anthropic-compatible endpoint, whichever proves most reliable

  A third provider (for example, a local Ollama) must be addable without
  changing callers.
- **Structured output.** A call can supply a JSON schema. The layer
  validates the reply against it and raises an error on failure. It never
  repairs the reply silently. Whether it retries once on invalid output is
  configurable, and every retry is recorded.
- **Tool calls.** The layer exposes the provider's tool-calling interface in
  one neutral form, which rung 2 needs.
- **Usage records.** Every call produces a record containing:
  - profile, provider and model
  - input, cached-input and output tokens
  - computed cost
  - latency
  - outcome (ok, provider error, schema failure, budget stop)

  Cost is computed from a price table kept in configuration and dated, with
  each rate citing its source page. Where the provider reports its own cost,
  both values are recorded, and any difference is itself a finding.
- **Spend guard.** A run is given a budget. The layer adds up the cost of
  each call and refuses the next call once the budget is reached, raising a
  budget-stop error. The guard runs on our side and does not depend on any
  provider feature. Provider-side spend limits are also used wherever a
  provider offers them.
- **Credentials.** API keys are read from environment variables. They are
  delivered the way the fleet already delivers tokens to containers and VMs,
  and are never written to a repository, a prompt or a usage record.

### 5.3 Headless Claude Code on another model

For rungs 3 and 4, Claude Code runs non-interactively inside one of the
fleet's dev containers. It is configured to send its model requests to a
chosen endpoint and model: Ollama Cloud's Anthropic-compatible endpoint, or
the Anthropic API with a Haiku or Sonnet model.

How to point Claude Code at another endpoint, and which settings it honours,
has not been verified. Establishing that is the first step of rung 3, and the
answer is a finding.

Headless runs:

- use the same identity, wrappers and hook guards as an interactive agent
  session
- run under the same rule that agents never submit or merge PRs
- end in a branch, and at most a `report-ready` handoff

### 5.4 The experiment ladder

Each rung is a set of experiments. The next rung starts only once the current
rung's results are recorded. The human may stop the climb at any rung, and
reporting where the climb stopped is a valid finding.

1. **Single structured-output call.** A Python utility sends a small prompt
   with fleet context and gets back JSON that our code acts on. The model
   never touches the system. Candidate tasks (the builder picks two or three
   with real, checkable answers):
   - suggest an intake route for a past intake item, compared with how the
     human actually routed it
   - classify a deprecation warning
   - draft a PR summary or release-note line from a diff, compared with what
     the human actually wrote
2. **Narrow tool loop.** A Python loop gives the model a few tools that our
   code controls, such as read a file, list files, propose an edit and run a
   validation. The model uses them to finish a small, mechanical change, for
   example fixing a lint finding or a broken doc link in a scratch clone.
   Success is judged mechanically: the change applies and validation passes.
3. **Headless Claude Code, small task.** First establish the configuration
   from section 5.3. Then give the runtime a small, well-bounded fleet task:
   one skill step or one tiny ad-hoc-sized change on a scratch branch.
   Compare cheap models against Haiku and Sonnet on the same task.
4. **`issue-implement`.** Run it only if rung 3 shows it is realistic. Take a
   few real, already-closed ad-hoc tasks and replay them from their original
   issue text on a scratch branch. Compare the result with the change the
   human-driven session actually merged.

For every task at every rung, the sweep covers at least one Claude model and
at least two Ollama Cloud open-weight models, so the report can compare
across providers. Off-peak pricing is noted wherever a model offers it.

Repeat runs: a task runs more than once per model wherever the budget allows,
because single runs hide variance. How many times is the builder's call
within the spend cap.

### 5.5 Recording and the findings report

- Every experiment run writes its usage records and outcome to a
  machine-readable results file. The report's figures are computed from
  those files, not typed in by hand.
- The experiments' code and results live in `vergil-tooling`, outside the
  shipped package. `.github` holds documents only.
- `findings.md` in this epic's directory is the epic's main output. It keeps
  data and judgment in labelled sections, cites the results files, and closes
  with recommendations and open questions for bookend #710.

### 5.6 Terms of service

Section 4 of Ollama's terms forbids accessing the service by "automated
means… without permission", yet Ollama markets the API "for your own tools".
Before any automated Ollama Cloud run, the human asks Ollama in writing
whether scripted, unattended API use on a paid or pay-as-you-go account is
permitted, and records the answer on the planning issue. This is a human
action and a precondition, not an agent step.

## 6. Constraints

- **Accounts.** Experiments use pay-as-you-go access on both providers. No
  subscription is bought during the epic; choosing a plan is a finding.
- **Budget.** The fleet's running budget for automated work is about
  $400–500 a month. The epic's total experiment budget is $100. Each rung's
  cap comes out of that total and is set before the rung's first run.
- **Data.** Only public fleet repositories and issues go into prompts.
- **Engineering principles.** The fleet's rules apply in full: no silent
  failures, `vrg-git`/`vrg-gh` only, and validation through
  `vrg-container-run -- vrg-validate`.
- **Prices change.** Every cost figure in the report carries the date of the
  price table it used.

## 7. Delivery

The plan turns this into tasks. In outline:

1. The model layer, with its configuration, usage records and spend guard,
   in `vergil-tooling`.
2. Rung 1 experiments and their results.
3. Rung 2 experiments and their results.
4. Rung 3: the headless Claude Code configuration, then the experiments.
5. Rung 4, conditional on rung 3's results.
6. The findings report in `.github`.
7. The bookends: the documentation review (vergil-project/vergil-tooling#3810),
   the follow-on brainstorm (vergil-project/.github#710) and the
   retrospective (vergil-project/.github#711).

The terms confirmation (section 5.6) blocks the first Ollama Cloud run.

## 8. Left to the builder

- The module layout, configuration format and price-table format of the
  model layer.
- Which Ollama endpoint style to use, and which specific models to sweep
  (with the section 5.4 minimum).
- The exact candidate tasks for each rung, and the repeat counts within the
  spend cap.
- The results-file format.
- How headless Claude Code is launched inside the container.
