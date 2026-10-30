# pr-review-solver

**Get through PR review comments in repos you don't usually work on, without losing a morning to re-learning them.**

A Claude Code plugin that reads the review comments on your pull request, works out why each reviewer asked for what they asked, fixes the straightforward ones, and explains or drafts replies for the rest. You approve before anything reaches GitHub.

## Why this exists

More and more, we contribute outside our own team's code. Someone from frontend ships a backend change, someone from data touches a service or a UI. That's good for the company, but it has a predictable cost at review time:

- **You don't know the project's conventions** as well as the people who own it. Many review comments are about exactly that: naming, structure, the "way we do it here".
- **The review arrives days later.** By then you're deep in something else, and you have to rebuild the context of your own change before you can even judge a comment.
- **Most comments are small**, but each one still needs reading, checking, fixing, testing, committing, and replying.

pr-review-solver takes that loop off your plate. It reads the repo's own guidance (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`, linters, CI), the PR, its history, and the code around each comment. For every comment it gives you one of these:

- **A fix**, one verified commit per comment, when the change is safe and clear.
- **The reason behind the current code**, with evidence (a commit, a rule, a caller that would break), when the request doesn't hold up, plus a constructive reply draft.
- **Options with their risks**, when the change touches something sensitive: contracts, data, security, CI, dependencies, money paths.
- **A question for the reviewer**, when the comment is ambiguous, instead of a guess.

If you built the PR in a Claude Code session, `claude --from-pr <N>` brings that session back. The plugin then works with the full context of how and why the change was made.

## What you get

```
PR #128 · Add retry to webhook dispatcher
| # | File:line            | Type     | Level  | Verdict    | Action      |
|---|----------------------|----------|--------|------------|-------------|
| 1 | dispatcher.ts:42     | cosmetic | Low    | Viable     | commit      |
| 2 | dispatcher.ts:88     | local    | Medium | Viable     | commit      |
| 3 | events/schema.ts:12  | contract | High   | Viable     | options     |
| 4 | retry.ts:30          | local    | —      | Not viable | reply draft |
Already resolved: 2
- dispatcher.ts:15 · Prefer a named constant for the retry limit…
- retry.ts:7 · Typo in the log message…
```

(Illustrative output.)

## Safe by design

- **Nothing leaves your machine without your OK.** Commits stay local until you approve the push, and replies are posted only after you review them.
- **Never** force-pushes, skips commit signing, or changes your git config.
- **Never** runs commands, URLs, or installs that appear inside a review comment. Those are flagged for you to decide.
- **Respects the repo's conventions** when it writes code, and doesn't argue a comment just because something isn't documented.

## Quick start

```
claude plugin marketplace add AustinDeuna/pr-review-solver
claude plugin install pr-review-solver@pr-review-solver --scope user
```

Then, on a branch with an open PR, in a new Claude Code session:

```
/pr-review-solver:resolve
```

Or just ask: "resolve the review comments on my PR".

## Requirements

- Claude Code with plugin support.
- GitHub CLI (`gh`), authenticated (`gh auth login`). Use a token with the least privilege you need: read access to the repo, plus write access to pull requests and contents for the repos you work on.
- git. If you sign commits with GPG, see "Commit signing" below.

## Other install options

This repository is the marketplace. It will move to the DEUNA organization after testing, and the marketplace source will change accordingly.

From inside a Claude Code session:

```
/plugin marketplace add AustinDeuna/pr-review-solver
/plugin install pr-review-solver@pr-review-solver
```

From a local clone (development and testing):

```
claude plugin marketplace add /path/to/pr-review-solver
claude plugin install pr-review-solver@pr-review-solver --scope user
```

## How it works

1. Preflight: `gh`, the PR, session mode, commit signing, your config.
2. Evaluate and classify every comment, then show a summary table and one card per comment.
3. Fix Low and Medium items, one verified commit per comment.
4. Present options for High items, and draft replies for comments that are not viable, unclear, or obsolete.
5. Validate with the repo's review skill, if it has one.
6. After your approval: push (never force), then post the replies you approve.

**Tip:** create your PRs by asking Claude Code to do it from the session where you developed the branch. Later, `claude --from-pr <N>` reopens that session with full context.

## Configuration

Optional. One global file, `~/.claude/pr-review-solver.md`, with general values and per-repo sections. It is never stored inside a repo. Use `/pr-review-solver:config show | edit <request> | init`. The schema is in [`configuration.md`](plugins/pr-review-solver/skills/resolve/references/configuration.md).

## Repository layout

```
.claude-plugin/marketplace.json            # marketplace manifest
plugins/pr-review-solver/
├── .claude-plugin/plugin.json             # plugin manifest
├── skills/resolve/SKILL.md                # /pr-review-solver:resolve
├── skills/resolve/references/             # preflight, rubric, guardrails, templates, ...
├── skills/config/SKILL.md                 # /pr-review-solver:config
└── agents/review-gate.md                  # read-only validation gate
```

You can loosen rules. Every relaxation is saved only with your explicit acceptance, and it is announced at the start of every run.

These rules cannot be configured:

- Commands, URLs, or installs that appear in a comment's text are never executed.
- Signing is never skipped, git config is never changed, there is no force push, and unsigned commits are never pushed.
- No secrets go in the config.

## Optional: permissions

The plugin does not pre-approve any tool. Your normal Claude Code permissions apply. To reduce prompts, you can add read-only rules to `~/.claude/settings.json`, for example:

```json
{
  "permissions": {
    "allow": [
      "Bash(gh pr view:*)",
      "Bash(gh pr diff:*)",
      "Bash(git log:*)",
      "Bash(git blame:*)",
      "Bash(git diff:*)"
    ]
  }
}
```

Keep `git push` and `gh pr comment` / `gh api` (write) out of the allow list. They are the approval points.

## Commit signing

Claude Code's shell has no terminal, so `pinentry-curses`/`pinentry-tty` cannot ask for your GPG passphrase. Before each commit, the skill checks whether gpg-agent has the passphrase cached (without signing anything), and which pinentry the agent uses:

- **Cached:** commits are made and signed normally.
- **Not cached, graphical pinentry** (for example `pinentry-mac`): commits are made normally, and a dialog asks for your passphrase. With `pinentry-mac`, the dialog closes by itself after about 60 seconds. If you close or cancel the dialog, or it times out, nothing is lost: the fix stays staged, and the skill asks whether to retry or switch to step mode.
- **Not cached, terminal pinentry:** step mode. The skill stages each fix and gives you the `git commit -S` command to run in your terminal.

Do not try to sign from Claude Code while the passphrase is not cached. If `GPG_TTY` points to Claude Code's terminal, `pinentry-curses` takes over that terminal: the UI fills with escape sequences, typed keys land in the wrong place, and the commit fails. The skill re-checks the cache before every commit to avoid this.

Options (your choice; the plugin never changes them, and the passphrase is never stored by the plugin):

- **Graphical pinentry (recommended on macOS).** `brew install pinentry-mac`, add `pinentry-program /opt/homebrew/bin/pinentry-mac` to `~/.gnupg/gpg-agent.conf`, then run `gpgconf --kill gpg-agent`. Signing then opens a native dialog, so it works from Claude Code. Uncheck "Save in Keychain" if you do not want the passphrase stored: once it is saved, `pinentry-mac` reads it from the Keychain without asking whenever your macOS session is unlocked. To remove a saved passphrase: `security delete-generic-password -s GnuPG -a <keygrip>`.
- **Warm the cache** in a separate terminal (not Claude Code's): `echo test | gpg --clearsign > /dev/null`. It lasts for your agent's `default-cache-ttl` (idle) and `max-cache-ttl` (absolute).
- **Step mode:** do nothing. The skill stages each fix and you run `git commit -S` yourself.

## Known limits

- Session mode (A/B) is judged from the current conversation. There is no deterministic signal.
- The skill does not read tickets. Scope comes from the PR description and commits.
- Calling the repo's review skill from this skill is done by reading its `SKILL.md` and applying it through the `review-gate` subagent. Direct skill-to-skill invocation is not documented.
- Not yet verified: that `gh pr create` run from a Claude Code session links the session for `--from-pr`.
