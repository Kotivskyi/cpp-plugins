---
name: marketplace-manage
description: Use when bumping plugin versions, validating marketplace consistency, syncing the registry, adding new plugins to the marketplace, or before pushing plugin changes to main. Triggers on "bump version", "publish plugin", "release", "validate marketplace", "sync registry", "add to marketplace", "pre-push check".
---

# Marketplace Management

Manage the cpp-plugins marketplace lifecycle: version bumping, validation, registry sync, and pre-push checks.

**Marketplace repo:** `github:Kotivskyi/cpp-plugins`
**Marketplace root:** the current working directory (or find `cpp-plugins` root via git)

## Key Files

| File | Purpose |
|------|---------|
| `.claude-plugin/marketplace.json` | Master registry — lists all plugins with name, description, version, source, category |
| `plugins/<name>/.claude-plugin/plugin.json` | Per-plugin metadata — name, description, version, author |
| `README.md` | Marketplace root README with plugins table |

**Critical rule from CLAUDE.md:** Bump the plugin version in BOTH `plugin.json` AND `marketplace.json` before pushing to main.

## Capabilities

### 1. Version Bump

When the user wants to bump a plugin version (or you detect plugin changes before a push):

1. Identify the plugin name. If ambiguous, ask.
2. Determine bump type: `patch` (default), `minor`, or `major`.
3. Read the current version from `plugins/<name>/.claude-plugin/plugin.json`.
4. Compute the new version using semver rules:
   - `patch`: 1.1.0 → 1.1.1
   - `minor`: 1.1.0 → 1.2.0
   - `major`: 1.1.0 → 2.0.0
5. Update the version in **both** files:
   - `plugins/<name>/.claude-plugin/plugin.json` → update `"version"` field
   - `.claude-plugin/marketplace.json` → update `"version"` in the matching plugin entry
6. Report: `<plugin-name>: <old> → <new>`

**If the plugin has no version field yet**, add `"version": "1.0.0"` to both files, then apply the bump.

### 2. Validate

Run a consistency check across all plugins. For each directory under `plugins/`:

#### Checks (run all, report all):

| # | Check | Severity |
|---|-------|----------|
| 1 | `plugins/<name>/.claude-plugin/plugin.json` exists | ERROR |
| 2 | `plugin.json` has `name` and `description` fields | ERROR |
| 3 | Plugin is registered in `marketplace.json` | ERROR |
| 4 | Directory name matches `plugin.json` `name` | ERROR |
| 5 | Directory name matches `marketplace.json` entry `name` | ERROR |
| 6 | `plugin.json` version matches `marketplace.json` version | ERROR |
| 7 | `plugin.json` description matches `marketplace.json` description | WARN |
| 8 | Plugin appears in README.md plugins table | WARN |
| 9 | Plugin has at least one skill in `skills/` | WARN |

#### Output format:

```
Marketplace Validation Report
=============================

nightwatch-health-audit
  ✓ plugin.json exists
  ✓ Required fields present
  ✓ Registered in marketplace.json
  ✓ Names match across files
  ✓ Versions match (1.1.0)
  ✓ Description in sync
  ✓ Listed in README
  ✓ Has skills (1)

plugin-writing-skill
  ✓ plugin.json exists
  ✓ Required fields present
  ✓ Registered in marketplace.json
  ✓ Names match across files
  ✗ Version mismatch: plugin.json=1.0.1, marketplace.json=1.0.0
  ⚠ Description differs
  ✓ Listed in README
  ✓ Has skills (1)

Summary: 2 plugins checked, 1 error, 1 warning
```

**If errors are found**, offer to fix them automatically.

### 3. Registry Sync

Detect and register unregistered plugins:

1. List all directories under `plugins/` that contain `.claude-plugin/plugin.json`
2. Compare against entries in `.claude-plugin/marketplace.json`
3. For each unregistered plugin:
   a. Read its `plugin.json` for name, description, version
   b. Ask the user for the category: `monitoring`, `development`, `productivity`, or `integration`
   c. Add an entry to `marketplace.json`:
      ```json
      {
        "name": "<name>",
        "description": "<from plugin.json>",
        "version": "<from plugin.json>",
        "source": "./plugins/<name>",
        "category": "<user-chosen>"
      }
      ```
   d. Add a row to the README plugins table
4. Report what was added

### 4. Pre-push Check

Run before pushing to main. Combines validation with change detection:

1. Run `git diff --name-only HEAD` (staged + unstaged) to find changed files
2. For each changed file under `plugins/<name>/`:
   - Check if the version in `plugin.json` was bumped compared to the last commit
   - If files changed but version didn't bump → **ERROR**: "Plugin `<name>` has changes but version was not bumped"
3. Run full validation (section 2)
4. If all clear: "Ready to push."
5. If errors found: list them and offer to fix (bump versions, sync registry)

## Workflow

When triggered, determine what the user needs:

- Explicit "bump version" / "release" → **Version Bump**
- Explicit "validate" / "check marketplace" → **Validate**
- Explicit "sync" / "add to marketplace" → **Registry Sync**
- About to push / "pre-push" → **Pre-push Check**
- Unclear → Run **Validate** first, then offer specific actions based on findings

## Common Mistakes

- **Bumping only one file** — ALWAYS update both `plugin.json` and `marketplace.json`
- **Forgetting the README** — every plugin must appear in the marketplace README table
- **Name mismatches** — directory name, `plugin.json` name, and `marketplace.json` name must all be identical
- **Not running validation after changes** — always validate before pushing
