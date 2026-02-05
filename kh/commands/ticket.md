---
description: View Jira ticket details and transition status
arguments:
  id: Ticket ID (required, e.g., KH-123, PROJ-456)
  --move: Transition ticket to new status (e.g., --move "In Progress")
  --verbose: Show full history, comments, and linked PRs
  --comment: Add a comment to the ticket
---

# Ticket Command

View Jira ticket details, transition status, and add comments without leaving Claude Code.

## Usage

```
/kh:ticket <id> [flags]
```

## Workflow

### Step 1: Parse and Validate Ticket ID

Accept ticket ID with or without project prefix and validate format.

**Parse ticket ID:**
```python
import re

def parse_ticket_id(ticket_id):
    """Parse and validate Jira ticket ID.

    Accepts:
    - KH-123 (with project prefix)
    - PROJ-456 (any project prefix)
    - 123 (number only - requires default project from config)

    Returns: normalized ticket ID or error message
    """
    if not ticket_id:
        return None, "Ticket ID is required. Usage: /kh:ticket <id>"

    ticket_id = ticket_id.strip().upper()

    # Pattern: uppercase letters followed by hyphen and digits
    pattern = r'^[A-Z]+-\d+$'
    if re.match(pattern, ticket_id):
        return ticket_id, None

    # Check if just a number (needs default project)
    if ticket_id.isdigit():
        config = load_kh_config()
        default_project = config.get('jira_project') if config else None
        if default_project:
            return f"{default_project}-{ticket_id}", None
        return None, f"Invalid ticket format. Use PROJECT-NUMBER (e.g., KH-123)"

    return None, f"Invalid ticket format. Use PROJECT-NUMBER (e.g., KH-123)"
```

### Step 2: Fetch Ticket Details via Atlassian MCP

Query Jira for complete ticket information.

**MCP call:**
```
Use atlassian_get_issue with issue_key: {ticket_id}
Fields to retrieve:
  - key (ticket ID)
  - summary (ticket title)
  - status (current status name)
  - assignee (display name and account ID)
  - reporter (display name)
  - description (full description text)
  - created (creation timestamp)
  - updated (last update timestamp)
  - priority (priority name)
  - labels (array of labels)
  - comments (array of comment objects with author, body, created)
  - transitions (available transitions from current status)
```

**Parse ticket response:**
```python
def parse_ticket_response(response):
    """Extract relevant fields from Jira API response."""
    fields = response.get('fields', {})

    # Get assignee info
    assignee = fields.get('assignee')
    assignee_name = assignee.get('displayName', 'Unassigned') if assignee else 'Unassigned'

    # Get reporter info
    reporter = fields.get('reporter')
    reporter_name = reporter.get('displayName', 'Unknown') if reporter else 'Unknown'

    # Get status
    status = fields.get('status', {})
    status_name = status.get('name', 'Unknown')

    # Get priority
    priority = fields.get('priority')
    priority_name = priority.get('name', 'None') if priority else 'None'

    # Parse timestamps
    created = fields.get('created', '')[:10]  # YYYY-MM-DD
    updated = fields.get('updated', '')[:10]  # YYYY-MM-DD

    # Get labels
    labels = fields.get('labels', [])

    # Get comments
    comment_data = fields.get('comment', {})
    comments = comment_data.get('comments', []) if isinstance(comment_data, dict) else []

    return {
        'key': response.get('key'),
        'summary': fields.get('summary', 'No summary'),
        'status': status_name,
        'assignee': assignee_name,
        'reporter': reporter_name,
        'description': fields.get('description', 'No description'),
        'created': created,
        'updated': updated,
        'priority': priority_name,
        'labels': labels,
        'comments': comments,
    }
```

**Error handling:**
```python
def fetch_ticket(ticket_id):
    """Fetch ticket details from Jira via Atlassian MCP."""
    try:
        # MCP call to atlassian_get_issue
        response = atlassian_get_issue(issue_key=ticket_id)

        if not response:
            return None, f"Ticket {ticket_id} not found. Check the ticket ID."

        return parse_ticket_response(response), None

    except Exception as e:
        error_msg = str(e).lower()
        if 'not found' in error_msg or '404' in error_msg:
            return None, f"Ticket {ticket_id} not found. Check the ticket ID and project key."
        elif 'not configured' in error_msg or 'mcp' in error_msg or 'authentication' in error_msg:
            return None, "Jira not configured. Ensure Atlassian MCP is set up in your Claude Code settings."
        elif 'permission' in error_msg or 'forbidden' in error_msg or '403' in error_msg:
            return None, f"You don't have permission to view ticket {ticket_id}."
        else:
            return None, f"Error fetching ticket: {str(e)}"
```

### Step 3: Fetch Available Transitions

Get the list of valid status transitions for the ticket.

**MCP call:**
```
Use atlassian_get_transitions with issue_key: {ticket_id}
Returns array of transition objects with:
  - id (transition ID for execution)
  - name (display name like "In Progress", "Testing Staging")
  - to (target status object)
```

**Parse transitions:**
```python
def fetch_transitions(ticket_id):
    """Fetch available transitions for a ticket."""
    try:
        response = atlassian_get_transitions(issue_key=ticket_id)
        transitions = response.get('transitions', [])

        return [
            {
                'id': t.get('id'),
                'name': t.get('name'),
                'to_status': t.get('to', {}).get('name', t.get('name'))
            }
            for t in transitions
        ], None

    except Exception as e:
        return [], f"Could not fetch transitions: {str(e)}"
```

### Step 4: Fetch Linked PRs from GitHub

Search for PRs that mention this ticket.

**gh CLI command:**
```bash
# Search for PRs mentioning this ticket in any state
gh pr list --state all --search "KH-123" --json number,title,state,url --limit 10
```

**Parse PR results:**
```python
def fetch_linked_prs(ticket_id):
    """Find GitHub PRs that mention this ticket."""
    try:
        result = subprocess.run(
            ['gh', 'pr', 'list', '--state', 'all', '--search', ticket_id,
             '--json', 'number,title,state,url', '--limit', '10'],
            capture_output=True,
            text=True,
            timeout=30
        )

        if result.returncode != 0:
            error = result.stderr.strip().lower()
            if 'auth' in error or 'login' in error:
                return [], "GitHub not authenticated"
            return [], None  # Silent failure for other errors

        import json
        prs = json.loads(result.stdout)

        # Filter to only PRs that actually mention the ticket ID
        linked = []
        for pr in prs:
            title = pr.get('title', '').upper()
            if ticket_id.upper() in title:
                linked.append({
                    'number': pr.get('number'),
                    'title': pr.get('title'),
                    'state': pr.get('state', 'OPEN'),
                    'url': pr.get('url'),
                })

        return linked, None

    except subprocess.TimeoutExpired:
        return [], "GitHub search timed out"
    except FileNotFoundError:
        return [], "gh CLI not installed"
    except Exception as e:
        return [], str(e)
```

### Step 5: Display Basic Ticket Information

Format and display the ticket details.

**Display format:**
```python
def display_ticket(ticket, linked_prs, verbose=False):
    """Display ticket details."""
    key = ticket['key']
    summary = ticket['summary']

    # Header with title
    print(f"{key}: {summary}")
    print("=" * (len(key) + len(summary) + 2))

    # Core fields
    print(f"Status:     [{ticket['status']}]")
    print(f"Assignee:   @{ticket['assignee'].lower().replace(' ', '.')}")
    print(f"Priority:   {ticket['priority']}")
    print(f"Created:    {ticket['created']}")
    print(f"Updated:    {ticket['updated']}")
    print()

    # Description
    print("Description:")
    description = ticket['description']
    if description:
        # Truncate long descriptions unless verbose
        if not verbose and len(description) > 300:
            description = description[:300] + "..."
        print(description)
    else:
        print("No description provided.")
    print()

    # Labels
    labels = ticket['labels']
    if labels:
        print(f"Labels: {', '.join(labels)}")
        print()

    # Linked PRs
    if linked_prs:
        print("Linked PRs:")
        for pr in linked_prs:
            state = pr['state'].capitalize()
            state_label = "[Merged]" if state == "Merged" else "[Open]" if state == "Open" else f"[{state}]"
            print(f"- #{pr['number']} {state_label} {pr['title']}")
        print()
```

**Output example:**
```
KH-123: Implement user authentication
======================================
Status:     [In Progress]
Assignee:   @alice.smith
Priority:   High
Created:    2024-01-15
Updated:    2024-01-25

Description:
As a user, I want to authenticate using OAuth2 so that I can
securely access my account without managing passwords.

Labels: backend, security, sprint-5

Linked PRs:
- #456 [Merged] feat: add auth endpoint
- #789 [Open]   fix: auth token expiry
```

### Step 6: Handle --move Transition

Transition the ticket to a new status.

**Validate and execute transition:**
```python
def handle_move(ticket_id, target_status, available_transitions):
    """Transition ticket to new status."""
    if not available_transitions:
        return None, "No transitions available for this ticket."

    # Normalize target status for matching
    target_normalized = target_status.lower().strip()

    # Map shorthand to full status names
    shorthand_map = {
        'progress': 'in progress',
        'staging': 'testing staging',
        'prod': 'testing prod',
        'done': 'done',
        'blocked': 'blocked',
        'todo': 'to do',
        'to do': 'to do',
    }

    if target_normalized in shorthand_map:
        target_normalized = shorthand_map[target_normalized]

    # Find matching transition
    matched_transition = None
    for transition in available_transitions:
        if transition['name'].lower() == target_normalized:
            matched_transition = transition
            break
        if transition['to_status'].lower() == target_normalized:
            matched_transition = transition
            break

    if not matched_transition:
        available_names = [t['name'] for t in available_transitions]
        return None, f"Cannot move to '{target_status}'. Available: {', '.join(available_names)}"

    # Execute transition via MCP
    try:
        atlassian_transition_issue(
            issue_key=ticket_id,
            transition_id=matched_transition['id']
        )
        return matched_transition['to_status'], None

    except Exception as e:
        error_msg = str(e).lower()
        if 'permission' in error_msg or 'forbidden' in error_msg:
            return None, f"You don't have permission to transition this ticket."
        return None, f"Failed to transition: {str(e)}"
```

**Display transition result:**
```python
def display_transition_result(ticket_id, from_status, to_status):
    """Display transition success message."""
    print(f"Transitioned {ticket_id} from '{from_status}' to '{to_status}'")
    print()
```

**Output example:**
```
Transitioned KH-123 from 'In Progress' to 'Testing Staging'

KH-123: Implement user authentication
Status:     [Testing Staging]
...
```

### Step 7: Handle --verbose Mode

Show extended information including comments and activity.

**Display comments:**
```python
def display_comments(comments, limit=5):
    """Display recent comments."""
    if not comments:
        return

    # Sort by created date, newest first
    sorted_comments = sorted(
        comments,
        key=lambda c: c.get('created', ''),
        reverse=True
    )[:limit]

    print(f"Recent Comments ({len(sorted_comments)}):")
    print("-" * 15)

    for comment in sorted_comments:
        author = comment.get('author', {})
        author_name = author.get('displayName', 'Unknown') if isinstance(author, dict) else 'Unknown'
        created = comment.get('created', '')[:10]
        body = comment.get('body', '')

        # Truncate long comments
        if len(body) > 200:
            body = body[:200] + "..."

        print(f"@{author_name.lower().replace(' ', '.')} ({created}):")
        print(body)
        print()
```

**Display activity (verbose mode):**
```python
def display_activity(ticket):
    """Display ticket activity history."""
    print("Activity:")
    # Note: Full activity requires changelog which may need separate API call
    # For now, show key dates
    print(f"- {ticket['updated']}: Last updated")
    print(f"- {ticket['created']}: Created by @{ticket['reporter'].lower().replace(' ', '.')}")
    print()
```

**Verbose output example:**
```
KH-123: Implement user authentication
======================================
Status:     [In Progress]
Assignee:   @alice.smith
Priority:   High
Created:    2024-01-15
Updated:    2024-01-25

Description:
As a user, I want to authenticate using OAuth2 so that I can
securely access my account without managing passwords.

Acceptance criteria:
- Support Google OAuth2
- Support GitHub OAuth2
- Session management with secure cookies
- Rate limiting on auth endpoints

Labels: backend, security, sprint-5

Linked PRs:
- #456 [Merged] KH-123: Add auth endpoint
- #789 [Open] KH-123: Fix auth token expiry

Recent Comments (3):
---------------
@bob.smith (2024-01-25):
PR approved, ready for staging deploy.

@alice.smith (2024-01-24):
Tests passing, requesting review.

@pm.person (2024-01-20):
Adding to sprint 5, priority high.

Activity:
- 2024-01-25: Last updated
- 2024-01-15: Created by @pm.person
```

### Step 8: Handle --comment

Add a comment to the ticket.

**Add comment via MCP:**
```python
def handle_comment(ticket_id, comment_text):
    """Add a comment to the ticket."""
    if not comment_text or not comment_text.strip():
        return None, "Comment text is required."

    try:
        atlassian_add_comment(
            issue_key=ticket_id,
            body=comment_text.strip()
        )

        from datetime import datetime
        now = datetime.now().strftime("%Y-%m-%d %H:%M")

        return {
            'text': comment_text.strip(),
            'time': now,
        }, None

    except Exception as e:
        error_msg = str(e).lower()
        if 'permission' in error_msg or 'forbidden' in error_msg:
            return None, "You don't have permission to comment on this ticket."
        return None, f"Failed to add comment: {str(e)}"
```

**Display comment result:**
```python
def display_comment_result(ticket_id, comment_info):
    """Display comment success message."""
    print(f"Added comment to {ticket_id}")
    print()
    print(f"Comment: {comment_info['text']}")
    print(f"Author: @me")
    print(f"Time: {comment_info['time']}")
```

**Output example:**
```
Added comment to KH-123

Comment: Deployed to staging, ready for QA
Author: @me
Time: 2024-01-26 10:30
```

## Complete Workflow

```python
def run_ticket_command(args):
    """Main ticket command entry point."""
    # Step 1: Parse ticket ID
    ticket_id = args.get('id')
    ticket_id, error = parse_ticket_id(ticket_id)
    if error:
        print(f"[Error] {error}")
        return

    # Step 2: Fetch ticket details
    ticket, error = fetch_ticket(ticket_id)
    if error:
        print(f"[Error] {error}")
        return

    original_status = ticket['status']

    # Step 3: Fetch available transitions (needed for --move)
    transitions, _ = fetch_transitions(ticket_id)

    # Step 4: Fetch linked PRs
    linked_prs, pr_error = fetch_linked_prs(ticket_id)
    if pr_error and 'not authenticated' in pr_error.lower():
        print(f"[Note] GitHub PRs unavailable: {pr_error}")

    # Handle --move flag first (if provided)
    if args.get('move'):
        target_status = args['move']
        new_status, error = handle_move(ticket_id, target_status, transitions)
        if error:
            print(f"[Error] {error}")
            return
        display_transition_result(ticket_id, original_status, new_status)
        # Refresh ticket data after transition
        ticket, _ = fetch_ticket(ticket_id)
        if not ticket:
            return

    # Handle --comment flag (if provided)
    if args.get('comment'):
        comment_text = args['comment']
        comment_info, error = handle_comment(ticket_id, comment_text)
        if error:
            print(f"[Error] {error}")
            return
        display_comment_result(ticket_id, comment_info)
        return

    # Step 5: Display ticket
    verbose = args.get('verbose', False)
    display_ticket(ticket, linked_prs, verbose)

    # Step 7: Show comments and activity in verbose mode
    if verbose and ticket.get('comments'):
        display_comments(ticket['comments'])
        display_activity(ticket)
```

## Workflow Statuses

The standard KeeperHub Jira workflow supports these transitions:

| From | To | When |
|------|----|------|
| To Do | In Progress | Starting work on ticket |
| In Progress | Testing Staging | PR merged to staging |
| Testing Staging | Testing Prod | Staging verified, promoted to prod |
| Testing Prod | Done | Production verified and complete |

Any status can move to "Blocked" if work is blocked.

## Examples

**View ticket details:**
```
/kh:ticket KH-123
```

Output:
```
KH-123: Implement user authentication
======================================
Status:     [In Progress]
Assignee:   @alice.smith
Priority:   High
Created:    2024-01-15
Updated:    2024-01-25

Description:
As a user, I want to authenticate using OAuth2 so that I can
securely access my account without managing passwords.

Labels: backend, security, sprint-5

Linked PRs:
- #456 [Open] KH-123: Add auth endpoint
```

**Transition to a new status:**
```
/kh:ticket KH-123 --move "Testing Staging"
```

Output:
```
Transitioned KH-123 from 'In Progress' to 'Testing Staging'

KH-123: Implement user authentication
======================================
Status:     [Testing Staging]
Assignee:   @alice.smith
Priority:   High
Created:    2024-01-15
Updated:    2024-01-26

Description:
As a user, I want to authenticate using OAuth2 so that I can
securely access my account without managing passwords.

Labels: backend, security, sprint-5

Linked PRs:
- #456 [Merged] KH-123: Add auth endpoint
```

**View with full history:**
```
/kh:ticket KH-123 --verbose
```

Output:
```
KH-123: Implement user authentication
======================================
Status:     [In Progress]
Assignee:   @alice.smith
Priority:   High
Created:    2024-01-15
Updated:    2024-01-25

Description:
As a user, I want to authenticate using OAuth2 so that I can
securely access my account without managing passwords.

Acceptance criteria:
- Support Google OAuth2
- Support GitHub OAuth2
- Session management with secure cookies
- Rate limiting on auth endpoints

Labels: backend, security, sprint-5

Linked PRs:
- #456 [Merged] KH-123: Add auth endpoint
- #789 [Open] KH-123: Fix auth token expiry

Recent Comments (3):
---------------
@bob.smith (2024-01-25):
PR approved, ready for staging deploy.

@alice.smith (2024-01-24):
Tests passing, requesting review.

@pm.person (2024-01-20):
Adding to sprint 5, priority high.

Activity:
- 2024-01-25: Last updated
- 2024-01-15: Created by @pm.person
```

**Add a comment:**
```
/kh:ticket KH-123 --comment "Deployed to staging, ready for QA"
```

Output:
```
Added comment to KH-123

Comment: Deployed to staging, ready for QA
Author: @me
Time: 2024-01-26 10:30
```

**Use shorthand transition:**
```
/kh:ticket KH-123 --move staging
```

Output:
```
Transitioned KH-123 from 'In Progress' to 'Testing Staging'

KH-123: Implement user authentication
Status:     [Testing Staging]
...
```

**Invalid transition:**
```
/kh:ticket KH-123 --move "Done"
```

Output:
```
[Error] Cannot move to 'Done'. Available: Testing Staging, Blocked
```

## Quick Transitions

For common transitions, you can use shorthand:
- `/kh:ticket KH-123 --move progress` - "In Progress"
- `/kh:ticket KH-123 --move staging` - "Testing Staging"
- `/kh:ticket KH-123 --move prod` - "Testing Prod"
- `/kh:ticket KH-123 --move done` - "Done"
- `/kh:ticket KH-123 --move blocked` - "Blocked"

## Error States

**Ticket not found:**
```
/kh:ticket INVALID-999
```
Output:
```
[Error] Ticket INVALID-999 not found. Check the ticket ID and project key.
```

**Jira not configured:**
```
/kh:ticket KH-123
```
Output:
```
[Unavailable] Jira not configured. Ensure Atlassian MCP is set up in your Claude Code settings.
```

**Permission denied:**
```
/kh:ticket KH-123 --move "Done"
```
Output:
```
[Error] You don't have permission to transition this ticket.
```

**Invalid transition:**
```
/kh:ticket KH-123 --move "Done"
```
Output:
```
[Error] Cannot move to 'Done'. Available: Testing Staging, Blocked
```

**Invalid ticket format:**
```
/kh:ticket invalid
```
Output:
```
[Error] Invalid ticket format. Use PROJECT-NUMBER (e.g., KH-123)
```

## Aliases

| Alias | Command |
|-------|---------|
| /kh:t | /kh:ticket |
| /kh:issue | /kh:ticket |
