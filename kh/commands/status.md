---
description: View unified dashboard showing Jira tickets and GitHub PRs
arguments:
  --jira: Show only Jira tickets section
  --prs: Show only GitHub PRs section
  --deploys: Show only deploy status section (coming soon)
  --service: Override service auto-detection (e.g., --service=keeperhub-api)
  --verbose: Show detailed output with dates, URLs
  --all: Show all PRs (not just mine)
  --team: Show team-related PRs
---

# Status Command

Display a unified dashboard showing your Jira tickets and GitHub PRs with optional filtering and service context.

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

    # JSON fields to retrieve - note: reviewRequests for reviewer detection
    json_fields = "number,title,headRefName,statusCheckRollup,reviewRequests,url,author"
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
      Branch: fix/memory-leak | Author: @me
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
                print(f"      Branch: {branch} | Author: @{author}")
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

### Step 8: Render Deploys Section (Placeholder)

**Skip if --jira only or --prs only specified.**

Deploy status integration is coming in the next update.

```
Deploy Status
-------------
[Coming Soon] Deploy status will be available in the next update.

To check manually:
- kubectl get deployments -n <namespace>
- gh run list --repo <repo>
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

    if 'jira' in sections:
        jira_actions = gather_jira_actions(service_context)

    if 'prs' in sections:
        pr_actions = gather_pr_actions(service_context, args)

    needs_attention = jira_actions + pr_actions

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
        render_deploys_section(service_context)
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

Deploy Status
-------------
[Coming Soon] Deploy status will be available in the next update.

---
Tickets: 2 | PRs: 2 | Deploys: --
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

Deploy Status
-------------
[Coming Soon] Deploy status will be available in the next update.

---
Tickets: [Error] | PRs: 1 | Deploys: --
```

## Aliases

| Alias | Command |
|-------|---------|
| /kh:s | /kh:status |
