---
description: List Jira tickets assigned to you with optional status filtering
arguments:
  --status: Filter by status (e.g., --status "In Progress", --status todo)
  --project: Filter by project (e.g., --project KH)
  --all: Show all tickets (not just mine)
  --verbose: Show descriptions, dates, and URLs
  --limit: Maximum tickets to show (default: 20)
---

# Tickets Command

List Jira tickets assigned to you with optional status filtering and grouping.

## Usage

```
/kh:tickets [flags]
```

## Workflow

### Step 1: Build JQL Query

Construct the JQL query based on provided flags.

```python
def build_tickets_jql(status=None, project=None, all_tickets=False):
    """Build JQL query for ticket listing."""
    clauses = []

    # Filter by assignee unless --all
    if not all_tickets:
        clauses.append("assignee = currentUser()")

    # Filter by project if specified
    if project:
        clauses.append(f"project = {project}")

    # Filter by status if specified
    if status:
        # Map shorthand to full status names
        status_map = {
            'todo': 'To Do',
            'progress': 'In Progress',
            'staging': 'Testing Staging',
            'prod': 'Testing Prod',
            'done': 'Done',
            'blocked': 'Blocked',
            'wontdo': "Won't do",
        }
        resolved_status = status_map.get(status.lower(), status)
        clauses.append(f'status = "{resolved_status}"')
    else:
        # Default: exclude Done and Won't do (completed/cancelled work)
        clauses.append('status NOT IN (Done, "Won\'t do")')

    # Build final JQL
    jql = " AND ".join(clauses)
    jql += " ORDER BY updated DESC"

    return jql
```

**Example JQL outputs:**
- Default: `assignee = currentUser() AND status NOT IN (Done, "Won't do") ORDER BY updated DESC`
- With status: `assignee = currentUser() AND status = "In Progress" ORDER BY updated DESC`
- With project: `assignee = currentUser() AND project = KH AND status NOT IN (Done, "Won't do") ORDER BY updated DESC`
- All tickets: `status NOT IN (Done, "Won't do") ORDER BY updated DESC`

### Step 2: Execute JQL via Atlassian MCP

Query Jira using the Atlassian MCP.

```
Use atlassian_search_issues with:
- jql: {built_jql}
- limit: {args.limit or 20}
- fields: key, summary, status, assignee, priority, updated, description, project
```

**Field mapping:**
```python
def extract_ticket_fields(issue):
    """Extract relevant fields from Jira issue."""
    return {
        'key': issue.get('key'),
        'summary': issue.get('fields', {}).get('summary', ''),
        'status': issue.get('fields', {}).get('status', {}).get('name', 'Unknown'),
        'assignee': issue.get('fields', {}).get('assignee', {}).get('displayName', 'Unassigned'),
        'priority': issue.get('fields', {}).get('priority', {}).get('name', 'None'),
        'updated': issue.get('fields', {}).get('updated', ''),
        'description': issue.get('fields', {}).get('description', ''),
        'project': issue.get('fields', {}).get('project', {}).get('key', ''),
        'url': f"https://yourorg.atlassian.net/browse/{issue.get('key')}",
    }
```

### Step 3: Group Tickets by Status

When multiple statuses are present, group for readability.

```python
def group_tickets_by_status(tickets):
    """Group tickets by their status."""
    groups = {}
    status_order = ['Blocked', 'In Progress', 'Testing Staging', 'Testing Prod', 'To Do']

    for ticket in tickets:
        status = ticket['status']
        if status not in groups:
            groups[status] = []
        groups[status].append(ticket)

    # Return ordered groups
    ordered = []
    for status in status_order:
        if status in groups:
            ordered.append((status, groups.pop(status)))

    # Append any remaining statuses
    for status, tickets in groups.items():
        ordered.append((status, tickets))

    return ordered
```

### Step 4: Format Ticket Display (Compact)

Default output format for quick scanning.

```python
def format_ticket_compact(ticket):
    """Format a single ticket in compact form."""
    key = ticket['key']
    status = ticket['status']
    summary = ticket['summary']
    assignee = ticket['assignee']

    # Truncate summary at 40 characters
    if len(summary) > 40:
        summary = summary[:37] + "..."

    # Format assignee
    if assignee == 'Unassigned':
        assignee_str = '@unassigned'
    else:
        # Extract first name or use full name
        assignee_str = f"@{assignee.split()[0].lower()}" if assignee else '@me'

    return f"{key}  {summary:<40}  {assignee_str}"
```

**Compact output (grouped by status):**
```
My Tickets (5)
==============

In Progress (2):
KH-123  Implement user authentication         @me
KH-345  Add rate limiting to API              @me

To Do (2):
KH-456  Update API documentation              @me
KH-678  Review security audit findings        @me

Blocked (1):
KH-012  Integrate payment gateway             @me
```

**Compact output (single status filter):**
```
My Tickets - In Progress (2)
============================

KH-123  Implement user authentication         @me
KH-345  Add rate limiting to API              @me
```

**Compact output (with project filter):**
```
My Tickets - KH (3)
===================

INFRA-45  [In Progress]  Upgrade Kubernetes cluster    @me
INFRA-67  [To Do]        Add monitoring dashboards     @me
INFRA-89  [To Do]        Configure auto-scaling        @me
```

### Step 5: Format Ticket Display (Verbose)

Extended output with descriptions and URLs.

```python
def format_ticket_verbose(ticket):
    """Format a single ticket with full details."""
    key = ticket['key']
    status = ticket['status']
    summary = ticket['summary']
    assignee = ticket['assignee']
    priority = ticket['priority']
    updated = ticket['updated']
    description = ticket['description']
    url = ticket['url']

    # Format assignee
    if assignee == 'Unassigned':
        assignee_str = '@unassigned'
    else:
        assignee_str = f"@{assignee.split()[0].lower()}" if assignee else '@me'

    # Truncate summary at 40 characters
    if len(summary) > 40:
        summary_short = summary[:37] + "..."
    else:
        summary_short = summary

    # Format date
    date_str = format_date(updated)

    # Truncate description
    if description and len(description) > 80:
        description_short = description[:77] + "..."
    else:
        description_short = description or "No description"

    lines = [
        f"{key}  [{status}]  {summary_short}  {assignee_str}",
        f"        Priority: {priority} | Updated: {date_str}",
        f"        {description_short}",
        f"        {url}",
    ]

    return "\n".join(lines)
```

**Verbose output:**
```
My Tickets (5)
==============

KH-123  [In Progress]  Implement user authentication         @me
        Priority: High | Updated: 2024-01-25
        As a user, I want to authenticate using OAuth2...
        https://yourorg.atlassian.net/browse/KH-123

KH-456  [To Do]        Update API documentation              @me
        Priority: Medium | Updated: 2024-01-20
        Document all REST endpoints with examples...
        https://yourorg.atlassian.net/browse/KH-456

KH-789  [Testing Staging]  Fix memory leak in worker         @me
        Priority: High | Updated: 2024-01-22
        Memory usage increases over time in the background worker...
        https://yourorg.atlassian.net/browse/KH-789
```

### Step 6: Build Header

Generate appropriate header based on filters.

```python
def build_header(count, status=None, project=None, all_tickets=False):
    """Build the header line based on active filters."""
    parts = []

    if all_tickets:
        parts.append("All Tickets")
    else:
        parts.append("My Tickets")

    if project:
        parts.append(project)

    if status:
        parts.append(status)

    header = " - ".join(parts) + f" ({count})"
    separator = "=" * len(header)

    return f"{header}\n{separator}"
```

### Step 7: Handle Empty Results

Provide helpful guidance when no tickets match.

```
My Tickets
==========
No tickets found matching your criteria.

Try: /kh:tickets --status "In Progress"
     /kh:tickets --all --project KH
     /kh:tickets --status done
```

### Step 8: Complete Render Function

```python
def render_tickets(args):
    """Main render function for tickets command."""
    status = args.get('status')
    project = args.get('project')
    all_tickets = args.get('all', False)
    verbose = args.get('verbose', False)
    limit = args.get('limit', 20)

    # Step 1: Build JQL
    jql = build_tickets_jql(status, project, all_tickets)

    # Step 2: Execute query
    try:
        results = atlassian_search_issues(jql=jql, limit=limit)
    except Exception as e:
        return handle_error(e)

    # Handle empty results
    if not results:
        print(build_header(0, status, project, all_tickets))
        print("\nNo tickets found matching your criteria.\n")
        print("Try: /kh:tickets --status \"In Progress\"")
        print("     /kh:tickets --all --project KH")
        print("     /kh:tickets --status done")
        return

    # Extract ticket data
    tickets = [extract_ticket_fields(issue) for issue in results]

    # Build header
    header = build_header(len(tickets), status, project, all_tickets)
    print(header)
    print()

    # Render tickets
    if status:
        # Single status - no grouping needed
        for ticket in tickets:
            if verbose:
                print(format_ticket_verbose(ticket))
                print()
            else:
                print(format_ticket_compact(ticket))
    else:
        # Multiple statuses - group them
        groups = group_tickets_by_status(tickets)
        for group_status, group_tickets in groups:
            if not verbose:
                print(f"{group_status} ({len(group_tickets)}):")
            for ticket in group_tickets:
                if verbose:
                    print(format_ticket_verbose(ticket))
                    print()
                else:
                    print(format_ticket_compact(ticket))
            if not verbose:
                print()
```

## Error Handling

**MCP not configured:**
```
Jira Tickets
============
[Unavailable] Jira not configured. Ensure Atlassian MCP is set up.

Run /kh:setup to configure Jira access.
```

**Invalid status shortcut:**
```
Unknown status 'inprog'.

Valid shortcuts: todo, progress, staging, prod, done, blocked
Or use full status name: --status "In Progress"
```

**Permission error:**
```
[Error] Cannot access Jira. Check your authentication.

Try refreshing your Atlassian MCP connection.
```

**JQL syntax error:**
```
[Error] Invalid JQL query. Check your filters.

Project '{project}' may not exist or you may not have access.
```

```python
def handle_error(e):
    """Handle errors from Jira queries."""
    error_msg = str(e).lower()

    if 'not configured' in error_msg or 'mcp' in error_msg:
        print("Jira Tickets")
        print("============")
        print("[Unavailable] Jira not configured. Ensure Atlassian MCP is set up.")
        print("\nRun /kh:setup to configure Jira access.")

    elif 'unauthorized' in error_msg or 'forbidden' in error_msg or '401' in error_msg:
        print("[Error] Cannot access Jira. Check your authentication.")
        print("\nTry refreshing your Atlassian MCP connection.")

    elif 'jql' in error_msg or 'syntax' in error_msg:
        print("[Error] Invalid JQL query. Check your filters.")

    else:
        print(f"[Error] {str(e)}")
```

## Status Shortcuts

For convenience, you can use shorthand status names:

| Shorthand | Full Status |
|-----------|-------------|
| todo | To Do |
| progress | In Progress |
| staging | Testing Staging |
| prod | Testing Prod |
| done | Done |
| blocked | Blocked |
| wontdo | Won't do |

Example: `/kh:tickets --status progress` is equivalent to `/kh:tickets --status "In Progress"`

## Examples

**List all my open tickets (default):**
```
/kh:tickets
```

Output:
```
My Tickets (5)
==============

In Progress (2):
KH-123  Implement user authentication         @me
KH-345  Add rate limiting to API              @me

To Do (2):
KH-456  Update API documentation              @me
KH-678  Review security audit findings        @me

Blocked (1):
KH-012  Integrate payment gateway             @me
```

**Filter by status:**
```
/kh:tickets --status progress
```

Output:
```
My Tickets - In Progress (2)
============================

KH-123  Implement user authentication         @me
KH-345  Add rate limiting to API              @me
```

**Filter by project:**
```
/kh:tickets --project INFRA
```

Output:
```
My Tickets - INFRA (3)
======================

INFRA-45  [In Progress]  Upgrade Kubernetes cluster    @me
INFRA-67  [To Do]        Add monitoring dashboards     @me
INFRA-89  [To Do]        Configure auto-scaling        @me
```

**Show all team tickets:**
```
/kh:tickets --all --project KH
```

Output:
```
All Tickets - KH (12)
=====================

In Progress (4):
KH-123  Implement user authentication         @alice
KH-345  Add rate limiting to API              @bob
KH-567  Fix login timeout                     @carol
KH-789  Update user profile page              @dave
...
```

**Verbose output:**
```
/kh:tickets --verbose
```

Output:
```
My Tickets (5)
==============

KH-123  [In Progress]  Implement user authentication         @me
        Priority: High | Updated: 2024-01-25
        As a user, I want to authenticate using OAuth2...
        https://yourorg.atlassian.net/browse/KH-123

KH-456  [To Do]        Update API documentation              @me
        Priority: Medium | Updated: 2024-01-20
        Document all REST endpoints with examples...
        https://yourorg.atlassian.net/browse/KH-456
```

**Combine filters:**
```
/kh:tickets --status staging --project KH
```

Output:
```
My Tickets - KH - Testing Staging (2)
=====================================

KH-234  Feature X complete, testing on staging    @me
KH-456  Bug fix for auth timeout                  @me
```

**Limit results:**
```
/kh:tickets --limit 5
```

Shows only the 5 most recently updated tickets.

**Show completed tickets:**
```
/kh:tickets --status done
```

Output:
```
My Tickets - Done (8)
=====================

KH-100  Initial project setup                     @me
KH-101  Configure CI/CD pipeline                  @me
KH-102  Set up monitoring                         @me
...
```

## Quick Tips

- Use `/kh:ticket KH-123` to view full details of a specific ticket
- Use `/kh:ticket KH-123 --move staging` to transition after deploying to staging
- Default filter excludes Done and Won't do tickets; use `--status done` or `--status wontdo` to see those
- Tickets are sorted by last updated date (newest first)
- Use `--all` to see team workload, not just your tickets

## Aliases

| Alias | Command |
|-------|---------|
| /kh:ts | /kh:tickets |
| /kh:issues | /kh:tickets |
| /kh:my-tickets | /kh:tickets |
