---
name: design
description: Overall design step for a feature, run before the implementation plan. Decides whether the design space is open enough to compare approaches, names the design decisions, lays out options for each with trade-offs, and waits for the user to choose before any implementation detail is written. Invoke with /frugal:design, or load it whenever a new feature, schema or data change, or cross-cutting refactor reaches planning.
user-invocable: true
---

# design

Planning has two steps: **overall design**, then **implementation details**. This skill is the first. It ends when the user has chosen an option for every open decision; the implementation plan (files, line numbers, steps, PR slices) is written only after that, and never in the same document section.

The failure this prevents: a plan built on the first workable design, approved because it looked vetted, then reworked after a code reviewer proposes an approach nobody compared. A proposal from the user or a teammate is one option to weigh, not the answer, even when it ends up winning.

## 1. Inputs

Collect what the user gave: feature summary, product or business requirements, user stories and tickets (read linked Monday items with the monday tools when they are available), acceptance criteria, and any proposed designs from the user or teammates. Missing acceptance criteria is the one gap worth a question before starting; ask it with AskUserQuestion and stop.

## 2. Investigate

Read enough code to answer two things for the feature:

- **What already exists that could carry this change.** For each question the feature raises, find what the codebase already uses to answer the same kind of question: a discriminator column or enum that decides how records are treated, an existing table holding the data, a pipeline or pattern that already does a similar job. These are the options code reviewers most often raise.
- **Where a proposal and the code disagree.** Gaps that stop a proposed design from working as stated (missing columns, events not recorded, a table that does not hold what was assumed).

Read directly. Use `frugal:scout` only for a surface too large to read without crowding out later work.

## 3. Gate: is the design space open?

**Closed** when one approach is dictated by the code or the requirements: a bug fix with a known root cause, a column added to an existing table, an endpoint that follows an existing pattern, an existing mechanism extended the obvious way. Say so in one line, naming the existing pattern the change follows, so the user can disagree, then go straight to the implementation plan.

**Open** when any of these holds:

- two or more existing mechanisms could carry the change;
- a proposal needs something the code lacks, and the gap can be filled more than one way;
- the change defines a new data shape, file format, or contract that other components or teams will depend on;
- a requirement implies a decision nobody stated (if Ops uploads reuse the Insights+ pipeline, something must tell the two apart).

When unsure, treat it as open. A one-decision design note costs minutes; a rework after code review costs the PR.

## 4. Name the decisions

A design decision is a question whose answer changes the shape of the solution: the data model, which mechanism carries a behaviour, where data is sourced from, the contract between components, or what is in scope. Write each as a question. Examples from past features:

- How are Ops uploads told apart from Insights+ uploads?
- How do we find rows that changed: timestamps or full row comparison?
- Where are changes sourced from: the audit schema tables or the source tables?
- How are delta files structured, and how are updates told apart from deletes?
- Which changes are tracked?

Implied decisions count as much as stated ones. Not decisions: names, which file or class code lives in, method signatures, test layout, PR order. Those belong to the implementation plan.

Expect two to five. More than six usually means implementation details have crept in, or the feature should be split. Order them so decisions that constrain others come first, and note each dependency ("if changes come from the source tables, deletes need their own detection").

## 5. Options

Give each decision two or three options. Always include:

- every proposed design (the user's, teammates');
- the option that reuses whatever the codebase already uses for the same kind of question, found in step 2;
- a third only when it is materially different. Do not pad. A decision with only one real option is not a decision; move it to **Settled**.

**Independent read.** When the gate opened on more than one decision, or a proposal is in play, spawn one `frugal:designer` in the background while you draft. Brief it with the requirements, acceptance criteria, your decision questions, the relevant code areas, and the facts from step 2. Do **not** give it any proposed design or your own leaning; its value is the options an anchored reader would not consider. Merge any distinct option and any missed decision it returns; drop duplicates silently. Skip it when the gate barely opened (one decision, two obvious options).

**Trade-offs.** Judge every option on the same criteria, one short line each, and only where options differ:

- **Size**: rough number of PRs, tables or columns added, components touched. Not files.
- **Reuse**: existing abstractions, tables, and patterns it builds on, or new ones it introduces.
- **Future fit**: what known upcoming functionality it makes easier or harder.
- **Performance**: only when the difference is material.
- **Risk**: migration, backfill, behaviour change for existing users.
- **Gaps**: what it cannot do without extra work.

**Recommend** one option per decision in one sentence. Default preferences, in order: smaller and fewer pull requests; reuse of existing abstractions, patterns, and database tables; performance considered, but not at a large cost in simplicity.

## 6. Write the design doc

The doc is meant to be shared with the team as is. Keep it at design altitude: name mechanisms, tables, columns, and concepts; leave out file paths, line numbers, and method names. Every fact a decision rests on goes in the **Evidence** section at the end, so everything above it can be copied without trimming. Aim for something a teammate reads in five minutes; no option gets more than five bullets.

```markdown
# <Feature>: design

## Requirements
3 to 6 bullets: what must be true when this ships, from the ticket and acceptance criteria.

## Settled
One line each: things with one sensible answer, and the existing pattern they follow.

## Decisions

### 1. <question>
- **A. <short name>** (proposed by <who>, if anyone): one sentence.
  - + <advantage>
  - − <cost or gap>
- **B. <short name>**: one sentence.
  - + ...
  - − ...

Recommendation: B, because <one sentence>.
Depends on: <decision number, or nothing>.

## Open questions
Questions only product or another team can answer.

## Evidence
- <fact a decision rests on>: `path:line`
```

**Where it goes.** The design doc is always saved as a markdown file in the repo, never only in chat: `<docs dir>/<feature>-design.md`. Use the directory where the repo already keeps design docs or plans (for example `docs/<area>/`); if it has none, use `docs/`. Name the feature in kebab case from the ticket or feature name. Plan mode can only write its own plan file, so in plan mode draft the design doc there, and write the repo file as the first action once plan mode ends (step 8). Do not call ExitPlanMode for the design doc alone; it is not a plan yet.

## 7. Iterate and choose

Present the doc and end the turn. The user will ask questions, propose new options, or push back; revise the doc each time instead of answering only in chat, so it stays the single shareable version. When the conversation turns to choosing, ask with AskUserQuestion, one question per open decision (at most four per call), recommended option first.

Record each choice in the doc: mark the chosen option and keep each rejected option with one line on why it lost. Reviewers ask about the alternatives, and the PR description can reuse those lines.

## 8. Hand off to the implementation plan

Write the implementation plan as a second markdown file in the repo, next to the design doc: `<docs dir>/<feature>-plan.md`. The design doc stays as the short, shareable one; the plan holds the detail. The plan opens with a `## Design` section that links the design doc and repeats the chosen option for each decision in one line each, so it can be read alone. The plan adds what the design left out: files to change, line numbers, ordered steps, and pull request slices, each slice as small as it can be while still shipping on its own. Write it so a new session can execute it without this conversation: every fact it relies on stays in the file.

In plan mode, the plan file holds both documents while you work: the design doc on top, the implementation plan below it. Once the user approves with ExitPlanMode, the first action, before any code, is to write the two repo files from it: the design part to `<feature>-design.md` and the implementation part to `<feature>-plan.md`.

If implementation research shows a chosen option does not hold, stop and reopen that one decision with the user. Never switch designs silently inside the plan.

From here the normal flow continues: adversarial plan review, approval (including the PR split), then implementation, which may happen in a later session.
