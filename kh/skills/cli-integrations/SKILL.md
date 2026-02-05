---
name: cli-integrations
description: Patterns for GitHub, AWS, and Kubernetes CLI operations in KeeperHub workflows
---

# CLI Integrations

This skill documents the patterns for using external CLI tools in KeeperHub workflows. All integrations follow the on-demand validation philosophy: try the operation, handle failures gracefully.

## GitHub CLI (gh) Patterns

The GitHub CLI handles PR operations, repository queries, and CI status checks.

### Authentication

Verify authentication status before operations:

```bash
gh auth status
```

If not authenticated, guide the user:
> "GitHub CLI not authenticated. Run `gh auth login` to authenticate."

### PR Operations

**List open PRs:**
```bash
gh pr list --repo owner/repo --state open --json number,title,author,headRefName,url
```

**View PR details:**
```bash
gh pr view <number> --repo owner/repo --json title,body,state,mergeable,reviews,statusCheckRollup
```

**Create PR:**
```bash
gh pr create --repo owner/repo --title "..." --body "..." --base staging --head feature-branch
```

**Check CI status:**
```bash
gh pr checks <number> --repo owner/repo
```

**Merge PR:**
```bash
gh pr merge <number> --repo owner/repo --squash --delete-branch
```

### Repository Operations

**Get repo info:**
```bash
gh repo view owner/repo --json name,defaultBranch,url,description
```

**List branches:**
```bash
gh api repos/owner/repo/branches --jq '.[].name'
```

### API Fallback

For complex queries not covered by gh commands, use the API directly:

```bash
gh api repos/owner/repo/pulls --jq '.[] | {number, title, state}'
gh api repos/owner/repo/actions/runs --jq '.workflow_runs[:5] | .[] | {id, status, conclusion}'
```

## AWS CLI Patterns

The AWS CLI handles EKS context setup and AWS resource queries.

### Authentication

Verify credentials and identity:

```bash
aws sts get-caller-identity
```

If not authenticated, guide the user:
> "AWS CLI not authenticated. Run `aws configure` or check your IAM credentials."

### EKS Context Setup

Update kubeconfig for EKS cluster access:

```bash
aws eks update-kubeconfig --name cluster-name --region us-east-1
```

### Common Patterns

**Always use JSON output for parsing:**
```bash
aws ecs describe-services --cluster my-cluster --services my-service --output json
```

**Use --query for filtering:**
```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}'
```

**List EKS clusters:**
```bash
aws eks list-clusters --region us-east-1 --output json
```

## Kubernetes CLI (kubectl) Patterns

The kubectl CLI handles deployment status, pod inspection, and cluster operations.

### Context Verification

Check current context before operations:

```bash
kubectl config current-context
```

If context is wrong or not set:
> "kubectl context not set or incorrect. Run `aws eks update-kubeconfig --name <cluster> --region <region>` to configure."

### Deployment Status

**Get deployment details:**
```bash
kubectl get deployment <name> -n <namespace> -o json
```

**Check rollout status:**
```bash
kubectl rollout status deployment/<name> -n <namespace>
```

**View deployment history:**
```bash
kubectl rollout history deployment/<name> -n <namespace>
```

### Pod Operations

**List pods for a service:**
```bash
kubectl get pods -n <namespace> -l app=<name> -o wide
```

**Get pod details:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

**View pod logs:**
```bash
kubectl logs <pod-name> -n <namespace> --tail=100
```

### Event Monitoring

**Recent events for troubleshooting:**
```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

**Events for specific resource:**
```bash
kubectl get events -n <namespace> --field-selector involvedObject.name=<name>
```

## Error Handling Patterns

All CLI integrations should handle errors gracefully and provide actionable guidance.

### Authentication Failures

| Tool | Error Pattern | User Guidance |
|------|---------------|---------------|
| gh | "gh auth login" / "401" / "Not logged in" | Run `gh auth login` |
| aws | "Unable to locate credentials" / "ExpiredToken" | Run `aws configure` or refresh SSO with `aws sso login` |
| kubectl | "Unable to connect" / "Unauthorized" | Check kubeconfig with `kubectl config view` |

### Network Failures

For transient network issues:
- Retry the operation once after a brief delay
- If still failing, report the specific error
- Suggest checking network connectivity or VPN status

### Permission Errors

When operations fail due to permissions:
- Report the specific permission denied message
- Suggest what permission or role might be needed
- For AWS, point to IAM policy requirements
- For GitHub, suggest checking repository access

### Timeout Handling

For long-running operations:
- Use appropriate timeout flags where available
- `kubectl --request-timeout=30s`
- `gh api --timeout 30s`
- Report timeout clearly, suggest retry

## Output Parsing

### JSON Output

Always request JSON output for programmatic access:

```bash
# GitHub
gh pr view 123 --repo owner/repo --json title,state,url

# AWS
aws ecs describe-services --cluster my-cluster --services my-service --output json

# Kubernetes
kubectl get deployment my-app -n prod -o json
```

### Handling Results

When parsing CLI output:
- Parse JSON with appropriate tools (jq for bash, json module for Python)
- Handle empty results: check array length before accessing elements
- Handle missing fields: use default values or optional chaining

### Empty Results

When a query returns no results:
- Report clearly: "No open PRs found" vs. failing silently
- Distinguish between "none found" and "error occurred"

## On-Demand Validation

Per the KeeperHub design philosophy, integrations are validated on-demand rather than upfront.

### Philosophy

- **Don't pre-validate** at plugin load or command start
- **Try the operation**, handle the failure
- **Show partial results** when some integrations work and others don't

### Example: Mixed Results

When checking status across multiple tools:

```
Jira: [OK] 3 tickets in progress
GitHub: [OK] 2 open PRs
AWS/EKS: [Unavailable] Run `aws configure` to enable deployment status
```

This allows users to get value from working integrations while being informed about unavailable ones.

### Graceful Degradation

Commands should work with whatever integrations are available:
- `/kh:status` shows Jira + GitHub even if AWS is not configured
- Missing integrations are noted but don't block the command
- Error messages explain how to enable the missing integration
