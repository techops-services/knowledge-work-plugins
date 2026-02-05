---
description: View unified dashboard showing Jira tickets, GitHub PRs, and deployment status
arguments:
  --jira: Show only Jira tickets section
  --prs: Show only GitHub PRs section
  --deploys: Show only deployment status section
  --service: Override service auto-detection (e.g., --service=keeperhub-api)
  --verbose: Show detailed output with dates, URLs, and extended info
  --all: Show all PRs (not just mine)
  --team: Show team-related PRs
---

# Status Command

Display a unified dashboard showing your Jira tickets, GitHub PRs, and deployment status with optional filtering and service context.

## Usage

```
/kh:status [flags]
```

## Workflow

### Step 1: Determine Service Context

Check for explicit service override, then attempt auto-detection.

**If --service flag provided:**
```python
# Use the specified service directly
service_name = args.get('service')
config = load_services_config()
if service_name not in config.get('services', {}):
    print(f"[Warning] Service '{service_name}' not found in config.")
    print("Available services:", list(config.get('services', {}).keys()))
    service_context = None
else:
    service_context = config['services'][service_name]
```

**Otherwise, auto-detect from git remote:**
```python
def get_current_service():
    """Detect current service based on git remote."""
    config = load_services_config()
    if not config:
        return None, "No services configured. Run /kh:setup to configure."

    try:
        result = subprocess.run(
            ['git', 'remote', 'get-url', 'origin'],
            capture_output=True,
            text=True
        )
        if result.returncode != 0:
            return None, "Not in a git repository or no remote configured."

        remote_url = result.stdout.strip()
        current_repo = parse_github_repo(remote_url)

        for name, svc_config in config.get('services', {}).items():
            if svc_config.get('github_repo') == current_repo:
                return name, svc_config

        return None, "Repository not in services config."
    except Exception as e:
        return None, f"Error detecting service: {e}"
```

**Continue without service context:**
If no service detected and no override, commands still work with user-level data.

### Step 2: Determine Which Sections to Show

Parse filter flags to decide what to display.

```python
def get_sections_to_show(args):
    """Determine which sections to render based on flags."""
    explicit_filters = []

    if args.get('jira'):
        explicit_filters.append('jira')
    if args.get('prs'):
        explicit_filters.append('prs')
    if args.get('deploys'):
        explicit_filters.append('deploys')

    # If no explicit filters, show all sections
    if not explicit_filters:
        return ['jira', 'prs', 'deploys']

    return explicit_filters
```

### Step 3: Gather Actionable Items for "Needs Attention" Section

Collect items requiring user action across all sections.

```python
needs_attention = []

# This list is populated during section gathering
# Items marked [ACTION] also appear in their normal sections
```

### Step 4: Display Header

Show service context if detected.

```
=== KeeperHub Status ===
```

If service context available:
```
=== KeeperHub Status: keeperhub-api ===
```

### Step 5: Render Needs Attention Section (Conditional)

Only display if actionable items exist.

```
Needs Attention
---------------
PR #789   Review requested: Add OAuth support
KH-456    New comment from @teammate
```

If no items need attention, skip this section entirely.

### Step 6: Render Jira Section

**Skip if --prs only or --deploys only specified.**

Use Atlassian MCP to search for assigned tickets.

**Build JQL query:**
```python
def build_jira_jql(service_context):
    """Build JQL for current user's tickets."""
    base_jql = "assignee = currentUser() AND status != Done ORDER BY updated DESC"

    if service_context:
        jira_project = service_context.get('jira_project')
        if jira_project:
            return f"project = {jira_project} AND {base_jql}"

    return base_jql
```

**MCP call:**
```
Use atlassian_search_issues with JQL: {jql_query}
Limit results to 10 for compact display.
```

**Display format (compact):**
```
Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication
KH-456  [To Do]        Update API documentation
KH-789  [In Review]    Fix memory leak in worker
```

**Display format (--verbose):**
```
Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication
        Assignee: @me | Created: 2024-01-15
        https://jira.company.com/browse/KH-123

KH-456  [To Do]        Update API documentation
        Assignee: @me | Created: 2024-01-20
        https://jira.company.com/browse/KH-456
```

**Error handling:**
```python
def render_jira_section(service_context, verbose=False):
    """Render Jira tickets section."""
    try:
        # Attempt MCP call
        jql = build_jira_jql(service_context)
        # mcp.atlassian_search_issues(jql=jql, limit=10)
        results = atlassian_search_issues(jql=jql, limit=10)

        if not results:
            print("Jira Tickets")
            print("------------")
            print("No tickets assigned to you.")
            return []

        print("Jira Tickets")
        print("------------")
        action_items = []

        for ticket in results:
            status = ticket.get('status', 'Unknown')
            key = ticket.get('key')
            summary = ticket.get('summary', '')

            # Check for actionable items (e.g., new comments)
            if ticket.get('has_new_comments'):
                action_items.append(f"{key}    New comment")
                print(f"{key}  [{status}]  {summary}  [ACTION]")
            else:
                print(f"{key}  [{status}]  {summary}")

            if verbose:
                assignee = ticket.get('assignee', '@me')
                created = ticket.get('created', 'Unknown')
                url = ticket.get('url', '')
                print(f"        Assignee: {assignee} | Created: {created}")
                if url:
                    print(f"        {url}")
                print()

        return action_items

    except Exception as e:
        error_msg = str(e)
        if 'not configured' in error_msg.lower() or 'mcp' in error_msg.lower():
            print("Jira Tickets")
            print("------------")
            print("[Unavailable] Jira not configured - run /kh:setup")
        else:
            print("Jira Tickets")
            print("------------")
            print(f"[Error] {error_msg}")
        return []
```

### Step 7: Render GitHub PRs Section

**Skip if --jira only or --deploys only specified.**

Use gh CLI to list open PRs.

**Get current GitHub username:**
```bash
gh api user --jq '.login'
```

This returns the authenticated user's login for reviewer detection.

**Build gh command:**
```python
def build_gh_pr_command(service_context, all_prs=False, team=False):
    """Build gh pr list command with appropriate filters."""
    base_cmd = ["gh", "pr", "list", "--state", "open"]

    # JSON fields to retrieve - note: reviewRequests for reviewer detection, reviews for review count
    json_fields = "number,title,headRefName,statusCheckRollup,reviewRequests,reviews,url,author"
    base_cmd.extend(["--json", json_fields])

    # Filter by author unless --all specified
    if not all_prs:
        base_cmd.extend(["--author", "@me"])

    # Add team filter if requested
    if team:
        base_cmd.extend(["--search", "team-review-requested:@me"])

    # Scope to specific repo if service context available
    if service_context:
        github_repo = service_context.get('github_repo')
        if github_repo:
            base_cmd.extend(["--repo", github_repo])

    return base_cmd
```

**Execute command:**
```bash
gh pr list --author @me --state open --json number,title,headRefName,statusCheckRollup,reviewRequests,url,author
```

**Reviewer detection logic:**
```python
def get_current_github_user():
    """Get current authenticated GitHub username."""
    result = subprocess.run(
        ['gh', 'api', 'user', '--jq', '.login'],
        capture_output=True,
        text=True
    )
    if result.returncode == 0:
        return result.stdout.strip()
    return None

def is_review_requested(pr, current_user):
    """Check if current user is requested as reviewer."""
    review_requests = pr.get('reviewRequests', [])
    for request in review_requests:
        # reviewRequests contains user objects with 'login' field
        if request.get('login') == current_user:
            return True
    return False
```

**CI status mapping:**
```python
def get_ci_status(status_check_rollup):
    """Map statusCheckRollup to display status."""
    if not status_check_rollup:
        return "[CI: None]"

    # statusCheckRollup is a list of check states
    states = [check.get('state', '').upper() for check in status_check_rollup]

    if all(s == 'SUCCESS' for s in states):
        return "[CI: Pass]"
    elif any(s == 'FAILURE' or s == 'ERROR' for s in states):
        return "[CI: Fail]"
    elif any(s == 'PENDING' or s == 'EXPECTED' for s in states):
        return "[CI: Pending]"
    else:
        return "[CI: Unknown]"
```

**Display format (compact):**
```
GitHub PRs
----------
#123  [Open] [CI: Pass]     feat: add login endpoint
#456  [Open] [CI: Fail]     fix: memory leak in worker
#789  [Open] [CI: Pending]  [ACTION] chore: update dependencies
```

**Display format (--verbose):**
```
GitHub PRs
----------
#123  [Open] [CI: Pass]     feat: add login endpoint
      Branch: feature/login | Author: @me
      https://github.com/techops/keeperhub-api/pull/123

#456  [Open] [CI: Fail]     fix: memory leak in worker
      Branch: fix/memory-leak | Author: @me | Reviews: 2/3
      https://github.com/techops/keeperhub-api/pull/456
```

**Full render function:**
```python
def render_github_section(service_context, verbose=False, all_prs=False, team=False):
    """Render GitHub PRs section."""
    try:
        # Get current user for reviewer detection
        current_user = get_current_github_user()
        if not current_user:
            print("GitHub PRs")
            print("----------")
            print("[Unavailable] GitHub not authenticated - run `gh auth login`")
            return []

        # Build and execute command
        cmd = build_gh_pr_command(service_context, all_prs, team)
        result = subprocess.run(cmd, capture_output=True, text=True)

        if result.returncode != 0:
            error = result.stderr.strip()
            print("GitHub PRs")
            print("----------")
            if 'auth' in error.lower() or 'login' in error.lower():
                print("[Unavailable] GitHub not authenticated - run `gh auth login`")
            else:
                print(f"[Error] {error}")
            return []

        prs = json.loads(result.stdout)

        if not prs:
            print("GitHub PRs")
            print("----------")
            print("No open PRs found.")
            return []

        print("GitHub PRs")
        print("----------")
        action_items = []

        for pr in prs:
            number = pr.get('number')
            title = pr.get('title', '')
            ci_status = get_ci_status(pr.get('statusCheckRollup', []))

            # Check if review requested from current user
            needs_review = is_review_requested(pr, current_user)

            if needs_review:
                action_items.append(f"PR #{number}   Review requested: {title[:40]}")
                print(f"#{number}  [Open] {ci_status}  [ACTION] {title}")
            else:
                print(f"#{number}  [Open] {ci_status}  {title}")

            if verbose:
                branch = pr.get('headRefName', '')
                author = pr.get('author', {}).get('login', '@me')
                url = pr.get('url', '')
                # Count reviews (approved/pending/changes_requested)
                reviews = pr.get('reviews', [])
                review_requests = pr.get('reviewRequests', [])
                reviews_completed = len([r for r in reviews if r.get('state') in ['APPROVED', 'CHANGES_REQUESTED']])
                reviews_requested = len(review_requests) + reviews_completed
                review_info = f" | Reviews: {reviews_completed}/{reviews_requested}" if reviews_requested > 0 else ""
                print(f"      Branch: {branch} | Author: @{author}{review_info}")
                if url:
                    print(f"      {url}")
                print()

        return action_items

    except json.JSONDecodeError:
        print("GitHub PRs")
        print("----------")
        print("[Error] Failed to parse GitHub response")
        return []
    except FileNotFoundError:
        print("GitHub PRs")
        print("----------")
        print("[Unavailable] gh CLI not installed - install from https://cli.github.com")
        return []
    except Exception as e:
        print("GitHub PRs")
        print("----------")
        print(f"[Error] {str(e)}")
        return []
```

### Step 8: Render Deploys Section

**Skip if --jira only or --prs only specified.**

**Requires service context:** Deployment status needs namespace and cluster information from service configuration.

**If no service context:**
```
Deployments
-----------
[Service context required] Use --service=name or run from a configured repo
```

**Get derived configuration:**
```python
def get_deploy_config(service_name, service_context):
    """Get deployment configuration with derived defaults."""
    return {
        'eks_cluster': service_context.get('eks_cluster', f'keeperhub-prod'),
        'namespace_staging': service_context.get('namespace_staging', f'{service_name}-staging'),
        'namespace_prod': service_context.get('namespace_prod', service_name),
        'aws_region': service_context.get('aws_region', 'us-east-1'),
    }
```

**Check kubectl context:**
```python
def check_kubectl_context(expected_cluster):
    """Verify kubectl is configured for the correct cluster."""
    result = subprocess.run(
        ['kubectl', 'config', 'current-context'],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        return None, "kubectl not configured"

    current_context = result.stdout.strip()
    # EKS contexts typically contain the cluster name
    if expected_cluster not in current_context:
        return current_context, f"context mismatch (expected {expected_cluster})"

    return current_context, None
```

**Get deployment status for an environment:**
```bash
# Get deployment details with JSON output
kubectl get deployment {service-name} -n {namespace} -o json

# Check rollout status (with timeout to avoid hanging)
kubectl rollout status deployment/{service-name} -n {namespace} --timeout=5s
```

**Parse deployment JSON:**
```python
def parse_deployment_status(deployment_json):
    """Extract status from kubectl deployment JSON."""
    spec = deployment_json.get('spec', {})
    status = deployment_json.get('status', {})

    # Get replica counts
    desired = spec.get('replicas', 0)
    ready = status.get('readyReplicas', 0)
    available = status.get('availableReplicas', 0)

    # Get image version from first container
    containers = spec.get('template', {}).get('spec', {}).get('containers', [])
    image = containers[0].get('image', '') if containers else ''
    version = image.split(':')[-1] if ':' in image else 'latest'

    # Determine status
    conditions = status.get('conditions', [])
    for condition in conditions:
        if condition.get('type') == 'Available':
            if condition.get('status') == 'True':
                deploy_status = 'Deployed'
            else:
                deploy_status = 'Pending'
            break
        if condition.get('type') == 'Progressing':
            if condition.get('reason') == 'ProgressDeadlineExceeded':
                deploy_status = 'Failed'
                break
    else:
        deploy_status = 'Unknown'

    # Get last update time
    last_update = None
    for condition in conditions:
        if condition.get('lastUpdateTime'):
            last_update = condition.get('lastUpdateTime')
            break

    return {
        'status': deploy_status,
        'version': version,
        'image': image,
        'replicas_ready': ready,
        'replicas_desired': desired,
        'last_update': last_update,
    }
```

**Calculate relative time:**
```python
def relative_time(iso_timestamp):
    """Convert ISO timestamp to relative time (e.g., '2h ago', '3d ago')."""
    from datetime import datetime, timezone

    if not iso_timestamp:
        return 'unknown'

    try:
        # Parse ISO 8601 timestamp
        dt = datetime.fromisoformat(iso_timestamp.replace('Z', '+00:00'))
        now = datetime.now(timezone.utc)
        delta = now - dt

        seconds = delta.total_seconds()
        if seconds < 60:
            return 'just now'
        elif seconds < 3600:
            mins = int(seconds / 60)
            return f'{mins}m ago'
        elif seconds < 86400:
            hours = int(seconds / 3600)
            return f'{hours}h ago'
        else:
            days = int(seconds / 86400)
            return f'{days}d ago'
    except:
        return 'unknown'
```

**Full render function:**
```python
def render_deploys_section(service_name, service_context, verbose=False):
    """Render deployment status section."""

    # Check for service context
    if not service_context:
        print("Deployments")
        print("-----------")
        print("[Service context required] Use --service=name or run from a configured repo")
        return []

    deploy_config = get_deploy_config(service_name, service_context)
    action_items = []

    # Check kubectl context
    current_context, context_error = check_kubectl_context(deploy_config['eks_cluster'])
    if context_error:
        print("Deployments")
        print("-----------")
        if context_error == "kubectl not configured":
            cluster = deploy_config['eks_cluster']
            region = deploy_config['aws_region']
            print(f"[Unavailable] kubectl not configured - run `aws eks update-kubeconfig --name {cluster} --region {region}`")
        else:
            print(f"[Context mismatch] kubectl context is {current_context}, expected {deploy_config['eks_cluster']}")
            print(f"Run: aws eks update-kubeconfig --name {deploy_config['eks_cluster']} --region {deploy_config['aws_region']}")
        return []

    environments = [
        ('staging', deploy_config['namespace_staging']),
        ('prod', deploy_config['namespace_prod']),
    ]

    print("Deployments")
    print("-----------")

    for env_name, namespace in environments:
        try:
            # Get deployment JSON
            result = subprocess.run(
                ['kubectl', 'get', 'deployment', service_name, '-n', namespace, '-o', 'json'],
                capture_output=True,
                text=True,
                timeout=10
            )

            if result.returncode != 0:
                error = result.stderr.strip()
                if 'not found' in error.lower():
                    print(f"{env_name:8} [Not Found]  No deployment in namespace {namespace}")
                elif 'forbidden' in error.lower() or 'unauthorized' in error.lower():
                    print(f"{env_name:8} [No Access]  Permission denied for namespace {namespace}")
                else:
                    print(f"{env_name:8} [Error]  {error[:50]}")
                continue

            deployment = json.loads(result.stdout)
            status_info = parse_deployment_status(deployment)

            # Format status display
            status_label = status_info['status']
            version = status_info['version']
            relative = relative_time(status_info['last_update'])

            # Check for failed state - add to action items
            if status_label == 'Failed':
                action_items.append(f"{env_name} [Failed] - deployment rollout failed")
                print(f"{env_name:8} [Failed]    {version}  {relative}")
            elif status_label == 'Pending':
                print(f"{env_name:8} [Pending]   {version}  {relative}")
            else:
                print(f"{env_name:8} [Deployed]  {version}  {relative}")

            if verbose:
                image = status_info['image']
                ready = status_info['replicas_ready']
                desired = status_info['replicas_desired']
                last_update = status_info['last_update'] or 'unknown'
                print(f"         Image: {image}")
                print(f"         Replicas: {ready}/{desired} ready  Updated: {last_update}")
                print()

        except subprocess.TimeoutExpired:
            print(f"{env_name:8} [Timeout]   kubectl command timed out")
        except json.JSONDecodeError:
            print(f"{env_name:8} [Error]     Failed to parse kubectl output")
        except FileNotFoundError:
            print("Deployments")
            print("-----------")
            print("[Unavailable] kubectl not installed")
            return []
        except Exception as e:
            print(f"{env_name:8} [Error]     {str(e)[:50]}")

    return action_items
```

**Display format (compact):**
```
Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
prod     [Deployed]  v1.2.2  3d ago
```

**Display format (--verbose):**
```
Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
         Image: registry.io/app:v1.2.3
         Replicas: 3/3 ready  Updated: 2024-01-25T14:30:00Z

prod     [Deployed]  v1.2.2  3d ago
         Image: registry.io/app:v1.2.2
         Replicas: 5/5 ready  Updated: 2024-01-22T09:15:00Z
```

**Error states:**

kubectl not configured:
```
Deployments
-----------
[Unavailable] kubectl not configured - run `aws eks update-kubeconfig --name keeperhub-prod --region us-east-1`
```

Wrong context:
```
Deployments
-----------
[Context mismatch] kubectl context is minikube, expected keeperhub-prod
Run: aws eks update-kubeconfig --name keeperhub-prod --region us-east-1
```

Permission denied:
```
Deployments
-----------
staging  [No Access]  Permission denied for namespace keeperhub-api-staging
prod     [Deployed]   v1.2.2  3d ago
```

Deployment not found:
```
Deployments
-----------
staging  [Not Found]  No deployment in namespace keeperhub-api-staging
prod     [Deployed]   v1.2.2  3d ago
```

**Failed deployments in Needs Attention:**

When a deployment has failed, it appears in the Needs Attention section:
```
Needs Attention
---------------
prod [Failed] - deployment rollout failed
PR #789   Review requested: Add OAuth support
```

### Step 9: Summary Footer

Show counts and any errors that occurred.

```
---
Tickets: 3 | PRs: 2 | Deploys: --
```

If errors occurred:
```
---
Tickets: 3 | PRs: [Error] | Deploys: --
```

## Complete Workflow Example

```python
def run_status_command(args):
    """Main status command entry point."""
    # Step 1: Get service context
    if args.get('service'):
        service_name, service_context = get_explicit_service(args['service'])
    else:
        service_name, service_context = get_current_service()

    # Step 2: Determine sections
    sections = get_sections_to_show(args)

    # Step 3: Initialize needs attention list
    needs_attention = []

    # Step 4: Print header
    if service_name:
        print(f"=== KeeperHub Status: {service_name} ===")
    else:
        print("=== KeeperHub Status ===")
    print()

    # Gather data from all sections first
    jira_actions = []
    pr_actions = []
    deploy_actions = []

    if 'jira' in sections:
        jira_actions = gather_jira_actions(service_context)

    if 'prs' in sections:
        pr_actions = gather_pr_actions(service_context, args)

    if 'deploys' in sections:
        deploy_actions = gather_deploy_actions(service_name, service_context)

    needs_attention = deploy_actions + jira_actions + pr_actions  # Deploys first (most urgent)

    # Step 5: Render needs attention (if any)
    if needs_attention:
        print("Needs Attention")
        print("---------------")
        for item in needs_attention:
            print(item)
        print()

    # Step 6-8: Render sections
    if 'jira' in sections:
        render_jira_section(service_context, verbose=args.get('verbose'))
        print()

    if 'prs' in sections:
        render_github_section(
            service_context,
            verbose=args.get('verbose'),
            all_prs=args.get('all'),
            team=args.get('team')
        )
        print()

    if 'deploys' in sections:
        render_deploys_section(service_name, service_context, verbose=args.get('verbose'))
        print()

    # Step 9: Footer
    print("---")
    # Counts rendered based on what was shown
```

## Examples

**Basic status check:**
```
/kh:status
```

Output:
```
=== KeeperHub Status: keeperhub-api ===

Needs Attention
---------------
PR #789   Review requested: Add OAuth support

Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication
KH-456  [To Do]        Update API documentation

GitHub PRs
----------
#123  [Open] [CI: Pass]  feat: add login endpoint
#789  [Open] [CI: Pass]  [ACTION] Add OAuth support

Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
prod     [Deployed]  v1.2.2  3d ago

---
Tickets: 2 | PRs: 2 | Deploys: 2
```

**Just Jira tickets:**
```
/kh:status --jira
```

**Just GitHub PRs:**
```
/kh:status --prs
```

**All team PRs (not just mine):**
```
/kh:status --prs --all
```

**Detailed output:**
```
/kh:status --verbose
```

Output:
```
=== KeeperHub Status: keeperhub-api ===

Needs Attention
---------------
PR #789   Review requested: Add OAuth support

Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication
        Assignee: @john.doe | Created: 2024-01-15
        https://yourorg.atlassian.net/browse/KH-123

KH-456  [To Do]        Update API documentation
        Assignee: @jane.doe | Created: 2024-01-20
        https://yourorg.atlassian.net/browse/KH-456

GitHub PRs
----------
#123  [Open] [CI: Pass]  feat: add login endpoint
      Branch: feature/login | Author: @johndoe
      https://github.com/org/repo/pull/123

#456  [Open] [CI: Fail]  fix: memory leak
      Branch: fix/memory | Author: @janedoe | Reviews: 2/3
      https://github.com/org/repo/pull/456

Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
         Image: registry.io/app:v1.2.3
         Replicas: 3/3 ready  Updated: 2024-01-25T14:30:00Z

prod     [Deployed]  v1.2.2  3d ago
         Image: registry.io/app:v1.2.2
         Replicas: 5/5 ready  Updated: 2024-01-22T09:15:00Z

---
Tickets: 2 | PRs: 2 | Deploys: 2
```

**Override service context:**
```
/kh:status --service=keeperhub-worker
```

## Error States

**No service context, no config:**
```
=== KeeperHub Status ===

[Note] No service configured. Showing user-level data only.
Run /kh:setup to configure services for service-scoped status.

Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication

GitHub PRs
----------
#123  [Open] [CI: Pass]  feat: add login endpoint

---
Tickets: 1 | PRs: 1 | Deploys: --
```

**Mixed availability:**
```
=== KeeperHub Status: keeperhub-api ===

Jira Tickets
------------
[Unavailable] Jira not configured - run /kh:setup

GitHub PRs
----------
#123  [Open] [CI: Pass]  feat: add login endpoint

Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
prod     [Deployed]  v1.2.2  3d ago

---
Tickets: [Error] | PRs: 1 | Deploys: 2
```

**Failed deployment:**
```
=== KeeperHub Status: keeperhub-api ===

Needs Attention
---------------
prod [Failed] - deployment rollout failed

Jira Tickets
------------
KH-123  [In Progress]  Implement user authentication

GitHub PRs
----------
#123  [Open] [CI: Pass]  feat: add login endpoint

Deployments
-----------
staging  [Deployed]  v1.2.3  2h ago
prod     [Failed]    v1.2.4  5m ago

---
Tickets: 1 | PRs: 1 | Deploys: 2
```

## Aliases

| Alias | Command |
|-------|---------|
| /kh:s | /kh:status |
