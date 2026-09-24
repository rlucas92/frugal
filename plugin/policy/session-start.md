# frugal orchestration policy

Main-session policy. Plugin roles (`frugal:scout`, `plan-reviewer`, `coder`, `prototyper`, `runner`, `verifier`, `advisor`) are leaves: do the assignment you were given, never delegate, ignore this section.

## Triage every task before working

Read the request and any plan file it names, then sort the task into one bucket. Say which bucket in one line before starting.

**Direct.** Do it yourself in the main session, on whatever model the session is already running; never switch models or hand off to a coder just to change model. Signals: one chain of dependent steps; the outcome and "done" are clear; it fits comfortably in context; you can name the files. Covers executing an existing approved plan step by step, single-bug debugging, minor tweaks, config changes. Delegate only test runs (`frugal:runner`) and wide read-only lookups (`frugal:scout`).

**Multi-agent.** Load `/frugal:orchestrate` and follow it. Any one signal is enough:
- the work splits into independent pieces (files, modules, questions) that need no shared state;
- reading or doing it would fill your context with material later steps do not need;
- a piece needs a different tool set or domain (security, performance, docs);
- a plan is worth adversarial review before code is written: new feature, cross-cutting refactor, schema or data change.

**Ask.** Use AskUserQuestion, recommendation first, one question only. Ask when the outcome or acceptance is unclear, when the request wording and the codebase disagree (a "small tweak" that touches a dozen call sites; a "big feature" that is one function), or when exactly one multi-agent signal applies weakly. Never ask when the bucket is obvious.

Multi-agent runs cost 3 to 10 times the tokens of one agent, spent on duplicated context, coordination, and summarizing at each handoff. A tie between Direct and Multi-agent with a clear outcome goes to Direct; escalate if the first pass exposes real independence or a plan worth review. A handoff is worth it only when the receiving agent needs less context than you hold, never more: a scout that reads what you would read, or a reviewer that must rebuild your understanding of the plan, is coordination without gain.

## Routing rules (every bucket)

- Direct tasks run on the session model, whichever it is. Fable 5.1 is chosen as a session model for complex, long-horizon planning (multi-agent orchestration, the advisor role); Opus 5.5 at medium effort is the everyday default. That choice is the user's, made with `/model`, not the policy's.
- Code is written by `frugal:coder` (Opus 5.5) or `frugal:prototyper` (Sonnet 5). Opus 5 is never used; Opus 5.5 always runs at medium effort. The main session edits directly only when a brief would cost more than the edit: roughly 20 lines across at most 2 files already in context.
- `frugal:runner` (Haiku) runs tests and routine maintenance. `frugal:scout` (Haiku) does read-only lookups; prefer it over the built-in Explore agent, which inherits the main model.
- Omit the `model` parameter when calling a plugin role; the role file owns routing.
- Brief a subagent once and completely: goal, constraints, done criteria, relevant paths, and the facts you already established (file:line references, behaviour you confirmed) so it starts from them instead of re-exploring. Commit to the delegation; never redo or re-derive its work.
- Launch independent subagents back to back in the background and keep working. Collect every result before dependent work. Parallel writers get `isolation: "worktree"`.
- A blocked leaf reports "blocked because X" with a specific question. You answer and re-task it with SendMessage so it keeps its context. That is the advisor loop; do not take the work over on the first block.
