# github-review

A **self-contained** [Claude Code](https://claude.com/claude-code) skill that addresses unresolved inline review threads on a Pull Request. It is the **"In review" stage worker** of [talos-loop](https://github.com/qiaolei1973/talos-loop): the orchestrator re-dispatches it on every poll **while unresolved threads remain**, so resolving the threads you fixed is how you signal "done". It pushes fixes to the **existing** PR branch and **never moves the board**.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI installed
- `gh` CLI installed and authenticated (`gh auth login` — needs `repo` scope)
- `git` configured with `user.name` and `user.email`
- An isolated git **worktree** on the PR's **existing** feature branch (the PR head), with the `TALOS_*` environment variables set (provisioned by talos-loop)

## Installation

```
/install-skill qiaolei1973/skills/github-review
```

For talos-loop, install it at the **user level** (`~/.claude/skills/github-review`) — the orchestrator runs each job inside a worktree whose cwd is outside any single repo checkout, so a project-level skill would not be found.

## Usage

### The normal path — invoked by talos-loop

The orchestrator launches this skill directly in print mode whenever a `done`-state issue still has unresolved review threads:

```
claude -p "/github-review 处理 issue：<ISSUE_URL>" --dangerously-skip-permissions \
       --output-format=stream-json --verbose
```

You do not run it yourself in production. A clean exit (exit 0) tells the orchestrator the pass is complete; unresolved threads are re-probed on the next poll and re-dispatched if any remain.

### Manual / test invocation

```
/github-review
```

with the `TALOS_*` variables exported (cwd inside the worktree on the PR branch).

## Context — environment variables

| Variable          | Meaning                                                                |
|-------------------|------------------------------------------------------------------------|
| `TALOS_ISSUE_URL` / `TALOS_SOURCE_ID` | The issue this PR belongs to.                            |
| `TALOS_BRANCH`    | The **existing** PR branch (e.g. `feat/issue-33`). Push here — no new branch. |
| `TALOS_TARGET_REPO` | The repository name.                                                 |

`owner/name` is derived from `git remote get-url origin`.

## How it works

```
1. gh pr list --head <TALOS_BRANCH> --state open    → find the PR
2. graphql reviewThreads(isResolved==false)         → list unresolved threads
3. for each thread: read path/line/feedback → fix the code in the worktree
4. run the repo's own checks; git commit + git push origin <TALOS_BRANCH>
5. graphql resolveReviewThread(threadId) for each thread you fixed
6. exit 0
```

Threads are read straight from GitHub's native `reviewThreads` — that is both the trigger and the termination signal. Outdated threads are skipped (or resolved as moot); threads you disagree with are **replied to** and left open for the human reviewer. Partial progress is acceptable.

## Safety

- **Never** moves the GitHub Projects board — the issue stays "In review" until the PR merges.
- Pushes to the **same** branch only — **no** new branch or PR.
- **Never** merges, closes, or force-pushes.
- Resolves only the threads it actually fixed; leaves the rest for the reviewer.
