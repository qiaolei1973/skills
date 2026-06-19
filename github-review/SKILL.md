---
name: github-review
description: Self-contained talos-loop review-fix skill. For a PR that is "In review" and has unresolved inline review threads, address each thread's feedback in the current worktree, push to the existing PR branch (no new branch/PR), and resolve the addressed threads. Never moves the board.
---

# github-review — address PR review threads

You are running inside a **talos-loop** session. Your working directory is a git **worktree** on the PR's **existing** feature branch (the PR head). A reviewer left unresolved inline review threads on the PR. Your job: address them, push to the **same** branch, resolve the threads you fixed — then stop.

This is a perpetual worker: the orchestrator re-dispatches you on later polls **only while unresolved threads remain**, so resolving threads is how you tell it "I'm done".

## Context — read from the environment

| Variable | Meaning |
|----------|---------|
| `TALOS_ISSUE_URL` / `TALOS_SOURCE_ID` | The issue this PR belongs to. |
| `TALOS_BRANCH` | The existing PR branch (e.g. `feat/issue-33`). Push here — do **not** create a new branch. |
| `TALOS_TARGET_REPO` | The repository name. |

Derive `owner/name` from the remote:
```bash
git remote get-url origin
# → git@github.com:owner/name.git
OWNER=<owner>; NAME=<repo>   # parse from the URL above
```

## Steps

### 1. Find the open PR for this branch
```bash
gh pr list --head "$TALOS_BRANCH" --state open --json number,title,url,baseRefName
```
If there is no open PR, there is nothing to fix — stop (exit cleanly).

### 2. List the unresolved review threads
```bash
PR=<number>   # from step 1
gh api graphql -f query='
query($o:String!,$n:String!,$p:Int!){
  repository(owner:$o,name:$n){
    pullRequest(number:$p){
      reviewThreads(first:100){
        nodes{ id isResolved isOutdated path line
               comments(first:20){ nodes{ body author{ login } } } }
      }
    }
  }
}' -F o="$OWNER" -F n="$NAME" -F p="$PR" \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved==false)'
```
Each thread carries: `id` (needed to resolve it), `path`, `line`, and `comments[].body` (the reviewer's feedback). Read the feedback carefully.

### 3. Address each unresolved thread
For each thread, open `path` near `line`, understand the reviewer's point, and fix the code in the worktree.

- **Skip** `isOutdated` threads (the code has already moved past that line) — unless the underlying concern still applies somewhere. Resolve an outdated thread only if it's now moot; otherwise leave it for a human.
- If a comment is a question, answer it by fixing the code (preferred) or by replying (step 5) — don't ignore it.

### 4. Verify + push to the SAME branch
Verify against the repo's own conventions (typecheck/test/build). Only fail on regressions you introduced.

```bash
git add -A
git commit -m "review: address review feedback"
git push origin "$TALOS_BRANCH"
```
Pushing to the same branch keeps PR history continuous. Do **not** open a new branch or PR.

### 5. Resolve the threads you addressed
Resolving is what stops the orchestrator from re-dispatching you, so resolve every thread you actually fixed:
```bash
# for each thread id you addressed:
gh api graphql -f query='
mutation($t:ID!){ resolveReviewThread(input:{threadId:$t}){ thread{ isResolved } } }' \
  -F t="<thread-id>"
```
Optionally reply first so the reviewer has context:
```bash
gh pr comment "$PR" --body "..."
```

### Threads you deliberately leave open
If you disagree with a thread or can't safely fix it, **reply** explaining why and leave it unresolved — a human will see your reply on the next pass. That's fine: partial progress is acceptable.

## When you're done

Stop. **Never** move the GitHub Projects board — the issue stays "In review" until the PR merges. Resolve the threads you fixed so the orchestrator stops re-dispatching you; leave the rest for the human reviewer.
