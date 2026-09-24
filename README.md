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
│   └── code-comments/SKILL.md  /frugal:code-comments — comment rules, preloaded into coders
└── agents/                     one role per file, referenced as frugal:<name>
```

| Role | Model | Effort | Does | Cannot |
|---|---|---|---|---|
| `scout` | Haiku | default | read-only lookups with `file:line` answers | write, run, delegate |
| `runner` | Haiku | default | runs tests, builds, formatters; reports pass/fail | edit source, diagnose, delegate |
| `prototyper` | Sonnet 5 | medium | executes a clear spec fast: prototypes, scripts, pattern edits, convention-following tests | delegate |
| `coder` | Opus 5.5 | medium | features, root-caused fixes, refactors with design decisions | delegate |
| `plan-reviewer` | Opus 5.5 | medium | adversarial read-only plan review, returns READY or REVISE | write, run, delegate |
| `verifier` | Opus 5.5 | medium | fresh-context verification of an exact claim: CONFIRMED, REFUTED, INCONCLUSIVE | edit, delegate |
| `advisor` | Fable 5.1 | high | plan or course correction for a session running on a cheaper model | write, run |

The orchestrator is the main session, so the frontier model is whatever `/model` is set to.

## How a task flows

1. **Triage** (session-start policy, automatic). Every task lands in one bucket before work starts:
   - *Direct*: one chain of dependent steps, clear outcome, fits in context. Executing an existing plan, single-bug debugging, minor tweaks. Done in the main session on whatever model it is already running. Only test runs and wide lookups get delegated.
   - *Multi-agent*: the work splits into independent pieces, or would flood the context, or needs a different tool set, or has a plan worth adversarial review. Loads the `orchestrate` skill.
   - *Ask*: outcome unclear, request wording disagrees with the codebase, or exactly one weak multi-agent signal. One AskUserQuestion with a recommendation. Ties with a clear outcome go to Direct.
2. **Shape** (`orchestrate` §1). Independent pieces get the orchestrator shape. A serial chain that is hard in spots gets the advisor shape, which by default runs inside the orchestrator: one coder owns the chain and escalates "blocked because X" to the main session, which answers and re-tasks it. The literal advisor pattern (cheap main session plus `advisor`) needs a `/model` switch, so it is recommended, not automatic.
3. **Frontier model** (`orchestrate` §2). See the rule of thumb below.
4. **Phases**: research (orchestrator reads directly; scouts only for large or several independent surfaces) → plan (main session) → adversarial review (one plan-reviewer; more only on risk triggers or multi-slice plans) → approval → implement (coder or prototyper per piece, briefed with the facts already established; worktrees for parallel writers) → verify (coder runs its piece's tests, runner runs the full suite once, verifier only on risk triggers) → report.

Every handoff duplicates context and summarizes on the way back, which is where the 3 to 10x multi-agent token cost comes from. The rule throughout: a handoff is worth it only when the receiving agent needs less context than the sender holds. A typical mid-size feature runs 4 to 6 agents, not 8 or 9.

## Opus 5.5 vs Fable 5.1: rule of thumb

Direct tasks run on the session model, whichever it is. Choose Fable 5.1 as the session model for complex, long-horizon planning; Opus 5.5 at medium effort is the everyday default. Within multi-agent runs, default to **Fable 5.1 at medium effort** for orchestration and advising. Switch to **Opus 5.5 at medium effort** when Fable has refused a benign request, or for long research loops over external sources. Opus 5.5 runs cyber and bio safety classifiers similar to Fable's, so it is a retry path for a false positive, not a way around the classifiers. Opus 5 is never used anywhere in the plugin; Opus 5.5 at medium effort replaces it.

Why:

| Finding | Source |
|---|---|
| Fable 5.1 cached input is $0.25/M vs $0.20/M for Opus 5.5, and cached input is the largest term in an agent loop; the two are close, so the choice is about behaviour, not cost | pricing page |
| Frontier safety classifiers can refuse defensive security work mid-task; pilotfish routes security to Opus for this reason. Since Opus 5.5 ships the same classifier families (cyber, bio), the Opus switch is a retry after a false positive, not a guaranteed pass | pilotfish design notes; Opus 5.5 launch notes |
| Opus 5.5 defaults to medium effort, and at medium it beat Opus 5 at high on agentic coding with about half the tokens; cache reads are $0.20/M against Opus 5's $0.50/M | Opus 5.5 launch notes |

The plugin does not ask you to choose unless an Opus trigger applies.

## Your model preferences vs the research

| Your preference | Verdict | Notes |
|---|---|---|
| Fable 5.1 for large features, complex refactors, performance work | Kept, as the orchestrator only | Fable plans, reviews, and coordinates those; the code is written by Opus 5.5 or Sonnet 5. Anything triaged as Direct runs on the session model as-is. |
| Opus 5 is never used; Opus 5.5 at medium effort is the alternative frontier | Kept | Every role and rule that named Opus 5 now names Opus 5.5 at medium effort. |
| Opus 5.5 for high-accuracy coding | Kept, pinned | Pinned to `claude-opus-5-5` so the floating `opus` alias can never resolve to Opus 5, which is never used. Same tokenizer as Fable. |
| Sonnet 5 for fast iteration and clear plans | Kept | Matches the plan-big, execute-small executor role. |
| Haiku for tests and routine maintenance | Kept | Also used for read-only recon, as pilotfish does. |
| Fable never writes code unless the change is tiny | Kept, with a ceiling | Main session edits directly at roughly 20 lines across 2 files already in context. One data point to revisit: on SWE-bench Pro, Fable 5.1 at low effort cost $0.54 per solved task against Sonnet 5's $0.84, because it solves more first try. Measure on your own work before changing the rule. |

## Changing a role's model

Edit the `model:` line in `plugin/agents/<role>.md`. Aliases (`haiku`, `sonnet`, `opus`, `fable`) float to the current release; full IDs (`claude-opus-5-5`) pin. The coding roles are pinned on purpose.

## Comment rules

`skills/code-comments/SKILL.md` is preloaded into `coder` and `prototyper`. Comments explain purpose, non-obvious logic, and considerations a reader cannot infer from the code. They never reference plan files, sprints, tickets, design docs, or conversations; the test is whether the comment stays true and useful if everything outside the repository is lost. A `TODO` for a real gap may point at where the design lives, and only there.
