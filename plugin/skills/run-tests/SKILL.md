---
name: run-tests
description: Hands a requested test run to the frugal:test-runner agent on a cheaper model, so the session model never reads raw test output. The agent runs unit tests or platform-api-test API specs, triages failures with git/GitHub history, and proposes obvious fixes; the main session reviews its report and proposes next steps. Invoke with /frugal:run-tests, or load it whenever the user asks to run tests outside an orchestrate run.
user-invocable: true
---

# run-tests

The user asked for tests. Do not run them yourself. Running and reading test output is mechanical work; the session model's job is judging the triage afterward.

Inside an orchestrate run, the Verify phase rules apply instead (coders run their own tests, `frugal:runner` runs the full suite once).

## 1. Brief the agent

Launch one `frugal:test-runner` in the foreground (you need its result before anything else) and omit `model`. Give it:

- **Repo**: absolute path. API tests live in `/Users/rj/Pearl/projects/platform-api-test`; unit tests are in the repo the user names, or the current one.
- **Scope**: exactly what the user asked to run: a test, class, file, module, or spec file. For API tests, name the spec file(s). If the user asked for "the API tests" with no file, ask which spec or area before launching rather than letting the agent guess; the full suite is never run locally.
- **Context you already have**: the branch and what it changes, files you just edited, a failure you already suspect. One or two lines saves the agent re-deriving it.

Several independent repos or areas: one agent each, launched back to back in the background.

## 2. Review the report

The agent's triage is a first pass from a cheaper model; check it before relaying it.

- Verify each classification against its evidence. A "stale test" needs a code change that is newer than the test and intentional; a "regression" needs a plausible code change. Open a cited file or commit only when the claim does not hold up as written.
- For API tests, the scenario the user defined is the spec. Reject any proposed fix that changes an assertion to match current behaviour unless the history shows the behaviour change was intended.
- Treat a cascade (later phases of a serial data share suite failing after one phase) as one failure.
- If the report is thin or a claim needs one more check, re-task the same agent with SendMessage and the specific question; it keeps its context. Do not take over the investigation.

## 3. Propose next steps

Tell the user: the verdict and counts, each failure with your judgment of its cause (agreeing with or correcting the agent), and what you propose per failure: apply the fix, update the test, fix the app, fix the environment (for example `scripts/api.sh restart` or a reset, which you never run without asking), or investigate further. Apply nothing until the user picks.
