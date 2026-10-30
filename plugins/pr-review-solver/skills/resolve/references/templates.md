# Templates

Fill in the placeholders. Keep the structure; trim empty fields.

## Run header

```
PR #<N> · <title> · <url>
Branch <head> → <base> · Mode <A|B> · Signing <normal|step|not configured>
Review skill: <name> (<path>) | none, using internal rubric
Verification: test=<cmd> (<source>) · lint=<cmd> (<source>) · typecheck=<cmd|none>
```

## Relaxed-rules notice

```
Relaxed rules active (view with /pr-review-solver:config show):
- <key>: <value> (default <default>). Effect: <one line>. Scope: <defaults|owner/repo>. <Expires <date>.>
```

## Summary table

```
| # | File:line | Type | Level | Verdict | Action |
|---|-----------|------|-------|---------|--------|
| 1 | src/a.ts:42 | cosmetic | Low | Viable | commit |
| 2 | api/b.py:10 | contract | High | Viable | options |
| 3 | lib/c.go:7 | local | — | Not viable | reply draft |
```

## Comment card

```
#<n> · <file>:<line> · <Level> · <Verdict>   <author>[ (bot)] · <link>
Asks:          <one line>
Reviewer's reason: <one line>
Type:          <type>          Scope: <files>; outside refs: <none|list>
Runtime change: <yes|no>       Coverage: <yes|partial|no|unknown>
Confidence:    <high|medium|low>   Reversible: <yes|no>
Tests to run:  <commands|none>
Risk:          <one line>
Evidence:      <file:line | SHA | quoted rule>
Action:        <commit | options | reply draft | question | confirm obsolete>
Adjustment:    <only for Viable with adjustment: what differs and why>
Relaxed rules: <only if this item was resolved because of a relaxation>
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
