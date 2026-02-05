# KeeperHub Development Plugin

A Claude Code plugin that unifies the KeeperHub development workflow by integrating Jira, GitHub, AWS/EKS, and deployment operations into a single command interface.

## Installation

```
claude plugins add knowledge-work-plugins/kh
```

## What It Does

This plugin gives you a unified view of your development workflow:

- **Status visibility** - See your Jira tickets, open PRs, and deployment status in one place
- **PR workflow** - Create PRs, check CI status, and manage reviews without leaving Claude
- **Deployment coordination** - Promote staging to production, trigger rollbacks, verify deploys
- **Jira integration** - View and transition tickets, with suggestions based on Git activity

## Developer Workflow

The KeeperHub plugin provides a seamless workflow from ticket to deployment:

```
┌─────────────────────────────────────────────────────────────────┐
│                     DEVELOPMENT LIFECYCLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Jira: To Do          Jira: In Progress        Jira: Review     │
│       │                      │                      │           │
│       ▼                      ▼                      ▼           │
│  ┌─────────┐           ┌─────────┐            ┌─────────┐       │
│  │ Backlog │  ──────▶  │  Code   │  ───────▶  │   PR    │       │
│  └─────────┘           └─────────┘            └─────────┘       │
│       │                      │                      │           │
│       │               /kh:start               /kh:pr            │
│       │                      │                      │           │
│       │                      ▼                      ▼           │
│       │              Creates branch          Creates PR         │
│       │              TICKET-123-desc         Links Jira         │
│       │                                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Jira: Testing Staging      Jira: Testing Prod      Jira: Done  │
│       │                           │                      │      │
│       ▼                           ▼                      ▼      │
│  ┌─────────┐              ┌─────────────┐         ┌────────┐   │
│  │ Staging │  ─────────▶  │ Production  │ ──────▶ │ Done   │   │
│  └─────────┘              └─────────────┘         └────────┘   │
│       │                           │                             │
│  /kh:merge                  /kh:promote                         │
│       │                     (Phase 4)                           │
│       ▼                                                         │
│  Squash merges PR                                               │
│  Deletes branch                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Quick Start

**1. Start work on a ticket:**
```bash
/kh:start KEEP-123
```
- Fetches ticket details from Jira
- Transitions ticket to "In Progress"
- Creates branch: `KEEP-123-ticket-summary`
- Checks out the branch

**2. Create a Pull Request:**
```bash
/kh:pr
```
- Auto-detects Jira ticket from branch name
- Generates PR title and body
- Links Jira ticket in PR description
- Transitions ticket to "Review"

**3. Merge to staging:**
```bash
/kh:merge
```
- Squash merges the PR
- Deletes the feature branch
- Transitions ticket to "Testing Staging"

### Status Commands

Check your current work state at any time:

| Command | Description |
|---------|-------------|
| `/kh:status` | Unified view: tickets, PRs, deploys |
| `/kh:tickets` | List your Jira tickets |
| `/kh:prs` | List your open PRs |
| `/kh:ticket KEEP-123` | View/transition a specific ticket |

### Command Reference

| Command | Action | Jira Transition |
|---------|--------|-----------------|
| `/kh:start <ticket>` | Create branch, start work | → In Progress |
| `/kh:pr` | Create PR from branch | → Review |
| `/kh:merge [pr]` | Squash merge PR | → Testing Staging |
| `/kh:ticket <id> --move <status>` | Manual transition | → Any status |

## Commands

| Command | What it does |
|---------|--------------|
| `/kh:status` | Unified dashboard: tickets, PRs, and deploy status |
| `/kh:pr` | Create PR, view status, check CI results |
| `/kh:prs` | List open PRs for user or team |
| `/kh:ticket` | View or update a Jira ticket |
| `/kh:tickets` | List tickets (in progress, to do, etc.) |
| `/kh:promote` | Promote staging to production |
| `/kh:rollback` | Trigger rollback via GitHub Actions |
| `/kh:setup` | Verify tool configuration and auth status |

## Prerequisites

The plugin requires these CLI tools to be installed and authenticated:

| Tool | Install | Auth |
|------|---------|------|
| GitHub CLI | `brew install gh` | `gh auth login` |
| AWS CLI | `brew install awscli` | `aws configure` or SSO |
| kubectl | `brew install kubectl` | Via `aws eks update-kubeconfig` |

Jira access is provided via the Atlassian MCP server (configured automatically).

## Quick Start

1. Install the plugin: `claude plugins add knowledge-work-plugins/kh`
2. Run `/kh:setup` to verify your tool configuration
3. Run `/kh:status` to see your current work state

## Configuration

Service configuration is stored in `~/.claude/kh/services.yaml`. The plugin auto-detects which service you're working on based on the current Git repository.

For tool connector details, see [CONNECTORS.md](CONNECTORS.md).
