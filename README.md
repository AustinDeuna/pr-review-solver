# pr-review-solver

Claude Code plugin that helps you resolve the review comments left on your pull request. It evaluates each comment against the code and the repo's own conventions, classifies its risk, fixes the safe ones with one signed local commit per comment, proposes options for the risky ones, drafts replies, and pushes or posts only after you approve.

## Requirements

- Claude Code with plugin support.
- GitHub CLI (`gh`), authenticated (`gh auth login`). Use a token with the least privilege you need: read access to the repo, plus write access to pull requests and contents for the repos you work on.
- git. If you sign commits with GPG, see "Commit signing" below.

## Install

From GitHub (this repository is the marketplace):

```
/plugin marketplace add AustinDeuna/pr-review-solver
/plugin install pr-review-solver@pr-review-solver
```

The repository will move to the DEUNA organization after testing; the marketplace source will change accordingly.

From a local clone (development and testing):

```
claude plugin marketplace add /path/to/pr-review-solver
claude plugin install pr-review-solver@pr-review-solver --scope user
```

## Usage

On a branch that has an open PR:

```
/pr-review-solver:resolve
```

Or ask in plain language, for example "resolve the review comments on my PR". The skill can also be triggered automatically.

Flow:

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
