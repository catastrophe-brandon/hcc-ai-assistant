---
name: post-pr
description: Consolidates post-PR-creation bookkeeping (task updates, JIRA transitions, Slack notifications, learning storage) into a single efficient operation
---

# Post-PR Skill

Handles all post-PR-creation bookkeeping in a single script to reduce tool calls per implementation cycle.

## Purpose

After creating a PR, this skill executes a predictable sequence of operations without requiring LLM reasoning:
1. Update GitHub PR (add labels, JIRA link, request reviewers)
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
- `--reviewers`: Comma-separated list of GitHub usernames to request as reviewers
- `--skip`: Comma-separated list of operations to skip (e.g., "slack,memory")
- `--dry-run`: Show what would be done without executing
- `--json`: Output result as JSON instead of human-readable format

### Examples

```bash
# Basic usage
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 123 RHCLOUD-456 "Add vector search caching"

# With reviewers
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/124 124 RHCLOUD-457 "Fix timeout" --reviewers=user1,user2

# Custom Slack channel
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/125 125 RHCLOUD-458 "Update deps" --slack-channel=#hcc-alerts

# Skip operations
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/126 126 RHCLOUD-459 "Refactor" --skip=slack,memory

# Dry run to preview
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/127 127 RHCLOUD-460 "Add feature" --dry-run

# JSON output
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/128 128 RHCLOUD-461 "Bug fix" --json
```

## Operations

### 1. GitHub PR Update (`task_update`)
- Adds labels: `code-review`, `awaiting-review`
- Appends JIRA ticket link to PR description
- Requests reviewers (if specified)
- Uses GitHub REST API v3

### 2. JIRA Transition (`jira_transition_issue`)
- Moves ticket to "Code Review" status
- Uses JIRA Cloud API v3 with Basic auth
- Validates transition is available
- Supports custom target status

### 3. JIRA Comment (`jira_add_comment`)
- Posts comment with PR link and summary
- Uses Atlassian Document Format (ADF)
- JIRA Cloud API v3 with Basic auth

### 4. Slack Notification (`slack_notify`)
- Sends formatted notification to webhook
- Includes PR number, link, and summary
- Customizable channel (default: #hcc-ai-assistant)
- Uses Slack attachment format

### 5. Memory Storage (`memory_store`)
- Saves implementation learnings to JSON file
- Records PR URL, ticket ID, timestamp
- Appends to existing memories (accumulative)

### 6. Bot Status Update (`bot_status_update`)
- Updates status JSON file
- Sets status to `idle` with timestamp
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
    ticket_id="RHCLOUD-456",
    summary="Add vector search caching",
    github_token=None,  # Falls back to GITHUB_TOKEN env var
    jira_token=None,    # Falls back to POST_PR_JIRA_TOKEN env var
    slack_channel="#hcc-ai-assistant",
    reviewers=["user1", "user2"],  # Optional
    skip_operations=[],  # Or ["slack", "memory"] to skip
    dry_run=False,
)

# Result contains success status and operation details
if result.success:
    print(f"✅ Workflow completed successfully")
    for op in result.operations:
        print(f"  {op.operation}: {op.message}")
else:
    print(f"❌ Workflow failed")
    failed = [op for op in result.operations if op.status.value == "failed"]
    for op in failed:
        print(f"  {op.operation}: {op.message}")
```

## Testing

Run tests with:

```bash
cd .claude/skills/post-pr

# Unit tests (21 tests - individual operations)
uv run pytest tests/test_operations.py -v

# Integration tests (12 tests - full workflow scenarios)
uv run pytest tests/test_integration.py -v

# All tests (35 total)
uv run pytest tests/ -v

# With coverage report
uv run pytest tests/ --cov=scripts --cov-report=html -v
```

## Configuration

Environment variables for API integration:

**Required:**
- `GITHUB_TOKEN` or `GH_TOKEN`: GitHub API token for PR operations
- `POST_PR_JIRA_TOKEN`: JIRA API token for issue transitions and comments
- `POST_PR_JIRA_EMAIL`: Email address for JIRA Basic authentication
- `POST_PR_SLACK_WEBHOOK`: Slack incoming webhook URL for notifications

**Optional:**
- `POST_PR_JIRA_URL`: JIRA instance URL (default: https://redhat.atlassian.net)
- `POST_PR_MEMORY_STORE`: Memory storage path (default: /tmp/memory.json)

## Notes

- All inputs are known at PR creation time; no LLM reasoning needed
- Operations execute sequentially with fail-fast error handling
- Designed for speed: ~5-6 tool calls → 1 script execution
- Full API integration: GitHub REST API, JIRA Cloud API v3, Slack webhooks
- Uses Basic authentication for JIRA Cloud (email + API token)
- Supports both dry-run mode and skip operations for flexibility
- Logging to stdout for visibility in Claude Code output
