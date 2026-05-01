---
name: proxyuser
description: Creates and monitors continuous scenarios via ProxyUser API. Use when the user just installed the proxyuser skill, says "set up ProxyUser", "add ProxyUser to this project", "get started with ProxyUser", or otherwise mentions onboarding to ProxyUser; when setting up continuous monitoring for user flows; when scenarios fail and need diagnosis; or when adding scenarios alongside new features.
---

# ProxyUser Continuous Monitoring

ProxyUser is continuous monitoring for web apps — automated proof your app works. Describe scenarios in plain English; AI executes them like a human would, on a recurring schedule.

## First-time use

When the user mentions ProxyUser without a specific task — *"I installed the proxyuser skill"*, *"set up ProxyUser"*, *"get me started"*, or similar — do **not** ask them what to do first. Drive the full onboarding yourself:

1. **Resolve the API key** (env var → `~/.proxyuser/config.json` → signup flow). See [Authentication](#authentication).
2. **Run signup if no key was found** — collect email, call `/agent/signup`, walk them through OTP. See [Signup flow](#signup-flow-no-key-found).
3. **Then ask once: *"What URL should I monitor, and what user flows matter most?"*** — and proceed to [Starter Scenarios Workflow](#starter-scenarios-workflow).

The user came in cold from a marketing page. They don't know what `/proxyuser` is, what a "scenario" is, or what to type. Owning the next 60 seconds end-to-end is the entire job.

## Quick Start


### Authentication

The skill provisions and stores its own API key. Do not send the user to a dashboard to create one.

**Precedence — check in this order:**

1. **`PROXYUSER_API_KEY` environment variable.** If set, use it and skip signup entirely.
2. **`~/.proxyuser/config.json`.** File shape: `{ "api_key": "sk_live_…", "organization_id": "org_…" }`. Mode `0600` (owner read/write only). If missing or empty, fall through.
3. **Run the signup flow** (next section).

**Key handling rules:**

- Never log, echo, or include the raw key in any user-facing output. When you must reference it, mask as `sk_live_••••<last4>`.
- Keys have scope. A key returned from `/agent/signup` starts as `pending` and only works against `/agent/verify`. After a successful `verify`, the *same key* upgrades to `full` scope — keep using it; no new key is issued.
- Throughout this doc, curl examples use `$PROXYUSER_API_KEY` as a placeholder for the resolved key. Substitute whichever value you got from the precedence above (env var, config file, or fresh from signup+verify). If you loaded from `~/.proxyuser/config.json`, you can either export it into the shell for the duration of the call or splice it inline.

### Signup flow (no key found)

When neither the env var nor `~/.proxyuser/config.json` has a key, walk the user through signup. Do not skip ahead and do not invent a key.

**Step 1: Ask for an email.**

> "What email should I use to sign you up for ProxyUser? You'll get a 6-digit code there."

**Step 2: Call `POST /api/v1/agent/signup`** (no auth header):

```bash
curl -X POST "https://proxyuser.com/api/v1/agent/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "agent_source": "claude-code"
  }'
```

`agent_source` identifies the calling agent for analytics. Best-effort detection — use one of: `claude-code`, `cursor`, `copilot`, `cline`, `windsurf`, `unknown`.

**Step 3: Persist the key on success (200).**

Response shape:

```json
{
  "api_key": "sk_live_xxx",
  "organization_id": "org_xxx",
  "organization_name": "Acme Inc",
  "status": "pending_verification",
  "expires_in": 600
}
```

Write `~/.proxyuser/config.json` immediately with mode `0600`. **Substitute the literal `api_key` and `organization_id` values from the response** — do not write the placeholders below verbatim. Set `umask 077` before the write so the file is owner-only from the moment it exists (a permissive umask + post-hoc `chmod` leaves a brief window where the file with the raw key is world-readable):

```bash
( umask 077 && \
  mkdir -p ~/.proxyuser && \
  printf '{"api_key":"%s","organization_id":"%s"}' "$API_KEY" "$ORG_ID" \
    > ~/.proxyuser/config.json )
```

`$API_KEY` and `$ORG_ID` here stand in for the values from the signup response — splice them in directly rather than relying on shell variables if that's simpler.

If `umask` / `chmod` are unavailable on the platform (e.g. some Windows shells), warn the user that the key file may be world-readable and proceed.

Then tell the user:

> "I sent a 6-digit code to `<email>`. Read it back to me when you've got it."

**Step 4: Handle non-success responses.**

- **409 Conflict (account exists):** Do not retry. Tell the user: *"You already have a ProxyUser account. Sign in at https://proxyuser.com/agent-key, copy the API key it gives you, and paste it here. I'll save it for you."* Accept the pasted `sk_live_…` token, validate it (below), then persist. Do not skip the validation.
- **429 Too Many Requests:** Surface the `Retry-After` header value to the user verbatim. Do not auto-retry. Limits: 5/min per IP, 3/hour per email.
- **5xx or network failure:** Tell the user the request failed; offer to retry once.

**Validating a pasted key (409 path):**

```bash
curl -s -o /dev/null -w '%{http_code}' \
  "https://proxyuser.com/api/v1/projects" \
  -H "Authorization: Bearer <pasted_key>"
```

Persist to `~/.proxyuser/config.json` (mode `0600`) only if the status is `200`. On `401`, tell the user the key didn't work and ask them to re-copy it from `/agent-key`.

### OTP verify

Once the user pastes a 6-digit code, verify it with the *same* key from signup (still `pending` scope). Strip whitespace from the code before sending.

```bash
curl -X POST "https://proxyuser.com/api/v1/agent/verify" \
  -H "Authorization: Bearer <pending_api_key>" \
  -H "Content-Type: application/json" \
  -d '{"otp": "123456"}'
```

**On success (200):** the same key is now `full` scope. Response: `{ api_key_scope: "full", user, organization, dashboard_url }`. **Do not re-persist** — the file from signup already has the right key. Continue with the user's original task.

**Failure handling:**

- **403 invalid_otp:** Response body includes `{ attempts_remaining }`. Tell the user: *"That code didn't match. Try again — `<n>` attempts left."* If `attempts_remaining == 0`, the key is dead. Restart signup from Step 1 and **ask the user for a new email** — do not silently retry the same address.
- **410 expired:** OTP timed out (10 minutes). Restart signup from Step 1.
- **429 Too Many Requests:** Surface `Retry-After`. Limit: 10/min per API key.

### After signup

Tell the user:

> "You're all set. I'll add scenarios as you describe them. Your dashboard is at https://proxyuser.com/dashboard if you ever want to look — but you shouldn't need to."

Then continue with the user's original task — typically `POST /api/v1/projects` to create a project followed by `POST /api/v1/projects/:id/scenarios`. Do not redirect the user to the dashboard before they have a scenario.

### Verify Setup

```bash
curl -s https://proxyuser.com/api/v1/projects \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.projects[].name'
```

### Key Concepts

| Concept | Description | ID Format |
|---------|-------------|-----------|
| **Project** | Groups scenarios for an application | `proj_xxx` |
| **Scenario** | Plain English description of a user flow + URL | `scen_xxx` |
| **Run** | Single scenario execution with status and video | `run_xxx` |
| **Folder** | Groups scenarios with shared instructions | `fold_xxx` |

---

## When Invoked Without Instructions

If the user invokes `/proxyuser` without specifying what they want to do:

**Ask the user to choose:**

1. **Generate starter scenarios** - Analyze the codebase and create foundational monitoring scenarios
2. **Create scenario for a feature** - Write a scenario for something specific
3. **Run scenarios** - Execute existing scenarios against a URL
4. **Check recent failures** - Diagnose and investigate failed scenarios

### Starter Scenarios Workflow

When the user chooses "Generate starter scenarios":

**Step 1: Understand the application**

Analyze the codebase to identify:
- App type (e-commerce, SaaS, content site, etc.)
- Main user-facing routes and pages
- Authentication patterns (email/password, OAuth, magic link)
- Core features that deliver user value

Look at: route files, navigation components, page directories, and any existing scenario coverage.

**Step 2: Check existing coverage**

List all existing scenarios in the project to avoid duplicates:

```bash
curl -s "https://proxyuser.com/api/v1/projects/$PROJECT_ID/scenarios" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.scenarios[].prompt'
```

**Step 3: Suggest foundational scenarios**

Recommend scenarios following the prioritization order:

1. **Revenue-critical paths** - Checkout, payment, upgrade flows
2. **User acquisition** - Signup, onboarding
3. **Core product value** - The main thing users come to do
4. **Authentication** - Login, logout, password reset

Limit initial suggestions to 5-10 high-impact scenarios. Quality over quantity.

**Step 4: Create scenarios with proper organization**

Group related scenarios into folders. Create folders first, then add scenarios to them.

Example structure for an e-commerce app:
```
📁 Authentication
   - User can sign up with email
   - User can log in with credentials
📁 Shopping
   - User can add item to cart
   - User can view cart
📁 Checkout
   - User can complete purchase
```

---

## Workflows

### Create Scenario for New Feature

When implementing a feature that needs monitoring coverage:

```
- [ ] **List existing scenarios and review coverage before suggesting new ones**
- [ ] Identify the user journey the feature enables
- [ ] Extract the starting URL from the user's prompt or app config; if missing, ask before creating the scenario. Do not invent URLs.
- [ ] Write scenario with specific, observable assertions
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
    "url": "https://example.com/products/widget-1"
  }'
```

The first run auto-enqueues on creation. Poll `GET /scenarios/:id` to see the latest run status, or watch the dashboard.

---

### Run All Scenarios on Preview Deploy

Re-run your entire scenario suite against a preview/staging URL:

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
    "target_url": "https://staging.example.com",
    "folder_ids": ["fold_abc123", "fold_def456"]
  }'
```

---

### Diagnose a Failed Scenario

Quick lookup. For the full investigation (verifier verdict, screenshot, trace, cost, recording, snooze/acknowledge follow-ups), use [Investigate a failed run](#investigate-a-failed-run) under Operations.

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

## Operations

These workflows replace the dashboard for the day-to-day. Drive them from chat — the only browser steps are Stripe Checkout and Slack OAuth (handled via the [`action_required` envelope](#action_required-envelope)).

### Investigate a failed run

When the user asks *"why did the X scenario fail?"*, fetch the latest failure and surface a human paragraph — not raw JSON.

**Step 1: Locate the run.** If the user named a scenario, look it up first (`GET /projects/:id/scenarios`), then list its recent failures:

```bash
curl -s "https://proxyuser.com/api/v1/runs?scenario_id=$SCENARIO_ID&status=failed&limit=5" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.runs[] | {id, completed_at}'
```

Pick the latest. If the user gave a run ID directly, skip to Step 2.

**Step 2: Pull the full diagnosis.**

```bash
curl -s "https://proxyuser.com/api/v1/runs/$RUN_ID" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

The response now includes `verifier_verdict`, `verifier_reasoning`, `final_screenshot_url`, `trace_url`, `cost_usd`, `duration_ms`, `heals[]`, `replay_plan` summary, and `acknowledged_at` alongside the existing `ai_diagnosis`.

**Step 3: Render a human paragraph.** Do **not** dump JSON. A good response covers, in this order:

1. Scenario prompt (what we were testing)
2. AI diagnosis (one sentence)
3. Verifier verdict + reasoning (one sentence)
4. What it cost / how long it took (one short clause)
5. Whether the agent self-healed during the run (`heals[]` non-empty → mention briefly)
6. Links to the final screenshot and trace, on their own lines

**Step 4: Offer next actions.** End with a short menu:

> *"Want me to (a) walk through the recorded steps, (b) acknowledge this so the recovery alert is suppressed, or (c) snooze the scenario while you fix it?"*

- **Recording** → `GET /api/v1/runs/$RUN_ID/recording` returns rrweb events. Render a high-level summary (event count, last URL, last action) rather than dumping raw events.
- **Acknowledge** → see [Snooze and acknowledge](#snooze-and-acknowledge) below.
- **Snooze** → see [Snooze and acknowledge](#snooze-and-acknowledge) below.

If `acknowledged_at` is already set on the run, mention it: *"You've already acknowledged this one — the recovery alert won't fire."*

---

### Snooze and acknowledge

Four sub-flows. Pick based on the user's ask.

**1. *"Mute this scenario for the next 2 hours"*** — per-scenario snooze. Compute an ISO8601 timestamp `now + 2h`:

```bash
UNTIL=$(date -u -v+2H +"%Y-%m-%dT%H:%M:%SZ")  # macOS
# UNTIL=$(date -u -d '+2 hours' +"%Y-%m-%dT%H:%M:%SZ")  # Linux

curl -X POST "https://proxyuser.com/api/v1/scenarios/$SCENARIO_ID/snooze" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"until\": \"$UNTIL\"}"
```

**2. *"Mute the whole project for the rest of the day"*** — project-wide snooze via `paused_until`:

```bash
curl -X PATCH "https://proxyuser.com/api/v1/projects/$PROJECT_ID/alert_settings" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"paused_until": "2024-01-15T23:59:59Z"}'
```

**3. *"Stop notifying me about this run"*** — per-run acknowledge. **Important:** this only suppresses the *recovery* notification for this failure streak — future failures still alert. Tell the user that explicitly:

```bash
curl -X POST "https://proxyuser.com/api/v1/runs/$RUN_ID/acknowledge" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

> *"Done — acknowledged. You won't get a recovery ping when the next pass comes back green. New failures will still alert you."*

**4. *"Bump the threshold to 3 failures before alerting"*** — change project alert threshold:

```bash
curl -X PATCH "https://proxyuser.com/api/v1/projects/$PROJECT_ID/alert_settings" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"threshold": 3}'
```

Read the current settings any time with `GET /api/v1/projects/$PROJECT_ID/alert_settings`.

---

### Hit the cap → upgrade

Two triggers:

- **Reactive** — `POST /api/v1/projects/$PROJECT_ID/scenarios` returned a cap error.
- **Proactive** — `GET /api/v1/billing` shows `data.active_scenario_count >= data.plan.scenario_cap`. Worth checking before bulk creates.

**Upgrade flow:**

```bash
curl -X POST "https://proxyuser.com/api/v1/billing/checkout" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"plan_id": "pro"}'
```

Response is the `action_required` envelope. Follow the [canonical pattern](#action_required-envelope):

1. Show `reason` and `url` to the user.
2. Wait for "done" / "ok" / similar — never click for them.
3. Poll `GET /api/v1/billing` every 3s, up to 40 times, until `data.plan.id` differs from where it started.
4. Confirm: *"You're on the Pro plan now. Resuming."*

If the response comes back as `422 non_checkout_plan`, the requested plan isn't self-serve — see Error Handling.

**Manage existing subscription** (cancel, change card, download invoices):

```bash
curl -X POST "https://proxyuser.com/api/v1/billing/portal" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

Same envelope, same polling pattern. The portal is also the path for downgrades and cancellations — the skill never tries to do those itself.

---

### Connect / disconnect Slack

Org-wide, single channel. One Slack connection per organization.

**Connect:**

```bash
curl -X POST "https://proxyuser.com/api/v1/slack_settings/connect" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

Returns the [`action_required` envelope](#action_required-envelope). Show URL → wait for confirmation → poll `GET /api/v1/slack_settings` every 3s until `data.connected == true` (max 2 min).

After connection, optionally verify with a test message:

```bash
curl -X POST "https://proxyuser.com/api/v1/slack_settings/test" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

Then ask the user: *"See the test message in the channel?"* — don't assume delivery.

**Disconnect** (no browser step):

```bash
curl -X DELETE "https://proxyuser.com/api/v1/slack_settings" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

`POST /slack_settings/connect` may return `503 slack_not_configured` on environments where Slack OAuth isn't configured — see Error Handling.

---

### Invite a teammate

```bash
curl -X POST "https://proxyuser.com/api/v1/team/invitations" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "mike@example.com", "role": "member"}'
```

Roles: `owner`, `member`. Confirmation copy:

> *"Sent an invite to mike@example.com. They'll get an email; once they accept, they'll show up in `team/members`."*

**Promote / demote / remove** an existing member:

```bash
# Promote Alice to owner
curl -s "https://proxyuser.com/api/v1/team/members" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  | jq '.data.members[] | select(.email == "alice@example.com") | .id'
# → mem_xxx

curl -X PATCH "https://proxyuser.com/api/v1/team/members/mem_xxx" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"role": "owner"}'
```

Last-owner protection is server-side. Demoting or removing the only owner returns `422 last_owner` — see Error Handling. Pending invitations are listed at `GET /api/v1/team/invitations` and can be revoked with `DELETE /api/v1/team/invitations/:id`.

---

### Per-app memory (advanced)

ProxyUser builds up a per-project memory of `known_locators` (DOM hints the agent has confirmed for a given URL) and `reflections` (free-form notes from prior runs). Most users never touch this — surface it when the agent is making the wrong call.

**Read** the current memory (e.g. when the user asks *"why is ProxyUser clicking the wrong button?"*):

```bash
curl -s "https://proxyuser.com/api/v1/projects/$PROJECT_ID/memory" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

Render the locators relevant to the URL the user mentioned — not the whole map. Offer to edit.

**Edit** — e.g. *"Tell ProxyUser that on this app, the sign-in button reads 'Continue', not 'Sign in'"*:

```bash
curl -X PATCH "https://proxyuser.com/api/v1/projects/$PROJECT_ID/memory" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "known_locators": {
      "https://app.example.com/login": [
        { "intent": "submit_login", "role": "button", "name": "Continue" }
      ]
    }
  }'
```

PATCH is **additive** — existing locators for the URL are preserved; new ones are upserted; `confirmed_runs` increments on repeat. To remove a locator, the user has to do it from the dashboard (or call the API with the locator omitted under a deliberate full-replacement strategy — out of scope here).

---

### Bulk scenario creation

When the user describes several flows at once — *"add scenarios for sign-in, signup, checkout, password reset on `https://app.example.com`"* — use bulk_create instead of looping over single-create.

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/scenarios/bulk_create" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://app.example.com",
    "prompts": [
      "User can sign in with {{EMAIL}} and {{PASSWORD}}",
      "User can sign up with a new email address",
      "User can complete a Stripe test-mode checkout",
      "User can request a password reset"
    ]
  }'
```

**Transactional**: if any prompt fails validation, none persist. Surface which prompt(s) failed (response `error.details` will name them) — do not silently retry with a subset.

---

### Project environment variables

Test secrets the agent uses inside scenarios — e.g. `STRIPE_TEST_KEY`, `TEST_USER_TOKEN`. Distinct from the `{{VAR}}` placeholder syntax in scenario prompts (see [Environment Variables](#environment-variables) for that).

**Set** a value:

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/environment_variables" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"key": "STRIPE_TEST_KEY", "value": "sk_test_..."}'
```

Keys must match `^[A-Z][A-Z0-9_]*$`. The server may normalize a user-supplied key (e.g. `stripe_test_key` → `STRIPE_TEST_KEY`); when that happens, tell the user *"Saved as `STRIPE_TEST_KEY` (normalized from your input)."*

**Update** an existing key with PATCH (same body shape). **Delete** with DELETE.

**List** — values are encrypted at rest and **never returned** by the API. Listing returns names only:

```bash
curl -s "https://proxyuser.com/api/v1/projects/$PROJECT_ID" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  | jq '.data.project.environment_variables'
```

When the user asks *"show me my env vars,"* be explicit:

> *"Here are the keys — I can't show you the values; ProxyUser stores them encrypted and never hands them back. If you've lost a value, you'll need to re-set it."*

---

### Rotate the API key

The order matters. **Create a new key first, switch the local config to it, then revoke the old one** — the API rejects deleting the calling key with `422 self_revoke_forbidden`.

**Step 1: Create the new key.** Capture the raw token immediately — it's returned **once** and is not retrievable afterward.

```bash
curl -X POST "https://proxyuser.com/api/v1/api_keys" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"claude-code rotation $(date -u +%Y-%m-%d)\"}"
```

Response shape:

```json
{
  "data": {
    "api_key": {
      "id": "key_new",
      "name": "claude-code rotation 2024-01-15",
      "token": "sk_live_..."
    }
  }
}
```

**Step 2: Persist the new key to `~/.proxyuser/config.json`** with mode 0600 (same `umask 077` idiom as signup at the top of this doc):

```bash
( umask 077 && \
  printf '{"api_key":"%s","organization_id":"%s"}' "$NEW_TOKEN" "$ORG_ID" \
    > ~/.proxyuser/config.json )
```

Substitute `$NEW_TOKEN` (from `data.api_key.token`) and `$ORG_ID` (read it from the existing `~/.proxyuser/config.json` before overwriting). From this point on, all subsequent calls use the new key.

**Step 3: Identify the old key's ID, then revoke it** with the *new* key now in effect:

```bash
# List keys, find the one that isn't the just-created new key (match by id or created_at)
curl -s "https://proxyuser.com/api/v1/api_keys" \
  -H "Authorization: Bearer $NEW_TOKEN" \
  | jq '.data.api_keys[] | {id, name, created_at}'

curl -X DELETE "https://proxyuser.com/api/v1/api_keys/$OLD_KEY_ID" \
  -H "Authorization: Bearer $NEW_TOKEN"
```

If the listing makes it ambiguous which key is the old one (e.g. the user has rotated before and has several), confirm with the user before deleting — better to leave a stale key than nuke an active one.

If the user asks to delete their *current* key in isolation (without rotating first), the API returns `422 self_revoke_forbidden` — see Error Handling. Walk them through the rotate flow above instead of trying again.

---

## Writing Effective Scenarios

### Keep It Simple

ProxyUser's AI browses like a human. Trust it to figure out the details—don't write step-by-step instructions.

**Good:**
```
User can sign up with a valid email address
```

**Bad:**
```
User enters "test@example.com" in email field, clicks Submit button,
sees "Check your inbox" confirmation message on screen
```

### Code-to-Scenario Translation

When you implement a feature, ask: "What capability does this give the user?"

| Code Change | Scenario Prompt |
|-------------|-------------|
| Add delete button | "User can delete an item from the list" |
| Implement search | "User can search for items by name" |
| Add form validation | "User sees error when submitting with empty email" |
| Add pagination | "User can load more items at bottom of list" |
| Implement login | "User can log in with {{EMAIL}} and {{PASSWORD}}" |

### Environment Variables

Use `{{VAR}}` for credentials (configured in ProxyUser dashboard):

```
User can log in with {{EMAIL}} and {{PASSWORD}}
```

### Common Patterns

See [examples/prompt-patterns.md](examples/prompt-patterns.md) for a library of reusable scenario templates.

### Error Handling

When the API rejects a request, surface the error to the user — don't silently retry forever.

**401 Unauthorized**

The API key in `~/.proxyuser/config.json` (or `$PROXYUSER_API_KEY`) is invalid, revoked, or still in `pending` scope. Either run the signup flow again, or have the user paste a fresh full-scope key from https://proxyuser.com/agent-key. Validate the pasted key with `GET /api/v1/projects` before persisting.

**403 invalid_otp**

The user typed the wrong 6-digit code. Response body includes `{ attempts_remaining }`. Surface the count: *"That code didn't match. Try again — `<n>` attempts left."* If `attempts_remaining == 0`, restart the signup flow with a fresh email request.

**409 Conflict (account exists)**

Returned by `/agent/signup` when the email already has a ProxyUser account. Do not retry. Route the user to https://proxyuser.com/agent-key to copy a key, then validate (`GET /api/v1/projects`) and persist to `~/.proxyuser/config.json`.

**410 Gone (otp_expired)**

The 6-digit code expired (10-minute window). Restart the signup flow.

**429 Too Many Requests**

Surface the `Retry-After` header verbatim and do not auto-retry. Limits: signup is 5/min per IP and 3/hour per email; verify is 10/min per key; the rest of the API is 100/min per key.

**422 Unprocessable Entity**

The request had invalid data. The response body shape:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "Validation failed: Url can't be blank",
    "hint": "Check the details field for specific field errors",
    "details": {
      "url": ["can't be blank"]
    }
  }
}
```

Surface `error.details` to the user verbatim — it lists which fields failed validation.

**5xx / network failure**

Retry once. If it fails again, surface the failure with the request body so the user can re-run when the API recovers. Do not loop silently.

**422 last_owner**

Returned by `PATCH /api/v1/team/members/:id` (demote) or `DELETE /api/v1/team/members/:id` when the target is the only owner. Surface the server message verbatim and stop — do not auto-retry. Tell the user: *"That's the only owner on the org. Promote someone else to owner first, then try again."*

**422 self_revoke_forbidden**

Returned by `DELETE /api/v1/api_keys/:id` when the target is the key the agent is currently authenticating with. Tell the user: *"I can't revoke the key I'm currently using. Create a new key first, switch to it, then revoke the old one."* This is the order the **Rotate the API key** runbook follows; if the user asked to delete their *current* key in isolation, walk them through the rotate flow instead.

**503 slack_not_configured**

Returned by `POST /api/v1/slack_settings/connect` when the Slack OAuth env isn't set up on this ProxyUser environment. Not a user error — tell them: *"Slack isn't enabled on this ProxyUser environment yet. Contact support."* Do not retry.

**422 non_checkout_plan**

Returned by `POST /api/v1/billing/checkout` when the requested `plan_id` isn't a self-serve checkout plan. Free plans have no checkout (use **Hit the cap → upgrade** to move *off* free); enterprise plans are managed by sales. Surface accordingly and don't retry.

### Prioritizing Scenarios

Focus on high-impact scenarios first. If this breaks, does the business suffer?

**Cover these first (in order):**

1. **Revenue-critical paths** - Checkout, payment, subscription upgrades
2. **User acquisition** - Signup, onboarding completion
3. **Core product value** - The main thing users come to do
4. **Authentication** - Login, logout, password reset
5. **Key error states** - Validation errors, failure messages users need to see

**Skip these (low impact):**
- Cosmetic hover states
- Admin-only features (cover manually)
- Edge cases that affect <1% of users

### Scenario Guardrails

**Before suggesting any new scenarios:**

1. **List existing scenarios first** - Run the list scenarios API call and review what's already covered
2. **Check for overlapping coverage** - Don't suggest scenarios that duplicate existing functionality
3. **Limit recommendations** - Recommend a maximum of 20 scenarios per session. Quality > quantity.

**When reviewing existing scenarios, look for:**
- Similar user journeys (e.g., "User can add item" vs "User adds product to cart")
- Overlapping scenario coverage (e.g., login covered in multiple scenarios)
- Scenarios that could be consolidated

**Red flags that you're over-covering:**
- More than 5 scenarios for a single feature
- Scenarios checking minor variations of the same flow
- Scenarios for edge cases affecting <1% of users

### Scenario Boundaries

**Only test what can be completed entirely within the browser.**

ProxyUser controls a browser. It cannot check email inboxes, receive SMS codes, or interact with external services.

❌ **Do NOT create scenarios that require:**

- Checking email (verification links, password reset emails)
- SMS or phone verification
- Third-party OAuth (Google, Apple, Facebook login buttons)
- External service responses (Slack notifications, webhook deliveries)
- Real payment processing (use test/sandbox modes instead)
- Mobile app interactions
- Desktop notifications or OS-level prompts

✅ **Instead, test the browser-side behavior:**

| Don't test this | Test this instead |
|-----------------|-------------------|
| "User receives verification email" | "User sees 'Check your email' confirmation" |
| "User logs in with Google" | "User sees Google login button and it's clickable" |
| "Payment completes successfully" | "User can submit payment form" (in test mode) |
| "User gets Slack notification" | "User sees 'Notification sent' message in UI" |

---

## Organizing Scenarios

### Use Folders to Group Related Scenarios

Avoid creating a flat list of scenarios. Group scenarios by feature area, user flow, or page section.

**Good organization:**
```
📁 Authentication
   - User can sign up with email
   - User can log in with credentials
   - User can reset password
📁 Shopping Cart
   - User can add item to cart
   - User can remove item from cart
   - User can update quantity
📁 Checkout
   - User can complete purchase
   - User can apply discount code
```

**Bad organization:**
```
- User can sign up
- User can add to cart
- User can log in
- User can checkout
- User can reset password
- User can remove from cart
... (12 more unorganized scenarios)
```

### Before Creating Scenarios

**Step 1: List existing folders**

```bash
curl -s "https://proxyuser.com/api/v1/projects/$PROJECT_ID/folders" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" | jq '.data.folders[] | {id, name, scenarios_count}'
```

To see scenarios within each folder, add `?include_scenarios=true`.

**Step 2: Choose the right folder**

- If a folder exists that matches the feature area → use it
- If no folder fits → create one (see below)
- If adding multiple related scenarios → create a folder for them

### Creating a Folder

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/folders" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Shopping Cart",
    "instructions": "All scenarios start from the product catalog page",
    "schedule": "0 9 * * *"
  }'
```

Parameters:
- `name` (required) - Folder name
- `instructions` (optional) - Shared context for all scenarios in the folder
- `schedule` (optional) - Cron expression for recurring scenario runs

**For folders requiring authentication:**

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/folders" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "User Dashboard",
    "instructions": "First, log in with {{EMAIL}} and {{PASSWORD}}"
  }'
```

**Note:** Set the scenario URLs to the login page (e.g., `https://example.com/login`) when using auth instructions.

### Folder Instructions and Inheritance

When you set `instructions` on a folder, **all scenarios in that folder inherit those instructions**. This is ideal for shared setup steps like authentication.

#### When to Use Folder-Level Authentication

Use folder-level instructions when:
- Multiple scenarios in the folder require the same login
- All scenarios start from an authenticated state
- You want DRY (Don't Repeat Yourself) scenario organization

Use scenario-level authentication when:
- Only one or two scenarios need authentication
- Different scenarios need different credentials
- The auth flow IS what you're testing

#### Example: Folder with Authentication

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/folders" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Dashboard Features",
    "instructions": "First, log in with {{EMAIL}} and {{PASSWORD}}"
  }'
```

Now scenarios in this folder are simple:
```
User can view analytics chart
User can export data to CSV
User can update notification settings
```

The AI will log in first (from folder instructions), then execute the scenario.

#### Before/After: DRY Principle

**Before (repetitive):**
```
📁 Dashboard (no instructions)
   - User logs in with {{EMAIL}} and {{PASSWORD}}, then views analytics
   - User logs in with {{EMAIL}} and {{PASSWORD}}, then exports data
   - User logs in with {{EMAIL}} and {{PASSWORD}}, then updates settings
```

**After (using folder instructions):**
```
📁 Dashboard
   instructions: "First, log in with {{EMAIL}} and {{PASSWORD}}"
   - User can view analytics chart
   - User can export data to CSV
   - User can update notification settings
```

#### Starting URL for Authentication

When scenarios require authentication, set the scenario's starting URL to the login page:

| Scenario Type | Starting URL |
|--------------|--------------|
| Public page scenario | `https://example.com/` |
| Authenticated scenario | `https://example.com/login` |
| Deep-link after auth | `https://example.com/login` (AI navigates after login) |

**Note:** Even if your folder instructions say to log in, the browser needs to start on a page where login is possible.

### Running All Scenarios in a Folder

```bash
curl -X POST "https://proxyuser.com/api/v1/folders/$FOLDER_ID/run" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY"
```

All triggered scenarios share a `batch_id` for unified tracking.

### Assigning Scenarios to Folders

Include `folder_id` when creating scenarios:

```bash
curl -X POST "https://proxyuser.com/api/v1/projects/$PROJECT_ID/scenarios" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "User can add item to cart",
    "url": "https://example.com/products",
    "folder_id": "fold_abc123"
  }'
```

### Moving Scenarios Between Folders

To move an existing scenario to a different folder:

```bash
curl -X PATCH "https://proxyuser.com/api/v1/scenarios/$SCENARIO_ID" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"folder_id": "fold_newFolder123"}'
```

To remove a scenario from its folder (move to root/unfiled):

```bash
curl -X PATCH "https://proxyuser.com/api/v1/scenarios/$SCENARIO_ID" \
  -H "Authorization: Bearer $PROXYUSER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"folder_id": null}'
```

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
| GET | `/projects/:id/folders` | List folders |
| POST | `/projects/:id/folders` | Create folder |
| GET | `/folders/:id` | Get folder details |
| PATCH | `/folders/:id` | Update folder |
| DELETE | `/folders/:id` | Delete folder |
| POST | `/folders/:id/run` | Run all scenarios in folder |
| GET | `/scenarios/:id` | Get scenario with recent runs |
| PATCH | `/scenarios/:id` | Update scenario (including folder assignment) |
| DELETE | `/scenarios/:id` | Delete scenario |
| GET | `/runs` | List runs (filter: status, project_id, scenario_id) |
| GET | `/runs/:id` | Get run with AI diagnosis, verifier verdict, screenshot, trace, cost, heals |
| GET | `/runs/:id/recording` | rrweb events for replay |
| POST | `/runs/:id/acknowledge` | Suppress recovery alert for this failure streak |
| DELETE | `/runs/:id` | Delete a run |
| POST | `/scenarios/:id/snooze` | Snooze a single scenario (`{ until }`) |
| POST | `/projects/:id/scenarios/bulk_create` | Create multiple scenarios in one transactional call (`{ prompts, url }`) |
| GET / PATCH | `/projects/:id/alert_settings` | Read or update `threshold` and `paused_until` |
| GET / PATCH | `/projects/:id/memory` | Read or upsert `known_locators` and `reflections` (PATCH is additive) |
| POST / PATCH / DELETE | `/projects/:id/environment_variables` | Test-secret CRUD; `GET /projects/:id` lists names only |
| GET | `/slack_settings` | Inspect org-wide Slack connection |
| POST | `/slack_settings/connect` | Returns `action_required` (OAuth URL) |
| POST | `/slack_settings/test` | Send a test message to the connected channel |
| DELETE | `/slack_settings` | Disconnect Slack |
| GET | `/billing` | Plan, usage, status |
| POST | `/billing/checkout` | Returns `action_required` (Stripe Checkout URL) |
| POST | `/billing/portal` | Returns `action_required` (Stripe Customer Portal URL) |
| GET | `/team/members` | List team members |
| PATCH | `/team/members/:id` | Update role (`{ role }`) |
| DELETE | `/team/members/:id` | Remove member |
| GET | `/team/invitations` | List pending invitations |
| POST | `/team/invitations` | Invite (`{ email, role }`) |
| DELETE | `/team/invitations/:id` | Revoke a pending invitation |
| GET | `/api_keys` | List the org's API keys |
| POST | `/api_keys` | Create a new key — raw token returned **once** in `data.api_key.token` |
| DELETE | `/api_keys/:id` | Revoke a key (cannot be the calling key) |

### Scenario Object

```json
{
  "id": "scen_xxx",
  "project_id": "proj_xxx",
  "folder_id": "fold_xxx",  // null if not in a folder
  "prompt": "User can sign up with email",
  "url": "https://example.com",
  "is_active": true,
  "schedule": null,
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

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

### action_required envelope

A small set of endpoints — `POST /billing/checkout`, `POST /billing/portal`, `POST /slack_settings/connect` — return 200 with an `action_required` block instead of the normal resource payload. The action requires a real browser (Stripe Checkout, Stripe Customer Portal, Slack OAuth), which the skill cannot drive. Shape:

```json
{
  "data": {
    "action_required": {
      "type": "browser",
      "url": "https://...",
      "reason": "One-line user-facing description",
      "poll_path": "/api/v1/billing"
    }
  }
}
```

**Three rules — apply every time you see `action_required`:**

1. **Surface `reason` and `url` to the user.** Verbatim. Do not paraphrase the URL or wrap it in markdown link syntax that could obscure it.
2. **Never automate clicking the `url`.** The user must complete it in a real browser. Wait for them to say "done" / "ok" / similar.
3. **Poll `poll_path` until the relevant state changes.** Every 3 seconds, give up after 2 minutes (40 attempts). On timeout, tell the user: *"I didn't see the change yet — try again, or check `https://proxyuser.com/dashboard` if you're stuck."*

**Per-endpoint state-change predicate** (what to check on each poll):

| Endpoint | poll_path | Predicate (state has changed when…) |
|---|---|---|
| `/billing/checkout` | `/api/v1/billing` | `data.plan.id` differs from the value at the start of polling |
| `/billing/portal` | `/api/v1/billing` | `data.plan.id` *or* `data.status` differs from the start (user may cancel, change card, etc.) |
| `/slack_settings/connect` | `/api/v1/slack_settings` | `data.connected == true` |

**Canonical polling snippet:**

```bash
# After the user confirms they completed the browser step:
for i in $(seq 1 40); do
  RESP=$(curl -s "https://proxyuser.com$POLL_PATH" \
    -H "Authorization: Bearer $PROXYUSER_API_KEY")
  # ...check the predicate against $RESP using jq...
  if [ "$CHANGED" = "true" ]; then break; fi
  sleep 3
done
```

If the loop exits without the change, do not retry the action — surface the timeout message above.

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

## Triggering scenarios after a deploy

Continuous monitoring runs on a schedule — that's the default. If you also want to re-run the suite immediately after a deploy completes, use `POST /projects/:id/run_all` from your deploy hook (Vercel Deploy Hooks, Netlify build notifications, GitHub `deployment_status` webhooks, your own deploy script). See [Run All Scenarios on Preview Deploy](#run-all-scenarios-on-preview-deploy) above for the request shape.

ProxyUser is not a pre-merge PR gate — failures don't block deploys. The model is "monitor what's live and tell me when something breaks," not "fail CI on a flaky run."


---

## Troubleshooting

### Scenario fails against a preview URL

- Check if `target_url` override is correct
- Verify environment variables are set in ProxyUser dashboard
- Check if preview deploy is fully ready before scenarios run

### Video not available

Videos take ~30 seconds to process after scenario completion. Poll the run endpoint until `video_available: true`.

### Rate limits

The API allows 100 requests per minute per API key. For bulk operations, use `run_all` instead of individual scenario runs.

---

## Resources

- **Full API Docs**: https://proxyuser.com/docs
- **Dashboard**: https://proxyuser.com
