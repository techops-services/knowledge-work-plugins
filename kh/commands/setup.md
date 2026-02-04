---
description: Configure KeeperHub services and verify tool integrations
---

# Setup Command

Configure your KeeperHub services and verify that all required tools are available. This command helps debug connection issues and establish service configuration.

> For configuration schema and auto-detection details, see the [service-config skill](../skills/service-config/SKILL.md).

## Instructions

### 1. Check Existing Configuration

Check for existing services configuration:

```bash
ls ~/.claude/kh/services.yaml
```

**If exists:**
- Read and display current configuration
- Show number of services configured
- Offer to add new service or reconfigure existing

**If missing:**
- Note this is a fresh setup
- Continue to service configuration (step 4)

### 2. Verify CLI Tools

Check availability of required CLI tools. Don't fail on missing tools - just report what's available.

**GitHub CLI:**
```bash
gh --version
```
- [Available] gh version X.Y.Z
- [Missing] GitHub CLI not found. Install: https://cli.github.com

**AWS CLI:**
```bash
aws --version
```
- [Available] aws-cli/X.Y.Z
- [Missing] AWS CLI not found. Install: https://aws.amazon.com/cli

**Kubernetes CLI:**
```bash
kubectl version --client
```
- [Available] Client Version: vX.Y.Z
- [Missing] kubectl not found. Install: https://kubernetes.io/docs/tasks/tools

**Summary:**
```
CLI Tools:
  gh       [Available] v2.40.0
  aws      [Available] aws-cli/2.15.0
  kubectl  [Missing] - Kubernetes commands will be unavailable
```

Note which features depend on each tool:
- `gh`: PR operations, GitHub Actions status
- `aws`: AWS resource queries, EKS authentication
- `kubectl`: Kubernetes pod status, logs

### 3. Verify MCP Connections

Test MCP connectivity for Jira integration.

**Atlassian MCP:**

Try a simple Jira query to verify connectivity:
```
Use Atlassian MCP to query: GET /rest/api/3/myself
```

**If successful:**
```
Atlassian MCP:
  [Connected] Authenticated as: user@example.com
```

**If authentication fails:**
```
Atlassian MCP:
  [Not authenticated] Jira commands will be unavailable.
  To configure: Set up Atlassian MCP in your Claude configuration.
  See: https://github.com/anthropics/mcp-atlassian
```

**If MCP unavailable:**
```
Atlassian MCP:
  [Not configured] Jira integration not available.
  To add: Configure Atlassian MCP in .mcp.json
```

### 4. Interactive Service Configuration

If no services configured or user wants to add a new service:

**Prompt for service details:**
```
What KeeperHub service would you like to configure?

Enter a name for this service (e.g., keeperhub-api):
```

**Collect required fields:**

| Field | Prompt | Validation |
|-------|--------|------------|
| Service name | "Service name:" | No spaces, lowercase, hyphen-separated |
| Jira project | "Jira project key (e.g., KH):" | 2-10 uppercase letters |
| GitHub repo | "GitHub repo (owner/repo):" | Format: `owner/repo` |

**Validate GitHub repo:**
```bash
gh repo view techops/keeperhub-api --json name
```

- [Valid] Repository exists
- [Invalid] Repository not found - check the owner/repo format

**Write configuration:**

Create or update `~/.claude/kh/services.yaml`:

```bash
mkdir -p ~/.claude/kh
```

```yaml
services:
  keeperhub-api:
    jira_project: KH
    github_repo: techops/keeperhub-api
```

If file exists, append new service (don't overwrite existing).

### 5. Test Auto-Detection

If currently in a git repository, test auto-detection:

```bash
git remote get-url origin
```

**Parse and match:**
```
Current repository: techops/keeperhub-api

Auto-detection test:
  [Matched] Service: keeperhub-api
  Jira project: KH
  GitHub repo: techops/keeperhub-api
```

**If no match:**
```
Current repository: techops/other-repo

Auto-detection test:
  [No match] This repository is not configured as a KeeperHub service.
  Add it with: /kh:setup (run again to add)
```

**If not in git repo:**
```
Auto-detection test:
  [Skipped] Not in a git repository.
  Auto-detection works when you're in a service repository.
```

### 6. Report Results

Summarize setup status:

```
KeeperHub Plugin Setup Complete
===============================

Services: 3 configured
  - keeperhub-api (KH, techops/keeperhub-api)
  - keeperhub-web (KH, techops/keeperhub-web)
  - keeperhub-worker (KH, techops/keeperhub-worker)

CLI Tools:
  gh       [Available]
  aws      [Available]
  kubectl  [Available]

MCP Connections:
  Atlassian [Connected]

Current Context:
  Repository: techops/keeperhub-api
  Service: keeperhub-api
  Jira: KH

Next Steps:
  /kh:status - See your current work state
  /kh:tickets - View your assigned tickets
  /kh:prs - List your open pull requests
```

**If gaps exist:**
```
Gaps Identified:
  - kubectl missing: Kubernetes pod status unavailable
  - Atlassian MCP not authenticated: Jira commands unavailable

These features will work once the tools/connections are configured.
All other commands work normally.
```

## Notes

- Setup is optional - commands work without it (with reduced functionality)
- Run again anytime to add services or recheck connections
- Service configuration persists in `~/.claude/kh/services.yaml`
- Auto-detection is tested but not required - use `--service` flag as override
