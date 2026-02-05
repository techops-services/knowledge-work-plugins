# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. For example, `~~jira` might mean Jira Cloud, Jira Server, or any other Atlassian project tracker with an MCP server.

Plugins are **tool-agnostic** in documentation but **specific** in implementation. The `.mcp.json` configures MCP servers, while CLI tools are expected to be configured locally.

## Connectors for this plugin

| Category | Placeholder | Connection type | Tool/Server |
|----------|-------------|-----------------|-------------|
| Jira | `~~jira` | MCP | Atlassian (included in `.mcp.json`) |
| GitHub | `~~github` | CLI | `gh` CLI (local installation) |
| AWS | `~~aws` | CLI | `aws` CLI (local installation) |
| Kubernetes | `~~kubernetes` | CLI | `kubectl` CLI (local installation) |

## Configuration details

### MCP Servers (automatic)

**Atlassian (Jira)**
- Server: `https://mcp.atlassian.com/v1/mcp`
- Authentication: OAuth via Atlassian MCP
- Used for: Ticket queries, status transitions, project lookups

### CLI Tools (manual setup required)

**GitHub CLI (`gh`)**
- Install: `brew install gh`
- Auth: `gh auth login`
- Used for: PR creation, status, CI results, repo operations

**AWS CLI (`aws`)**
- Install: `brew install awscli`
- Auth: `aws configure` or SSO setup
- Used for: EKS cluster access, deployment status

**Kubernetes (`kubectl`)**
- Install: `brew install kubectl`
- Auth: Via AWS EKS context (`aws eks update-kubeconfig`)
- Used for: Pod status, deployment verification, logs
