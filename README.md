# ProxyUser Skills

Agent Skills for [ProxyUser](https://proxyuser.com) — continuous proof your app works.

Describe scenarios in plain English. AI executes them like a human would, on a recurring schedule.

## Installation

### Using skills CLI

```bash
npx skills add proxyuserai/skills
```

### Manual installation

Skills must live in a folder whose name matches the `name` field in `SKILL.md` (here, `proxyuser`). Copy the file like this:

```bash
# Project-level (recommended) — works in Claude Code, Cursor, and Copilot
mkdir -p .agents/skills/proxyuser && cp SKILL.md .agents/skills/proxyuser/

# User-level (available across all your repos)
mkdir -p ~/.agents/skills/proxyuser && cp SKILL.md ~/.agents/skills/proxyuser/
```

Tool-specific paths also work:

| Tool | Project location | User location |
|------|------------------|---------------|
| Claude Code | `.claude/skills/proxyuser/SKILL.md` | `~/.claude/skills/proxyuser/SKILL.md` |
| Cursor | `.cursor/skills/proxyuser/SKILL.md` | `~/.cursor/skills/proxyuser/SKILL.md` |
| Copilot (VS Code) | `.github/skills/proxyuser/SKILL.md` | `~/.copilot/skills/proxyuser/SKILL.md` |


## Quick Start

1. **Install the skill** (above).

2. **Ask your AI assistant** to create a scenario:
   > "Create a ProxyUser scenario for the sign-in flow on https://example.com/login"

   The skill handles signup itself: it asks for your email, sends a 6-digit code there, takes the code back from you, and saves the resulting API key to `~/.proxyuser/config.json` (mode `0600`). No dashboard required.

That's it. Subsequent runs reuse the saved key.

### Want to override the saved key?

Set `PROXYUSER_API_KEY` in your environment. If it's set, the skill uses it and ignores `~/.proxyuser/config.json`. Useful for CI, ephemeral environments, or running with a different organization's key.

## How is my key stored?

The skill writes a single file:

```
~/.proxyuser/config.json   (mode 0600, owner read/write only)
```

Shape:

```json
{
  "api_key": "sk_live_…",
  "organization_id": "org_…"
}
```

The key is never logged or echoed — when the agent has to mention it, it masks the value as `sk_live_••••<last4>`. To delete the key, remove the file. To rotate, run the signup flow again or paste a new key from `/cli-onboard`.

## Existing accounts

If you already have a ProxyUser account, the agentic signup will return HTTP 409 Conflict. The skill is built for this:

1. Agent: "You already have a ProxyUser account. Sign in at https://proxyuser.com/cli-onboard, copy the API key it gives you, and paste it here."
2. You: paste the `sk_live_…` token.
3. Agent: validates the key against `GET /api/v1/projects`, saves it to `~/.proxyuser/config.json` if it works, and continues with your task.

## What you can do without the dashboard

With this skill, your AI assistant can drive ProxyUser end-to-end. The dashboard is optional — the only browser steps are Stripe Checkout and Slack OAuth.

- **Investigate a failed run** — verdict, reasoning, final screenshot, trace, cost, and the rrweb recording
- **Snooze or acknowledge** — mute a scenario for N hours, pause project alerts for the day, or acknowledge a single failure
- **Upgrade your plan** — Stripe Checkout opens in your browser; the agent polls until the plan flips, then resumes
- **Connect Slack alerts** — Slack OAuth opens in your browser; the agent polls until connected and can send a test message
- **Invite teammates and manage roles** — invite by email, promote/demote, with last-owner protection
- **Edit the agent's per-app memory** — *"the sign-in button reads 'Continue', not 'Sign in'"*
- **Bulk-create scenarios** — describe several flows at once; the agent submits them in a single transactional call
- **Manage project environment variables** — test secrets like `STRIPE_TEST_KEY`; values are encrypted at rest and never returned
- **Rotate your ProxyUser API key** — new key, swap the local config, then revoke the old one (in that order)

Plus the original scenario authoring: create monitoring scenarios from code and re-run them against preview deploys via `POST /projects/:id/run_all`.

## Examples

Three flows the skill handles cleanly. Each shows the user prompt and the resulting API request body.

### Signup flow

> "Add a ProxyUser scenario for email/password signup on https://example.com/signup, ending on the dashboard."

```json
{
  "prompt": "User signs up with email and password, then sees the dashboard.",
  "url": "https://example.com/signup"
}
```

### Sign-in flow

> "Add a ProxyUser scenario for the sign-in flow on https://example.com/login."

```json
{
  "prompt": "User signs in with email and password, then sees the dashboard.",
  "url": "https://example.com/login"
}
```

### Billing-checkout flow

> "Add a ProxyUser scenario for checkout starting at https://example.com/products."

```json
{
  "prompt": "User adds an item to the cart, proceeds to checkout, completes a Stripe test payment, and sees the order confirmation page.",
  "url": "https://example.com/products"
}
```

In all cases the skill sends only `{ prompt, url }`. The first run auto-enqueues; results appear in the dashboard within ~30 seconds.

## Usage

### Slash command

In tools that expose skills as slash commands (Claude Code, Cursor), invoke directly:
```
/proxyuser
```

### Natural language

> "Create scenarios for the new checkout feature I just implemented"

> "Run all ProxyUser scenarios against the preview URL at https://staging.example.com"

> "Why did the login scenario fail? Check the ProxyUser diagnosis"

> "Mute the login scenario for 2 hours while I fix this"

## Documentation

- [SKILL.md](SKILL.md) - Full skill instructions
- [examples/prompt-patterns.md](examples/prompt-patterns.md) - Scenario prompt templates
- [ProxyUser Docs](https://proxyuser.com/docs) - Complete API reference

## Errors

**401 Unauthorized** — Your saved key is invalid, revoked, or still in `pending` scope (signup wasn't verified). The agent will re-run signup or accept a fresh key pasted from https://proxyuser.com/cli-onboard.

**403 invalid_otp** — You typed the wrong 6-digit code. The response includes `attempts_remaining`; the agent surfaces the count and asks you to retry. After the limit, the agent restarts signup with a fresh email request.

**409 Conflict (account exists)** — The email you gave already has a ProxyUser account. The agent routes you to https://proxyuser.com/cli-onboard to copy a key, then validates and saves it. Original signup is not retried.

**410 Gone (otp_expired)** — The 6-digit code timed out (10-minute window). The agent restarts signup.

**429 Too Many Requests** — Rate limit hit. The agent surfaces the `Retry-After` header verbatim and does not auto-retry. Limits: signup is 5/min per IP and 3/hour per email; verify is 10/min per key; the rest of the API is 100/min per key.

**422 Unprocessable Entity** — The request had invalid data. The response body's `error.details` lists which fields failed and why. Example:

```json
{
  "error": {
    "code": "validation_failed",
    "message": "Validation failed: Url can't be blank",
    "details": { "url": ["can't be blank"] }
  }
}
```

Surface `error.details` to the user — don't paraphrase it.

**5xx / network failure** — Retry once. If it fails again, surface the request body so the user can re-run when the API recovers. The skill never retries silently in a loop.

**422 last_owner** — Returned by team-member ops when the target is the only owner. The skill stops and asks you to promote someone else first.

**422 self_revoke_forbidden** — Returned when you try to delete the API key the skill is currently authenticating with. The skill rotates instead: creates a new key, swaps `~/.proxyuser/config.json`, then deletes the old one.

**503 slack_not_configured** — Returned by Slack connect when the ProxyUser environment doesn't have Slack OAuth wired up. Not a user error — contact support.

**422 non_checkout_plan** — Returned by `/billing/checkout` for free plans (no checkout — use the upgrade flow) and enterprise plans (managed by sales). The skill explains and stops.

## License

MIT
