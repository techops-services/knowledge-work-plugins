---
phase: 03-pr-jira-workflow
plan: 10
subsystem: workflow
tags: [gh-cli, atlassian-mcp, jira, github, pr-merge]

# Dependency graph
requires:
  - phase: 03-pr-jira-workflow
    provides: PR creation, ticket transition commands
provides:
  - Unified PR merge and Jira ticket transition command
  - Squash merge workflow with branch cleanup
  - Automatic ticket status updates after merge
affects: [phase-04-deployment-automation]

# Tech tracking
tech-stack:
  added: []
  patterns: [Gap closure pattern - combining manual steps into single command]

key-files:
  created: [kh/commands/merge.md]
  modified: [kh/commands/help.md]

key-decisions:
  - "Squash merge as default (preserves clean history)"
  - "Auto-transitions to Testing Staging (matches team workflow)"
  - "Branch deletion default (prevents stale branches)"
  - "Non-blocking Jira transition (PR merge succeeds even if ticket transition fails)"

patterns-established:
  - "Command composition: combine gh CLI + MCP in single workflow"
  - "Graceful degradation: core action (merge) succeeds even if secondary action (transition) fails"
  - "Ticket detection from branch or title for flexible workflows"

# Metrics
duration: 2min
completed: 2026-02-05
---

# Phase 3 Plan 10: Merge PR and Transition Ticket Summary

**Single command merges PR with squash, deletes branch, and transitions linked Jira ticket to Testing Staging**

## Performance

- **Duration:** 2 min
- **Started:** 2026-02-05T03:17:16Z
- **Completed:** 2026-02-05T03:19:27Z
- **Tasks:** 1
- **Files modified:** 2

## Accomplishments
- Created comprehensive merge command documentation (625 lines)
- Merges PR using `gh pr merge --squash` with configurable merge strategy
- Auto-detects Jira ticket from branch name or PR title
- Transitions ticket to "Testing Staging" via Atlassian MCP after successful merge
- Updated help.md with merge command in Quick Reference and Common Workflows
- Handles errors gracefully with user-friendly guidance

## Task Commits

Each task was committed atomically:

1. **Task 1: Create merge command with PR merge and Jira transition** - `19a6452` (feat)

## Files Created/Modified
- `kh/commands/merge.md` - PR merge command with automatic Jira transition
- `kh/commands/help.md` - Updated Quick Reference, PR Workflow section, and Common Workflows

## Decisions Made
- **Squash merge default:** Matches team workflow for clean commit history
- **Branch deletion default:** Prevents accumulation of stale feature branches
- **Non-blocking transition:** PR merge succeeds even if Jira transition fails (user gets warning with manual command)
- **Auto-detection from branch or title:** Flexible ticket linking works with different branching conventions
- **Warnings not errors:** CI/review status shown as warnings to inform user, but doesn't block merge (GitHub enforces actual requirements)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## Next Phase Readiness

Phase 3 is now complete with full PR and Jira workflow integration:
- PR creation with auto-linking
- PR listing with status
- Ticket viewing and transitions
- Ticket listing with filtering
- PR merge with automatic ticket transition

Ready for Phase 4 (deployment automation).

---
*Phase: 03-pr-jira-workflow*
*Completed: 2026-02-05*
