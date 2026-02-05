---
description: Start work on a Jira ticket - transition to In Progress and create branch
arguments:
  ticket: Ticket ID (required, e.g., KEEP-123, KH-456)
  --no-transition: Skip Jira transition (just create branch)
  --base: Base branch to create from (default: staging)
---

# Start Command

Start work on a Jira ticket by transitioning it to In Progress and creating a feature branch with a descriptive name.

## Usage

```
/kh:start <ticket> [flags]
```

## Workflow

### Step 1: Parse and Validate Ticket ID

Accept ticket ID and validate format.

**Parse ticket ID:**
```python
import re

def parse_ticket_id(ticket_id):
    """Parse and validate Jira ticket ID.

    Accepts:
    - KH-123 (with project prefix)
    - PROJ-456 (any project prefix)

    Returns: normalized ticket ID or error message
    """
    if not ticket_id:
        return None, "Ticket ID is required. Usage: /kh:start <ticket>"

    ticket_id = ticket_id.strip().upper()

    # Pattern: uppercase letters followed by hyphen and digits
    pattern = r'^[A-Z]+-\d+$'
    if re.match(pattern, ticket_id):
        return ticket_id, None

    return None, f"Invalid ticket format. Use PROJECT-NUMBER (e.g., KEEP-123)"
```

### Step 2: Fetch Ticket Details via Atlassian MCP

Query Jira for ticket information to confirm it exists and get the summary.

**MCP call:**
```
Use atlassian_get_issue with issue_key: {ticket_id}
Fields to retrieve:
  - key (ticket ID)
  - summary (ticket title for branch name)
  - status (current status name)
  - assignee (display name)
```

**Parse ticket response:**
```python
def fetch_ticket(ticket_id):
    """Fetch ticket details from Jira via Atlassian MCP."""
    try:
        response = atlassian_get_issue(issue_key=ticket_id)

        if not response:
            return None, f"Ticket {ticket_id} not found. Check the ticket ID."

        fields = response.get('fields', {})
        status = fields.get('status', {})

        return {
            'key': response.get('key'),
            'summary': fields.get('summary', 'No summary'),
            'status': status.get('name', 'Unknown'),
        }, None

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

### Step 3: Check Current Status

Warn user if ticket is in a problematic state.

**Status check:**
```python
def check_status(current_status):
    """Check if current status requires user confirmation."""
    # These statuses might indicate work shouldn't be started
    problematic_statuses = ['done', 'closed', 'won\'t do', 'cancelled']

    if current_status.lower() in problematic_statuses:
        return f"[Warning] Ticket is currently '{current_status}'. Are you sure you want to start work on it?"

    if current_status.lower() == 'in progress':
        return f"[Note] Ticket is already 'In Progress'"

    return None
```

### Step 4: Verify Git State

Ensure we're in a valid repository and on an appropriate branch.

**Verify git repository:**
```python
def verify_git_state():
    """Verify git repository state before branch creation."""
    # Check if in git repo
    result = subprocess.run(
        ['git', 'rev-parse', '--git-dir'],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        return None, "Not in a git repository. Navigate to a repo directory."

    # Get current branch name
    result = subprocess.run(
        ['git', 'branch', '--show-current'],
        capture_output=True,
        text=True
    )
    if result.returncode != 0 or not result.stdout.strip():
        return None, "Cannot determine current branch. Are you in detached HEAD state?"

    branch_name = result.stdout.strip()

    return branch_name, None
```

**Check for uncommitted changes:**
```python
def check_uncommitted_changes():
    """Check if there are uncommitted changes."""
    result = subprocess.run(
        ['git', 'status', '--porcelain'],
        capture_output=True,
        text=True
    )
    if result.stdout.strip():
        return "You have uncommitted changes. Commit or stash them before creating a new branch."
    return None
```

**Check if already on a feature branch:**
```python
def check_current_branch(current_branch, base_branch):
    """Warn if already on a feature branch."""
    # Base branches are OK
    base_branches = ['staging', 'main', 'master', 'develop']

    if current_branch not in base_branches and current_branch != base_branch:
        return f"Already on branch '{current_branch}'. Checkout {base_branch} first or use this branch."

    return None
```

### Step 5: Generate Branch Name

Create a descriptive branch name from ticket ID and summary.

**Generate branch name:**
```python
import re

def generate_branch_name(ticket_id, summary):
    """Generate branch name from ticket ID and summary.

    Format: TICKET-123-slugified-summary

    Examples:
    - KEEP-123: Implement user authentication → KEEP-123-implement-user-authentication
    - KH-456: Fix login timeout issue → KH-456-fix-login-timeout-issue
    """
    # Slugify the summary
    slug = summary.lower()

    # Replace non-alphanumeric characters with dashes
    slug = re.sub(r'[^a-z0-9]+', '-', slug)

    # Remove leading/trailing dashes
    slug = slug.strip('-')

    # Limit length to 40 characters for the slug part
    slug = slug[:40].rstrip('-')

    # Combine ticket ID and slug
    return f"{ticket_id}-{slug}"
```

**Output example:**
```
KEEP-123: Implement user authentication
→ KEEP-123-implement-user-authentication

KH-456: Fix login timeout issue on mobile devices
→ KH-456-fix-login-timeout-issue-on-mobile-dev
```

### Step 6: Transition Ticket to In Progress

Update the Jira ticket status unless --no-transition flag is set.

**Fetch available transitions:**
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

**Execute transition:**
```python
def transition_to_in_progress(ticket_id, available_transitions):
    """Transition ticket to In Progress."""
    # Find the "In Progress" transition
    target_normalized = 'in progress'
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
        return None, f"Cannot move to 'In Progress'. Available: {', '.join(available_names)}"

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

### Step 7: Create and Checkout Branch

Create the new branch from the base branch.

**Create branch:**
```python
def create_branch(branch_name, base_branch):
    """Create and checkout a new branch from base."""
    try:
        # Fetch latest from remote
        result = subprocess.run(
            ['git', 'fetch', 'origin', base_branch],
            capture_output=True,
            text=True,
            timeout=30
        )

        if result.returncode != 0:
            return None, f"Failed to fetch {base_branch}: {result.stderr.strip()}"

        # Create and checkout new branch
        result = subprocess.run(
            ['git', 'checkout', '-b', branch_name, f'origin/{base_branch}'],
            capture_output=True,
            text=True
        )

        if result.returncode != 0:
            error = result.stderr.strip()
            if 'already exists' in error.lower():
                return None, f"Branch '{branch_name}' already exists. Use a different name or checkout the existing branch."
            return None, f"Failed to create branch: {error}"

        return branch_name, None

    except subprocess.TimeoutExpired:
        return None, f"Timeout fetching {base_branch} from remote"
    except Exception as e:
        return None, f"Error creating branch: {str(e)}"
```

### Step 8: Display Success

Show the user what was accomplished.

**Success output:**
```python
def display_success(ticket_id, summary, branch_name, base_branch, transitioned=True):
    """Display success message."""
    print(f"Starting work on {ticket_id}: {summary}")
    print()

    if transitioned:
        print(f"✓ Transitioned {ticket_id} to 'In Progress'")

    print(f"✓ Created branch: {branch_name}")
    print(f"✓ Checked out from: {base_branch}")
    print()
    print("Ready to code!")
```

**Output example:**
```
Starting work on KEEP-123: Implement user authentication

✓ Transitioned KEEP-123 to 'In Progress'
✓ Created branch: KEEP-123-implement-user-authentication
✓ Checked out from: staging

Ready to code!
```

## Complete Workflow

```python
def run_start_command(args):
    """Main start command entry point."""
    # Step 1: Parse ticket ID
    ticket_id = args.get('ticket')
    ticket_id, error = parse_ticket_id(ticket_id)
    if error:
        print(f"[Error] {error}")
        return

    # Step 2: Fetch ticket details
    ticket, error = fetch_ticket(ticket_id)
    if error:
        print(f"[Error] {error}")
        return

    # Step 3: Check current status
    status_warning = check_status(ticket['status'])
    if status_warning:
        print(status_warning)
        # For "Done" or similar, could prompt for confirmation
        # For now, continue with a note

    # Step 4: Verify git state
    current_branch, error = verify_git_state()
    if error:
        print(f"[Error] {error}")
        return

    # Check for uncommitted changes
    error = check_uncommitted_changes()
    if error:
        print(f"[Error] {error}")
        return

    # Get base branch
    base_branch = args.get('base', 'staging')

    # Check if already on a feature branch
    warning = check_current_branch(current_branch, base_branch)
    if warning:
        print(f"[Warning] {warning}")
        return

    # Step 5: Generate branch name
    branch_name = generate_branch_name(ticket_id, ticket['summary'])

    # Step 6: Transition ticket (unless --no-transition)
    transitioned = False
    if not args.get('no_transition'):
        # Only transition if not already in progress
        if ticket['status'].lower() != 'in progress':
            transitions, error = fetch_transitions(ticket_id)
            if error:
                print(f"[Warning] {error}")
                print("Continuing with branch creation...")
            else:
                new_status, error = transition_to_in_progress(ticket_id, transitions)
                if error:
                    print(f"[Warning] {error}")
                    print("Continuing with branch creation...")
                else:
                    transitioned = True

    # Step 7: Create and checkout branch
    created_branch, error = create_branch(branch_name, base_branch)
    if error:
        print(f"[Error] {error}")
        return

    # Step 8: Display success
    display_success(ticket_id, ticket['summary'], branch_name, base_branch, transitioned)
```

## Examples

**Start work on a ticket:**
```
/kh:start KEEP-123
```

Output:
```
Starting work on KEEP-123: Implement user authentication

✓ Transitioned KEEP-123 to 'In Progress'
✓ Created branch: KEEP-123-implement-user-authentication
✓ Checked out from: staging

Ready to code!
```

**Start from different base branch:**
```
/kh:start KEEP-123 --base main
```

Output:
```
Starting work on KEEP-123: Implement user authentication

✓ Transitioned KEEP-123 to 'In Progress'
✓ Created branch: KEEP-123-implement-user-authentication
✓ Checked out from: main

Ready to code!
```

**Just create branch (skip Jira transition):**
```
/kh:start KEEP-123 --no-transition
```

Output:
```
Starting work on KEEP-123: Implement user authentication

✓ Created branch: KEEP-123-implement-user-authentication
✓ Checked out from: staging

Ready to code!
```

**Ticket already in progress:**
```
/kh:start KEEP-456
```

Output:
```
[Note] Ticket is already 'In Progress'

Starting work on KEEP-456: Fix login timeout

✓ Created branch: KEEP-456-fix-login-timeout
✓ Checked out from: staging

Ready to code!
```

**Long ticket summary gets truncated:**
```
/kh:start KH-789
```

Output (ticket summary: "Implement comprehensive user authentication system with OAuth2 support for Google and GitHub providers"):
```
Starting work on KH-789: Implement comprehensive user authentication system with OAuth2 support for Google and GitHub providers

✓ Transitioned KH-789 to 'In Progress'
✓ Created branch: KH-789-implement-comprehensive-user-authent
✓ Checked out from: staging

Ready to code!
```

## Error States

**Invalid ticket format:**
```
/kh:start invalid
```
Output:
```
[Error] Invalid ticket format. Use PROJECT-NUMBER (e.g., KEEP-123)
```

**Ticket not found:**
```
/kh:start INVALID-999
```
Output:
```
[Error] Ticket INVALID-999 not found. Check the ticket ID and project key.
```

**Not in git repository:**
```
/kh:start KEEP-123
```
Output:
```
[Error] Not in a git repository. Navigate to a repo directory.
```

**Uncommitted changes:**
```
/kh:start KEEP-123
```
Output:
```
[Error] You have uncommitted changes. Commit or stash them before creating a new branch.
```

**Already on feature branch:**
```
/kh:start KEEP-123
```
Output (on branch: feature/old-work):
```
[Warning] Already on branch 'feature/old-work'. Checkout staging first or use this branch.
```

**Branch already exists:**
```
/kh:start KEEP-123
```
Output:
```
[Error] Branch 'KEEP-123-implement-user-authentication' already exists. Use a different name or checkout the existing branch.
```

**Jira transition fails but branch succeeds:**
```
/kh:start KEEP-123
```
Output:
```
[Warning] Cannot move to 'In Progress'. Available: Testing Staging, Blocked
Continuing with branch creation...

Starting work on KEEP-123: Implement user authentication

✓ Created branch: KEEP-123-implement-user-authentication
✓ Checked out from: staging

Ready to code!

[Note] Branch created but Jira transition failed. Manually run: /kh:ticket KEEP-123 --move progress
```

**Ticket is Done:**
```
/kh:start KEEP-999
```
Output:
```
[Warning] Ticket is currently 'Done'. Are you sure you want to start work on it?

Starting work on KEEP-999: Previously completed task

✓ Transitioned KEEP-999 to 'In Progress'
✓ Created branch: KEEP-999-previously-completed-task
✓ Checked out from: staging

Ready to code!
```

**Jira not configured:**
```
/kh:start KEEP-123
```
Output:
```
[Error] Jira not configured. Ensure Atlassian MCP is set up in your Claude Code settings.
```

**Permission denied:**
```
/kh:start KEEP-123
```
Output:
```
[Error] You don't have permission to view ticket KEEP-123.
```

## Aliases

| Alias | Command |
|-------|---------|
| /kh:begin | /kh:start |
| /kh:work | /kh:start |
