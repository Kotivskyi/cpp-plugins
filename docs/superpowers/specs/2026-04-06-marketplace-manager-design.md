# Marketplace Manager Plugin — Design Spec

**Date:** 2026-04-06
**Status:** Approved
**Plugin name:** `marketplace-manager`
**Marketplace:** cpp-plugins (`github:Kotivskyi/cpp-plugins`)

## Problem

Managing plugin versions and marketplace registry entries is manual and error-prone:
- Versions must be bumped in both `plugin.json` and `marketplace.json` before pushing
- New plugins must be registered in the marketplace index and README
- Name/description mismatches between files go unnoticed
- Desktop/web app users have no easy way to pull marketplace updates (no CLI `/plugin marketplace update`)

## Solution

A new plugin with two skills:

### Skill 1: `marketplace-manage` (model-invoked)

Automatically triggers when Claude detects marketplace maintenance work.

**Trigger conditions:**
- User says "bump version", "publish plugin", "release", "validate marketplace", "sync registry"
- User is about to push changes to a plugin
- User adds a new plugin directory under `plugins/`

**Capabilities:**

#### Version Bumping
1. Accept a plugin name and bump type (patch/minor/major, default: patch)
2. Read the current version from `plugins/<name>/.claude-plugin/plugin.json`
3. Compute the new semver version
4. Update version in both:
   - `plugins/<name>/.claude-plugin/plugin.json`
   - `.claude-plugin/marketplace.json` (matching entry by name)
5. Report the change: `nightwatch-health-audit: 1.1.0 → 1.2.0`

#### Validation
Check all plugins for consistency. For each plugin directory under `plugins/`:
1. `plugin.json` exists and has required fields (`name`, `description`)
2. Plugin is registered in `marketplace.json` (has an entry with matching `name`)
3. Name matches across: directory name, `plugin.json` name, `marketplace.json` name
4. Description in `marketplace.json` matches `plugin.json` (warn if diverged)
5. Version in `marketplace.json` matches `plugin.json` (error if diverged)
6. Plugin appears in the README plugins table
7. Report all findings: passes, warnings, errors

#### Registry Sync
1. Scan `plugins/` for directories that have `.claude-plugin/plugin.json` but no entry in `marketplace.json`
2. For each unregistered plugin, add an entry to `marketplace.json` with name, description, version, source path
3. Prompt for category (monitoring/development/productivity/integration)
4. Add the plugin to the README plugins table

#### Pre-push Check
Combines validation + version check:
1. Run full validation
2. Check if any plugin files changed (via `git diff`) without a version bump
3. If changes detected without bump, warn and offer to bump

### Skill 2: `/cpp-update` (user-invoked slash command)

A slash command for users who have the cpp-plugins marketplace installed and want to pull the latest updates. Designed for Desktop/web app users who can't run CLI plugin commands.

**What it does:**

1. Find the cpp-plugins marketplace cache directory:
   - Check `~/.claude/plugins/cache/` for a directory matching `cpp-plugins` or `Kotivskyi-cpp-plugins`
   - If not found, tell user to install the marketplace first
2. Run `git -C <cache-dir> pull origin main` to fetch latest
3. Parse `marketplace.json` to list all plugins and their versions
4. Compare against installed plugin versions in the cache
5. Report what changed:
   ```
   cpp-plugins marketplace updated.
   
   Updated plugins:
   - nightwatch-health-audit: 1.0.0 → 1.1.0
   
   Up to date:
   - plugin-writing-skill: 1.0.0
   
   Restart Claude Code to pick up changes.
   ```
6. If git pull fails (no git, network error), provide clear error message

**Edge cases:**
- Marketplace not installed → guide user to install it
- No internet → report fetch failure clearly
- Cache directory structure varies → search common patterns

## File Structure

```
plugins/marketplace-manager/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    ├── marketplace-manage/
    │   └── SKILL.md
    └── cpp-update/
        └── SKILL.md
```

## Out of Scope

- Publishing to external registries (npm, etc.)
- Managing other marketplaces (this is scoped to cpp-plugins)
- Automated CI/CD integration
- Plugin dependency management
