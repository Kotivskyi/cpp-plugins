# cpp-plugins

A Claude Code (Cowork) plugin marketplace by Vitalii Kotivskyi.

## Plugins

| Plugin | Description |
|--------|-------------|
| [nightwatch-health-audit](plugins/nightwatch-health-audit/) | Laravel Nightwatch health auditing with automatic Linear issue creation |
| [plugin-writing-skill](plugins/plugin-writing-skill/) | Guide for creating Cowork / Claude Code plugins from scratch |

## Installation

### Add the marketplace

```bash
claude /install-marketplace github:vkotivskiy/cpp-plugins
```

### Install a plugin

```bash
claude /install nightwatch-health-audit@cpp-plugins
claude /install plugin-writing-skill@cpp-plugins
```

## Plugin Details

### nightwatch-health-audit

Pulls all issue categories from [Laravel Nightwatch](https://nightwatch.laravel.com/) (exceptions, failed jobs, commands, scheduled tasks, routes), compiles prioritized bug reports, and creates [Linear](https://linear.app/) issues for untracked bugs.

**Requires MCP setup:**
- `NIGHTWATCH_MCP_URL` — your Nightwatch MCP endpoint
- `NIGHTWATCH_API_TOKEN` — your Nightwatch API token
- Linear MCP — uses OAuth via `https://mcp.linear.app/mcp`

See [nightwatch-health-audit/README.md](plugins/nightwatch-health-audit/README.md) for full setup instructions.

### plugin-writing-skill

A reference skill that teaches Claude how to create new plugins. Covers:
- `plugin.json` schema
- `.mcp.json` formats (HTTP, stdio, Docker)
- Skills, hooks, agents, and commands
- README best practices for MCP-dependent plugins

## Structure

```
cpp-plugins/
├── .claude-plugin/
│   └── marketplace.json           # Marketplace index
├── plugins/
│   ├── nightwatch-health-audit/   # Nightwatch + Linear plugin
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── .mcp.json
│   │   └── skills/
│   │       └── nightwatch-health-audit/
│   │           └── SKILL.md
│   └── plugin-writing-skill/      # Plugin authoring guide
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── plugin-writing-skill/
│               └── SKILL.md
├── README.md
└── LICENSE
```

## License

MIT
