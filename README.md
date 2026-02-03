# ProxyUser Skills

Agent Skills for [ProxyUser](https://proxyuser.com) - AI-assisted E2E testing.

Describe tests in plain English. AI executes them as a human would.

## Installation

### Using skills CLI

```bash
npx skills add proxyuserai/skills
```

### Manual installation

Copy the `SKILL.md` file to your project:

```bash
# For Claude Code
mkdir -p .claude/skills && cp SKILL.md .claude/skills/

# For GitHub Copilot
mkdir -p .github/skills && cp SKILL.md .github/skills/

# For Cursor
mkdir -p .cursor/skills && cp SKILL.md .cursor/skills/
```

## Quick Start

1. **Get an API key** at https://proxyuser.com/settings/api_keys

2. **Set the environment variable**:
   ```bash
   export PROXYUSER_API_KEY="sk_live_xxx"
   ```

3. **Ask your AI assistant** to create a test:
   > "Create a ProxyUser test scenario for the login flow on https://myapp.com/login"

## What This Skill Enables

With this skill, your AI coding assistant can:

- **Create test scenarios** from code changes you're implementing
- **Run tests** against preview deployments or staging
- **Diagnose failures** using AI analysis and video recordings
- **Set up CI/CD** integration with GitHub Actions

## Example Prompts

> "Create E2E tests for the new checkout feature I just implemented"

> "Run all ProxyUser tests against the preview URL"

> "Why did the login test fail? Check the ProxyUser diagnosis"

> "Set up GitHub Actions to run ProxyUser tests on every PR"

## Documentation

- [SKILL.md](SKILL.md) - Full skill instructions
- [examples/prompt-patterns.md](examples/prompt-patterns.md) - Test scenario templates
- [ProxyUser Docs](https://proxyuser.com/docs) - Complete API reference

## License

MIT
