---
name: config
description: View or edit the pr-review-solver user configuration at ~/.claude/pr-review-solver.md (show, edit, init).
argument-hint: "[show | edit <request> | init]"
disable-model-invocation: true
---

# pr-review-solver configuration

The schema, defaults, precedence, validation, and relaxation rules are defined in `../resolve/references/configuration.md`. Read that file first. The fixed rules in `../resolve/references/guardrails.md` cannot be configured.

The only file this skill may write is `~/.claude/pr-review-solver.md`. Never write inside a repository.

## `show` (default when no argument is given)

1. Read `~/.claude/pr-review-solver.md`. If it is missing, say so and show the plugin defaults.
2. If inside a git repo with a GitHub remote, resolve `nameWithOwner` (`gh repo view --json nameWithOwner --jq .nameWithOwner`) and show the effective config for that repo. Show each key's value and its source (plugin default, `defaults`, or `repos.<owner/repo>`).
3. List active relaxations: key, value, default, effect, scope, and expiry.
4. Report validation problems (invalid YAML, unknown keys, invalid values, expired relaxations, values that look like secrets). Never echo a value that looks like a secret.

## `edit <request>`

1. Translate the request into the minimal YAML change and decide where it goes: `defaults`, or `repos.<owner/repo>` (default: the current repo, if the request is about it).
2. If the request targets a fixed rule (signing, git config, force push, executing comment text, secrets), refuse and explain in one or two sentences why it is fixed.
3. Show:
   - the exact entry,
   - whether it **tightens** or **loosens** a rule,
   - its consequence in one line,
   - and, for a loosening, offer an optional expiry date.
4. Write only after an explicit yes. If the user declines, write nothing.
5. Keep the rest of the file, including its comments, unchanged.

## `init`

1. If the file exists, do not overwrite it. Offer `show` or `edit` instead.
2. Otherwise, show the file to be created (frontmatter with `version: 1`, an empty `defaults: {}` and `repos: {}`, and a short notes section) and create it only after an explicit yes.
