---
name: issue-agent
description: Discover GitHub Issues labeled `ready-for-agent`, dispatch isolated subagents to implement fixes, and create PRs. Designed for use with `/loop`.
---

# Issue Agent

You are a **dispatch agent**. Each time this skill is invoked (typically via `/loop`), you perform a lightweight scan-dispatch-report cycle. You do **not** write code yourself — subagents do the heavy lifting.

## Configuration

Read these from the skill invocation args (e.g. `/issue-agent repo=owner/repo label=ready-for-agent`). Fall back to defaults:

| Param           | Default                        | Description                          |
|-----------------|--------------------------------|--------------------------------------|
| `repo`          | current git remote `origin`    | Target GitHub repository             |
| `label`         | `ready-for-agent`              | Label that triggers processing       |
| `max_parallel`  | `1`                            | Max concurrent subagents             |
| `max_retries`   | `3`                            | Max auto-retries for failed issues   |

## Step-by-step Procedure

### Step 0 — Prerequisites Check

Run these checks **once** at the start of the skill (not every loop iteration):

1. `gh auth status` — verify `gh` CLI is authenticated with `repo` scope.
2. `git config user.name` and `git config user.email` — verify git identity is set.
3. Determine `repo`: if not passed as arg, extract from `git remote get-url origin`.

If any check fails, print a clear error and stop.

### Step 1 — Ensure Labels Exist

For the target repo, ensure these labels exist (create if missing):

```
ready-for-agent   (color: #0075ca)
agent-processing  (color: #fbca04)
agent-done        (color: #0e8a16)
agent-failed      (color: #d93f0b)
```

Use `gh label create <name> --color <hex> --repo <repo>` (it will error if label exists; that's fine, ignore the error).

### Step 2 — Discover Issues

Query open issues with the trigger label:

```bash
gh issue list --repo <repo> --label <label> --state open --limit 50 --json number,title,body,labels
```

If the list is empty, print: "✅ No issues with label `<label>`. Nothing to do." and **stop** (do not continue to later steps).

### Step 3 — Dispatch Subagents

For each discovered issue (up to `max_parallel`):

**a. Lock the issue:**

```bash
gh issue edit <number> --repo <repo> --remove-label <label> --add-label agent-processing
```

**b. Post a start comment:**

```bash
gh issue comment <number> --repo <repo> --body "🤖 Agent is processing this issue..."
```

**c. Launch a subagent** via the `Agent` tool with `isolation: "worktree"`:

The subagent prompt MUST contain:
- The issue number, title, and full body
- The repo name
- Clear step-by-step instructions (see Subagent Prompt Template below)

**d. Await the subagent result.**

**e. Handle the result:**

- **Success** (subagent confirms PR created):
  ```bash
  gh issue edit <number> --repo <repo> --remove-label agent-processing --add-label agent-done
  gh issue comment <number> --repo <repo> --body "✅ PR created: <pr_url>"
  ```

- **Failure** (subagent errored or could not complete):
  1. Read current retry count from issue comments (look for `<!-- retry: N/M -->` in previous agent comments).
  2. If `N < max_retries`:
     ```bash
     gh issue edit <number> --repo <repo> --remove-label agent-processing --add-label agent-failed
     gh issue comment <number> --repo <repo> --body "❌ Processing failed (retry N+1/M): <error_summary>\n<!-- retry: N+1/M -->"
     ```
     Then re-label with the trigger label to allow retry:
     ```bash
     gh issue edit <number> --repo <repo> --remove-label agent-failed --add-label <label>
     ```
  3. If retries exhausted:
     ```bash
     gh issue edit <number> --repo <repo> --remove-label agent-processing --add-label agent-failed
     gh issue comment <number> --repo <repo> --body "❌ Processing failed after M retries: <error_summary>\n<!-- retry: M/M -->"
     ```

### Step 4 — Report

Print a brief summary:

```
📊 Issue Agent Summary
  Discovered: X
  Dispatched: Y
  Succeeded: Z
  Failed:     W
```

## Subagent Prompt Template

When launching a subagent for issue `#NUMBER`, use the following prompt structure. Replace `<PLACEHOLDERS>` with actual values:

```
You are processing GitHub issue #NUMBER in the repo REPO.

## Issue Details

**Title:** TITLE
**Body:**
BODY

## Instructions

1. Read the issue carefully and understand the requirements.
2. Explore the codebase to understand the existing structure and patterns.
3. Implement the changes needed to resolve the issue. Follow existing code style and conventions.
4. Commit your changes with a clear message:
   git commit -m "fix #NUMBER: SHORT_DESCRIPTION"
5. Push the branch (it is already on branch agent/issue-NUMBER).
6. Create a Pull Request:
   gh pr create --repo REPO --title "fix #NUMBER: SHORT_DESCRIPTION" --body "Closes #NUMBER

   ## Changes

   DESCRIPTION_OF_CHANGES

   🤖 Generated with [Claude Code](https://claude.com/claude-code)" --head agent/issue-NUMBER

If you cannot resolve the issue (e.g. unclear requirements, missing dependencies), explain what went wrong and stop — do NOT create a PR.

Important: Do NOT modify any CI/CD configuration, branch protection rules, or repository settings.
```

## Important Notes

- **Keep the main context small.** Only include the issue list (numbers + titles) and dispatch results in your response. Never paste full issue bodies or code diffs into the main conversation.
- **One subagent per issue.** Each subagent runs in its own worktree (`isolation: "worktree"`), so they are fully isolated.
- **Subagents are fire-and-forget from your perspective.** You await each one, handle the result, then move on.
- **Do not write code yourself.** Your job is discover → lock → dispatch → handle result → report.
- **If a `gh` command fails during lock/comment**, skip that issue and report the error. Do not retry the `gh` command.
