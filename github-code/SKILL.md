---
name: github-code
description: Self-contained talos-loop coding skill. Given a GitHub issue (context injected as TALOS_* env vars), implement the change in the current git worktree, verify it against the repo's own conventions, push the feature branch, and open a Pull Request. Drives itself end-to-end and never calls back into any server.
---

# github-code — implement an issue and open a PR

You are running inside a **talos-loop** session. Your working directory is an isolated git **worktree** already checked out on a fresh feature branch cut from the repo's default branch. Your one job: implement the issue, verify, push the branch, open a PR — then stop.

The orchestrator treats your **clean process exit (exit 0)** as success and advances the board to "In review". It does not look at what you did — it trusts the exit code — so only finish cleanly once a PR actually exists.

## Context — read from the environment

| Variable | Meaning |
|----------|---------|
| `TALOS_ISSUE_URL` | The GitHub issue URL (also restated in the launch prompt). |
| `TALOS_SOURCE_ID` | The issue number (e.g. `33`). |
| `TALOS_TARGET_REPO` | The repository name (e.g. `talos-loop`). |
| `TALOS_BRANCH` | The feature branch you are on (e.g. `feat/issue-33`). |
| `TALOS_REPO_PATH` | The main clone (reference only — **work in the current worktree**, which is your cwd). |

Derive the repo `owner/name` from the worktree's own remote (don't assume):

```bash
git remote get-url origin
# → git@github.com:owner/name.git   or   https://github.com/owner/name.git
```

## Steps

### 1. Read the issue
```bash
gh issue view "$TALOS_SOURCE_ID"
```
Understand the acceptance criteria. If the issue links a spec / PRD / design doc, read it before coding. Reproduce or at least locate the relevant code.

### 2. Explore & plan
You are already inside the worktree. Read `CLAUDE.md` / `README.md` for conventions. Identify the minimal set of files to change. Keep the change **scoped to the issue** — no drive-by refactors.

### 3. Implement
Make the change. Match the surrounding code's style, naming, and comment density. Prefer editing existing code over adding new files unless the issue asks for a new file.

### 4. Verify against the repo's own conventions
Do not invent new tooling. If a `package.json` (or equivalent) exists, run the checks it already defines:
- typecheck / lint / test / build — whichever the repo uses.
- Only treat the task as **blocked by your change** if you introduced a failure. **Pre-existing** failures are out of scope: note them in the PR body and move on.
- For a docs-only or marker-file change, a build/test run is optional.

### 5. Commit
Stage and commit only your change:
```bash
git add -A
git commit -m "feat: <concise imperative summary> (#$TALOS_SOURCE_ID)"
```
Do **not** rewrite or amend existing history.

### 6. Push the branch
```bash
git push -u origin "$TALOS_BRANCH"
```

### 7. Open the PR — base = the repo's default branch
```bash
BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
gh pr create --base "$BASE" --head "$TALOS_BRANCH" \
  --title "<concise summary>" \
  --body "Closes #$TALOS_SOURCE_ID

## What changed
- ...

## How to verify
- ..."
```

## When you're done

Stop. You're finished when exactly **one** PR is open for `$TALOS_BRANCH`. Do **not**:
- move any GitHub Projects board field (the orchestrator does `queued → processing → In review`),
- open more than one PR,
- merge or close anything.

## If you genuinely cannot proceed

If the issue is unclear, a duplicate, blocked, or out of scope: post a single clarifying comment and stop — do **not** open an empty PR.

```bash
gh issue comment "$TALOS_SOURCE_ID" --body "..."
```

Then make the final action of your session a deliberately failing command so the orchestrator sees a non-zero exit and parks the issue for a human rather than advancing it to "In review":

```bash
echo "talos-loop: could not complete issue #$TALOS_SOURCE_ID — see comment above" >&2
false
```
