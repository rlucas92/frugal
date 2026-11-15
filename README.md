# frugal

A Claude Code plugin for triage-first multi-agent orchestration. It decides on every task whether a multi-agent run is worth the 3 to 10x token cost, asks you only when the call is genuinely unclear, picks orchestrator or advisor shape, and routes each kind of work to a specific model. The frontier model keeps planning, review, and judgment; cheaper models do the reading, coding, and test runs.

The layout follows [pilotfish](https://github.com/Nanako0129/pilotfish): one file per role, a compact policy injected at session start, and a plugin that installs and uninstalls cleanly per project or globally. The decision rules come from Anthropic's [multi-agent guidance](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them), the [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) and [plan big, execute small](https://github.com/anthropics/claude-cookbooks/blob/main/managed_agents/CMA_plan_big_execute_small.ipynb) patterns, and the measured results in [Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence).

## Install

Add the repo as a marketplace once, then install per scope:

```bash
claude plugin marketplace add https://github.com/rlucas92/frugal
claude plugin install frugal@frugal --scope user      # every project
claude plugin install frugal@frugal --scope project   # this project only, shared via .claude/settings.json
claude plugin install frugal@frugal --scope local     # this project only, gitignored
```

Working from a local clone instead, pass its path to `marketplace add`. To try it for one session without installing:

```bash
claude --plugin-dir /path/to/frugal/plugin
```

Disable, re-enable, or remove:

```bash
claude plugin disable frugal
claude plugin enable frugal
claude plugin uninstall frugal --scope user
```

The plugin depends on [ponytail](https://github.com/DietrichGebert/ponytail), so installing or enabling it also installs and enables ponytail at the same scope, and ponytail cannot be disabled while frugal is enabled. Add the ponytail marketplace first if it is not already known:

```bash
claude plugin marketplace add https://github.com/DietrichGebert/ponytail
```

Restart Claude Code after installing or updating so the agents and the session hook reload. Do not set `CLAUDE_CODE_SUBAGENT_MODEL`; it overrides every role's model.

## What is in the plugin

```
plugin/
├── .claude-plugin/plugin.json
├── hooks/hooks.json            SessionStart: prints policy/session-start.md into context
├── policy/session-start.md     the triage rules, ~40 lines, loaded every session
├── skills/
│   ├── orchestrate/SKILL.md    /frugal:orchestrate — the full multi-agent procedure
│   ├── design/SKILL.md         /frugal:design — overall design options before the implementation plan
│   ├── review-plan/SKILL.md    /frugal:review-plan — adversarial plan review on Fable 5.2
│   ├── run-tests/SKILL.md      /frugal:run-tests — test runs on a cheaper model, triaged before review
│   └── code-comments/SKILL.md  /frugal:code-comments — comment rules, preloaded into coders
└── agents/                     one role per file, referenced as frugal:<name>
```

| Role | Model | Effort | Does | Cannot |
|---|---|---|---|---|
| `scout` | Haiku | default | read-only lookups with `file:line` answers | write, run, delegate |
| `runner` | Haiku | default | runs tests, builds, formatters; reports pass/fail | edit source, diagnose, delegate |
| `test-runner` | Sonnet 5.5 | medium | runs requested tests, triages failures with git/GitHub history, proposes obvious fixes | edit, delegate |
| `prototyper` | Sonnet 5.5 | medium | executes a clear spec fast: prototypes, scripts, pattern edits, convention-following tests | delegate |
| `coder` | Opus 5.5 | medium | features, root-caused fixes, refactors with design decisions | delegate |
| `plan-reviewer` | Fable 5.2 | high | adversarial read-only plan review, returns READY or REVISE | write, run, delegate |
| `designer` | Opus 5.5 | medium | independent design options from the requirements alone, without the proposed design | write, run, delegate |
| `verifier` | Opus 5.5 | medium | fresh-context verification of an exact claim: CONFIRMED, REFUTED, INCONCLUSIVE | edit, delegate |
| `advisor` | Fable 5.2 | high | plan or course correction for a session running on a cheaper model | write, run |

The orchestrator is the main session, so the frontier model is whatever `/model` is set to.

## How a task flows

1. **Triage** (session-start policy, automatic). Every task lands in one bucket before work starts:
   - *Direct*: one chain of dependent steps, clear outcome, fits in context. Executing an existing plan, single-bug debugging, minor tweaks. Done in the main session on whatever model it is already running. Only test runs and wide lookups get delegated.
   - *Multi-agent*: the work splits into independent pieces, or would flood the context, or needs a different tool set, or has a plan worth adversarial review. Loads the `orchestrate` skill.
   - *Ask*: outcome unclear, request wording disagrees with the codebase, or exactly one weak multi-agent signal. One AskUserQuestion with a recommendation. Ties with a clear outcome go to Direct.
2. **Shape** (`orchestrate` §1). Independent pieces get the orchestrator shape. A serial chain that is hard in spots gets the advisor shape, which by default runs inside the orchestrator: one coder owns the chain and escalates "blocked because X" to the main session, which answers and re-tasks it. The literal advisor pattern (cheap main session plus `advisor`) needs a `/model` switch, so it is recommended, not automatic.
3. **Frontier model** (`orchestrate` §2). See the rule of thumb below.
4. **Phases**: research (orchestrator reads directly; scouts only for large or several independent surfaces) → design (see below) → plan (main session) → adversarial review (one plan-reviewer when a review trigger applies; one per slice on multi-slice plans) → approval → implement (coder or prototyper per piece, briefed with the facts already established; worktrees for parallel writers) → verify (coder runs its piece's tests, runner runs the full suite once, verifier only on risk triggers) → report.

Every handoff duplicates context and summarizes on the way back, which is where the 3 to 10x multi-agent token cost comes from. The rule throughout: a handoff is worth it only when the receiving agent needs less context than the sender holds. A typical mid-size feature runs 4 to 6 agents, not 8 or 9.

## Design before the implementation plan

Planning a new feature, a schema or data change, or a cross-cutting refactor runs in two steps: overall design, then implementation details. The session policy loads `/frugal:design` before any such plan, in plan mode too, and you can invoke it directly.

1. You give the feature summary, requirements, tickets, acceptance criteria, and any proposed design (yours or a teammate's).
2. It investigates and decides whether the design space is open. A bug fix or a column added to an existing table gets one line naming the pattern it follows, then goes straight to the plan.
3. If the space is open, it names the design decisions as questions ("How are Ops uploads told apart from Insights+ uploads?"). For each one it lists options with trade-offs and a recommendation. One option always reuses the mechanism the codebase already uses for that kind of question. A `designer` agent drafts options without seeing the proposed design, so the proposal is weighed instead of assumed.
4. The result is a short design doc, saved in the repo as `<docs dir>/<feature>-design.md`, that you can share with the team as is. It names mechanisms and tables, not files or line numbers; evidence sits in a separate section at the end.
5. You ask questions and iterate, then choose an option for each decision. Rejected options stay in the doc with one line on why they lost.
6. The implementation plan (files, line numbers, steps, PR slices) is saved next to it as `<feature>-plan.md`, opening with a link to the design doc and the chosen options, and goes through the usual review and approval. In plan mode both are drafted in the plan file and written to the repo right after approval. It can be executed in a later session.

Defaults when recommending: smaller and fewer PRs first, then reuse of existing abstractions, patterns, and tables, then performance where it does not cost much simplicity.

## Plan review

A plan is reviewed by `plan-reviewer`, an agent that did not write it, always on Fable 5.2 at high effort whatever the session model is. It checks the plan's claims against the code and returns READY or REVISE with blockers, each with evidence and the smallest revision that removes it.

- **Automatic.** In any bucket, plan mode included, the session policy loads `/frugal:review-plan` after the plan is written and before it is presented for approval, when a trigger applies: a schema, data, or migration change; a security boundary or an irreversible path; a changed contract between components, several modules, or more than one pull request; a design step that had open decisions; or a step that rests on code behaviour the session inferred and did not read. The verdict is shown with the plan. When no trigger applies, the plan carries one line saying the review was skipped and why.
- **Manual.** `/frugal:review-plan [plan file]` reviews any plan, before or after you accept it. If a blocker changes a plan you already accepted, the revised plan comes back for approval before any code is written.

## Running tests

Asking for tests in any session ("run the unit tests", "run the datashare spec") loads `/frugal:run-tests` instead of running them on the session model.

1. The main session briefs one `test-runner` (Sonnet 5.5) with the repo, the exact scope, and what it already knows.
2. The agent runs the narrowest command that covers the request, then triages each failure: reads the assertion and the code under test, compares the last commits to the test against the last commits to the code (and merged PRs via `gh`), and classifies it as stale test, regression, environment, flaky, or unclear. It proposes a fix only when it is obvious and never edits.
3. The main session checks the classifications against their evidence, re-tasks the agent with SendMessage if something needs one more look, and proposes next steps. Nothing is applied until you choose.

For `platform-api-test` the agent follows that repo's CLAUDE.md rules: one spec file at a time, never the whole suite or `teardown` against a local API, `scripts/api.sh status` before running, `../platform-api-under-test` as the source of truth, never bending an assertion to match current behaviour, the first failing phase of a serial data share suite as the one to triage, and datastore 500s as environment.

## Opus 5.5 vs Fable 5.2: rule of thumb

Direct tasks run on the session model, whichever it is. Choose Fable 5.2 as the session model for complex, long-horizon planning; Opus 5.5 at medium effort is the everyday default. Within multi-agent runs, default to **Fable 5.2 at medium effort** for orchestration and advising; plan review always runs on Fable 5.2. Switch to **Opus 5.5 at medium effort** when Fable has refused a benign request, or for long research loops over external sources. Opus 5.5 runs cyber and bio safety classifiers similar to Fable's, so it is a retry path for a false positive, not a way around the classifiers. Opus 5 is never used anywhere in the plugin; Opus 5.5 at medium effort replaces it.

Why:

| Finding | Source |
|---|---|
| Fable 5.2 cache reads are $0.20/M, the same as Opus 5.5, and cached input is the largest term in an agent loop. Uncached input and output cost twice as much ($8/$40 per M against $4/$20), so Fable stays on short-output judgment roles: orchestration, advising, plan review | pricing page |
| Fable 5.2 is up to 30% faster than Fable 5.1, uses fewer tokens at medium effort, and has about 55% fewer classifier blocks; Anthropic recommends Opus 5.5 as the daily driver and Fable as advisor or orchestrator | [Fable 5.2 launch notes](https://www.anthropic.com/claude-fable-and-mythos-5-2) |
| Frontier safety classifiers can refuse defensive security work mid-task; pilotfish routes security to Opus for this reason. Since Opus 5.5 ships the same classifier families (cyber, bio), the Opus switch is a retry after a false positive, not a guaranteed pass | pilotfish design notes; Opus 5.5 launch notes |
| Opus 5.5 defaults to medium effort, and at medium it beat Opus 5 at high on agentic coding with about half the tokens; cache reads are $0.20/M against Opus 5's $0.50/M | Opus 5.5 launch notes |

The plugin does not ask you to choose unless an Opus trigger applies.

## Your model preferences vs the research

| Your preference | Verdict | Notes |
|---|---|---|
| Fable 5.2 for large features, complex refactors, performance work | Kept, as the orchestrator only | Fable plans, reviews, and coordinates those; the code is written by Opus 5.5 or Sonnet 5.5. Anything triaged as Direct runs on the session model as-is. |
| Opus 5 is never used; Opus 5.5 at medium effort is the alternative frontier | Kept | Every role and rule that named Opus 5 now names Opus 5.5 at medium effort. |
| Opus 5.5 for high-accuracy coding | Kept, pinned | Pinned to `claude-opus-5-5` so the floating `opus` alias can never resolve to Opus 5, which is never used. Same tokenizer as Fable. |
| Sonnet 5.5 for fast iteration and clear plans | Kept | Matches the plan-big, execute-small executor role. |
| Haiku for tests and routine maintenance | Kept, except triage | Haiku runs builds, formatters, and the orchestrate full-suite pass, and does read-only recon as pilotfish does. Test runs you ask for go to Sonnet 5.5 because the same agent triages the failures. |
| Fable never writes code unless the change is tiny | Kept, with a ceiling | Main session edits directly at roughly 20 lines across 2 files already in context. One data point to revisit: on SWE-bench Pro, Fable 5.1 at low effort cost $0.54 per solved task against Sonnet 5's $0.84 (measured before Sonnet 5.5), because it solves more first try. Measure on your own work before changing the rule. |

## Changing a role's model

Edit the `model:` line in `plugin/agents/<role>.md`. Aliases (`haiku`, `sonnet`, `opus`, `fable`) float to the current release; full IDs (`claude-opus-5-5`, `claude-fable-5-2`) pin. The coding roles and `plan-reviewer` are pinned on purpose; `advisor` floats on the `fable` alias, so it follows the current Fable release.

## Comment rules

`skills/code-comments/SKILL.md` is preloaded into `coder` and `prototyper`. Comments explain purpose, non-obvious logic, and considerations a reader cannot infer from the code. They never reference plan files, sprints, tickets, design docs, or conversations; the test is whether the comment stays true and useful if everything outside the repository is lost. A `TODO` for a real gap may point at where the design lives, and only there.
