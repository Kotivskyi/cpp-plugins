---
name: plugin-writing-skill
description: Use when creating a new Cowork / Claude Code plugin from scratch, packaging skills with MCP servers, or converting an existing skill into a distributable plugin. Also use when user says "create a plugin", "make a plugin", "package this as a plugin".
---

# Writing Cowork / Claude Code Plugins

## Overview

A plugin is a distributable package that bundles **skills**, **MCP server configs**, **commands**, **hooks**, and **agents** for Claude Code (Cowork). Every plugin MUST be created inside a marketplace so it can be discovered and installed. The user's marketplace is **cpp-plugins** at `github:Kotivskyi/cpp-plugins`.

## Marketplace Structure

A marketplace is a git repo that indexes multiple plugins:

```
cpp-plugins/                          # Marketplace root
├── .claude-plugin/
│   └── marketplace.json              # REQUIRED: master index of all plugins
├── plugins/
│   ├── plugin-a/                     # Each plugin in its own directory
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── .mcp.json                 # Optional: MCP servers
│   │   ├── skills/
│   │   │   └── skill-name/
│   │   │       └── SKILL.md
│   │   └── README.md
│   └── plugin-b/
│       └── ...
├── README.md
└── LICENSE
```

## Publishing Checklist (MANDATORY)

Every new plugin requires BOTH steps — a plugin without a marketplace entry is invisible:

1. **Create the plugin** under `plugins/<plugin-name>/` with all required files
2. **Register in marketplace.json** — add an entry to `.claude-plugin/marketplace.json` `plugins` array
3. **Update marketplace README.md** — add the plugin to the plugins table
4. **Commit and push** to the marketplace repo

### marketplace.json Entry Format

```json
{
  "name": "plugin-name",
  "description": "What the plugin does — must match plugin.json description",
  "source": "./plugins/plugin-name",
  "category": "monitoring|development|productivity|integration"
}
```

**Critical rules:**
- `name` MUST match the directory name under `plugins/`
- `name` MUST match `name` in the plugin's `.claude-plugin/plugin.json`
- `source` MUST be a relative path: `"./plugins/<plugin-name>"`
- `description` should match or closely align with the plugin.json description

### Full marketplace.json Example

```json
{
  "name": "cpp-plugins",
  "description": "Claude Code plugins by Vitalii Kotivskyi",
  "owner": {
    "name": "Vitalii Kotivskyi",
    "email": "vitalii@kotivskyi.dev"
  },
  "plugins": [
    {
      "name": "my-new-plugin",
      "description": "What this plugin does",
      "source": "./plugins/my-new-plugin",
      "category": "development"
    }
  ]
}
```

## Plugin Directory Structure

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Required: plugin metadata
├── .mcp.json                # Optional: MCP server declarations
├── skills/
│   └── skill-name/
│       └── SKILL.md         # Skill content (agentskills.io format)
├── commands/
│   └── command-name.md      # Legacy slash commands (prefer skills/)
├── hooks/
│   └── hooks.json           # Event-driven hooks
├── agents/
│   └── agent-name.md        # Agent definitions
├── README.md                # Setup instructions (required if MCP)
└── LICENSE                  # License file
```

## plugin.json (Required)

Minimal metadata file inside `.claude-plugin/`:

```json
{
  "name": "plugin-name",
  "description": "What the plugin does — shown in plugin listings",
  "version": "1.0.0",
  "author": {
    "name": "Author Name",
    "email": "optional@email.com"
  },
  "homepage": "https://github.com/...",
  "repository": "https://github.com/...",
  "license": "MIT",
  "keywords": ["tag1", "tag2"]
}
```

Only `name` and `description` are required. Keep description under 300 chars.

## .mcp.json (MCP Server Config)

Declares MCP servers the plugin needs. Placed at plugin root. Three formats:

### HTTP MCP (remote API)

```json
{
  "server-name": {
    "type": "http",
    "url": "https://mcp.service.com/api"
  }
}
```

### HTTP with auth headers

```json
{
  "server-name": {
    "type": "http",
    "url": "${SERVICE_MCP_URL}",
    "headers": {
      "Authorization": "Bearer ${SERVICE_API_TOKEN}"
    }
  }
}
```

### Stdio MCP (local process)

```json
{
  "server-name": {
    "command": "npx",
    "args": ["-y", "@some/mcp-server"],
    "env": {
      "API_KEY": "${API_KEY}"
    }
  }
}
```

### Docker-based MCP

```json
{
  "server-name": {
    "command": "docker",
    "args": [
      "run", "-i", "--rm",
      "-e", "TOKEN=${TOKEN}",
      "image:tag"
    ]
  }
}
```

**Key rules:**
- Use `${ENV_VAR}` syntax for secrets — never hardcode tokens
- Use `${CLAUDE_PLUGIN_ROOT}` for paths relative to the plugin
- Server names become the MCP tool prefix: `mcp__server-name__tool_name`

## Skills (SKILL.md)

Follow the [agentskills.io specification](https://agentskills.io/specification):

```yaml
---
name: skill-name
description: Use when [triggering conditions]
---

# Skill Name

Content here...
```

**Frontmatter fields:**
| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Lowercase + hyphens, max 64 chars, must match directory name |
| `description` | Yes | Max 1024 chars, start with "Use when..." |
| `version` | No | Semver |
| `license` | No | License reference |
| `compatibility` | No | Environment requirements |
| `allowed-tools` | No | Space-delimited pre-approved tools |
| `argument-hint` | No | For user-invoked skills (slash commands) |

**Model-invoked vs user-invoked:**
- Model-invoked: activates automatically based on description match
- User-invoked: has `argument-hint`, invoked as `/skill-name`

## Hooks (hooks.json)

```json
{
  "version": 1,
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/scripts/init.sh\"",
            "async": false
          }
        ]
      }
    ]
  }
}
```

Reference in plugin.json: `"hooks": "./hooks/hooks.json"`

## README Best Practices

Every plugin with MCP servers needs a README covering:

1. **What it does** — one paragraph
2. **Installation** — how to install the plugin
3. **MCP setup** — step-by-step for each MCP server:
   - Where to get API tokens
   - Which env vars to set (`export` or `settings.json`)
   - How to verify connection (`/mcp`)
4. **Usage** — example prompts that trigger the skill
5. **Troubleshooting** — common connection issues

## Quick Reference

| What | Where | Format |
|------|-------|--------|
| Plugin identity | `.claude-plugin/plugin.json` | JSON |
| MCP servers | `.mcp.json` (root) | JSON |
| Skills | `skills/<name>/SKILL.md` | YAML frontmatter + Markdown |
| Slash commands | `skills/<name>/SKILL.md` with `argument-hint` | YAML frontmatter + Markdown |
| Legacy commands | `commands/<name>.md` | Markdown |
| Hooks | `hooks/hooks.json` | JSON |
| Agents | `agents/<name>.md` | Markdown |

## Common Mistakes

- **Plugin not in marketplace.json** — plugin exists but nobody can find or install it. ALWAYS add to `.claude-plugin/marketplace.json`
- **Name mismatch** — directory name, plugin.json `name`, marketplace.json `name`, and SKILL.md frontmatter `name` must all be consistent
- **Missing `.claude-plugin/plugin.json`** — plugin won't be recognized
- **Hardcoded secrets in `.mcp.json`** — always use `${ENV_VAR}` placeholders
- **Wrong skill directory name** — must match the `name` field in SKILL.md frontmatter
- **Forgetting README with MCP setup** — users won't know which tokens to configure
- **Using `commands/` instead of `skills/`** — commands is legacy, prefer skills directory
- **Forgetting to update marketplace README** — the plugins table in the marketplace root README must list all plugins
- **Not pushing after adding plugin** — the marketplace is a git repo; changes must be committed and pushed to be available
