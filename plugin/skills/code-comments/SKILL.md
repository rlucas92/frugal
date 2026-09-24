---
name: code-comments
description: Rules for writing code comments and doc comments in any language. Comments explain purpose, non-obvious logic, and considerations a reader cannot infer from the code, in the codebase's own vocabulary and plain words. They never reference plan files, sprints, tickets, design docs, or conversations. Preloaded into the coder and prototyper agents; invoke directly when writing or reviewing comments yourself.
user-invocable: true
---

# code-comments

A comment exists for a reader who has only the repository. It explains what the code cannot say on its own. If the code already says it, the comment is noise and should not be written.

## What a comment may contain

- **Purpose.** What this function, type, or block is for, in one sentence, when the name alone does not carry it.
- **Logic that is not obvious.** Why this branch exists, why the order matters, why the obvious approach was not taken, what invariant a loop maintains.
- **Considerations.** Units, nullability, ordering guarantees, side effects, thread-safety, failure modes, performance ceilings, external constraints (an API that rejects empty lists, a file format quirk).

## What a comment must never contain

- References to plan files, sprints, tickets, issues, PRs, design docs, meeting notes, chat threads, or people ("per the plan", "see design doc", "as discussed", "sprint 12", "TICKET-482").
- Dates, authors, or change history. Version control owns that.
- A restatement of the code. `i++ // increment i` and `// returns the user` above `return user` are deleted, not written.
- Commented-out code.
- Narration of the editing session ("added this to fix the test", "moved from OldClass").

The test: would the comment still be true and useful if every document outside the repository were lost? If not, cut it.

## The one exception: a real gap

When something is deliberately incomplete, mark it so a reader does not mistake it for finished:

```
// TODO: handle the multi-tenant case; the batch endpoint currently assumes one tenant per request.
```

Say what is missing and, if it matters, what the finished version needs to do. Pointing at where the design lives is acceptable here and only here, because the gap is the reason the reader needs it.

## Doc comments (Javadoc, docstrings, JSDoc, rustdoc, and similar)

Describe the contract a caller depends on: what it does, what it accepts, what it returns, how it fails.

- First sentence is the summary and stands alone. Third-person verb phrase: `Returns the active session.` Not `This method returns…`.
- Most members need only the summary. Add a paragraph only for a fact the signature cannot show: a side effect, a caching rule, an ordering guarantee.
- When writing a paragraph after the first sentence, be as concise as possible
- Parameter and return tags carry contract facts (nullability, units, valid range, edge cases such as empty versus null). A tag that only repeats the name adds nothing; leave it off.
- Document exceptions that are part of the contract.
- Skip trivial getters, setters, and overrides that add no behavior.
- A repository's own doc-comment conventions take precedence over these where they conflict.

## Plain language

Use the words the codebase and the domain already use for a concept: in comments, identifiers, commit messages, and prose. Do not invent umbrella terms or metaphors for something that already has a name. When a concept is genuinely new, use the common word for it, and say so rather than adopting a coined term silently.

- Prefer: a *paused* dataset, a *paused* file held in the *log*.
- Avoid: a "halted" drop "parked" in a "ledger".

## Inline comments

Explain why, not what. Put the comment above the line or block it describes. Keep it to one or two lines. If the explanation is growing into a paragraph, the code probably needs a clearer name or a smaller function instead.

## Examples

```java
// Bad: references outside the repo, restates code, narrates history
// Per design doc v3, section 4.2 (sprint 14): loop over users and send email
for (User u : users) { mailer.send(u); }

// Good: says the thing the code cannot
// Sends sequentially: the mail gateway rate-limits per connection and rejects bursts.
for (User u : users) { mailer.send(u); }
```

```python
# Bad
# TODO see PLAN.md

# Good
# TODO: retries are not implemented; a transient 503 from the upstream fails the whole batch.
```

```java
// Bad: padded summary, tags that echo names
/**
 * This method is used for getting the user by the ID.
 * @param userId The user id.
 * @return Returns a User.
 */

// Good: one line plus the tags that add a contract fact
/**
 * Retrieves the user with the given identifier.
 *
 * @param userId must not be {@code null}
 * @return the matching user, or {@code null} if none has that identifier
 */
```

## Reviewing comments

When asked whether comments are adequate, list the gaps and the violations and stop. Edit only when asked.
