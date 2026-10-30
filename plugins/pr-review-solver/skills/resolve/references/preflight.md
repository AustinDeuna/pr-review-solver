# Preflight

Run these steps in order. At the first blocking failure, stop, tell the user the exact fix, and do nothing else.

## 1. GitHub CLI

```bash
command -v gh && gh auth status
```

- `gh` missing: "Install the GitHub CLI (https://cli.github.com), then run this again."
- Not authenticated: "Run `! gh auth login` (or `gh auth login` in your terminal), then run this again."
- Never print, ask for, or handle a token.

## 2. Repository and pull request

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
gh pr view --json number,url,title,body,state,isDraft,author,headRefName,baseRefName,headRepository,headRepositoryOwner,isCrossRepository
gh repo view --json nameWithOwner --jq .nameWithOwner
gh api user --jq .login
```

- Not a git repo, or detached HEAD: stop.
- `gh pr view` fails because no PR exists: "Branch `<branch>` has no pull request. Nothing to resolve." Stop. Any other error: show it verbatim and stop.
- PR state is not `OPEN`: stop.
- PR author differs from the current user: warn that this is not the user's PR and ask whether to continue.
- Record PR number, URL, head and base branches, `nameWithOwner` (the config repo key), and whether the head lives in a fork (`isCrossRepository`).

Working tree and sync:

```bash
git status --porcelain
git fetch origin <headRefName> <baseRefName>
git rev-list --left-right --count HEAD...origin/<headRefName>
```

For a cross-repository PR, fetch from the remote that points to the head repository instead of `origin`.

- Uncommitted or untracked changes: stop and list them. The user commits or stashes them; you do not, because per-comment commits must not mix with unrelated work.
- Local branch behind the remote: stop and ask the user to pull. Do not pull, rebase, or reset yourself.
- Local branch ahead of the remote: note the unpushed commits; they will be part of the push and must be shown at approval time.

## 3. Mode A or Mode B

- **Mode A:** the current conversation already contains the development of this branch (earlier turns wrote or discussed these commits).
- **Mode B:** no trace of that development in the current conversation. When in doubt, Mode B.

In Mode B, print this once per session, without asking and without waiting (use the real PR number):

> Session without development context: I'm classifying comments using only the PR and the diff. If you created this PR from a Claude Code session, next time open it with `claude --from-pr <N>` to get that context.

The mode never changes any classification level. In Mode A, print nothing.

## 4. Commit signing

```bash
git config --get commit.gpgsign
git config --get gpg.format
git config --get user.signingkey >/dev/null && echo set || echo unset
```

Report only `set`/`unset` for the signing key, never its value.

- `commit.gpgsign` not `true`: signing is not configured. Say so once. Do not enable it. Commits will be unsigned, as they already are for this user.
- Format `openpgp` (or unset): check whether the agent can sign without a prompt. This is read-only and does not sign anything:

  ```bash
  K="$(git config --get user.signingkey)"
  for g in $(gpg --batch --with-colons --with-keygrip --list-secret-keys "$K" 2>/dev/null \
      | awk -F: '$1=="sec"||$1=="ssb"{cap=$12} $1=="grp" && cap ~ /s/ {print $10}'); do
    gpg-connect-agent "KEYINFO $g" /bye
  done
  ```

  In a line `S KEYINFO <grip> <type> <serial> <idstr> <cached> <protection> ...`, the passphrase is available when `<cached>` = `1` or `<protection>` is not `P`.

  Then find out which pinentry the agent uses:

  ```bash
  grep -E '^[[:space:]]*pinentry-program' ~/.gnupg/gpg-agent.conf 2>/dev/null
  ```

  It is **graphical** if the program's basename is `pinentry-mac`, `pinentry-gnome3`, `pinentry-qt`, `pinentry-gtk`/`pinentry-gtk-2`, or `pinentry-w32`. It is a **terminal** pinentry if the line is missing, or the program is `pinentry`, `pinentry-curses`, or `pinentry-tty`.

  | Passphrase | Pinentry | Mode |
  |---|---|---|
  | Available | any | normal |
  | Not available | graphical | normal (a dialog opens on each commit that needs the passphrase) |
  | Not available | terminal | step |
  | Check failed | any | step |

  Step mode lasts for the whole run (`commits-gpg.md`). With a terminal pinentry, never attempt a signed commit: pinentry would take over Claude Code's terminal.
- Format `ssh` or `x509`: normal mode. If the first commit fails with a signing error, switch to step mode.

When step mode is selected, tell the user once:

> Your GPG passphrase isn't cached and your pinentry needs a terminal, which my shell doesn't have, so I'll stage each fix and give you the exact commit command to run. Optional (your call): sign anything once in a separate terminal so gpg-agent caches the passphrase for its cache TTL, or switch to a graphical pinentry (on macOS: `brew install pinentry-mac`, add `pinentry-program /opt/homebrew/bin/pinentry-mac` to `~/.gnupg/gpg-agent.conf`, run `gpgconf --kill gpg-agent`). I will not change any git or gpg setting.

## 5. Configuration

Load the user config (`configuration.md`).
