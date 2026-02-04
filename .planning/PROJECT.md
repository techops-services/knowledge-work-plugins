# KeeperHub Development Plugin

## What This Is

A Claude Code plugin (`kh`) that unifies the KeeperHub development workflow by integrating Jira, GitHub, AWS/EKS, and Helm operations into a single command interface. Developers and PMs can view their work status, manage PRs, coordinate deployments, and handle incidents without context-switching between tools. The plugin leverages existing CLI tools (gh, jira-cli, aws, kubectl, helm, terraform) and MCPs (Atlassian, Terraform Cloud) for authentication and operations.

## Core Value

Developers can see their complete work state (tickets, PRs, deploys) and take action (promote, rollback, transition tickets) from a single interface without leaving Claude Code.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Status & Visibility**
- [ ] `/kh:status` unified dashboard showing Jira tickets, open PRs with CI status, recent deploys
- [ ] Status parameterized with `--jira`, `--prs`, `--deploys`, `--team` filters
- [ ] Auto-detect service from current directory with `--service=<name>` override
- [ ] Role-agnostic commands (same for devs and PMs)

**PR Workflow**
- [ ] `/kh:pr` - Create PR, view status, check CI results
- [ ] `/kh:prs` - List all open PRs for user or team
- [ ] Surface preview environment URL when `deploy-pr-environment` label applied

**Deployment**
- [ ] `/kh:promote` - Promote staging → prod (creates PR from staging to prod branch)
- [ ] `/kh:promote` validates CI passed and staging already deployed before allowing promotion
- [ ] `/kh:rollback` - Trigger rollback via GitHub Actions workflow
- [ ] Multi-service deployment coordination

**Jira Integration**
- [ ] `/kh:ticket` - View/update ticket, transition status
- [ ] `/kh:tickets` - List tickets (in progress, to do, etc.)
- [ ] Suggest Jira transitions based on Git activity (user confirms before executing)

### Out of Scope

- Infrastructure provisioning — handled by existing `deploy-service` skill
- Proactive notifications — command-only interaction model
- Time-based deploy restrictions (no Friday deploys, etc.)
- Incident ticket auto-creation
- Direct Helm/kubectl operations — uses GitHub Actions for all deploys
- Auto-transition Jira tickets — only suggest, user confirms

## Context

**Existing Codebase:**
- Repository: `knowledge-work-plugins` with 11 existing plugins
- Plugin structure: `.claude-plugin/plugin.json`, `.mcp.json`, `commands/`, `skills/`
- Commands: YAML frontmatter + markdown workflow in `commands/*.md`
- Skills: `skills/[skill-name]/SKILL.md` with optional `examples/`, `references/`

**Developer Environment:**
- CLI tools already configured: `gh`, `jira-cli`, `aws`, `kubectl`, `helm`, `terraform`
- MCPs available: Atlassian (Jira), Terraform Cloud
- Multiple KeeperHub services with separate repos/deploy pipelines

**Current Workflow:**
1. Jira tickets created in backlog, assigned by PM
2. Developer creates feature branch from staging
3. PR triggers GitHub Actions CI (lint, tests, build)
4. Optional: `deploy-pr-environment` label auto-provisions preview env
5. Merged to staging → GitHub Actions deploys via Helm to staging EKS
6. Promote: staging → prod branch merge triggers prod deploy
7. Developer responsible for verifying production and closing Jira ticket

**Pain Points Being Solved:**
- Context switching between Jira, GitHub, terminal
- Deployment coordination (staging → prod promotion)
- Status visibility (hard to see PR status, ticket state, deploy state together)

## Constraints

- **Auth**: Must leverage existing CLI tools for auth (gh, aws, kubectl) plus MCPs for Jira
- **Deploys**: All deployments happen via GitHub Actions, not direct Helm/kubectl
- **Rollback**: Must trigger via GitHub Actions workflow (mechanism TBD)
- **Config**: Plugin config in `~/.claude/`, service mappings in per-repo CLAUDE.md
- **Convention**: Follow existing plugin structure in this repository

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Single `kh` plugin vs multiple | Unified UX, simpler discovery | — Pending |
| Mix CLI + MCP for auth | Reuse existing credentials, no token management | — Pending |
| Suggest vs auto-transition Jira | Developer stays in control, reduces automation risk | — Pending |
| Rollback via GitHub Actions | Consistent with existing deploy pipeline | — Pending |
| Command-only (no notifications) | Simpler implementation, developer-initiated | — Pending |

---
*Last updated: 2026-02-04 after initialization*
