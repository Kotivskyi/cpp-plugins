---
name: nightwatch-health-audit
description: Use when the user asks to check application health, review bugs, audit exceptions, or generate a bug report from Laravel Nightwatch. Also use when connecting to Nightwatch MCP for the first time in a session to gather issues, or when user says "check nightwatch", "what's broken", "show me errors", "bug report". Automatically creates Linear issues for unregistered bugs.
---

# Nightwatch Health Audit

## Overview

Systematically pull all issue categories from Laravel Nightwatch MCP, enrich with occurrence counts, compile into a prioritized bug report, and **create Linear issues for any bugs not already tracked**. The goal is a single document that gives a complete picture of application health with all bugs flowing into Linear for tracking.

## When to Use

- User asks to check application health or generate a bug report
- User wants to know what's broken or needs attention
- User says "check nightwatch", "audit errors", "what are the problems"
- After connecting Nightwatch MCP (`/mcp` -> nightwatch)

## Workflow

```dot
digraph audit_flow {
    rankdir=TB;
    node [shape=box];

    discover [label="1. Discover application\nlist_applications"];
    parallel_fetch [label="2. Fetch ALL issue types in parallel\nexception, job, command,\nscheduled_task, route"];
    paginate [label="3. Paginate if next_cursor present\nRepeat until all pages fetched"];
    enrich [label="4. Enrich top issues with get_issue\nGet occurrence counts, stack traces"];
    compile [label="5. Compile report\nSort exceptions by count, flag gaps"];
    patterns [label="6. Identify patterns\nGroup related issues, find root causes"];
    linear_dedup [label="7. Deduplicate against Linear\nSearch existing issues by title/exception"];
    linear_create [label="8. Create Linear issues\nFor unregistered bugs only"];

    discover -> parallel_fetch;
    parallel_fetch -> paginate;
    paginate -> enrich;
    enrich -> compile;
    compile -> patterns;
    patterns -> linear_dedup;
    linear_dedup -> linear_create;
}
```

## Step-by-Step

### 1. Discover the Application

```
mcp__nightwatch__list_applications  # optionally with query filter
```

If multiple apps match, ask the user which one. Remember `application_id` for the session.

### 2. Fetch All Issue Types in Parallel

Fire ALL of these simultaneously - they are independent:

```
mcp__nightwatch__list_issues(application_id, type: "exception")
mcp__nightwatch__list_issues(application_id, type: "job")
mcp__nightwatch__list_issues(application_id, type: "command")
mcp__nightwatch__list_issues(application_id, type: "scheduled_task")
mcp__nightwatch__list_issues(application_id, type: "route")
```

**Key:** Use parallel tool calls. This is 5 independent requests - never run them sequentially.

### 3. Paginate Through All Results

If any response includes `next_cursor`, fetch the next page. Exceptions often have 25+ issues across multiple pages. Don't stop at the first page.

### 4. Enrich Key Issues

Call `mcp__nightwatch__get_issue` on the most important issues (batch in parallel, up to ~15 at a time):

- **All recent issues** (last 24-48 hours) - they're actively happening
- **Issues that appear to recur** (seen across a date range)
- **Issues in different categories** (get a representative sample)

What you're looking for from `get_issue`:
- **Occurrence counts** (last 24h, 7d, 30d) - this is the priority signal
- **Stack traces** - to understand root cause and group related issues
- **Code context** - to write actionable descriptions
- **Deploy info** - which environment is affected
- **Nightwatch URL** - to link from Linear issues

### 5. Compile the Report

Structure the output with these sections:

#### Exceptions (sorted by 30-day count descending)

| # | Ref | Exception | Count (30d) | Count (7d) | Last Seen | Location | Linear |
|---|-----|-----------|-------------|------------|-----------|----------|--------|

This is the most important section. Sort by count - highest first.

#### Failed Jobs

If no job issues found, explicitly state "No failed/slow jobs detected" BUT cross-reference with exceptions - jobs may be failing silently if exceptions are caught inside job handlers.

#### Failed Commands

List command-related exceptions separately. Include the command name, arguments, and error.

#### Scheduled Tasks

Report status. No issues = good news worth stating explicitly.

#### Outgoing Request Problems

Filter for HTTP client exceptions, timeouts, connection errors. These indicate external service issues.

### 6. Identify Patterns

After listing individual issues, add a "Key Patterns & Recommendations" section:

- **Group related issues** - e.g., 15 "undefined column" errors may all stem from one incomplete migration
- **Flag silent failures** - jobs that catch exceptions internally but still fail
- **Highlight severity** - Critical (data loss, broken core features) vs Medium (degraded UX) vs Low (transient/cosmetic)
- **Note environment** - staging-only vs production issues

### 7. Deduplicate Against Linear

Before creating any Linear issues, search for existing ones to avoid duplicates.

For each Nightwatch issue worth tracking, search Linear:

```
mcp__linear__list_issues(team: "<team>", query: "<exception class or issue title>")
```

**Matching strategy:**
- Search by the exception class name (e.g. `QueryException`, `ModelNotFoundException`)
- Also search by a distinctive fragment of the error message
- If an existing Linear issue's title contains the same exception class AND similar location, consider it a match
- When in doubt, it's better to skip creation than create a duplicate

**Run searches in parallel** - batch up to 10 at a time.

Compile two lists:
- **Already tracked:** Nightwatch issues that have a matching Linear issue (note the Linear identifier)
- **Unregistered:** Nightwatch issues with no matching Linear issue

### 8. Create Linear Issues for Unregistered Bugs

For each unregistered bug, create a Linear issue using `mcp__linear__save_issue`.

**Priority mapping:**
| Nightwatch Signal | Linear Priority |
|-------------------|-----------------|
| Critical: data loss, auth broken, payments failing | 1 (Urgent) |
| High: 100+ occurrences/30d, core feature broken | 2 (High) |
| Medium: 10-100 occurrences/30d, degraded UX | 3 (Normal) |
| Low: <10 occurrences/30d, cosmetic, transient | 4 (Low) |

**Issue creation parameters:**

```
mcp__linear__save_issue(
  team: "<team name>",
  title: "[NW-<ref>] <ExceptionClass> in <Location>",
  description: <see template below>,
  priority: <mapped priority>,
  labels: ["bug", "nightwatch"]
)
```

**Create issues in parallel** - batch up to 5 at a time.

#### Linear Issue Description Template

```markdown
## Bug Report (from Nightwatch)

**Exception:** `<Full\Exception\Class\Name>`
**Location:** `<file>:<line>` in `<method>()`
**Environment:** <production/staging>
**First seen:** <date>
**Last seen:** <date>

---

## Occurrence Stats

| Period | Count |
|--------|-------|
| Last 24h | <n> |
| Last 7d | <n> |
| Last 30d | <n> |

## Error Message

```
<The actual error message text>
```

## Stack Trace (top frames)

```
<Top 5-8 frames of the stack trace>
```

## Code Context

<If available from get_issue, include the code snippet around the error line>

## Reproduction Context

- **Route/Job/Command:** <what triggered it>
- **HTTP Method & URL:** <if route-related>
- **User context:** <if available>

## Related Issues

<List any Nightwatch issues that share the same root cause or pattern>

---

> Auto-generated from [Nightwatch #<ref>](<nightwatch_url>)
```

**Key rules for descriptions:**
- Always include the Nightwatch link at the bottom
- Always include occurrence stats - this is how engineers prioritize
- Include the actual error message verbatim, not paraphrased
- Stack trace should show the top frames (app code, not framework boilerplate)
- If multiple Nightwatch issues share a root cause, mention them all in one Linear issue

## Common Mistakes

- **Stopping at page 1** - Always check for `next_cursor` and paginate
- **Not fetching details** - The list only shows titles; `get_issue` gives counts and stack traces needed for prioritization
- **Sequential API calls** - All issue type fetches and detail fetches should be parallel
- **Ignoring "no issues" categories** - Explicitly report clean categories; "no failed jobs" is valuable information
- **Listing without analysis** - Raw lists aren't actionable. Group related issues and identify root causes
- **Missing cross-references** - An exception in a job handler means the job is broken even if Nightwatch shows zero job issues
- **Creating duplicate Linear issues** - Always search Linear first. Match on exception class + location
- **Vague Linear descriptions** - Every Linear issue must have occurrence stats, error message, stack trace, and Nightwatch link
- **Creating one Linear issue per Nightwatch issue blindly** - Group related issues that share a root cause into a single Linear issue
- **Skipping the "bug" label** - Always apply the "bug" label so issues are filterable
