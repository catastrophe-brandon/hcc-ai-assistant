---
name: post-pr
description: Consolidates post-PR-creation bookkeeping (task updates, JIRA transitions, Slack notifications, learning storage) into a single efficient operation
---

# Post-PR Skill

Handles all post-PR-creation bookkeeping in a single script to reduce tool calls per implementation cycle.

## Purpose

After creating a PR, this skill executes a predictable sequence of operations without requiring LLM reasoning:
1. Update task status (pr_open, pr_number, pr_url, metadata)
2. Transition JIRA issue to "Code Review"
3. Add comment to JIRA with PR link and summary
4. Notify team via Slack (pr_created event)
5. Store implementation learnings in memory
6. Update bot status to idle

## Usage

```bash
/post-pr <pr_url> <pr_number> <ticket_id> <summary> [options]
```

### Required Arguments

- `pr_url`: Full GitHub PR URL (e.g., https://github.com/RedHatInsights/hcc-ai-assistant/pull/123)
- `pr_number`: PR number (e.g., 123)
- `ticket_id`: JIRA ticket ID (e.g., TICKET-456)
- `summary`: Brief PR summary (quoted if contains spaces)

### Optional Arguments

- `--slack-channel`: Slack channel for notifications (default: #hcc-ai-assistant)
- `--task-id`: Task ID to update (default: auto-detect from context)
- `--skip`: Comma-separated list of operations to skip (e.g., "slack,memory")
- `--dry-run`: Show what would be done without executing

### Examples

```bash
# Basic usage
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 123 TICKET-456 "Add vector search caching"

# Custom Slack channel
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/124 124 TICKET-457 "Fix embedding timeout" --slack-channel=#hcc-alerts

# Skip Slack notification
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/125 125 TICKET-458 "Update dependencies" --skip=slack

# Dry run to preview
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/126 126 TICKET-459 "Refactor MCP discovery" --dry-run
```

## Operations

### 1. Task Update (`task_update`)
- Sets task status to `pr_open`
- Records `pr_number`, `pr_url` in metadata
- Updates timestamp

### 2. JIRA Transition (`jira_transition_issue`)
- Moves ticket to "Code Review" status
- Validates transition is allowed
- Records transition timestamp

### 3. JIRA Comment (`jira_add_comment`)
- Posts comment with PR link
- Includes PR summary
- Tags relevant team members (if configured)

### 4. Slack Notification (`slack_notify`)
- Sends `pr_created` event to configured channel
- Includes PR link, summary, author
- Thread-aware (if part of existing conversation)

### 5. Memory Storage (`memory_store`)
- Saves implementation learnings
- Records patterns, gotchas, decisions
- Available for future reference

### 6. Bot Status Update (`bot_status_update`)
- Sets bot status to `idle`
- Clears active task context
- Ready for next assignment

## Error Handling

**Fail fast approach**: If any operation fails, execution stops immediately and reports the error. This ensures consistency and prevents partial bookkeeping.

Error scenarios:
- **Task not found**: Returns clear error with available task IDs
- **JIRA API down**: Reports connectivity error, suggests retry
- **Invalid transition**: Shows allowed transitions for current ticket status
- **Slack webhook error**: Returns HTTP status and error message
- **Missing parameters**: Validation error with usage examples

## Implementation

The skill uses Python scripts in `scripts/post_pr_operations.py`:

```python
# Example usage
from scripts.post_pr_operations import execute_post_pr_workflow

result = execute_post_pr_workflow(
    pr_url="https://github.com/RedHatInsights/hcc-ai-assistant/pull/123",
    pr_number=123,
    ticket_id="TICKET-456",
    summary="Add vector search caching",
    slack_channel="#hcc-ai-assistant",
    skip_operations=[],
    dry_run=False
)
```

## Testing

Run tests with:

```bash
# Unit tests (individual operations)
cd .claude/skills/post-pr && python -m pytest tests/test_operations.py -v

# Integration tests (full workflow)
cd .claude/skills/post-pr && python -m pytest tests/test_integration.py -v

# All tests with coverage
cd .claude/skills/post-pr && python -m pytest tests/ --cov=scripts --cov-report=html -v
```

## Configuration

Optional environment variables for customization:

- `POST_PR_JIRA_URL`: JIRA instance URL (default: https://issues.redhat.com)
- `POST_PR_JIRA_TOKEN`: JIRA API token (required for real operations)
- `POST_PR_SLACK_WEBHOOK`: Slack webhook URL (required for real operations)
- `POST_PR_TASK_DB`: Task database path (default: /tmp/tasks.db)
- `POST_PR_MEMORY_STORE`: Memory storage path (default: /tmp/memory.json)

## Notes

- All inputs are known at PR creation time; no LLM reasoning needed
- Operations execute sequentially (dependencies between steps)
- Designed for speed: ~5-6 tool calls → 1 script execution
- Stub implementations provided; integrate with real APIs as needed
- Logging to stdout for visibility in Claude Code output
