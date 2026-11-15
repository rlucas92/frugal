---
name: review-plan
description: Adversarial review of an implementation plan by the frugal:plan-reviewer agent (Fable 5.2) in a fresh context. It checks the plan's claims against the code and returns READY or REVISE with concrete blockers; the main session checks the blockers, revises the plan, and shows the verdict to the user. Invoke with /frugal:review-plan [plan file] on any plan, proposed or already accepted, or load it whenever a plan review trigger in the session-start policy applies.
user-invocable: true
---

# review-plan

A plan is reviewed by an agent that did not write it. You wrote the plan, so your own re-read confirms it; the reviewer's value is the fresh context.

When the user invoked this skill, always run the review. When you loaded it yourself, a plan review trigger from the session-start policy applied; name that trigger in one line before launching.

## 1. Find the plan

In order: the file named in the arguments, the plan-mode plan file, the `<feature>-plan.md` written this session, the plan most recently presented in the conversation. If there is none, say so and stop.

## 2. Brief the reviewer

Launch one `frugal:plan-reviewer` in the foreground and omit `model`. Give it:

- **The plan**: the file path, or the full text when it exists only in the conversation.
- **Requirements and acceptance criteria** the plan must meet, and the design doc path when there is one.
- **Relevant paths**: the code areas the plan touches.

Leave out your reasons for believing the plan is right. The reviewer should test the plan against the code, not against your argument for it.

A plan with several independent slices can take one reviewer per slice, launched together in the background. Otherwise one reviewer; each extra one re-reads the plan and the code.

## 3. Act on the verdict

- **READY**: present the plan (or continue, when it was already accepted) with the line `Plan review: READY` and any notes worth the user's attention.
- **REVISE**: open the evidence for each blocker. Revise the plan for every blocker that holds; for one that does not, keep the plan and record why in a line. Re-task the same reviewer with SendMessage and the revised plan. Two REVISE rounds on the same plan means simplify, narrow, or split it; never resubmit an unchanged plan.

Then tell the user the verdict, each blocker, and what changed in the plan because of it.

If the plan was already accepted and a blocker changed it, the accepted plan no longer exists: present the revised plan and wait for approval before any source edit. If implementation had started, stop at the current step first.
