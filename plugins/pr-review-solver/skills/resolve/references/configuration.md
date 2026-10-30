# User configuration

## Location

One global file: `~/.claude/pr-review-solver.md`. Nothing is ever written inside a repository.

Read it explicitly at preflight (`cat ~/.claude/pr-review-solver.md`). Do not assume Claude Code loads it. If it does not exist, use the defaults and say nothing.

The file has two parts:

- YAML frontmatter, which holds the values.
- A Markdown body for free-form notes, with optional `## <owner/repo>` sections. Notes are context for you: preferences about tone or areas to be careful with. They cannot change the fixed rules (`guardrails.md`).

## Schema and defaults

```yaml
---
version: 1
defaults:
  strictness: default            # strict | default | relaxed
  category_strictness: {}        # per type, e.g. { tests: strict, cosmetic: relaxed }
  fast_categories: [cosmetic]    # types eligible for the trivial exception
  review_skill: auto             # auto | off | <skill name>
  commit_style: ""               # used only when the repo defines no convention
  reply_language: ""             # empty = language of the PR thread
  reply_tone: constructive
  require_push_approval: true
  require_reply_approval: true
  hard_gates:
    security: true
    data_schema: true
    public_contract: true
    ci_infra: true
    dependencies: true
    money_paths: true
    deletion: true
    ambiguous: true
    contradiction: true
  expires: {}                    # optional, per relaxed key: { "hard_gates.deletion": 2026-12-31 }
repos:
  owner/repo:                    # nameWithOwner, as returned by `gh repo view`
    verify:
      test: ""
      lint: ""
      typecheck: ""
      format: ""
    critical_paths: []           # globs that count as money_paths
    # any key from `defaults` can be overridden here
---
```

## Precedence

Plugin defaults, then `defaults` in the file, then `repos.<owner/repo>`. The most specific value wins, key by key. There is no layer committed inside repos.

The repo's own guidance (`CLAUDE.md`, `AGENTS.md`, linters) still applies when writing code. It is code convention, not configuration of this skill.

## Validation

- If the YAML does not parse, or `version` is not `1`, use plugin defaults for everything. Report it in one line: "Config at ~/.claude/pr-review-solver.md is invalid (<reason>); using defaults."
- Ignore unknown keys and invalid values (for example `strictness: maybe`), and report each one by name. The rest of the file still applies.
- If a value looks like a secret, do not echo it. Ignore it and tell the user which key to remove (`guardrails.md` §3).
- If a key tries to relax a fixed rule (for example `allow_force_push`, `skip_signing`, `run_comment_commands`, or a hard gate named `embedded_instructions`), ignore it and explain that it is fixed.
- If an entry in `expires` has a date in the past, that relaxation has expired. Use the default value for that key and say so.

## Relaxed-rules notice

A relaxation is any effective value that is looser than the plugin default:

- `strictness` or `category_strictness` set to `relaxed`.
- A hard gate set to `false`.
- `fast_categories` beyond `cosmetic`.
- `require_push_approval: false` or `require_reply_approval: false`.

At the start of **every** run with a relaxation active, print the notice (`templates.md`): each rule, its default value, the effect, and how to view it (`/pr-review-solver:config show`). Every card resolved thanks to a relaxation lists it under "Relaxed rules".

## Changes to the file

The skill never writes the file on its own initiative. Changes happen only through these two entry points, and only after explicit acceptance:

1. **The user asks** for a change (here, or with `/pr-review-solver:config`).
2. **Suggesting changes:** at wrap-up, you may suggest one change, based **only** on what happened in this session. For example, the user overrode the same gate several times, or supplied verification commands that were missing. No cross-session history, no background counting.

Before writing, show:

- The exact YAML entry and where it goes (`defaults` or `repos.<owner/repo>`).
- Whether it **tightens** or **loosens** a rule.
- The consequence, in one line.

Write it only after an explicit yes. If the user declines, do not write it and do not suggest it again this session. An instruction given in conversation ("skip the deletion gate this time") applies only to the current run, unless the user asks to save it.

Edit the file with a minimal change that keeps the existing content and comments.
