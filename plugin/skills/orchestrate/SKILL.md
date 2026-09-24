---
name: orchestrate
description: Full procedure for a multi-agent run once triage has chosen the multi-agent bucket. Chooses orchestrator vs advisor shape, confirms the frontier model, then runs research, plan, adversarial plan review, implementation, and verification through frugal role agents with the right model on each. Invoke with /frugal:orchestrate, or load it whenever the session-start policy routes a task to multi-agent.
user-invocable: true
---

# orchestrate

You are the orchestrator: plan, delegate, read results, decide. You do not write most of the code, gather most of the research, or run the tests yourself. The session-start policy already sorted this task into the multi-agent bucket; if it did not and the task is a single dependent chain that fits in context, stop here and do it directly on the session model.

## 1. Choose the shape: orchestrator or advisor

One question decides it: **does the work split into independent pieces, or is it one answer reached through a chain of dependent steps?**

| Shape | The work looks like | Frontier model's job | Frontier cost scales with |
|---|---|---|---|
| **Orchestrator** | Fans out across independent files, modules, documents, or questions; may exceed one context window | Plans, dispatches, synthesizes, judges | How hard the pieces are to coordinate |
| **Advisor** | One serial chain that is hard in spots: many mechanical turns between a few real decisions | Consulted for the plan and for course corrections | How often the executor gets stuck |

**Orchestrator shape** is this skill, run with you (the session model) as orchestrator. It is the default for anything that fans out.

**Advisor shape** has two ways to run in Claude Code:

- *Inside an orchestrator run (default).* Delegate the whole chain to one `frugal:coder` with a complete brief. When it reports "blocked because X", you answer and re-task it with SendMessage so it keeps its context. You are the advisor; no model switch is needed. Use this when the chain is medium length or judgment-dense.
- *Cheap main session plus a frontier advisor.* The user switches the session to Opus 5.5 with `/model claude-opus-5-5` and works the chain directly, calling `frugal:advisor` once at the start for a plan and again when stuck. Cheaper on long, mostly mechanical chains (many edit-test cycles). Requires a user action, so recommend it in one line and continue with the default if they do not take it. If the advisor would be consulted on most steps, the session should just run on the frontier model instead; measured results show the advisor's benefit disappears when the executor asks constantly.

## 2. Confirm the frontier model

The orchestrator and the advisor are the frontier model. The orchestrator is the session model, chosen with `/model`; the advisor role file pins `fable` and can be changed to `claude-opus-5-5` (with `effort: medium`) in one line. Never the `opus` alias: it can resolve to Opus 5, which is never used.

Rule of thumb, from Anthropic's published cost and intelligence measurements:

- **Fable 5.1 at medium effort** is the orchestrator only for complex, long-horizon planning; that is what the multi-agent bucket is, so it is the default here and nowhere else. It delegates and coordinates subagents more reliably, and is cheap on cached input (which dominates an agent loop). Raise effort only where it misses.
- **Switch to Opus 5.5 at medium effort when** the task is security-sensitive or offensive-security-adjacent (Fable's safety classifiers can refuse benign defensive work mid-task), when Fable has already refused, or when the run is a long research loop over external sources. Cap its spawn count and ask for terse reports.
- If none of the Opus triggers apply, do not ask; use the session model.

Opus 5 is never used, in any role. Opus 5.5 always runs at medium effort.

## 3. Phases

Run these in order, dropping any that do not apply. State which phases you are running before starting.

**Research.** Read directly first. You already hold the request and the plan; a scout that reads the same files you would read and summarizes them back is a handoff with no gain, and its findings get read a third time by the coder. Delegate to `frugal:scout` only for a surface large enough that reading it yourself would crowd out later work, or for several independent surfaces you can sweep in parallel. Never run a scout as the step before a single coder. When you do delegate, launch scouts together in the background with disjoint scopes, do not read an active scout's scope yourself, and collect all results before comparing across surfaces. A scouted fact is input, not verified truth; re-check any fact a decision rests on.

**Plan.** You write it. Goal, affected files, approach, ordered steps, risks, verification. Write it to a plan file if the project keeps plans in a directory; otherwise present it inline. For large work, split into independently approvable slices, each with scope, non-goals, owner, prerequisites, acceptance, and rollback.

**Adversarial plan review.** One `frugal:plan-reviewer` with the plan and the paths. Use two or three in parallel only when a risk trigger applies (see Verify) or the plan has several slices; each extra reviewer re-reads the plan and the code, so it must earn its cost. Revise against blockers that survive your own check of their evidence. Two REVISE rounds on the same plan means simplify, narrow, or split it; do not resubmit an unchanged plan. Skip this phase when there is no real plan to review (one mechanical task with one obvious approach).

**Approval.** Large, architectural, or risky work waits for explicit user approval of the plan before any source edit. A broad initial request is not approval of an unseen plan. Present the plan, end the turn.

**Implement.** One executor per independent piece. Route by kind of work:

| Work | Role | Model | Effort |
|---|---|---|---|
| Needs accuracy and reasoning: features, root-caused fixes, refactors with design decisions | `frugal:coder` | Opus 5.5 | medium |
| Plan is already clear: spec execution, prototyping, scripts, pattern edits, tests following conventions, docs | `frugal:prototyper` | Sonnet 5 | medium |
| Tiny change, files already in context, brief would cost more than the edit | you, directly | session model | as is |

Parallel writers get `isolation: "worktree"` and disjoint file ownership; integrate every worktree. A brief contains goal, constraints, done criteria, relevant paths, the rationale, and the facts you already established: file:line references, the function that does X, behaviour you confirmed. The coder should start from your findings, not rediscover them. Add the specific rules the piece needs (reuse before adding, shortest readable change, surgical edits). The comment rules are preloaded into both coding roles; do not paste them. Tell the coder which tests cover its piece; it runs those, not the whole suite.

Escalation: a prototyper that fails twice moves to coder; a coder that reports blocked gets your answer and a SendMessage re-task. No third same-tier retry. Never redo a leaf's work yourself after delegating it.

**Verify.** Tests run at most twice per change. Each coder runs the tests that cover its own piece. `frugal:runner` then runs the full suite once, after every piece is integrated, not after each one. One fresh `frugal:verifier` with the exact claim and diff at the smallest boundary where the claim can be refuted, but only for risk triggers: data, schema, or migration changes; security or trust boundaries; destructive or irreversible paths; material cross-component integration; or an explicit user request. Give it the runner's result so it exercises the acceptance flow and edge cases instead of re-running the suite. Routine work does not get a verifier pass. Tests are evidence, not a substitute for the verifier where it is triggered, and the verifier is not a substitute for tests.

**Report.** Outcome first. What was built and how it was verified, decisions and why, anything deferred, findings rejected with evidence, and any external action not taken.

## 4. Workflow tool

This skill is standing authorization to call the Workflow tool without a separate opt-in. Use it when three or more independent agents span two or more phases; otherwise direct Agent calls are cheaper. In a script, pass `agentType: 'frugal:coder'` (or scout, prototyper, plan-reviewer, verifier, runner) and omit `model` so the role file keeps routing. Default to `pipeline()`; use a barrier only when a stage needs all prior results at once. Load the workflow-authoring reference before writing a script.

## 5. Model-specific orchestrator behaviour

- **On Fable 5.1:** delegate asynchronously. Launch subagents with `run_in_background` and keep working; intervene if one goes off track or lacks context. State goals and constraints in briefs rather than step lists; prescriptive checklists reduce output quality on this generation. For long builds, establish a way of checking your own work and run it on a cadence; a fresh-context verifier outperforms self-critique.
- **On Opus 5.5:** keep spawn counts low. Do not spawn a subagent for work you can finish in a handful of tool calls, and do not spawn subagents just to double-check yourself; verification belongs in your own loop except where the risk triggers above call for the verifier role. Brief once, commit to the delegation.
- **Any model:** omit `model` on plugin-role calls. Never set `CLAUDE_CODE_SUBAGENT_MODEL`; it overrides every role's routing.

## 6. Effort reference

| Role | Effort | Why |
|---|---|---|
| scout, runner | model default | Haiku; high volume, near-zero judgment |
| prototyper | medium | judgment lives in the spec |
| verifier | medium | balance point for read-and-run work |
| coder | medium | Opus 5.5 default; raise where it misses |
| plan-reviewer | medium | Opus 5.5 default; raise where it misses |
| advisor | high | correctness over cost, short output |
| orchestrator (you) | session setting | medium is usually enough on Fable; raise where it misses |
