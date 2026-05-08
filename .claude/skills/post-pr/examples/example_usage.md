# Post-PR Skill Usage Examples

## Example 1: Basic Usage (Dry Run)

```bash
cd .claude/skills/post-pr
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 \
  123 \
  TICKET-456 \
  "Add vector search caching" \
  --dry-run
```

**Output:**
```
[post-pr] INFO: [DRY RUN] Would update task auto: {...}
[post-pr] ERROR: Failed to transition JIRA issue: JIRA token not configured

================================================================================
Post-PR Workflow: FAILED
================================================================================
PR: https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 (#123)
Ticket: TICKET-456
Timestamp: 2026-05-07T18:56:25.506274+00:00

Operations:
  ✓ task_update: Task auto updated to pr_open
  ✗ jira_transition_issue: JIRA transition failed: JIRA token not configured
================================================================================
```

**Explanation:** Workflow fails fast when JIRA token is not configured. This demonstrates the fail-fast error handling.

---

## Example 2: Skip JIRA and Slack (Dry Run)

```bash
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/124 \
  124 \
  TICKET-457 \
  "Fix timeout issue" \
  --dry-run \
  --skip=jira,slack
```

**Output:**
```
[post-pr] INFO: [DRY RUN] Would update task auto: {...}
[post-pr] INFO: [DRY RUN] Would store memory: {...}
[post-pr] INFO: [DRY RUN] Would update bot status to idle

================================================================================
Post-PR Workflow: SUCCESS
================================================================================
PR: https://github.com/RedHatInsights/hcc-ai-assistant/pull/124 (#124)
Ticket: TICKET-457
Timestamp: 2026-05-07T18:56:36.735245+00:00

Operations:
  ✓ task_update: Task auto updated to pr_open
  - jira_transition_issue: Skipped by user request
  - jira_add_comment: Skipped by user request
  - slack_notify: Skipped by user request
  ✓ memory_store: Stored implementation learnings
  ✓ bot_status_update: Bot status set to idle
================================================================================
```

**Explanation:** Skipping JIRA and Slack operations allows workflow to complete successfully even without API credentials.

---

## Example 3: JSON Output for Programmatic Consumption

```bash
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/125 \
  125 \
  TICKET-458 \
  "Update dependencies" \
  --dry-run \
  --skip=jira,slack \
  --json
```

**Output:**
```json
{
  "success": true,
  "pr_url": "https://github.com/RedHatInsights/hcc-ai-assistant/pull/125",
  "pr_number": 125,
  "ticket_id": "TICKET-458",
  "timestamp": "2026-05-07T18:56:43.687724+00:00",
  "operations": [
    {
      "operation": "task_update",
      "status": "success",
      "message": "Task auto updated to pr_open",
      "timestamp": "2026-05-07T18:56:43.687661+00:00",
      "details": {
        "status": "pr_open",
        "pr_url": "https://github.com/RedHatInsights/hcc-ai-assistant/pull/125",
        "pr_number": 125,
        "updated_at": "2026-05-07T18:56:43.687437+00:00",
        "summary": "Update dependencies",
        "ticket_id": "TICKET-458"
      }
    },
    ...
  ]
}
```

**Explanation:** JSON output is suitable for integration with other tools or CI/CD pipelines.

---

## Example 4: Custom Slack Channel

```bash
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/126 \
  126 \
  TICKET-459 \
  "Critical security fix" \
  --slack-channel=#hcc-security-alerts \
  --skip=jira \
  --dry-run
```

**Output:**
```
Operations:
  ✓ task_update: Task auto updated to pr_open
  - jira_transition_issue: Skipped by user request
  - jira_add_comment: Skipped by user request
  ✓ slack_notify: Sent notification to #hcc-security-alerts
  ✓ memory_store: Stored implementation learnings
  ✓ bot_status_update: Bot status set to idle
```

**Explanation:** You can route notifications to specific Slack channels for different types of PRs.

---

## Example 5: From Python Code

```python
from scripts.post_pr_operations import execute_post_pr_workflow

# Execute workflow
result = execute_post_pr_workflow(
    pr_url="https://github.com/RedHatInsights/hcc-ai-assistant/pull/127",
    pr_number=127,
    ticket_id="TICKET-460",
    summary="Add new MCP discovery feature",
    slack_channel="#hcc-ai-assistant",
    task_id="task-789",
    skip_operations=["jira", "slack"],
    dry_run=True,
)

# Check result
if result.success:
    print("✓ All operations completed successfully")
    for op in result.operations:
        print(f"  - {op.operation}: {op.message}")
else:
    print("✗ Workflow failed")
    failed_ops = [op for op in result.operations if op.status == "failed"]
    for op in failed_ops:
        print(f"  - {op.operation}: {op.message}")

# Convert to JSON for storage
import json
result_json = json.dumps(result.to_dict(), indent=2)
print(result_json)
```

**Output:**
```
✓ All operations completed successfully
  - task_update: Task task-789 updated to pr_open
  - jira_transition_issue: Skipped by user request
  - jira_add_comment: Skipped by user request
  - slack_notify: Skipped by user request
  - memory_store: Stored implementation learnings
  - bot_status_update: Bot status set to idle
```

---

## Example 6: Integration Test with Real Environment

```bash
# Set up environment
export POST_PR_JIRA_URL=https://issues.redhat.com
export POST_PR_JIRA_TOKEN=your-real-token
export POST_PR_SLACK_WEBHOOK=https://hooks.slack.com/services/YOUR/WEBHOOK
export POST_PR_TASK_DB=/tmp/test_tasks.db
export POST_PR_MEMORY_STORE=/tmp/test_memory.json

# Execute workflow (without --dry-run)
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/128 \
  128 \
  TICKET-461 \
  "Production-ready feature" \
  --slack-channel=#hcc-releases

# Check persisted data
cat /tmp/test_tasks.db
cat /tmp/test_memory.json
cat /tmp/bot_status.json
```

**Explanation:** With real API credentials, the workflow makes actual API calls to JIRA and Slack, and persists data to the configured storage locations.

---

## Common Scenarios

### Scenario 1: PR Creation After Bug Fix

```bash
/post-pr https://github.com/.../pull/101 101 BUG-789 "Fix race condition in MCP discovery"
```

Use case: Simple bug fix, no special routing needed.

### Scenario 2: Feature PR with Custom Notifications

```bash
/post-pr https://github.com/.../pull/102 102 FEAT-456 "New auth provider" --slack-channel=#hcc-auth
```

Use case: Feature that affects specific team, route notification to their channel.

### Scenario 3: Quick PR Without Bookkeeping

```bash
/post-pr https://github.com/.../pull/103 103 CHORE-123 "Update README" --skip=slack,memory
```

Use case: Trivial PR that doesn't need full bookkeeping.

### Scenario 4: Preview What Would Happen

```bash
/post-pr https://github.com/.../pull/104 104 TICKET-999 "Experimental feature" --dry-run
```

Use case: Test the workflow before running for real.

---

## Troubleshooting Examples

### Issue: JIRA Token Not Set

```bash
# Error:
[post-pr] ERROR: Failed to transition JIRA issue: JIRA token not configured

# Solution 1: Set environment variable
export POST_PR_JIRA_TOKEN=your-token

# Solution 2: Skip JIRA operations
--skip=jira
```

### Issue: Slack Webhook Not Set

```bash
# Error:
[post-pr] ERROR: Slack webhook not configured

# Solution:
export POST_PR_SLACK_WEBHOOK=https://hooks.slack.com/services/YOUR/WEBHOOK
# Or skip: --skip=slack
```

### Issue: Workflow Stopped Partway

```bash
# This is expected (fail-fast design)
# Check which operation failed:
Operations:
  ✓ task_update: Task auto updated to pr_open
  ✗ jira_transition_issue: JIRA API returned 401 Unauthorized

# Fix the issue and re-run
```
