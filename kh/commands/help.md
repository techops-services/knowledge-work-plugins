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

## Quick Reference

| Task | Command |
|------|---------|
| Check my work status | `/kh:status` |
| Create a PR | `/kh:pr` |
| List my PRs | `/kh:prs` |
| List team PRs | `/kh:prs --team` |
| View a ticket | `/kh:ticket KH-123` |
| Transition ticket | `/kh:ticket KH-123 --move staging` |
| List my tickets | `/kh:tickets` |
| Tickets in progress | `/kh:tickets --status progress` |

## Daily Driver Commands

| Command | Description |
|---------|-------------|
| `/kh:status` | View unified dashboard (Jira, PRs) |
| `/kh:pr` | Create PR from current branch |
| `/kh:ticket` | View or update a Jira ticket |
| `/kh:help` | Show this help |
| `/kh:setup` | Configure services and verify tools |

## Grouped Commands

### PR Workflow

| Command | Description |
|---------|-------------|
| `/kh:pr` | Create PR from current branch with auto-generated title and body |
| `/kh:prs` | List open PRs with CI status, review state, and preview URLs |

### Jira Tickets

| Command | Description |
|---------|-------------|
| `/kh:ticket <id>` | View ticket details and transition status |
| `/kh:tickets` | List your tickets with status filtering |

### Deployment

| Command | Description |
|---------|-------------|
| `/kh:promote` | Promote staging to production |
| `/kh:rollback` | Trigger rollback via GitHub Actions |

## Aliases

| Alias | Full Command |
|-------|--------------|
| `/kh:s` | `/kh:status` |
| `/kh:t` | `/kh:ticket` |
| `/kh:ts` | `/kh:tickets` |
| `/kh:issues` | `/kh:tickets` |
| `/kh:my-tickets` | `/kh:tickets` |

## Common Workflows

### Starting Work on a Ticket

1. Create feature branch: `git checkout -b KH-123-feature-name`
2. Plugin suggests: Move KH-123 to "In Progress"? [y/n]
3. Or manually: `/kh:ticket KH-123 --move progress`

### Submitting for Review

1. Create PR: `/kh:pr`
   - Auto-links KH-123 from branch name
   - Auto-generates title and body
2. Share PR link with team

### After Merge to Staging

1. Plugin suggests: Move KH-123 to "Testing Staging"? [y/n]
2. Or manually: `/kh:ticket KH-123 --move staging`
3. Verify on staging environment
4. When ready: `/kh:promote` (Phase 4)

### Completing Work

1. After prod deploy, verify functionality
2. `/kh:ticket KH-123 --move done`
3. Or use suggestion prompt after promotion

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

## Common Flags

| Flag | Description |
|------|-------------|
| --verbose, -v | Show detailed output with dates, URLs |
| --service=name | Override service auto-detection |
| --no-suggest | Disable workflow suggestions for this command |

## Examples

Quick status check:
```
/kh:status
```

Just my tickets:
```
/kh:status --jira
```

All team PRs:
```
/kh:status --prs --team
```

Create a PR:
```
/kh:pr
```

Move ticket to testing:
```
/kh:ticket KH-123 --move staging
```
