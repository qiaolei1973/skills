# issue-agent

A Claude Code skill that automatically discovers GitHub Issues labeled `ready-for-agent`, dispatches isolated subagents to implement the requested changes, and creates Pull Requests.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI installed
- `gh` CLI installed and authenticated (`gh auth login` — needs `repo` scope)
- `git` configured with `user.name` and `user.email`
- A GitHub repository with issues to process

## Installation

```
/install-skill qiaolei1973/skills/issue-agent
```

## Usage

### One-shot (test a single run)

```
/issue-agent
```

### Continuous loop (recommended, run inside tmux)

```
tmux new -s agent
claude
/loop 5m /issue-agent
```

Detach with `Ctrl+B → D`. Reattach with `tmux attach -t agent`.

## How It Works

```
Every loop tick:
  1. Query open issues with label `ready-for-agent`
  2. For each issue:
     a. Lock: label → agent-processing
     b. Post "processing" comment
     c. Launch isolated subagent in a worktree
     d. Subagent: reads issue → writes code → commits → pushes → creates PR
     e. On success: label → agent-done, comment PR link
     f. On failure: label → agent-failed, comment error (auto-retry up to 3×)
  3. Print summary
```

### Label State Machine

```
ready-for-agent ──→ agent-processing ──→ agent-done (PR created)
                         │
                         └──→ agent-failed (retry < max → back to ready-for-agent)
```

## Configuration

Pass parameters when invoking the skill:

```
/issue-agent repo=owner/repo label=ready-for-agent max_parallel=1 max_retries=3
```

| Parameter      | Default                  | Description                     |
|----------------|--------------------------|---------------------------------|
| `repo`         | current git remote       | Target GitHub repository        |
| `label`        | `ready-for-agent`        | Label that triggers processing  |
| `max_parallel` | `1`                      | Max concurrent subagents        |
| `max_retries`  | `3`                      | Max retries for failed issues   |

## Observability

- **Issue labels** show processing status at a glance
- **Issue comments** track start, success, and failure events
- **Loop summary** printed each iteration in the terminal

## Safety

- Subagents work in isolated git worktrees (auto-cleaned)
- PRs are **never auto-merged** — human review required
- Each issue is locked via label to prevent duplicate processing
