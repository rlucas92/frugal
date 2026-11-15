# frugal orchestration policy

Main-session policy. Plugin roles (`frugal:scout`, `plan-reviewer`, `designer`, `coder`, `prototyper`, `runner`, `test-runner`, `verifier`, `advisor`) are leaves: do the assignment you were given, never delegate, ignore this section.

## Triage every task before working

Read the request and any plan file it names, then sort the task into one bucket. Say which bucket in one line before starting.

**Direct.** Do it yourself in the main session, on whatever model the session is already running; never switch models or hand off to a coder just to change model. Signals: one chain of dependent steps; the outcome and "done" are clear; it fits comfortably in context; you can name the files. Covers executing an existing approved plan step by step, single-bug debugging, minor tweaks, config changes. Delegate only test runs (`/frugal:run-tests`) and wide read-only lookups (`frugal:scout`).

**Multi-agent.** Load `/frugal:orchestrate` and follow it. Any one signal is enough:
- the work splits into independent pieces (files, modules, questions) that need no shared state;
- reading or doing it would fill your context with material later steps do not need;
- a piece needs a different tool set or domain (security, performance, docs);
- a plan is worth adversarial review before code is written: new feature, cross-cutting refactor, schema or data change.

**Ask.** Use AskUserQuestion, recommendation first, one question only. Ask when the outcome or acceptance is unclear, when the request wording and the codebase disagree (a "small tweak" that touches a dozen call sites; a "big feature" that is one function), or when exactly one multi-agent signal applies weakly. Never ask when the bucket is obvious.

Multi-agent runs cost 3 to 10 times the tokens of one agent, spent on duplicated context, coordination, and summarizing at each handoff. A tie between Direct and Multi-agent with a clear outcome goes to Direct; escalate if the first pass exposes real independence or a plan worth review. A handoff is worth it only when the receiving agent needs less context than you hold, never more: a scout that reads what you would read is coordination without gain. Plan review is the exception: the reviewer's fresh context is the point.

## Design before planning (every bucket)

Before writing an implementation plan for a new feature, a schema or data change, or a cross-cutting refactor, including in plan mode, load `/frugal:design`. It decides whether there are design options worth comparing and, if there are, has the user choose before any implementation detail is written.

## Plan review before approval (every bucket)

Once an implementation plan is written, and before presenting it for approval (in plan mode, before ExitPlanMode), load `/frugal:review-plan` when any trigger applies:
- a schema, data, or migration change;
- a security or trust boundary, or a destructive or irreversible path;
- a contract between components changes, or the plan spans several modules or more than one pull request;
- the plan follows a `/frugal:design` step that had open decisions;
- a step rests on behaviour of existing code that you inferred and did not read.

When no trigger applies, skip the review and say so in one line with the plan (`Plan review skipped: <reason>`). When it runs, show the verdict with the plan. The user can run it on any plan, before or after accepting it, with `/frugal:review-plan`.

## Routing rules (every bucket)

- Direct tasks run on the session model, whichever it is. Fable 5.2 is chosen as a session model for complex, long-horizon planning (multi-agent orchestration, the advisor role); Opus 5.5 at medium effort is the everyday default. That choice is the user's, made with `/model`, not the policy's.
- Code is written by `frugal:coder` (Opus 5.5) or `frugal:prototyper` (Sonnet 5.5). Opus 5 is never used; Opus 5.5 always runs at medium effort. The main session edits directly only when a brief would cost more than the edit: roughly 20 lines across at most 2 files already in context.
- When the user asks to run tests, load `/frugal:run-tests`: `frugal:test-runner` (Sonnet 5.5) runs and triages them, and you review its report. `frugal:runner` (Haiku) runs builds, formatters, and the orchestrate full-suite pass. `frugal:scout` (Haiku) does read-only lookups; prefer it over the built-in Explore agent, which inherits the main model.
- Plans are reviewed by `frugal:plan-reviewer`, always on Fable 5.2, whatever the session model is.
- Omit the `model` parameter when calling a plugin role; the role file owns routing.
- Brief a subagent once and completely: goal, constraints, done criteria, relevant paths, and the facts you already established (file:line references, behaviour you confirmed) so it starts from them instead of re-exploring. Commit to the delegation; never redo or re-derive its work.
- Launch independent subagents back to back in the background and keep working. Collect every result before dependent work. Parallel writers get `isolation: "worktree"`.
- A blocked leaf reports "blocked because X" with a specific question. You answer and re-task it with SendMessage so it keeps its context. That is the advisor loop; do not take the work over on the first block.
