# github-code

A **self-contained** [Claude Code](https://claude.com/claude-code) skill that implements a GitHub issue and opens a Pull Request. It is the **"ready" stage worker** of [talos-loop](https://github.com/qiaolei1973/talos-loop): the orchestrator hands it an issue (via `TALOS_*` env vars) and it drives itself end-to-end — read, plan, implement, verify, push, open a PR — and **never calls back into any server**.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI installed
- `gh` CLI installed and authenticated (`gh auth login` — needs `repo` scope)
- `git` configured with `user.name` and `user.email`
- An isolated git **worktree** already checked out on a fresh feature branch, with the `TALOS_*` environment variables set (this is what talos-loop provisions; see below)

## Installation

```
/install-skill qiaolei1973/skills/github-code
```

For talos-loop, install it at the **user level** (`~/.claude/skills/github-code`) rather than per-project — the orchestrator runs each job inside a worktree whose cwd is outside any single repo checkout, so a project-level skill would not be found.

## Usage

### The normal path — invoked by talos-loop

The talos-loop orchestrator launches this skill directly in print mode:

```
claude -p "/github-code 处理 issue：<ISSUE_URL>" --dangerously-skip-permissions \
       --output-format=stream-json --verbose
```

You do not run it yourself in production. The orchestrator treats a **clean process exit (exit 0)** as success and advances the board to "In review" — it does not inspect what you did, only the exit code.

### Manual / test invocation

```
/github-code
```

with the `TALOS_*` variables exported in your shell (and your cwd inside the worktree on the feature branch).

## Context — environment variables

| Variable          | Meaning                                                          |
|-------------------|------------------------------------------------------------------|
| `TALOS_ISSUE_URL` | The GitHub issue URL (also restated in the launch prompt).       |
| `TALOS_SOURCE_ID` | The issue number (e.g. `33`).                                    |
| `TALOS_TARGET_REPO` | The repository name (e.g. `talos-loop`).                       |
| `TALOS_BRANCH`    | The feature branch the worktree is on (e.g. `feat/issue-33`).    |
| `TALOS_REPO_PATH` | The main clone (reference only — **work in the worktree** cwd).  |

`owner/name` is derived from the worktree's own `git remote get-url origin` — never assumed.

## How it works

```
1. gh issue view <TALOS_SOURCE_ID>          → read & understand the issue
2. explore the worktree (CLAUDE.md, README) → minimal scoped change plan
3. implement, matching surrounding style
4. run the repo's own checks (typecheck/lint/test/build)
5. git commit + git push -u origin <TALOS_BRANCH>
6. gh pr create --base <default-branch> --head <TALOS_BRANCH>   → one PR
7. exit 0
```

If the issue is genuinely unactionable (unclear, duplicate, blocked), the skill posts a single clarifying comment and ends with a deliberately **failing** command, so the orchestrator parks the issue for a human rather than advancing it to "In review".

## Safety

- **Never** moves the GitHub Projects board — the orchestrator owns `queued → processing → In review`.
- Opens **exactly one** PR per issue.
- **Never** merges, closes, or force-pushes.
- Verifies against the repo's *own* conventions; pre-existing failures are noted, not blocking.
