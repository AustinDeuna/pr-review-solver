# Commits and signing

Process Low and Medium items one at a time, in the order of the summary table. Finish an item (edit, verify, commit) before starting the next, so that each commit contains only one comment's changes.

## Per item

1. **Edit** following the repo's guidance (`context.md` §3). Touch only what the comment needs.
2. **Verify** with the commands for its level (`rubric.md`). Show the commands and the outcome (pass/fail plus relevant output lines).
   - On failure: try once to fix it within the same scope. If it still fails, discard only this item's own edits with `git restore --source=HEAD --staged --worktree -- <files you edited>` (the tree was clean at preflight). Then reclassify the item as High, explaining what failed. Never discard anything else.
3. **Announce** the commit before making it (template in `templates.md`): the files, the proposed message, and the comment it addresses (number and link).
4. **Commit message** follows the repo's convention, deduced from `git log -20 --format=%s` and `CONTRIBUTING` (for example Conventional Commits, ticket prefixes, imperative mood). If the repo has none, use `commit_style` from the config. If that is empty too, write an imperative summary under 72 characters. Never add trailers that the repo does not use.
5. **Stage only this item's files:** `git add -- <files>`. Never `git add -A` or `git add .`.
6. **Commit** according to the signing mode (`preflight.md` §4):
   - **Normal mode:** `git commit -m "<subject>" [-m "<body>"]`. Git signs according to the user's own config. Do not pass `-S` or `--no-gpg-sign`.
   - **Step mode:** see below.
7. Record the SHA: `git rev-parse --short HEAD`.

## Signing failure in normal mode

If the commit fails with a signing error (for example "gpg failed to sign the data", "No pinentry", "Inappropriate ioctl for device", "signing failed"):

- Do not retry the commit.
- Do not disable signing, change config, or use `--no-gpg-sign`.
- Switch to step mode for this item and every remaining one.

## Step mode

For each item:

1. Leave only this item's files staged (`git add -- <files>`). Confirm with `git diff --cached --stat`.
2. Print the exact command for the user to run in their own terminal:

   ```
   git commit -S -m "<subject>" [-m "<body>"]
   ```

3. Pause. Wait until the user says the commit is done ("done", "listo", or similar).
4. Verify it: `git log -1 --format='%h %s'`, and check that the commit has a signature with `git cat-file commit HEAD | grep -q '^gpgsig'`. If the commit is missing or the wrong files went in, tell the user and stop. Do not fix it by amending.
5. Continue with the next item.

## Never

- `--no-gpg-sign`, `-c commit.gpgsign=false`, `--no-verify`, `--amend` on commits that already exist, `git rebase`, `git reset` on commits.
- Asking for or receiving a passphrase.
- Committing files from more than one comment together.
