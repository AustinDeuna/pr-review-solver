# Stacked pull requests

A stack is a chain of open PRs where each PR's base branch is the head branch of another open PR. This file wraps workflow steps 3-8 so that a whole stack is resolved bottom-up, and each PR's fixes are merged into the PRs above it.

## 1. Detection (right after preflight §2)

```bash
gh pr list --author @me --state open --limit 100 \
  --json number,title,url,headRefName,baseRefName,isCrossRepository
```

1. PR X is the **parent** of PR Y when `Y.baseRefName == X.headRefName`.
2. From the current PR, walk down through parents to the root, a PR whose base is not the head of another open PR.
3. Collect every descendant of that root. The stack is the root plus all its descendants.
4. If the stack has a single PR, there is no stack: continue with the normal single-PR flow.
5. If any PR in the stack is cross-repository, stack mode is not available. Say so and continue with the single-PR flow.

Order: parents before children (topological order). Siblings are ordered by PR number.

If the stack has more than one PR, show it (`templates.md`, "Stack overview") with the unresolved comment count for each PR. Then ask once:

> PR #<N> is part of a stack of <k> PRs. Resolve the whole stack bottom-up (fixes are merged upward), or only PR #<N>?

If the user chooses **only this PR**, run the single-PR flow, and warn that the PRs above it will not receive the fixes until they are merged up.

## 2. Preconditions (checked before changing anything)

1. The working tree is clean (preflight). It must stay clean between branches.
2. Fetch every head in the stack: `git fetch origin <head1> <head2> ...`.
3. For each branch:
   - If there is no local branch, `git switch <head>` creates one that tracks the remote.
   - If the local branch is behind its remote (`git rev-list --left-right --count <head>...origin/<head>`), stop the whole run before any change and list the branches the user must pull.
4. **Merge policy.** If the repo's guidance (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`) requires rebasing feature branches, or forbids merge commits in them, do not merge upward. Explain that the upper branches would need a rebase plus a force push, which this skill never does. Offer either to resolve each PR without propagation, or to stop.
5. Record the starting branch. Switch back to it at the end, or whenever the run stops.

## 3. Flow

1. **Context and classification for the whole stack.** Gather context (`context.md`) and evaluate and classify the comments of every PR in the stack. Show one summary table grouped by PR, with each PR's "Already resolved" list (`templates.md`).
2. **For each PR, in order:**
   1. Switch to its branch: `git switch <head>`.
   2. If its parent in the stack has commits this branch does not contain (`git rev-list --count <head>..<parent>` > 0), merge the parent into it (§4).
   3. After a merge, re-evaluate this PR's comments that point to lines the merge changed (`git diff <pre-merge-sha>..HEAD -- <path>` overlaps the comment's lines). Note "re-evaluated after merge" in their cards.
   4. Resolve this PR's Low and Medium items (`commits-gpg.md`).
   5. Run the review gate on this PR's commits from this run, excluding merge commits (`git rev-list --no-merges`).
3. **High options and reply drafts**, grouped by PR.
4. **One approval for all branches** (`templates.md`, "Stack approval"). The user can approve all of them or a subset. Push bottom-up (`push-replies.md`). A child can be pushed only if its parent was pushed in this run, or its parent had nothing new to push. Otherwise its merge would publish the parent's unpushed commits through the child.
5. **Replies**, per PR, after that PR's push (`push-replies.md`).
6. Switch back to the starting branch.

## 4. Merging a parent into a child

Announce it first (`templates.md`, "Merge announcement"), then:

```bash
git switch <child>
git merge --no-edit <parent>
```

- Merge the **local** parent branch, which contains this run's commits.
- Keep git's default merge message, unless the repo's guidance defines one for merges.
- Signing works as it does for commits (`commits-gpg.md`): cache check, dialog notice, a 5-minute tool timeout. In step mode, print `git merge -S --no-edit <parent>` for the user to run, and wait.
- **Signing fails during a merge:** git leaves the merge in progress (`MERGE_HEAD` exists and the changes are staged). Ask the user: retry with `git commit --no-edit` (the dialog opens again), step mode (the user runs `git commit -S --no-edit`), or abort with `git merge --abort`. Do not retry automatically.
- **Conflicts:** run `git merge --abort`, which returns the branch to its state before the merge. Stop propagation for this child and all its descendants. List the conflicting files, and tell the user to resolve that merge manually. PRs earlier in the order keep their local commits and remain eligible for push. Never resolve merge conflicts automatically.
- Never rebase, reset, or force push.
