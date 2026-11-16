---
name: write-api-tests
description: Procedure for writing or extending platform-api-test API specs (Playwright + TypeScript against the platform REST API). The main session writes the scenario down as cases with the user's expected results, decides endpoint spec or suite, and briefs a frugal:prototyper or frugal:coder that looks the routes up in the API source, reuses the existing factories, writes the spec, and runs that one file; the main session then checks the spec against the cases and judges each failure. Invoke with /frugal:write-api-tests, or load it whenever the user asks to write, add, or extend API tests.
user-invocable: true
---

# write-api-tests

API tests live in `/Users/rj/Pearl/projects/platform-api-test`. Its `CLAUDE.md` is the authority on layout, commands, and rules; this skill is the procedure around it. Your job is the scenario and the judgment afterward; an agent writes the spec.

Inside an orchestrate run, sections 1 to 3 are the brief for the piece that writes API tests.

## 1. Write the cases down

The request defines what to test; the API source only shows how to call it. Turn the request into a numbered list of cases. Each case names who acts (role and org), what they do, and the expected result (status, response body, state afterward).

- Expected results come from the user's words: the request, a ticket, acceptance criteria. Never read one off the source. A test whose expectation was copied from the code cannot fail.
- Where the request leaves an expected result open, ask, with every open case in one question and the answer you think is meant as the recommendation. When every expected result is stated, do not ask; go on.

## 2. Decide where the spec goes

- **Endpoint spec**: one basic check of one route. `tests/endpoint/<url path>/<method>.spec.ts`, the folders mirroring the URL. Most of these files already exist as placeholders that only call `showComments('no test yet ...')`; fill the placeholder in place. `tests/endpoint/ping/get.spec.ts` is the template.
- **Suite**: anything with several steps, several roles, or its own data. `tests/suite/<area>/<area>.<topic>.spec.ts`, creating its own organization(s).
- **An existing suite**: when a suite or a scenario file (`data/datashare/*.json`) already covers the area, add a `test()` or a scenario entry there instead of a new file.

For a new suite, make the tests independent, each on its own organization, form, or dataset, so one failure does not skip the rest. Use `test.describe.configure({ mode: 'serial' })` only when each test is a phase that starts from the state the previous one left. A phase added to a serial suite changes the state every later phase starts from, so add it at the end or say which phases it affects.

## 3. Brief the agent

Who writes it:

- You, directly: roughly 20 lines in files already in context (a scenario entry, one endpoint placeholder).
- `frugal:prototyper`: an existing spec is the pattern and the helpers it needs exist.
- `frugal:coder`: the spec needs new factories, a scenario loader, or an expected-row model.

One agent per spec file, in the foreground; specs that change the same factory file go to one agent. Omit `model`. Give it:

- **Repo** and the instruction to read its `CLAUDE.md` first. An agent started from another repo does not load it.
- **The cases**, verbatim, and that the expected results are fixed: if the source suggests a case will fail, it writes the case as stated and says so in its report.
- **Where the spec goes** and the existing spec to follow as the pattern.
- **Source of truth**: `/Users/rj/Pearl/projects/platform-api-under-test`, on the branch it has checked out (`scripts/api.sh status`, or `git -C <dir> branch --show-current` when the API is down). Routes, roles, and request bodies are in `src/main/java/com/trilogymentors/api/endpoint/` (`@Path`, `@RolesAllowed`). It never switches branches and does not read the other `../platform-api*` checkouts.
- **Reuse**: `data/datafactory/` (and `data/datafactory/datashare/`), `common/auth.ts`, the org templates in `constants/organizationTypes.ts`, the scenario loaders. Extend a factory; do not add a parallel one.
- **Conventions the existing specs follow**:
  - chai `expect`, with a message on any assertion whose failure would not explain itself.
  - Endpoint spec: the title comes from `parseDirectory(__dirname)` and `parseFileName(__filename)`, with any path parameter appended (`/{tagType}`); the organization comes from `fetchEndpointTestingOrganization()`.
  - Suite: a header comment saying what the suite proves and the command that runs only this file; numbered test titles (`'1. ...'`); `test.setTimeout()` on setup that creates organizations; ids attached to the report with `test.info().attach()`, not printed; `okData` / `okPage` to unwrap the `{ code, data }` envelope.
  - A suite added or changed in `tests/suite/insight/` also updates its section in `tests/README.md` and its row in `tests/suite/insight/README.md`.
- **Checks**, in this order:
  1. `npx playwright test <spec> --list` loads the file and lists the expected titles. It needs no API, and it is the only compile check: nothing type-checks the TypeScript.
  2. `npx prettier --check <files it touched>`. Not `npm run prettier-check`, which fails on older files.
  3. `scripts/api.sh status`. If the API is up, run that one file: `npx playwright test <spec> --reporter=line --workers=1`. Never the whole suite or the `teardown` project. If the API is down, it stops here and says so; it never runs `restart`, `switch`, `use`, or `reset`.
- **Failures**: it fixes its own mistakes (setup, a wrong request body, a wrong helper call) and re-runs. It never changes an assertion that states a case's expected result.
- **Report**: files changed; each case with its test title and passed, failed (assertion message), or not run; cases the source suggests will fail and likely bugs it noticed along the way; helpers added or extended.

## 4. Review and report

- Read the diff against the cases. Every case has a test, and each assertion states the user's expected result as given: not loosened (`>= 400` where the case said 403), not narrowed to what the API returns today.
- Judge each failure. A mistake in the test goes back to the same agent with SendMessage. The app disagreeing with a case is the user's to triage: either the app or the scenario is wrong, so say which side looks wrong and why. A 500 from an endpoint that reads the Insights Plus datastore is environment.
- Tell the user: the cases, where the spec is, the result per case, each failure with your judgment, likely bugs noticed, and anything not run. Mark a failing test `test.fixme()` only when the user asks; `tests/suite/insight/README.md` describes how the existing suites do it.
- Later runs of the spec go through `/frugal:run-tests`.
