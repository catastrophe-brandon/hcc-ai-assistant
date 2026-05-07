# Post-PR Skill

Consolidates post-PR-creation bookkeeping into a single efficient operation, reducing 5-6 sequential tool calls into one script execution.

## Quick Start

```bash
# Install dependencies
pip install -e ".[dev]"

# Run tests
pytest -v

# Run with coverage
pytest --cov=scripts --cov-report=html -v

# Execute workflow
python scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 \
  123 \
  TICKET-456 \
  "Add vector search caching"
```

## What It Does

After creating a PR, this skill executes 6 operations in sequence:

1. **task_update** - Set task status to `pr_open`, record PR metadata
2. **jira_transition_issue** - Move JIRA ticket to "Code Review"
3. **jira_add_comment** - Post PR link and summary to JIRA
4. **slack_notify** - Send `pr_created` notification to team channel
5. **memory_store** - Save implementation learnings for future reference
6. **bot_status_update** - Set bot status to `idle`

All operations use **fail-fast error handling**: if any operation fails, execution stops immediately.

## Usage

### From Claude Code

```bash
/post-pr https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 123 TICKET-456 "Add caching"
```

### From Command Line

```bash
# Basic usage
python scripts/post_pr_operations.py PR_URL PR_NUMBER TICKET_ID SUMMARY

# With options
python scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/124 \
  124 \
  TICKET-457 \
  "Fix timeout" \
  --slack-channel=#hcc-alerts \
  --skip=slack,memory \
  --dry-run

# JSON output
python scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/125 \
  125 \
  TICKET-458 \
  "Update deps" \
  --json
```

### From Python

```python
from scripts.post_pr_operations import execute_post_pr_workflow

result = execute_post_pr_workflow(
    pr_url="https://github.com/RedHatInsights/hcc-ai-assistant/pull/123",
    pr_number=123,
    ticket_id="TICKET-456",
    summary="Add vector search caching",
    slack_channel="#hcc-ai-assistant",
    task_id="task-789",
    skip_operations=[],
    dry_run=False,
)

if result.success:
    print("✓ All operations completed successfully")
else:
    print(f"✗ Workflow failed: {result.operations}")
```

## Configuration

Set these environment variables to enable real API integrations:

```bash
# JIRA
export POST_PR_JIRA_URL=https://issues.redhat.com
export POST_PR_JIRA_TOKEN=your-token-here

# Slack
export POST_PR_SLACK_WEBHOOK=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# Storage (optional, defaults to /tmp)
export POST_PR_TASK_DB=/path/to/tasks.db
export POST_PR_MEMORY_STORE=/path/to/memory.json
```

Without these environment variables, the skill runs in stub mode (logs actions without making real API calls).

## Testing

```bash
# Run all tests
pytest -v

# Run specific test file
pytest tests/test_operations.py -v
pytest tests/test_integration.py -v

# Run with coverage
pytest --cov=scripts --cov-report=html -v

# View coverage report
open htmlcov/index.html
```

### Test Coverage

- **Unit tests** (`test_operations.py`): Test each operation individually
- **Integration tests** (`test_integration.py`): Test full workflow end-to-end

Current coverage: **100%** of executable code

## Architecture

### Design Principles

1. **Fail fast**: Stop on first error to maintain consistency
2. **No LLM reasoning**: All inputs known at PR creation time
3. **Sequential execution**: Operations have dependencies (e.g., JIRA transition before comment)
4. **Idempotent**: Safe to retry on failure
5. **Observable**: Logs all actions to stdout
6. **Testable**: Comprehensive unit and integration tests

### File Structure

```
.claude/skills/post-pr/
├── SKILL.md                     # Skill documentation (Claude Code entrypoint)
├── README.md                    # This file
├── pyproject.toml               # Dependencies and tool config
├── scripts/
│   ├── __init__.py
│   └── post_pr_operations.py   # Main implementation
└── tests/
    ├── __init__.py
    ├── test_operations.py       # Unit tests (19 tests)
    └── test_integration.py      # Integration tests (15 tests)
```

### Dependencies

- **Python 3.12+**: Required for modern type hints and language features
- **pytest**: Testing framework
- **pytest-cov**: Coverage reporting

No external dependencies for the core implementation (stub mode only).

## Integration with Real APIs

### JIRA API

To integrate with real JIRA API, update `jira_transition_issue` and `jira_add_comment` in `scripts/post_pr_operations.py`:

```python
import requests

def jira_transition_issue(self, ticket_id: str, target_status: str = "Code Review"):
    headers = {
        "Authorization": f"Bearer {self.jira_token}",
        "Content-Type": "application/json",
    }

    # Get available transitions
    resp = requests.get(
        f"{self.jira_url}/rest/api/2/issue/{ticket_id}/transitions",
        headers=headers,
    )
    transitions = resp.json()["transitions"]

    # Find target transition ID
    transition_id = next(
        (t["id"] for t in transitions if t["name"] == target_status),
        None,
    )

    # Execute transition
    requests.post(
        f"{self.jira_url}/rest/api/2/issue/{ticket_id}/transitions",
        headers=headers,
        json={"transition": {"id": transition_id}},
    )
```

### Slack Webhook

To integrate with real Slack webhook, update `slack_notify`:

```python
import requests

def slack_notify(self, pr_url: str, pr_number: int, summary: str, channel: str):
    message = {
        "channel": channel,
        "text": f"New PR created: #{pr_number}",
        "attachments": [{
            "color": "good",
            "fields": [
                {"title": "PR", "value": f"<{pr_url}|#{pr_number}>"},
                {"title": "Summary", "value": summary},
            ],
        }],
    }

    resp = requests.post(self.slack_webhook, json=message)
    resp.raise_for_status()
```

## Troubleshooting

### Common Issues

**Error: "JIRA token not configured"**
- Set `POST_PR_JIRA_TOKEN` environment variable
- Or run with `--skip=jira` to skip JIRA operations

**Error: "Slack webhook not configured"**
- Set `POST_PR_SLACK_WEBHOOK` environment variable
- Or run with `--skip=slack` to skip Slack notifications

**Workflow stops partway through**
- This is expected behavior (fail-fast)
- Check error message to identify which operation failed
- Fix the issue and re-run the workflow

### Dry Run Mode

Use `--dry-run` to preview what would happen without executing:

```bash
python scripts/post_pr_operations.py PR_URL PR_NUMBER TICKET_ID SUMMARY --dry-run
```

This logs all actions but doesn't make API calls or write files.

## Contributing

### Code Style

- **Line length**: 120 characters (black + ruff)
- **Type hints**: Required for all functions
- **Docstrings**: Google style for all public functions
- **Tests**: Required for all new operations

### Adding New Operations

1. Add method to `PostPROperations` class
2. Update `execute_post_pr_workflow` to call the new operation
3. Add unit tests in `tests/test_operations.py`
4. Add integration tests in `tests/test_integration.py`
5. Update SKILL.md documentation

### Running Tests Before Commit

```bash
# Format code
black .

# Lint
ruff check --fix .

# Run tests
pytest -v

# Check coverage
pytest --cov=scripts --cov-report=term-missing -v
```

## License

Same as parent project (hcc-ai-assistant).
