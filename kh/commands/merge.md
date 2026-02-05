---
description: Merge PR and transition Jira ticket to Testing Staging
arguments:
  pr: PR number (optional - auto-detects from current branch if not provided)
  --no-squash: Use regular merge instead of squash (default: squash)
  --no-transition: Skip Jira ticket transition
  --delete-branch: Delete branch after merge (default: true)
---

# Merge Command

Merge a GitHub Pull Request and transition the linked Jira ticket to "Testing Staging" in a single command.

## Usage

```
/kh:merge [pr] [flags]
```

## Workflow

### Step 1: Determine PR to Merge

If PR number is provided, use it directly. Otherwise, detect from the current branch.

**Detect PR from current branch:**
```python
def detect_pr_from_branch():
    """Get PR associated with current branch."""
    try:
        # Get PR details for current branch
        result = subprocess.run(
            ['gh', 'pr', 'view', '--json', 'number,headRefName,title,state,mergeable'],
            capture_output=True,
            text=True
        )

        if result.returncode != 0:
            error = result.stderr.strip()
            if 'no pull requests' in error.lower():
                return None, "No PR found for current branch. Provide PR number: /kh:merge 123"
            elif 'auth' in error.lower() or 'login' in error.lower():
                return None, "GitHub not authenticated. Run `gh auth login`"
            return None, f"Error detecting PR: {error}"

        import json
        pr_data = json.loads(result.stdout)

        return pr_data, None

    except FileNotFoundError:
        return None, "gh CLI not installed - install from https://cli.github.com"
    except Exception as e:
        return None, f"Error detecting PR: {str(e)}"
```

**Get PR by number:**
```python
def get_pr_by_number(pr_number):
    """Get PR details by number."""
    try:
        result = subprocess.run(
            ['gh', 'pr', 'view', str(pr_number), '--json', 'number,headRefName,title,state,mergeable'],
            capture_output=True,
            text=True
        )

        if result.returncode != 0:
            error = result.stderr.strip()
            if 'not found' in error.lower():
                return None, f"PR #{pr_number} not found."
            elif 'auth' in error.lower():
                return None, "GitHub not authenticated. Run `gh auth login`"
            return None, f"Error fetching PR: {error}"

        import json
        pr_data = json.loads(result.stdout)

        return pr_data, None

    except Exception as e:
        return None, f"Error fetching PR: {str(e)}"
```

### Step 2: Extract Jira Ticket from PR

Parse the branch name and PR title for Jira ticket ID.

**Extract ticket:**
```python
import re

def extract_jira_ticket(pr_data):
    """Extract Jira ticket ID from PR branch name or title.

    Patterns matched:
    - KH-123-add-feature (ticket at start of branch)
    - feature/KH-123-add-login (ticket after slash)
    - KH-456: Add feature (ticket at start of title)
    """
    # Pattern: uppercase letters followed by hyphen and digits
    pattern = r'([A-Z]+-\d+)'

    # Try branch name first (most reliable)
    branch_name = pr_data.get('headRefName', '')
    match = re.search(pattern, branch_name)
    if match:
        return match.group(1)

    # Fallback: try PR title
    title = pr_data.get('title', '')
    match = re.search(pattern, title)
    if match:
        return match.group(1)

    return None
```

### Step 3: Verify PR is Ready to Merge

Check PR state and mergeable status before attempting merge.

**Verify PR state:**
```python
def verify_pr_ready(pr_data):
    """Verify PR is ready to merge."""
    state = pr_data.get('state', '').upper()
    mergeable = pr_data.get('mergeable', '')

    # Check if PR is open
    if state != 'OPEN':
        return False, f"PR is {state}. Only open PRs can be merged."

    # Check if PR is mergeable
    if mergeable == 'CONFLICTING':
        return False, "PR has merge conflicts. Resolve conflicts first."

    return True, None
```

**Check CI and reviews (informational):**
```python
def check_pr_status(pr_number):
    """Get detailed PR status including CI and reviews."""
    try:
        result = subprocess.run(
            ['gh', 'pr', 'view', str(pr_number), '--json', 'statusCheckRollup,reviews'],
            capture_output=True,
            text=True
        )

        if result.returncode == 0:
            import json
            status_data = json.loads(result.stdout)

            # Check CI status
            checks = status_data.get('statusCheckRollup', [])
            if checks:
                states = [check.get('state', '').upper() for check in checks]
                if any(s in ['FAILURE', 'ERROR'] for s in states):
                    print("[Warning] CI checks failing. Merge anyway?")
                elif any(s in ['PENDING', 'EXPECTED', 'QUEUED'] for s in states):
                    print("[Warning] CI checks still running.")

            # Check reviews
            reviews = status_data.get('reviews', [])
            if reviews:
                has_approval = any(r.get('state') == 'APPROVED' for r in reviews)
                has_changes = any(r.get('state') == 'CHANGES_REQUESTED' for r in reviews)

                if has_changes:
                    print("[Warning] Changes requested on this PR.")
                elif not has_approval:
                    print("[Warning] No approvals yet.")
            else:
                print("[Warning] No reviews on this PR.")

    except Exception:
        # Non-critical, continue
        pass
```

### Step 4: Merge PR Using gh CLI

Execute the merge with appropriate flags.

**Build merge command:**
```python
def build_merge_command(pr_number, use_squash=True, delete_branch=True):
    """Build gh pr merge command with appropriate flags."""
    cmd = ['gh', 'pr', 'merge', str(pr_number)]

    # Merge strategy
    if use_squash:
        cmd.append('--squash')
    else:
        cmd.append('--merge')

    # Branch deletion
    if delete_branch:
        cmd.append('--delete-branch')

    return cmd
```

**Execute merge:**
```python
def execute_merge(cmd):
    """Execute the merge command."""
    try:
        result = subprocess.run(cmd, capture_output=True, text=True)

        if result.returncode != 0:
            error = result.stderr.strip()

            # Parse specific error cases
            if 'not mergeable' in error.lower() or 'conflict' in error.lower():
                return None, "PR has merge conflicts. Resolve conflicts first."
            elif 'required status check' in error.lower():
                return None, "Required CI checks have not passed. Override with --admin flag if needed."
            elif 'required review' in error.lower():
                return None, "Required reviews not satisfied. Get approvals first."
            elif 'auth' in error.lower():
                return None, "Authentication failed. Run `gh auth login`"
            elif 'permission' in error.lower() or 'forbidden' in error.lower():
                return None, "You don't have permission to merge this PR."
            else:
                return None, f"Merge failed: {error}"

        # Merge successful
        return result.stdout.strip(), None

    except Exception as e:
        return None, f"Error executing merge: {str(e)}"
```

### Step 5: Transition Jira Ticket to Testing Staging

After successful merge, transition the ticket unless --no-transition flag is set.

**Transition ticket:**
```python
def transition_ticket_to_staging(ticket_id):
    """Transition Jira ticket to Testing Staging status."""
    try:
        # Use Atlassian MCP to transition
        atlassian_transition_issue(
            issue_key=ticket_id,
            transition="Testing Staging"
        )
        return True, None

    except Exception as e:
        error_msg = str(e).lower()

        # Parse specific error cases
        if 'not found' in error_msg or '404' in error_msg:
            return False, f"Ticket {ticket_id} not found in Jira."
        elif 'transition' in error_msg and 'invalid' in error_msg:
            return False, f"Cannot transition {ticket_id} to Testing Staging from current status."
        elif 'permission' in error_msg or 'forbidden' in error_msg:
            return False, f"You don't have permission to transition {ticket_id}."
        elif 'not configured' in error_msg or 'mcp' in error_msg:
            return False, "Jira not configured. Ensure Atlassian MCP is set up."
        else:
            return False, f"Failed to transition ticket: {str(e)}"
```

### Step 6: Display Result

Show merge result and ticket transition status.

**Format output:**
```python
def display_merge_result(pr_data, ticket_id, merge_output, transition_result, transition_error):
    """Display comprehensive merge result."""
    pr_number = pr_data.get('number')
    pr_title = pr_data.get('title', '')
    branch_name = pr_data.get('headRefName', '')

    print(f"Merged PR #{pr_number}: {pr_title}")

    # Show merge details if available
    if merge_output and 'squash' in merge_output.lower():
        print(f"Merge type: Squash")
    elif merge_output:
        print(f"Merge type: Regular")

    # Show branch deletion
    if 'deleted' in merge_output.lower():
        print(f"Branch deleted: {branch_name}")

    print()

    # Show ticket transition result
    if ticket_id:
        if transition_result:
            print(f"Transitioned {ticket_id} to Testing Staging")
        elif transition_error:
            print(f"[Warning] PR merged but Jira transition failed: {transition_error}")
            print(f"Manually run: /kh:ticket {ticket_id} --move staging")
    else:
        print("[Note] No Jira ticket detected. Skipping transition.")

    print()
```

## Complete Workflow

```python
def run_merge_command(args):
    """Main merge command entry point."""
    pr_number = args.get('pr')
    use_squash = not args.get('no_squash', False)
    delete_branch = args.get('delete_branch', True)
    skip_transition = args.get('no_transition', False)

    # Step 1: Get PR data
    if pr_number:
        pr_data, error = get_pr_by_number(pr_number)
        if error:
            print(f"[Error] {error}")
            return
    else:
        pr_data, error = detect_pr_from_branch()
        if error:
            print(f"[Error] {error}")
            return

    pr_number = pr_data.get('number')
    pr_title = pr_data.get('title', '')
    branch_name = pr_data.get('headRefName', '')

    print(f"Merging PR #{pr_number} ({branch_name})...")
    print()

    # Step 2: Extract Jira ticket
    ticket_id = extract_jira_ticket(pr_data)
    if ticket_id:
        print(f"Detected Jira ticket: {ticket_id}")

    # Step 3: Verify PR is ready
    ready, error = verify_pr_ready(pr_data)
    if not ready:
        print(f"[Error] {error}")
        return

    # Check CI and review status (warnings only)
    check_pr_status(pr_number)

    # Step 4: Merge PR
    merge_cmd = build_merge_command(pr_number, use_squash, delete_branch)
    merge_output, error = execute_merge(merge_cmd)

    if error:
        print(f"[Error] {error}")
        return

    # Step 5: Transition Jira ticket (if detected and not skipped)
    transition_result = False
    transition_error = None

    if ticket_id and not skip_transition:
        transition_result, transition_error = transition_ticket_to_staging(ticket_id)

    # Step 6: Display result
    display_merge_result(pr_data, ticket_id, merge_output, transition_result, transition_error)
```

## Error Handling

**No PR found for current branch:**
```
/kh:merge
```
Output:
```
[Error] No PR found for current branch. Provide PR number: /kh:merge 123
```

**PR not mergeable:**
```
/kh:merge 123
```
Output:
```
Merging PR #123 (KH-456-add-auth)...

[Error] PR has merge conflicts. Resolve conflicts first.
```

**PR already merged:**
```
/kh:merge 123
```
Output:
```
Merging PR #123 (KH-456-add-auth)...

[Error] PR is MERGED. Only open PRs can be merged.
```

**CI checks failing:**
```
/kh:merge
```
Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456
[Warning] CI checks failing. Merge anyway?

[Error] Required CI checks have not passed. Override with --admin flag if needed.
```

**Missing required reviews:**
```
/kh:merge
```
Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456
[Warning] No approvals yet.

[Error] Required reviews not satisfied. Get approvals first.
```

**Jira transition fails:**
```
/kh:merge
```
Output:
```
Merged PR #123: KH-456: Add user auth
Merge type: Squash
Branch deleted: KH-456-add-auth

[Warning] PR merged but Jira transition failed: Cannot transition KH-456 to Testing Staging from current status.
Manually run: /kh:ticket KH-456 --move staging
```

**No Jira ticket detected:**
```
/kh:merge
```
Output:
```
Merged PR #123: Fix typo in docs
Merge type: Squash
Branch deleted: fix/typo

[Note] No Jira ticket detected. Skipping transition.
```

**GitHub not authenticated:**
```
/kh:merge
```
Output:
```
[Error] GitHub not authenticated. Run `gh auth login`
```

**Permission denied:**
```
/kh:merge 123
```
Output:
```
Merging PR #123 (KH-456-add-auth)...

[Error] You don't have permission to merge this PR.
```

## Examples

**Merge PR from current branch:**
```
/kh:merge
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456

Merged PR #123: KH-456: Add user auth
Merge type: Squash
Branch deleted: KH-456-add-auth

Transitioned KH-456 to Testing Staging
```

**Merge specific PR:**
```
/kh:merge 123
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456

Merged PR #123: KH-456: Add user auth
Merge type: Squash
Branch deleted: KH-456-add-auth

Transitioned KH-456 to Testing Staging
```

**Merge without squash:**
```
/kh:merge --no-squash
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456

Merged PR #123: KH-456: Add user auth
Merge type: Regular
Branch deleted: KH-456-add-auth

Transitioned KH-456 to Testing Staging
```

**Merge without Jira transition:**
```
/kh:merge --no-transition
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456

Merged PR #123: KH-456: Add user auth
Merge type: Squash
Branch deleted: KH-456-add-auth
```

**Keep branch after merge:**
```
/kh:merge --delete-branch=false
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456

Merged PR #123: KH-456: Add user auth
Merge type: Squash

Transitioned KH-456 to Testing Staging
```

**Merge with CI warnings:**
```
/kh:merge
```

Output:
```
Merging PR #123 (KH-456-add-auth)...
Detected Jira ticket: KH-456
[Warning] CI checks still running.
[Warning] No approvals yet.

Merged PR #123: KH-456: Add user auth
Merge type: Squash
Branch deleted: KH-456-add-auth

Transitioned KH-456 to Testing Staging
```

## Common Use Cases

**Standard merge flow (most common):**
```
/kh:merge
```
This merges the PR for the current branch, squashes commits, deletes the branch, and transitions the linked Jira ticket to "Testing Staging".

**Merge specific PR by number:**
```
/kh:merge 456
```
Useful when you're not on the PR's branch but want to merge it.

**Preserve commit history:**
```
/kh:merge --no-squash
```
Use regular merge instead of squash merge to preserve all commit messages.

**Keep branch for reference:**
```
/kh:merge --delete-branch=false
```
Merge PR but keep the feature branch (useful for long-lived branches or reference).

**Merge without updating Jira:**
```
/kh:merge --no-transition
```
Merge PR without transitioning the Jira ticket (useful when ticket status should remain unchanged).

## Workflow Integration

The `/kh:merge` command integrates seamlessly with your existing workflow:

1. **After PR approval:** `/kh:merge` - Merge and transition ticket to Testing Staging
2. **After staging verification:** `/kh:promote` - Promote to production (Phase 4)
3. **After prod verification:** `/kh:ticket KH-123 --move prod` - Move to Testing Prod
4. **After prod complete:** `/kh:ticket KH-123 --move done` - Mark as Done

## Aliases

| Alias | Command |
|-------|---------|
| /kh:m | /kh:merge |
| /kh:merge-pr | /kh:merge |
