# Templates

Fill in the placeholders. Keep the structure; trim empty fields. These templates define structure only: render every label in the user's session language, and write for someone who has never used this tool (`SKILL.md`, "Style").

## Run header

```
PR #<N> · <title> · <url>
Branch <head> → <base>
Session context: <yes, this session built the branch | no, working from the PR and diff only>
Commits: <I commit and sign them | you run each commit command (your signing needs a terminal) | unsigned (signing not configured)>
Review check before push: <skill name (path) | built-in checklist>
How I verify: compile=<cmd> · tests=<cmd> · lint=<cmd|none found> (sources: <files>)
```

## Legend (print once per run, right before the first summary table)

```
Risk
  Low     I fix it, then check with formatter/linter (and compile if a name changes).
  Medium  I fix it, then run the affected tests. Behavior changes are explained.
  High    I don't change code. You get 2-3 options with their risks to choose from.
Verdict
  Doable                 The request makes sense and breaks nothing.
  Doable, with a tweak   The reviewer's point is right; I'd do it slightly differently (explained).
  Not advisable          It would break something or goes against a rule/decision; I draft a reply with evidence.
  Unclear                I draft a question for the reviewer instead of guessing.
  Outdated               The code changed after the comment; you confirm before I reply.
  Waiting on #<n>        Depends on what you decide for comment #<n>.
```

## Relaxed-rules notice

```
Relaxed rules active (view with /pr-review-solver:config show):
- <key>: <value> (default <default>). Effect: <one line>. Scope: <defaults|owner/repo>. <Expires <date>.>
```

## Summary table

```
| # | Where | What it asks | Risk | Verdict | Next step |
|---|-------|--------------|------|---------|-----------|
| 1 | src/a.ts:42 | Rename `tmp` to `retryCount` | Low | Doable | I fix it |
| 2 | api/b.py:10 | Change the event payload field | High | Doable | You choose an option |
| 3 | lib/c.go:7 | Inline the helper | — | Not advisable | Reply draft |
| 4 | lib/c.go:20 | Update the comment wording | Low | Waiting on #2 | After you decide #2 |
```

Use "Waiting on #<n>" instead of raising a simple item to High just because its exact change depends on another comment's decision. Classify the item on its own merits.

Right after the table, list resolved threads. One line each, no card, no classification:

```
Already resolved: <count>
- src/a.ts:12 · Prefer a named constant for the retry limit…
- lib/c.go:30 · Typo in the error message…
```

If there are none, print `Already resolved: 0` and no list.

## Comment card

Lead with what a reader needs to decide. Technical classification goes last, in one line.

```
#<n> · <short title of the request>
<author>[ (bot)] · <file>:<line> · <link>

What they ask:     <one plain sentence>
Why they ask:      <the reviewer's underlying concern, one sentence>
What I propose:    <the concrete action: "I'll change X and run Y" | "Don't change code; reply explaining Z" | "Ask the reviewer W" | "Options below">
Why:               <2-4 bullets with evidence: file:line, commit SHA, quoted rule>
What could break:  <plain-language risk, or "nothing expected">
Your decision:     <only when something needs the user's choice>

Risk <Low|Medium|High> (<reason in a few words, e.g. "touches the database schema">) · <Verdict> · Tests: <yes|partial|none> · Confidence: <high|medium|low>
<Tweak: what differs from the literal request and why — only for "Doable, with a tweak">
<Relaxed rules: … — only if this item was resolved because of a relaxation>
```

Example:

```
#3 · Rename the legacy_mode column to run_mode
reviewer-a · ConfigLoader.scala:120 · <link>

What they ask:     Rename the database column legacy_mode to run_mode.
Why they ask:      Keep the column name consistent with the new name in code.
What I propose:    Don't do it in this PR; reply proposing a follow-up with the migration. Options below.
Why:               - Needs a migration in the schema repo; existing migration files can't be edited.
                   - Earlier in this PR we decided to keep the column name (abc1234).
What could break:  If code reading run_mode deploys before the migration, the query fails.
Your decision:     Rename now (with the migration), or keep it and reply.

Risk High (database schema change) · Doable · Tests: none for these queries · Confidence: high
```

## Commit announcement

```
Commit for #<n> (<link>)
Files:   <list>
Message: <subject>
         <body, if any>
Verified: <commands> → <pass>
<Medium only> Behavior change: <what changed>
```

## Step-mode instruction

```
Staged for #<n>: <files>
Run in your terminal:
  git commit -S -m "<subject>"
Tell me "done" when it's committed.
```

## High item options

```
#<n> · <why it is High: gate/coverage/confidence>
Option A: <change>. Risk: <…>. Validate: <full suite + specific checks>.
Option B: <change>. Risk: <…>. Validate: <…>.
Option C: reply without code change: <summary>.
```

## Approval request

```
Ready to push to <remote>/<head>:
  <sha> <subject>   → #<n>
  ...
Pre-existing unpushed commits: <list|none>
Signatures: all signed | not configured
Review gate: <skill|internal rubric> → <no findings | findings>
Pending High items: <list>
Push? (yes / no)
```

## Stack overview

```
Stack (bottom → top):
  #101 feat/a  → main     · 3 unresolved · 1 resolved
  #102 feat/b  → feat/a   · 2 unresolved · 0 resolved   ← current
  #103 feat/c  → feat/b   · 0 unresolved · 2 resolved
  #104 feat/d  → feat/c   · 1 unresolved · 0 resolved   (author: other-user, not modified)
Detected via: gh base/head links | session context (verified with gh)
```

For a stacked run, the summary table is repeated once per PR, under a `PR #<N> · <head>` heading, each with its own "Already resolved" list.

## Merge announcement

```
Merge #<parent> (<parent head>) into #<child> (<child head>)
Brings: <k> commits (<short SHAs>)
Command: git merge --no-edit <parent head>
```

## Stack approval

```
Ready to push (bottom → top):
  #101 feat/a:  <sha> <subject> → comment #<n>; ...
  #102 feat/b:  <sha> Merge branch 'feat/a' into feat/b; <sha> <subject> → comment #<n>
  #103 feat/c:  <sha> Merge branch 'feat/b' into feat/c
Signatures: all new commits signed | not configured
Not pushable: <branch: reason (conflict, parent not pushed)> | none
Review gate: <per PR result>
Pending High items: <per PR>
Push all, some (list), or none?
```

## Replies

Resolved:
> Done in <sha>: <one-line description>.

Resolved with adjustment:
> Addressed in <sha>. I went with <variant> instead of <literal request> because <reason, with evidence>. Let me know if that doesn't cover your concern.

Not viable:
> I looked into this: <finding with evidence, e.g. `path/file.py:120` calls this with …, or this was decided in <sha> because …>. Changing it as suggested would <consequence>. Does <alternative or existing behavior> cover your case?

Unclear:
> Quick question before I change this: <specific question>?

Obsolete:
> This line changed in <sha> after your comment; <how the concern is now handled | the concern no longer applies>. Happy to revisit if I missed something.

Contradiction:
> This seems to conflict with <other comment link | PR description>: <summary>. Which direction should I take?
