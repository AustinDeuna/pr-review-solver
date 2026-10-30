# Evaluating a comment (before classifying it)

Central rule: the reviewer has a reason. Find it, then check whether the requested change addresses it without breaking anything. Do not apply a change just because it was asked for. Do not dismiss one without evidence.

## Trivial exception

Apply without deep analysis only when **all** of these hold:

- The change is local to one file and has no runtime effect.
- Nothing outside the file references what changes.
- A formatter, linter, or compiler/typechecker can verify it.
- The comment type is in `fast_categories` (default: `cosmetic`).

Examples: renaming a local variable, fixing a typo in a comment or a string that is not user-facing, following an existing format convention.

## Analysis (everything else)

Check, citing evidence for each point:

1. **Reason.** What problem the reviewer is pointing at, in one sentence. If you cannot state it, the verdict is Unclear.
2. **Callers and references.** Search the repo for every use of the symbols that would change (definitions, imports, string references, configs, tests). A change that breaks a caller is not viable as written.
3. **Tests.** Which tests cover the touched code, and whether they would still pass.
4. **Contracts.** Exported signatures, APIs, events, schemas, serialized formats, database objects.
5. **Conventions.** Does the change follow or contradict `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, or the linter config? Quote the rule.
6. **Earlier decisions.** Do the commits, the PR description, or `git blame` show the current code was a deliberate choice? Cite the SHA or quote.
7. **Scope.** Is the request inside the scope stated by the PR description and commits?
8. **Staleness.** Has the line changed since the comment (`isOutdated`, or the diff hunk no longer matches the file)?
9. **Conflicts.** Does it contradict another comment or the PR description?

The absence of documentation in someone else's repo is never a reason to refute a comment.

## Verdicts

| Verdict | When | Action |
|---|---|---|
| Viable | The change addresses the reason and breaks nothing. | Apply according to its level. |
| Viable with adjustment | The reason is valid, but the literal change breaks something or contradicts a convention. A variant fixes both. | Apply the variant according to its level, and explain the difference in the card and the reply. |
| Not viable / counterproductive | The change breaks a caller, contradicts a convention or an earlier decision, or falls outside scope, and no variant satisfies the reason. | Do not change code. Draft a reply with the analysis and cited evidence (`file:line`, SHA, quoted rule). |
| Unclear | You cannot determine the reason or the exact request. | Draft a question to the reviewer. Do not guess. |
| Obsolete | The line changed after the comment, and the concern no longer applies or cannot be located. | Mark it and ask the user to confirm before replying. |

If the exact change for a comment depends on how another comment is decided, keep its own verdict and level, and mark it "Waiting on #<n>". Do not raise it to High only because of the dependency. Resolve it after the user decides #<n>.

Two comments that contradict each other, or a comment that contradicts the PR description, get a hard gate (`rubric.md`). Draft a reply that lays out the conflict and asks which way to go.

## Reply tone

- Constructive and specific. Example: "This was decided in `abc1234` because <reason>; does that cover your case?"
- Never confrontational, never dismissive, no sarcasm.
- Replies are drafts. They are never posted without the user's approval (unless the user config relaxes it, see `push-replies.md`).
