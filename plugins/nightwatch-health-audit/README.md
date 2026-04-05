# nightwatch-health-audit

Laravel Nightwatch health auditing with automatic Linear issue creation.

## What it does

- Pulls all issue categories from Nightwatch (exceptions, failed jobs, commands, scheduled tasks, routes)
- Enriches top issues with occurrence counts, stack traces, and code context
- Compiles a prioritized bug report sorted by severity
- Deduplicates against existing Linear issues
- Creates Linear issues for any unregistered bugs with full context

## MCP Setup

This plugin configures two MCP servers. You need to provide credentials for them.

### Nightwatch MCP

1. Log in to [Laravel Nightwatch](https://nightwatch.laravel.com/)
2. Go to **Settings** > **API Tokens** and create a new token
3. Copy the MCP URL from the Nightwatch dashboard
4. Set environment variables in `~/.claude/settings.json`:

```json
{
  "env": {
    "NIGHTWATCH_MCP_URL": "https://nightwatch.laravel.com/mcp",
    "NIGHTWATCH_API_TOKEN": "your-nightwatch-api-token"
  }
}
```

Or export in your shell profile:

```bash
export NIGHTWATCH_MCP_URL="https://nightwatch.laravel.com/mcp"
export NIGHTWATCH_API_TOKEN="your-nightwatch-api-token"
```

### Linear MCP

Linear uses OAuth authentication. No API token needed — you'll be prompted to authenticate in-browser on first use.

### Verify connections

In Claude Code, run `/mcp` — both `nightwatch` and `linear` should show as connected.

## Usage

Ask Claude:
- "Check nightwatch"
- "What's broken?"
- "Generate a bug report"
- "Audit application health"
