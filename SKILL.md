---
name: managing-proxyuser-tests
description: Creates, runs, and monitors E2E tests via ProxyUser API. Use when implementing features that need E2E test coverage, when tests fail and need diagnosis, when running tests against preview deployments, or when setting up automated QA for a web application.
---

# ProxyUser E2E Testing

ProxyUser is an AI-assisted E2E QA platform. Describe tests in plain English, and AI executes them as a human would.

## Quick Start

### Authentication

Store your API key in the environment:

```bash
export PROXYUSER_API_KEY="sk_live_your_key_here"
```

Create API keys at https://proxyuser.com/settings/api_keys

### Verify Setup

```bash
curl -s https://proxyuser.com/api/v1/projects \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.projects[].name'
```

### Key Concepts

| Concept | Description | ID Format |
|---------|-------------|-----------|
| **Project** | Groups scenarios for an application | `proj_xxx` |
| **Scenario** | Plain English test description + URL | `scen_xxx` |
| **Run** | Single test execution with status and video | `run_xxx` |
| **Folder** | Groups scenarios with shared instructions | `fold_xxx` |

---

## Workflows

### Create Test for New Feature

When implementing a feature that needs E2E coverage:

```
- [ ] List existing scenarios to avoid duplicates
- [ ] Identify the user journey the feature enables
- [ ] Write scenario with specific, observable assertions
- [ ] Run scenario to verify it passes
- [ ] Commit scenario alongside feature code
```

**Step 1: Check existing scenarios**

```bash
curl -s "https://proxyuser.com/api/v1/projects/$PROJECT_ID/scenarios" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.scenarios[].prompt'
```

**Step 2: Create the scenario**

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/scenarios" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "User clicks Add to Cart button, cart icon shows count of 1, checkout button becomes visible",
    "url": "https://myapp.com/products/widget-1"
  }'
```

**Step 3: Run and verify**

```bash
curl -X POST "https://proxyuser.com/api/v1/scenarios/$SCENARIO_ID/run" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

---

### Run Scenario and Wait for Results

Test runs are async. Trigger, then poll for completion:

```bash
# Trigger run
RUN_ID=$(curl -s -X POST "https://proxyuser.com/api/v1/scenarios/$SCENARIO_ID/run" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq -r '.data.run.id')

echo "Started run: $RUN_ID"

# Poll until complete (timeout after 5 minutes)
for i in {1..30}; do
  RESULT=$(curl -s "https://proxyuser.com/api/v1/runs/$RUN_ID" \
    -H "Authorization: Bearer $PROXYUSER_API_KEY")

  STATUS=$(echo $RESULT | jq -r '.data.run.status')

  case $STATUS in
    passed)
      echo "Test passed!"
      exit 0
      ;;
    failed)
      echo "Test failed:"
      echo $RESULT | jq -r '.data.run.ai_diagnosis'
      exit 1
      ;;
    *)
      echo "Status: $STATUS..."
      sleep 10
      ;;
  esac
done

echo "Timeout waiting for test"
exit 1
```

---

### Run All Tests on Preview Deploy

Run your entire test suite against a preview/staging URL:

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/run_all" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"target_url": "https://preview-abc123.vercel.app"}'
```

Filter to specific folders:

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/run_all" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "https://staging.myapp.com",
    "folder_ids": ["fold_abc123", "fold_def456"]
  }'
```

---

### Diagnose a Failed Test

Get the AI diagnosis and video for a failed run:

```bash
curl -s "https://proxyuser.com/api/v1/runs/$RUN_ID" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '{
    status: .data.run.status,
    diagnosis: .data.run.ai_diagnosis,
    video: .data.run.video_url,
    scenario: .data.run.scenario.prompt
  }'
```

**Common diagnosis patterns:**

| Diagnosis | Likely Cause | Action |
|-----------|--------------|--------|
| "Element not found" | Selector changed, element removed | Check if UI changed |
| "Timeout waiting for" | Slow load, async issue | Check performance |
| "Unexpected text" | Copy changed, display bug | Verify expected text |
| "Navigation failed" | Route broken, redirect issue | Check routing |

---

### List Recent Failures

```bash
curl -s "https://proxyuser.com/api/v1/runs?status=failed&limit=10" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.runs[] | {
    id: .id,
    scenario_id: .scenario_id,
    completed: .completed_at
  }'
```

---

## Writing Effective Scenarios

### Prompt Structure

Good prompts follow: **Action → Observable Result → Verification**

**Good:**
```
User enters "test@example.com" in email field, clicks Submit button,
sees "Check your inbox" confirmation message on screen
```

**Bad:**
```
Test the signup form
```

### Code-to-Test Translation

When you implement a feature, ask: "What would a user do to verify this works?"

| Code Change | Test Prompt |
|-------------|-------------|
| Add delete button | "User clicks delete icon on first item, confirmation dialog appears, clicks Confirm, item is removed from list" |
| Implement search | "User types 'widget' in search box, results update to show only items containing 'widget'" |
| Add form validation | "User leaves email empty, clicks Submit, sees 'Email is required' error" |
| Add pagination | "User scrolls to bottom, clicks Load More, additional items appear below" |
| Implement login | "User enters {{EMAIL}} in email field, enters {{PASSWORD}} in password, clicks Sign In, sees dashboard" |

### Environment Variables

Use `{{VAR}}` for credentials (configured in ProxyUser dashboard):

```
User enters {{TEST_EMAIL}} in email field, enters {{TEST_PASSWORD}} in password field,
clicks Login, sees Welcome message
```

### Common Patterns

See [examples/prompt-patterns.md](examples/prompt-patterns.md) for a library of reusable test scenario templates.

---

## API Reference

### Base URL

```
https://proxyuser.com/api/v1
```

### Authentication

```
Authorization: Bearer sk_live_xxx
```

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/projects` | List all projects |
| POST | `/projects` | Create project |
| GET | `/projects/:id` | Get project with scenarios |
| PATCH | `/projects/:id` | Update project |
| DELETE | `/projects/:id` | Delete project |
| POST | `/projects/:id/run_all` | Run all active scenarios |
| GET | `/projects/:id/scenarios` | List scenarios |
| POST | `/projects/:id/scenarios` | Create scenario |
| GET | `/scenarios/:id` | Get scenario with recent runs |
| PATCH | `/scenarios/:id` | Update scenario |
| DELETE | `/scenarios/:id` | Delete scenario |
| POST | `/scenarios/:id/run` | Trigger test run |
| GET | `/runs` | List runs (filter: status, project_id, scenario_id) |
| GET | `/runs/:id` | Get run with AI diagnosis |

### Response Format

```json
{
  "data": { ... },
  "meta": {
    "request_id": "abc-123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Run Status Values

- `pending` - Queued for execution
- `running` - Currently executing
- `passed` - Test completed successfully
- `failed` - Test failed (check ai_diagnosis)

### Error Response

```json
{
  "error": {
    "code": "scenario_not_found",
    "message": "Scenario with ID 'scen_xxx' not found",
    "hint": "Use GET /api/v1/projects/:id to see available scenarios"
  }
}
```

---

## CI/CD Integration

### GitHub Actions (Vercel Preview Deploys)

```yaml
name: E2E Tests
on:
  deployment_status:

jobs:
  test:
    if: github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: proxyuserai/action@v1
        with:
          api-key: ${{ secrets.PROXYUSER_API_KEY }}
          project-id: 'proj_xxx'
          target-url: ${{ github.event.deployment_status.target_url }}
          fail-on-test-failure: 'true'
```

### Other CI Systems

Use the API directly with the polling script from "Run Scenario and Wait for Results" above.

Full CI/CD documentation: https://proxyuser.com/docs#github-integration

---

## Troubleshooting

### Test passes locally but fails in CI

- Check if `target_url` override is correct
- Verify environment variables are set in ProxyUser dashboard
- Check if preview deploy is fully ready before tests run

### Video not available

Videos take ~30 seconds to process after test completion. Poll the run endpoint until `video_available: true`.

### Rate limits

The API allows 100 requests per minute per API key. For bulk operations, use `run_all` instead of individual scenario runs.

---

## Resources

- **Full API Docs**: https://proxyuser.com/docs
- **Dashboard**: https://proxyuser.com
- **GitHub Action**: https://github.com/proxyuserai/action
