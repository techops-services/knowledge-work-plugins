---
description: List all available KeeperHub commands
---

# Help Command

Display available KeeperHub commands and usage guidance.

## Overview

```
KeeperHub Development Plugin (kh)
Unified workflow for Jira tickets, GitHub PRs, and deployments.
```

## Daily Driver Commands

| Command | Description |
|---------|-------------|
| /kh:status | View unified dashboard (tickets, PRs, deploys) |
| /kh:pr | Create PR from current branch |
| /kh:ticket | View or update a Jira ticket |
| /kh:help | Show this help |
| /kh:setup | Configure services and verify tools |

## Short Aliases

| Alias | Expands To |
|-------|------------|
| /kh:s | /kh:status |

Additional aliases will be added as commands are built.

## Grouped Commands

### PR Management

| Command | Description |
|---------|-------------|
| /kh:prs | List all open PRs for user or team |

### Ticket Management

| Command | Description |
|---------|-------------|
| /kh:tickets | List tickets (in progress, to do, etc.) |

### Deployment

| Command | Description |
|---------|-------------|
| /kh:promote | Promote staging to production |
| /kh:rollback | Trigger rollback via GitHub Actions |

## Getting Started

**First time?**
Run `/kh:setup` to configure your services and verify tool access.

**Quick status check?**
Run `/kh:status` to see your tickets, PRs, and deployment status.

**Need help?**
Use `/kh:help` to see this list anytime.

## Output Format

All commands use plain text status labels:
- [Open], [Merged], [Closed] for PRs
- [In Progress], [Done], [To Do] for tickets
- [Deployed], [Pending], [Failed] for deployments

Use `--verbose` or `-v` flag with any command for detailed output.
