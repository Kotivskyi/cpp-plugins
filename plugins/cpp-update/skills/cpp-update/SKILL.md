---
name: cpp-update
description: Pull the latest cpp-plugins marketplace and update installed plugins. Use when user types /cpp-update or asks to update their cpp-plugins marketplace.
argument-hint: "[--check]"
---

# Update cpp-plugins Marketplace

Pull the latest version of the cpp-plugins marketplace and report what changed. Designed for Claude Code Desktop and web app users who don't have access to CLI plugin commands.

## Steps

### 1. Locate the Marketplace Cache

Search for the cpp-plugins marketplace in the local plugin cache:

```bash
find ~/.claude/plugins/cache/ -maxdepth 2 -name "marketplace.json" -exec grep -l "cpp-plugins" {} \; 2>/dev/null
```

The cache directory is typically at one of:
- `~/.claude/plugins/cache/cpp-plugins/`
- `~/.claude/plugins/cache/Kotivskyi-cpp-plugins/`
- `~/.claude/plugins/cache/github-Kotivskyi-cpp-plugins/`

**If not found:**
> The cpp-plugins marketplace isn't installed yet. To install it:
>
> **In Claude Code CLI:**
> ```
> /install-marketplace github:Kotivskyi/cpp-plugins
> ```
>
> **In Claude Code Desktop/Web:**
> Go to Settings → Plugins → Add Marketplace → enter `github:Kotivskyi/cpp-plugins`

Stop here if not installed.

### 2. Capture Current State

Before pulling, read the current `marketplace.json` to snapshot installed versions:

```bash
cat <cache-dir>/marketplace.json
```

Parse and store each plugin's name and version.

### 3. Pull Latest

```bash
git -C <cache-dir> pull origin main
```

**If this fails:**
- No git: "The cache directory isn't a git repo. Try removing `<cache-dir>` and reinstalling the marketplace."
- Network error: "Couldn't reach GitHub. Check your internet connection and try again."
- Merge conflict: "The local cache has conflicts. Try removing `<cache-dir>` and reinstalling the marketplace."

### 4. Read Updated State

Read the updated `marketplace.json` and each plugin's `plugin.json` for their versions.

### 5. Compare and Report

Compare before/after versions for each plugin:

```
cpp-plugins marketplace updated ✓

Changes:
  nightwatch-health-audit: 1.0.0 → 1.1.0

Up to date:
  plugin-writing-skill: 1.0.0
  marketplace-manager: 1.0.0

New plugins available:
  marketplace-manager: 1.0.0 (not installed)
```

**Categories:**
- **Changes** — version increased
- **Up to date** — same version
- **New plugins available** — in marketplace but not installed locally
- **Removed** — was installed but no longer in marketplace

### 6. Post-update Instructions

After reporting changes:

> **To apply updates:** Restart Claude Code (Cmd+Shift+P → "Reload" in Desktop, or close and reopen).
>
> **To install new plugins:**
> ```
> /install <plugin-name>@cpp-plugins
> ```

### `--check` Flag

If the user passes `--check`, only report what would change without actually pulling:

```bash
git -C <cache-dir> fetch origin main
git -C <cache-dir> diff HEAD origin/main -- .claude-plugin/marketplace.json
```

Report differences without applying them.

## Error Handling

| Situation | Response |
|-----------|----------|
| Marketplace not installed | Guide to install |
| No git in cache dir | Suggest remove + reinstall |
| Network unreachable | Clear error, suggest retry |
| Cache dir has local changes | Warn, offer `git reset --hard origin/main` with confirmation |
| No changes | "cpp-plugins marketplace is already up to date." |
