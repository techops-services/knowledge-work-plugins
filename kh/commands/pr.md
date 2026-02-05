---
description: Create a GitHub PR from the current branch with intelligent defaults
arguments:
  --draft: Create PR as draft (default: ready for review)
  --title: Override auto-generated title
  --body: Override auto-generated body
  --base: Target branch (default: staging)
  --no-link: Skip Jira ticket linking
  --no-transition: Skip Jira ticket status transition (default: transitions to Review)
---

# PR Command

Create a GitHub Pull Request from the current branch with intelligent defaults for title, body, and Jira ticket linking.

## Usage

```
/kh:pr [flags]
```

## Workflow

### Step 1: Verify Git State

Ensure we're in a valid git repository with a proper branch for PR creation.

**Check git repository:**
```python
def verify_git_state():
    """Verify git repository state before PR creation."""
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

    # Check if on base branch (staging or main)
    base_branches = ['staging', 'main', 'master']
    if branch_name in base_branches:
        return None, f"Cannot create PR from {branch_name}. Checkout a feature branch first."

    return branch_name, None
```

**Check for uncommitted changes:**
```python
def check_uncommitted_changes():
    """Warn if there are uncommitted changes."""
    result = subprocess.run(
        ['git', 'status', '--porcelain'],
        capture_output=True,
        text=True
    )
    if result.stdout.strip():
        print("[Warning] You have uncommitted changes. They will not be included in this PR.")
        return True
    return False
```

**Check remote configuration:**
```python
def check_remote():
    """Verify remote is configured."""
    result = subprocess.run(
        ['git', 'remote', 'get-url', 'origin'],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        return None, "No git remote configured. Run `git remote add origin <url>` first."
    return result.stdout.strip(), None
```

### Step 2: Detect Jira Ticket from Branch Name

Parse the branch name to find a Jira ticket reference.

**Extract ticket ID:**
```python
import re

def detect_jira_ticket(branch_name):
    """Extract Jira ticket ID from branch name.

    Patterns matched:
    - KH-123-add-feature (ticket at start)
    - feature/KH-123-add-login (ticket after slash)
    - fix/KH-456 (ticket at end after slash)
    - PROJ-999-description (any project prefix)
    """
    # Pattern: uppercase letters followed by hyphen and digits
    # Matches at start of branch or after a slash
    pattern = r'(?:^|/)([A-Z]+-\d+)'
    match = re.search(pattern, branch_name)

    if match:
        return match.group(1)
    return None
```

**Get Jira base URL:**
```python
def get_jira_base_url():
    """Get Jira base URL from config or default."""
    # Check kh config for jira_url
    config = load_kh_config()
    if config and config.get('jira_url'):
        return config['jira_url']

    # Default to common pattern
    return "https://yourorg.atlassian.net/browse"
```

### Step 3: Auto-generate PR Title

Generate a title from the branch name or commit history.

**Generate title from branch:**
```python
def generate_title_from_branch(branch_name, ticket_id):
    """Generate PR title from branch name.

    Examples:
    - KH-123-add-user-auth → "KH-123: Add user auth"
    - feature/add-login → "feat: add login"
    - fix/memory-leak → "fix: memory leak"
    """
    if ticket_id:
        # Remove ticket from branch name and convert rest to title
        # KH-123-add-user-auth → add-user-auth
        rest = branch_name.replace(ticket_id + '-', '').replace(ticket_id, '')
        rest = rest.lstrip('/-')

        # Convert hyphens to spaces, capitalize first word
        title_words = rest.replace('-', ' ').replace('_', ' ')
        if title_words:
            title_words = title_words[0].upper() + title_words[1:]

        return f"{ticket_id}: {title_words}" if title_words else ticket_id

    # No ticket - try to infer type prefix from branch
    prefix_map = {
        'feature/': 'feat: ',
        'feat/': 'feat: ',
        'fix/': 'fix: ',
        'bugfix/': 'fix: ',
        'hotfix/': 'fix: ',
        'chore/': 'chore: ',
        'docs/': 'docs: ',
        'refactor/': 'refactor: ',
    }

    for prefix, replacement in prefix_map.items():
        if branch_name.startswith(prefix):
            rest = branch_name[len(prefix):]
            title_words = rest.replace('-', ' ').replace('_', ' ')
            return replacement + title_words

    # Fallback: use branch name directly
    return branch_name.replace('-', ' ').replace('_', ' ')
```

**Generate title from first commit:**
```python
def generate_title_from_commits(base_branch):
    """Fallback: use first commit subject since divergence."""
    result = subprocess.run(
        ['git', 'log', f'{base_branch}..HEAD', '--format=%s', '--reverse'],
        capture_output=True,
        text=True
    )
    if result.returncode == 0 and result.stdout.strip():
        # Return first commit subject
        return result.stdout.strip().split('\n')[0]
    return None
```

### Step 4: Auto-generate PR Body

Build a PR body with optional Jira link and commit history.

**Build PR body:**
```python
def generate_pr_body(ticket_id, base_branch, skip_link=False):
    """Generate PR body with Jira link and commit messages.

    Format:
    ## Jira
    [KH-123](https://yourorg.atlassian.net/browse/KH-123)

    ## Changes
    - Commit message 1
    - Commit message 2
    """
    sections = []

    # Jira section (if ticket detected and not skipped)
    if ticket_id and not skip_link:
        jira_url = get_jira_base_url()
        sections.append(f"## Jira\n[{ticket_id}]({jira_url}/{ticket_id})")

    # Changes section from commit messages
    result = subprocess.run(
        ['git', 'log', f'{base_branch}..HEAD', '--format=%s'],
        capture_output=True,
        text=True
    )
    if result.returncode == 0 and result.stdout.strip():
        commits = result.stdout.strip().split('\n')
        changes = '\n'.join(f"- {commit}" for commit in commits)
        sections.append(f"## Changes\n{changes}")
    else:
        sections.append("## Changes\n- Initial implementation")

    return '\n\n'.join(sections)
```

### Step 5: Create PR Using gh CLI

Execute the PR creation command.

**Check for existing PR:**
```python
def check_existing_pr(branch_name):
    """Check if PR already exists for this branch."""
    result = subprocess.run(
        ['gh', 'pr', 'view', branch_name, '--json', 'url,number'],
        capture_output=True,
        text=True
    )
    if result.returncode == 0:
        import json
        pr_data = json.loads(result.stdout)
        return pr_data.get('url'), pr_data.get('number')
    return None, None
```

**Create the PR:**
```python
def create_pr(title, body, base_branch, draft=False):
    """Create PR using gh CLI."""
    cmd = [
        'gh', 'pr', 'create',
        '--title', title,
        '--body', body,
        '--base', base_branch
    ]

    if draft:
        cmd.append('--draft')

    result = subprocess.run(cmd, capture_output=True, text=True)

    if result.returncode != 0:
        error = result.stderr.strip()
        # Handle specific errors
        if 'not authenticated' in error.lower() or 'gh auth login' in error.lower():
            return None, "GitHub not authenticated. Run `gh auth login`"
        elif 'already exists' in error.lower():
            return None, "PR already exists for this branch"
        elif 'no commits' in error.lower():
            return None, "No commits to create PR from. Commit your changes first."
        else:
            return None, f"Failed to create PR: {error}"

    # Parse output for PR URL
    pr_url = result.stdout.strip()
    return pr_url, None
```

**Parse PR number from URL:**
```python
def parse_pr_number(pr_url):
    """Extract PR number from URL."""
    # URL format: https://github.com/org/repo/pull/123
    if '/pull/' in pr_url:
        return pr_url.split('/pull/')[-1]
    return None
```

### Step 6: Return PR URL and Number

Display the result to the user.

**Success output:**
```python
def display_success(pr_url, pr_number, title, draft=False):
    """Display success message."""
    pr_type = "draft PR" if draft else "PR"
    print(f"Created {pr_type} #{pr_number}: {title}")
    print(pr_url)
```

### Step 7: Transition Jira Ticket to Review

After successful PR creation, transition the linked Jira ticket to "Review" status.

**Transition function:**
```python
def transition_ticket_to_review(ticket_id):
    """Transition Jira ticket to Review status after PR creation."""
    try:
        # Use Atlassian MCP to transition the ticket
        atlassian_transition_issue(issue_key=ticket_id, transition="Review")
        print(f"Transitioned {ticket_id} to Review")
        return True
    except Exception as e:
        # Non-blocking - PR was created successfully, just log the transition failure
        print(f"[Warning] Could not transition {ticket_id} to Review: {str(e)}")
        return False
```

## Complete Workflow

```python
def run_pr_command(args):
    """Main PR command entry point."""
    # Step 1: Verify git state
    branch_name, error = verify_git_state()
    if error:
        print(f"[Error] {error}")
        return

    check_uncommitted_changes()

    remote_url, error = check_remote()
    if error:
        print(f"[Error] {error}")
        return

    # Step 2: Detect Jira ticket
    ticket_id = detect_jira_ticket(branch_name) if not args.get('no_link') else None

    # Get base branch
    base_branch = args.get('base', 'staging')

    # Step 3: Generate title
    if args.get('title'):
        title = args['title']
    else:
        title = generate_title_from_branch(branch_name, ticket_id)
        if not title:
            title = generate_title_from_commits(base_branch) or branch_name

    # Step 4: Generate body
    if args.get('body'):
        body = args['body']
    else:
        body = generate_pr_body(ticket_id, base_branch, skip_link=args.get('no_link'))

    # Progress output
    print(f"Creating PR from branch {branch_name}...")
    if ticket_id and not args.get('no_link'):
        print(f"Detected Jira ticket: {ticket_id}")
    print(f"Generated title: {title}")

    # Check for existing PR
    existing_url, existing_number = check_existing_pr(branch_name)
    if existing_url:
        print(f"\n[Note] PR already exists for this branch: {existing_url}")
        return

    # Step 5: Create PR
    draft = args.get('draft', False)
    pr_url, error = create_pr(title, body, base_branch, draft)

    if error:
        print(f"[Error] {error}")
        return

    # Step 6: Display result
    pr_number = parse_pr_number(pr_url)
    print()
    display_success(pr_url, pr_number, title, draft)

    # Step 7: Transition Jira ticket if detected and not skipped
    if ticket_id and not args.get('no_link') and not args.get('no_transition'):
        transition_ticket_to_review(ticket_id)
```

## Error Handling

**Not in git repository:**
```
/kh:pr
```
Output:
```
[Error] Not in a git repository. Navigate to a repo directory.
```

**On base branch:**
```
/kh:pr
```
Output:
```
[Error] Cannot create PR from staging. Checkout a feature branch first.
```

**No remote configured:**
```
/kh:pr
```
Output:
```
[Error] No git remote configured. Run `git remote add origin <url>` first.
```

**GitHub not authenticated:**
```
/kh:pr
```
Output:
```
[Error] GitHub not authenticated. Run `gh auth login`
```

**PR already exists:**
```
/kh:pr
```
Output:
```
Creating PR from branch KH-123-add-auth...
Detected Jira ticket: KH-123
Generated title: KH-123: Add auth

[Note] PR already exists for this branch: https://github.com/org/repo/pull/123
```

**No commits to create PR from:**
```
/kh:pr
```
Output:
```
[Error] No commits to create PR from. Commit your changes first.
```

**Uncommitted changes warning:**
```
/kh:pr
```
Output:
```
[Warning] You have uncommitted changes. They will not be included in this PR.
Creating PR from branch KH-123-add-auth...
...
```

## Examples

**Basic PR creation (auto-detects everything):**
```
/kh:pr
```

Output (branch: KH-123-add-auth):
```
Creating PR from branch KH-123-add-auth...
Detected Jira ticket: KH-123
Generated title: KH-123: Add auth

Created PR #456: KH-123: Add auth
https://github.com/org/repo/pull/456
Transitioned KH-123 to Review
```

**Create as draft:**
```
/kh:pr --draft
```

Output (branch: feature/login):
```
Creating PR from branch feature/login...
Generated title: feat: login

Created draft PR #457: feat: login
https://github.com/org/repo/pull/457
```

**Custom title:**
```
/kh:pr --title "Fix critical security bug"
```

Output:
```
Creating PR from branch fix/security-issue...
Generated title: Fix critical security bug

Created PR #458: Fix critical security bug
https://github.com/org/repo/pull/458
```

**Target different base branch:**
```
/kh:pr --base main
```

Output:
```
Creating PR from branch hotfix/urgent-fix...
Generated title: fix: urgent fix

Created PR #459: fix: urgent fix
https://github.com/org/repo/pull/459
```

**Skip Jira linking:**
```
/kh:pr --no-link
```

Output (branch: KH-123-add-auth):
```
Creating PR from branch KH-123-add-auth...
Generated title: KH-123: Add auth

Created PR #460: KH-123: Add auth
https://github.com/org/repo/pull/460
```

Note: PR body will not include the Jira link section when --no-link is used.

**Create PR without Jira transition:**
```
/kh:pr --no-transition
```

Output (branch: KH-123-add-auth):
```
Creating PR from branch KH-123-add-auth...
Detected Jira ticket: KH-123
Generated title: KH-123: Add auth

Created PR #461: KH-123: Add auth
https://github.com/org/repo/pull/461
```

Note: Jira ticket status remains unchanged when --no-transition is used.

**Custom body:**
```
/kh:pr --body "## Summary\nRefactored auth module for better security."
```

Output:
```
Creating PR from branch KH-789-refactor-auth...
Detected Jira ticket: KH-789
Generated title: KH-789: Refactor auth

Created PR #461: KH-789: Refactor auth
https://github.com/org/repo/pull/461
```

**Draft with custom title and different base:**
```
/kh:pr --draft --title "WIP: New feature" --base develop
```

Output:
```
Creating PR from branch feature/experimental...
Generated title: WIP: New feature

Created draft PR #462: WIP: New feature
https://github.com/org/repo/pull/462
```

## Error States

**On base branch:**
```
/kh:pr
```
Output:
```
[Error] Cannot create PR from staging. Checkout a feature branch first.
```

**PR already exists:**
```
/kh:pr
```
Output:
```
Creating PR from branch KH-123-add-auth...
Detected Jira ticket: KH-123
Generated title: KH-123: Add auth

[Note] PR already exists for this branch: https://github.com/org/repo/pull/123
```

**No remote configured:**
```
/kh:pr
```
Output:
```
[Error] No git remote configured. Run `git remote add origin <url>` first.
```

**GitHub not authenticated:**
```
/kh:pr
```
Output:
```
[Error] GitHub not authenticated. Run `gh auth login`
```

**Detached HEAD state:**
```
/kh:pr
```
Output:
```
[Error] Cannot determine current branch. Are you in detached HEAD state?
```

## Aliases

| Alias | Command |
|-------|---------|
| /kh:create-pr | /kh:pr |
