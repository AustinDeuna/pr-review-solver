# Push and replies

## Before asking for approval

1. **Signatures** (only if signing is configured). Every commit to be pushed must be signed:

   ```bash
   for c in $(git rev-list origin/<headRefName>..HEAD); do
     git cat-file commit "$c" | grep -q '^gpgsig' || echo "UNSIGNED $c"
   done
   ```

   If any commit is unsigned, do not push. Tell the user which ones.
2. **Remote unchanged:** run `git fetch origin <headRefName>` again. If the remote moved, stop. The user integrates those changes themselves. You never rebase or force push.
3. **Show the approval request** (`templates.md`): the target branch, every commit to be pushed (including unpushed commits that existed before this run), the comment each one resolves, the verification results, the review gate findings, and the pending High items.

## Push

Only after explicit approval, unless `require_push_approval: false` is in effect for this repo. In that case, announce the push and show the relaxation.

```bash
git push origin HEAD:<headRefName>
```

For a cross-repository PR, push to the remote that points to the head repository. Never use `--force`, `--force-with-lease`, or a `+` refspec. If the push is rejected, stop and report the error verbatim.

Then confirm each SHA is on the remote:

```bash
git branch -r --contains <sha>
```

## Replies

Draft one reply per processed comment (`templates.md`):

- Resolved: cite the commit SHA that resolves it (and the adjustment, if any).
- Not viable: the analysis with cited evidence.
- Unclear: the question.
- Obsolete: only after the user confirms.
- High: only if the user picks an option or asks for a reply.

Language: `reply_language` from the config, else the language of the PR thread. Tone: `reply_tone` from the config, else constructive.

Show every draft. The user can approve all, approve some, edit, or skip. Post only what is approved, unless `require_reply_approval: false` is in effect for this repo (announce it and show the relaxation). Disputed replies (Not viable, Unclear, contradictions) always need approval, whatever the config says.

Never post a reply that cites a SHA before that SHA is on the remote branch.

Posting:

```bash
# Reply in an inline review thread (comment_id = databaseId of the thread's first comment)
gh api repos/<owner>/<repo>/pulls/<N>/comments/<comment_id>/replies -F body=@<file>
# Top-level conversation comment
gh pr comment <N> --body-file <file>
```

Write each body to a temporary file under the session scratchpad and pass the file, never inline shell-escaped text.

## Threads

Never resolve threads. Resolving is the reviewer's decision. The only exception is a thread the user opened, and only if they ask.
