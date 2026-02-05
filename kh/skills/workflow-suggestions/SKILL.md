---
name: workflow-suggestions
description: Intelligent Jira transition suggestions based on Git workflow activity
---

# Workflow Suggestions

This skill enables intelligent suggestions for Jira ticket transitions based on Git activity. All suggestions require user confirmation before execution.

## Philosophy

- **Suggest, don't automate**: Users stay in control
- **Context-aware**: Suggestions based on actual workflow state
- **Non-blocking**: Suggestions appear after action completes, not before
- **Explicit confirmation**: Always require clear yes/no response

## Suggestion Triggers

### 1. Branch Creation with Ticket ID

**Trigger:** User creates or checks out a branch containing a Jira ticket ID.

**Detection:**
```bash
# Get current branch
git branch --show-current
# Parse for ticket pattern
branch_name | grep -oE '[A-Z]+-[0-9]+'
```

**Condition:** Ticket is in "To Do" status.

**Suggestion:**
```
Detected ticket KH-123 from branch name.
Suggest: Move KH-123 to "In Progress"? [y/n]
```

**On confirmation:** Execute `/kh:ticket KH-123 --move "In Progress"`

### 2. PR Merged to Staging

**Trigger:** User merges a PR to the staging branch (or PR with staging base is merged).

**Detection:**
```bash
# Check if PR was just merged
gh pr view {number} --json state,mergedAt,baseRefName
# baseRefName == "staging" and state == "MERGED"
```

**Condition:** Linked ticket is in "In Progress" status.

**Suggestion:**
```
PR #456 merged to staging.
Suggest: Move KH-123 to "Testing Staging"? [y/n]
```

**On confirmation:** Execute `/kh:ticket KH-123 --move "Testing Staging"`

### 3. Staging Merged to Production

**Trigger:** PR from staging to prod/main is merged.

**Detection:**
```bash
# Check PR base and head
gh pr view {number} --json state,headRefName,baseRefName
# headRefName == "staging" and baseRefName in ["prod", "main", "master"]
```

**Condition:** Linked ticket is in "Testing Staging" status.

**Suggestion:**
```
Staging promoted to production.
Suggest: Move KH-123 to "Testing Prod"? [y/n]
```

**On confirmation:** Execute `/kh:ticket KH-123 --move "Testing Prod"`

### 4. Production Verification Complete

**Trigger:** User explicitly indicates verification is complete (no automatic detection).

**Detection:** User runs `/kh:ticket KH-123 --verify` or similar explicit action.

**Condition:** Ticket is in "Testing Prod" status.

**Suggestion:**
```
Production verification complete.
Suggest: Move KH-123 to "Done"? [y/n]
```

**On confirmation:** Execute `/kh:ticket KH-123 --move "Done"`

## Suggestion Display Format

All suggestions follow this format:

```
{Context line explaining the trigger}
Suggest: {Action description}? [y/n]
```

**Response handling:**
- `y`, `yes`, `Y`, `Yes` -> Execute the transition
- `n`, `no`, `N`, `No` -> Skip, no action taken
- Any other input -> Repeat the prompt

## Integration Points

### With /kh:pr Command

After PR creation:
1. Parse branch name for ticket ID
2. If ticket found and in "To Do": suggest "In Progress"
3. Display suggestion after PR URL

```
Created PR #456: KH-123: Add auth endpoint
https://github.com/org/repo/pull/456

Suggest: Move KH-123 to "In Progress"? [y/n]
```

### With /kh:prs Command

After showing merged PRs (if --show-merged flag used):
1. For each recently merged PR
2. Check if linked ticket needs transition
3. Batch suggestions or show most relevant one

## Edge Cases

### No Ticket Detected

If no ticket ID found in branch name or PR:
- Do not suggest any transition
- User can manually transition with `/kh:ticket`

### Ticket Already in Expected Status

If ticket is already in the suggested status:
- Do not show suggestion
- Silently skip

### Multiple Tickets in Branch

If multiple ticket IDs detected (e.g., `KH-123-KH-456-feature`):
- Use the first ticket ID
- Mention: "Multiple tickets detected, suggesting for KH-123"

### Invalid Transition

If suggested transition is not available:
- Do not show suggestion
- Log internally for debugging

## Configuration

Suggestions are enabled by default. Users can disable:

```bash
# In ~/.claude/kh-config.json
{
  "workflow_suggestions": false
}
```

Or per-command with `--no-suggest` flag.

## Testing Suggestions

To test suggestion logic without executing:

```bash
# Dry run - shows what would be suggested
/kh:pr --dry-run
```
