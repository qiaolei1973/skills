# skills

A collection of self-contained [Claude Code](https://claude.com/claude-code) skills.

## Skills

| Skill | Description |
|-------|-------------|
| [issue-agent](./issue-agent) | Discover GitHub issues labeled `ready-for-agent`, dispatch isolated subagents to implement them, and create PRs. Designed for `/loop`. |
| [github-code](./github-code) | talos-loop **ready-stage** worker — implement an issue and open one PR. Self-contained; reads `TALOS_*` env, never calls back a server. |
| [github-review](./github-review) | talos-loop **In-review-stage** worker — address a PR's unresolved review threads on the existing branch and resolve them. Never moves the board. |

## Installation

```
/install-skill qiaolei1973/skills/<skill-name>
```
