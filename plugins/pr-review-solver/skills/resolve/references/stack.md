# Stacked pull requests

A stack is a chain of open PRs where each PR's base branch is the head branch of another open PR. This file wraps workflow steps 3-8 so that a whole stack is resolved bottom-up, and each PR's fixes are merged into the PRs above it.

## 1. Detection (right after preflight §2, always, starting from the PR given or the current branch's PR)

GitHub has no built-in "stack" object. The signal is the relationship between branches: PR X is the **parent** of PR Y when `Y.baseRefName == X.headRefName`. Walk both directions from the starting PR, with any author:

```bash
F=number,title,url,state,author,headRefName,baseRefName,isCrossRepository
# Children: open PRs whose base is this PR's head branch
gh pr list --state open --base <headRefName> --json $F
# Parent: an open PR whose head is this PR's base branch (skip when the base is the default branch)
gh pr list --state open --head <baseRefName> --json $F
```

1. Repeat for every PR you find, until no new PRs appear. Down to the root: a PR whose base is not the head of another open PR (usually the default branch). Up through every descendant.
2. **Session memory (Mode A only).** If the current conversation shows that this session created or worked on other PRs or branches in a chain (for example, it opened PR #101 on `feat/a`, then PR #102 on `feat/b` based on `feat/a`), add them as candidates. Verify each one with `gh pr view <n> --json $F`. Only links that `gh` confirms are part of the stack. A candidate that is merged, closed, or no longer linked is reported and left out. Never read session transcripts on disk.
3. If the stack has a single PR, there is no stack: continue with the normal single-PR flow.
4. If any PR in the stack is cross-repository, stack mode is not available. Say so and continue with the single-PR flow.
5. A PR in the stack whose author is not the user is shown, but it never receives commits, merges, or pushes. Propagation stops below it.

Order: parents before children (topological order). Siblings are ordered by PR number.

Show the stack (`templates.md`, "Stack overview") with the unresolved and resolved comment counts for each PR. Then ask once:

> PR #<N> is part of a stack of <k> PRs. How should I proceed?
> 1. Resolve every PR in the stack bottom-up, and merge the fixes upward.
> 2. Resolve only #<N>, then merge its fixes into the PRs above it.
> 3. Resolve only #<N>, without propagating.

Recommend option 1 when other PRs in the stack have unresolved comments, and option 2 otherwise. With option 3, warn that the PRs above will not receive the fixes until they are merged up.

## 2. Preconditions (checked before changing anything)

1. The working tree is clean (preflight). It must stay clean between branches.
2. Fetch every head in the stack: `git fetch origin <head1> <head2> ...`.
3. For each branch:
   - If there is no local branch, `git switch <head>` creates one that tracks the remote.
   - If the local branch is behind its remote (`git rev-list --left-right --count <head>...origin/<head>`), stop the whole run before any change and list the branches the user must pull.
4. **Merge policy.** If the repo's guidance (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING`) requires rebasing feature branches, or forbids merge commits in them, do not merge upward. Explain that the upper branches would need a rebase plus a force push, which this skill never does. Offer either to resolve each PR without propagation, or to stop.
5. Record the starting branch. Switch back to it at the end, or whenever the run stops.

## 3. Flow

The PRs in scope depend on the user's choice:

- **Option 1:** every PR in the stack resolves its comments and receives merges.
- **Option 2:** only #<N> resolves comments. The PRs above it only receive merges, which skips sub-steps 3 to 5 below for them.

1. **Context and classification** for the PRs that resolve comments. Gather context (`context.md`), and evaluate and classify their comments. Show one summary table grouped by PR, with each PR's "Already resolved" list (`templates.md`).
2. **For each PR, in order** (from the root for option 1, from #<N> for option 2):
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
