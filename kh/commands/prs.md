---
description: List open PRs with CI status, review state, and preview URLs
arguments:
  --mine: Show only my PRs (default)
  --team: Show all open PRs in the repository
  --needs-review: Show PRs where I'm requested as reviewer
  --ready: Show PRs that are ready to merge (CI passed, approved)
  --verbose: Show full details including description and comments
  --service: Override service auto-detection
---

# PRs Command

List open pull requests with CI status, review state, preview URLs, and linked Jira tickets in a unified view.

## Usage

```
/kh:prs [flags]
```

## Workflow

### Step 1: Determine Service Context

Check for explicit service override, then attempt auto-detection from git remote.

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
def get_current_repo():
    """Detect current repository from git remote."""
    try:
        result = subprocess.run(
            ['git', 'remote', 'get-url', 'origin'],
            capture_output=True,
            text=True
        )
        if result.returncode != 0:
            return None, "Not in a git repository or no remote configured."

        remote_url = result.stdout.strip()
        return parse_github_repo(remote_url), None

    except Exception as e:
        return None, f"Error detecting repository: {e}"

def parse_github_repo(remote_url):
    """Parse owner/repo from git remote URL (HTTPS or SSH)."""
    import re

    # HTTPS: https://github.com/owner/repo.git
    https_match = re.match(r'https://github\.com/([^/]+)/([^/]+?)(?:\.git)?$', remote_url)
    if https_match:
        return f"{https_match.group(1)}/{https_match.group(2)}"

    # SSH: git@github.com:owner/repo.git
    ssh_match = re.match(r'git@github\.com:([^/]+)/([^/]+?)(?:\.git)?$', remote_url)
    if ssh_match:
        return f"{ssh_match.group(1)}/{ssh_match.group(2)}"

    return None
```

**Continue without service context:**
If no service detected and no override, commands still work with the current repository.

### Step 2: Build Filter Criteria

Parse filter flags to determine which PRs to display.

```python
def get_filter_mode(args):
    """Determine PR filter mode based on flags."""
    if args.get('needs_review'):
        return 'needs-review'
    elif args.get('team'):
        return 'team'
    else:
        return 'mine'  # Default

def build_gh_pr_command(service_context, filter_mode, ready_only=False):
    """Build gh pr list command with appropriate filters."""
    base_cmd = ["gh", "pr", "list", "--state", "open"]

    # JSON fields to retrieve
    json_fields = "number,title,author,headRefName,url,labels,statusCheckRollup,reviews,reviewRequests,createdAt,body"
    base_cmd.extend(["--json", json_fields])

    # Apply filter based on mode
    if filter_mode == 'mine':
        base_cmd.extend(["--author", "@me"])
    elif filter_mode == 'needs-review':
        base_cmd.extend(["--search", "review-requested:@me"])
    # 'team' mode: no author filter, shows all open PRs

    # Scope to specific repo if service context available
    if service_context:
        github_repo = service_context.get('github_repo')
        if github_repo:
            base_cmd.extend(["--repo", github_repo])

    return base_cmd
```

### Step 3: Query PRs with gh CLI

Execute the command to fetch PR data.

```bash
gh pr list --state open --json number,title,author,headRefName,url,labels,statusCheckRollup,reviews,reviewRequests,createdAt,body
```

**Execute and parse:**
```python
def fetch_prs(cmd):
    """Execute gh command and parse results."""
    try:
        result = subprocess.run(cmd, capture_output=True, text=True)

        if result.returncode != 0:
            error = result.stderr.strip()
            if 'auth' in error.lower() or 'login' in error.lower() or '401' in error:
                return None, "GitHub not authenticated. Run `gh auth login`"
            return None, f"Error fetching PRs: {error}"

        prs = json.loads(result.stdout)
        return prs, None

    except json.JSONDecodeError:
        return None, "Failed to parse GitHub response"
    except FileNotFoundError:
        return None, "gh CLI not installed - install from https://cli.github.com"
    except Exception as e:
        return None, f"Error: {str(e)}"
```

### Step 4: Calculate Days Open

Convert creation timestamp to days since creation.

```python
def days_open(created_at):
    """Calculate number of days since PR was created."""
    from datetime import datetime, timezone

    if not created_at:
        return 0

    try:
        created = datetime.fromisoformat(created_at.replace('Z', '+00:00'))
        now = datetime.now(timezone.utc)
        delta = now - created
        return delta.days
    except:
        return 0

def format_days(days):
    """Format days count for display."""
    if days == 0:
        return 'today'
    elif days == 1:
        return '1d'
    else:
        return f'{days}d'
```

### Step 5: Extract Preview URL from Labels

Detect PRs with deploy-pr-environment label and derive preview URL.

```python
def get_preview_url(labels, pr_number):
    """Extract preview URL from labels if deploy-pr-environment is present."""
    for label in labels:
        label_name = label.get('name', '')
        if label_name == 'deploy-pr-environment':
            # Check if label description contains URL
            description = label.get('description', '')
            if description and description.startswith('http'):
                return description

            # Derive from PR number using convention
            # Typical pattern: https://pr-{number}.preview.example.com
            return f"https://pr-{pr_number}.preview.example.com"

    return None
```

### Step 6: Detect Linked Jira Ticket

Extract Jira ticket key from branch name or PR title.

```python
def extract_ticket_from_pr(pr):
    """Extract Jira ticket key from branch name or title."""
    import re

    # Check branch name first (most reliable)
    branch = pr.get('headRefName', '')
    match = re.search(r'([A-Z]+-\d+)', branch)
    if match:
        return match.group(1)

    # Check title
    title = pr.get('title', '')
    match = re.search(r'([A-Z]+-\d+)', title)
    if match:
        return match.group(1)

    return None
```

### Step 7: Map CI and Review Status

Convert raw status data to display labels.

```python
def get_ci_status(status_check_rollup):
    """Map statusCheckRollup to display status."""
    if not status_check_rollup:
        return "[CI: None]", None

    states = [check.get('state', '').upper() for check in status_check_rollup]

    if not states:
        return "[CI: None]", None

    if all(s == 'SUCCESS' for s in states):
        return "[CI: Pass]", 'pass'
    elif any(s == 'FAILURE' or s == 'ERROR' for s in states):
        return "[CI: Fail]", 'fail'
    elif any(s == 'PENDING' or s == 'EXPECTED' or s == 'QUEUED' for s in states):
        return "[CI: Pending]", 'pending'
    else:
        return "[CI: Unknown]", None

def get_review_status(reviews, review_requests):
    """Map reviews to display status."""
    if not reviews and not review_requests:
        return "[No Review]", None

    # Check for approvals and change requests
    approved = any(r.get('state') == 'APPROVED' for r in reviews)
    changes_requested = any(r.get('state') == 'CHANGES_REQUESTED' for r in reviews)

    if changes_requested:
        return "[Changes Req]", 'changes'
    elif approved:
        return "[Approved]", 'approved'
    elif review_requests:
        return "[Pending]", 'pending'
    else:
        return "[No Review]", None
```

### Step 8: Check Ready to Merge

Determine if PR is ready to merge (CI passed and approved).

```python
def is_ready_to_merge(pr):
    """Check if PR is ready to merge: CI passed and approved."""
    # Check CI status
    status_checks = pr.get('statusCheckRollup', [])
    ci_passed = False
    if status_checks:
        states = [check.get('state', '').upper() for check in status_checks]
        ci_passed = all(s == 'SUCCESS' for s in states) if states else False

    # Check reviews
    reviews = pr.get('reviews', [])
    has_approval = any(r.get('state') == 'APPROVED' for r in reviews)
    has_changes_requested = any(r.get('state') == 'CHANGES_REQUESTED' for r in reviews)

    return ci_passed and has_approval and not has_changes_requested
```

### Step 9: Display PRs (Compact Format)

Render PRs in compact format for quick scanning.

```python
def render_prs_compact(prs, filter_mode, ready_filter=False):
    """Render PRs in compact format."""

    # Apply ready filter if requested
    if ready_filter:
        prs = [pr for pr in prs if is_ready_to_merge(pr)]

    # Determine header based on filter mode
    if ready_filter:
        header = f"Ready to Merge ({len(prs)})"
    elif filter_mode == 'needs-review':
        header = f"PRs Needing Your Review ({len(prs)})"
    elif filter_mode == 'team':
        header = f"Open PRs ({len(prs)})"
    else:
        header = f"Open PRs ({len(prs)})"

    print(header)
    print("=" * len(header))
    print()

    if not prs:
        print("No open PRs matching your criteria.")
        return

    for pr in prs:
        number = pr.get('number')
        title = pr.get('title', '')[:40]
        author = pr.get('author', {}).get('login', 'unknown')
        branch = pr.get('headRefName', '')
        labels = pr.get('labels', [])
        reviews = pr.get('reviews', [])
        review_requests = pr.get('reviewRequests', [])
        created_at = pr.get('createdAt', '')

        # Get status indicators
        ci_label, _ = get_ci_status(pr.get('statusCheckRollup', []))
        review_label, _ = get_review_status(reviews, review_requests)
        days = format_days(days_open(created_at))

        # Get linked ticket
        ticket = extract_ticket_from_pr(pr)

        # Get preview URL
        preview_url = get_preview_url(labels, number)

        # Format author with @ prefix, pad for alignment
        author_display = f"@{author}"

        # Main line: #number  title  @author  [CI] [Review]  days
        print(f"#{number:<4} {title:<40} {author_display:<12} {ci_label} {review_label:<14} {days}")

        # Second line: branch and ticket info
        info_parts = [f"Branch: {branch}"]
        if ticket:
            info_parts.append(f"Ticket: {ticket}")
        print(f"      {' | '.join(info_parts)}")

        # Third line: preview URL if available
        if preview_url:
            print(f"      Preview: {preview_url}")

        print()
```

### Step 10: Display PRs (Verbose Format)

Render PRs with full details.

```python
def render_prs_verbose(prs, filter_mode, ready_filter=False):
    """Render PRs in verbose format with full details."""

    # Apply ready filter if requested
    if ready_filter:
        prs = [pr for pr in prs if is_ready_to_merge(pr)]

    # Determine header based on filter mode
    if ready_filter:
        header = f"Ready to Merge ({len(prs)})"
    elif filter_mode == 'needs-review':
        header = f"PRs Needing Your Review ({len(prs)})"
    elif filter_mode == 'team':
        header = f"Open PRs ({len(prs)})"
    else:
        header = f"Open PRs ({len(prs)})"

    print(header)
    print("=" * len(header))
    print()

    if not prs:
        print("No open PRs matching your criteria.")
        return

    for pr in prs:
        number = pr.get('number')
        title = pr.get('title', '')
        author = pr.get('author', {}).get('login', 'unknown')
        branch = pr.get('headRefName', '')
        labels = pr.get('labels', [])
        reviews = pr.get('reviews', [])
        review_requests = pr.get('reviewRequests', [])
        created_at = pr.get('createdAt', '')
        body = pr.get('body', '')
        url = pr.get('url', '')

        # Get status indicators
        ci_label, _ = get_ci_status(pr.get('statusCheckRollup', []))
        review_label, _ = get_review_status(reviews, review_requests)
        days = format_days(days_open(created_at))

        # Get linked ticket
        ticket = extract_ticket_from_pr(pr)

        # Get preview URL
        preview_url = get_preview_url(labels, number)

        # Count commits and comments (estimate from body)
        reviews_completed = len([r for r in reviews if r.get('state') in ['APPROVED', 'CHANGES_REQUESTED', 'COMMENTED']])

        # Format author with @ prefix
        author_display = f"@{author}"

        # Main line: #number  title  @author  [CI] [Review]  days
        print(f"#{number:<4} {title:<40} {author_display:<12} {ci_label} {review_label:<14} {days}")

        # Second line: branch and ticket info
        info_parts = [f"Branch: {branch}"]
        if ticket:
            info_parts.append(f"Ticket: {ticket}")
        print(f"      {' | '.join(info_parts)}")

        # Third line: preview URL if available
        if preview_url:
            print(f"      Preview: {preview_url}")

        # Fourth line: description excerpt
        if body:
            # Get first non-empty line, truncated
            description_lines = [line.strip() for line in body.split('\n') if line.strip()]
            if description_lines:
                excerpt = description_lines[0][:60]
                if len(description_lines[0]) > 60:
                    excerpt += '...'
                print(f"      Description: {excerpt}")

        # Fifth line: comment/review count
        print(f"      Comments: {reviews_completed} reviews")

        # Sixth line: URL
        if url:
            print(f"      {url}")

        print()
```

### Step 11: Main Workflow

Complete workflow combining all steps.

```python
def run_prs_command(args):
    """Main PRs command entry point."""

    # Step 1: Get service context
    if args.get('service'):
        service_name, service_context = get_explicit_service(args['service'])
    else:
        # Try to get service context, but don't require it
        github_repo, error = get_current_repo()
        service_context = {'github_repo': github_repo} if github_repo else None

    # Step 2: Determine filter mode
    filter_mode = get_filter_mode(args)
    ready_filter = args.get('ready', False)
    verbose = args.get('verbose', False)

    # Step 3: Build and execute command
    cmd = build_gh_pr_command(service_context, filter_mode)
    prs, error = fetch_prs(cmd)

    if error:
        print(f"Error: {error}")
        return

    # Step 4-10: Render output
    if verbose:
        render_prs_verbose(prs, filter_mode, ready_filter)
    else:
        render_prs_compact(prs, filter_mode, ready_filter)
```

## Error Handling

**GitHub not authenticated:**
```
Error: GitHub not authenticated. Run `gh auth login`
```

**No PRs found:**
```
Open PRs (0)
============

No open PRs matching your criteria.
```

**gh CLI not installed:**
```
Error: gh CLI not installed - install from https://cli.github.com
```

**API error:**
```
Error: Error fetching PRs: {error message}
```

## Examples

**List my open PRs (default):**
```
/kh:prs
```

Output:
```
Open PRs (2)
============

#123  KH-123: Add user auth              @me          [CI: Pass] [Approved]      2d
      Branch: KH-123-auth | Ticket: KH-123
      Preview: https://pr-123.preview.example.com

#456  KH-456: Fix memory leak            @me          [CI: Pending] [Pending]    1d
      Branch: KH-456-fix-memory | Ticket: KH-456
```

**List all team PRs:**
```
/kh:prs --team
```

Output:
```
Open PRs (5)
============

#123  KH-123: Add user auth              @alice       [CI: Pass] [Approved]      2d
      Branch: KH-123-auth | Ticket: KH-123
      Preview: https://pr-123.preview.example.com

#456  Fix memory leak                    @bob         [CI: Fail] [Pending]       5d
      Branch: fix/memory | Ticket: KH-456

#789  Update API docs                    @carol       [CI: Pass] [Changes Req]   1d
      Branch: docs/api

#890  Add caching layer                  @dave        [CI: Pass] [No Review]     3d
      Branch: feat/caching

#901  Refactor auth module               @eve         [CI: Pending] [Pending]    0d
      Branch: refactor/auth
```

**PRs where I'm requested to review:**
```
/kh:prs --needs-review
```

Output:
```
PRs Needing Your Review (2)
===========================

#456  Fix memory leak                    @bob         [CI: Pass] [Pending]       5d
      Branch: fix/memory | Ticket: KH-456

#789  Update API docs                    @carol       [CI: Pass] [Pending]       1d
      Branch: docs/api
```

**Ready to merge PRs:**
```
/kh:prs --ready
```

Output:
```
Ready to Merge (1)
==================

#123  KH-123: Add user auth              @alice       [CI: Pass] [Approved]      2d
      Branch: KH-123-auth | Ticket: KH-123
```

**Team PRs that are ready to merge:**
```
/kh:prs --team --ready
```

Output:
```
Ready to Merge (2)
==================

#123  KH-123: Add user auth              @alice       [CI: Pass] [Approved]      2d
      Branch: KH-123-auth | Ticket: KH-123

#567  Update dependencies                @frank       [CI: Pass] [Approved]      1d
      Branch: chore/deps
```

**Verbose output with details:**
```
/kh:prs --verbose
```

Output:
```
Open PRs (2)
============

#123  KH-123: Add user auth              @alice       [CI: Pass] [Approved]      2d
      Branch: KH-123-auth | Ticket: KH-123
      Preview: https://pr-123.preview.example.com
      Description: Implements OAuth2 authentication with refresh tokens...
      Comments: 3 reviews
      https://github.com/org/repo/pull/123

#456  KH-456: Fix memory leak            @me          [CI: Pending] [Pending]    1d
      Branch: KH-456-fix-memory | Ticket: KH-456
      Description: Fixes memory leak in worker process by properly closing...
      Comments: 1 reviews
      https://github.com/org/repo/pull/456
```

**Override service context:**
```
/kh:prs --service=keeperhub-worker
```

This queries PRs for a specific service repository regardless of your current working directory.

**Combined filters example:**
```
/kh:prs --needs-review --verbose
```

Output shows PRs needing your review with full details including description and comment count.

## Filter Precedence

Filters can be combined. When multiple filters are specified:
- `--needs-review` takes precedence over `--mine`
- `--ready` can be combined with any other filter
- `--team` shows all PRs regardless of author

Examples:
- `/kh:prs --team --ready` - All team PRs that are ready to merge
- `/kh:prs --needs-review --ready` - PRs you need to review that are ready

## Aliases

| Alias | Command | Notes |
|-------|---------|-------|
| /kh:pulls | /kh:prs | Short alias |
| /kh:list-prs | /kh:prs | Verbose alias |

All aliases support the same flags as the main command.
