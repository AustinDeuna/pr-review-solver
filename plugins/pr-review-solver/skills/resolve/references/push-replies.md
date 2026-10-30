# Push and replies

## Before asking for approval

1. **Signatures** (only if signing is configured). Every commit that is new to the remote must be signed, merge commits included:

   ```bash
   for c in $(git rev-list <headRefName> --not --remotes); do
     git cat-file commit "$c" | grep -q '^gpgsig' || echo "UNSIGNED $c"
   done
   ```

   Commits that already exist on any remote branch (for example, a parent's commits brought in by a stack merge) are not re-checked. If any new commit is unsigned, do not push that branch. Tell the user which commits.
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

## PR summary comment

After replies, offer the optional PR summary comment (`templates.md`, "PR summary comment"). It needs explicit approval, regardless of `require_reply_approval`.

## Threads

Never resolve threads. Resolving is the reviewer's decision. The only exception is a thread the user opened, and only if they ask.
