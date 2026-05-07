# Post-PR Skill - Quick Start

## TL;DR

```bash
# From Claude Code
/post-pr PR_URL PR_NUMBER TICKET_ID "Summary"

# From command line (dry-run)
cd .claude/skills/post-pr
python3 scripts/post_pr_operations.py \
  https://github.com/RedHatInsights/hcc-ai-assistant/pull/123 \
  123 \
  TICKET-456 \
  "Add caching" \
  --dry-run
```

## Installation

```bash
cd .claude/skills/post-pr
python3 -m venv venv
source venv/bin/activate
pip install -e ".[dev]"
```

## Run Tests

```bash
pytest -v                                    # All tests
pytest --cov=scripts --cov-report=html -v   # With coverage
```

**Expected**: ✓ 32 passed in ~0.03s

## Common Commands

### Preview (Dry-Run)
```bash
python3 scripts/post_pr_operations.py PR_URL PR_NUM TICKET "Summary" --dry-run
```

### Skip Operations
```bash
# Skip JIRA and Slack
--skip=jira,slack

# Skip only Slack
--skip=slack
```

### Custom Slack Channel
```bash
--slack-channel=#hcc-security
```

### JSON Output
```bash
--json
```

## Configuration (Optional)

```bash
export POST_PR_JIRA_URL=https://issues.redhat.com
export POST_PR_JIRA_TOKEN=your-token
export POST_PR_SLACK_WEBHOOK=https://hooks.slack.com/services/...
```

Without these, operations will fail (use `--skip` to bypass).

## What It Does

1. ✓ Update task status to `pr_open`
2. ✓ Transition JIRA to "Code Review"
3. ✓ Add JIRA comment with PR link
4. ✓ Notify Slack channel
5. ✓ Store implementation learnings
6. ✓ Set bot status to `idle`

**Fail-fast**: Stops on first error.

## Troubleshooting

| Error | Solution |
|-------|----------|
| JIRA token not configured | `export POST_PR_JIRA_TOKEN=...` or `--skip=jira` |
| Slack webhook not configured | `export POST_PR_SLACK_WEBHOOK=...` or `--skip=slack` |
| Workflow stopped partway | Expected (fail-fast). Check error message, fix issue, re-run |

## Files

- `SKILL.md` - Skill documentation
- `README.md` - Full developer guide
- `scripts/post_pr_operations.py` - Implementation (501 lines)
- `tests/test_operations.py` - Unit tests (21 tests)
- `tests/test_integration.py` - Integration tests (11 tests)
- `examples/example_usage.md` - Usage examples
- `IMPLEMENTATION_SUMMARY.md` - Build summary

## Next Steps

1. ✓ Tests pass
2. ⬜ Set environment variables for real APIs
3. ⬜ Integrate with real JIRA/Slack (see README.md)
4. ⬜ Use `/post-pr` in Claude Code after creating a PR
