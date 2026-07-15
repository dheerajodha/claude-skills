---
description: Generate a report of mid-sprint story additions for the current or most recent sprint, optionally post to Slack
argument-hint: [--post [#channel ...]] [--sprint <name>]
---

# Sprint Report — Mid-Sprint Additions

Query all issues in a sprint, find `[MID-SPRINT-ADD]` comments, and generate a report showing which stories were added after the sprint started, by whom, when, and why.

## Prerequisites

This command requires access to Jira via an MCP server (e.g. mcp-atlassian, plugin:atlassian:atlassian, or similar). The `--post` flag additionally requires a Slack MCP server.

If no Jira MCP tools are available, stop and tell the user:

> This command requires a Jira MCP server to be configured.
> See https://github.com/mcp-atlassian/mcp-atlassian for setup instructions,
> or install the Atlassian plugin via `/install-plugin atlassian`.
>
> To manually find mid-sprint additions, search Jira with:
> ```text
> sprint = <sprint-id> AND comment ~ "MID-SPRINT-ADD"
> ```

## Instructions

Parse `$ARGUMENTS` for optional flags:
- `--post [#channel ...]` — Post the report to Slack. Defaults to `#team-conforma` if no channels specified. Can list one or more channels (e.g. `--post #team-conforma #conforma-leads`).
- `--sprint <name>` — Target a specific sprint by name (e.g. `--sprint "Conforma 26-09"`) instead of the current active sprint. Partial matches are fine (e.g. `--sprint 26-09`).

## Step 1: Identify the Sprint

1. Get all sprints from the Conforma Team board (ID: `10131`) — fetch active, then closed if needed.
2. If `--sprint` was provided, find sprints whose name contains the provided text (case-insensitive).
   - If exactly one match: use it.
   - If no matches: list available sprint names and ask the user to clarify.
   - If multiple matches: list the matching sprints (name, dates, state) and ask the user to pick one.
3. If no `--sprint` flag, use the active sprint. If multiple active sprints exist, list them and ask the user to pick. If none active, use the most recently closed sprint.
4. Record: sprint name, ID, start date, end date, goal.

## Step 2: Fetch All Sprint Issues

Get the total count of issues in the sprint to use as the denominator for percentages. Query all issues in the sprint and record:
- Total issue count
- Sum of all story points (`customfield_10028`) across the sprint (total points)

Paginate using `start_at` until all results are fetched — the Jira API returns at most 50 items per request. If any page request fails, note the report as potentially incomplete.

## Step 3: Find Mid-Sprint Additions

Use JQL to search for issues in the sprint that have mid-sprint addition comments:

```text
sprint = <sprint-id> AND comment ~ "MID-SPRINT-ADD"
```

Paginate if needed. For each matching issue, fetch the full issue details including comments and record:
- Issue key, summary, status, assignee
- Issue type, priority, story points (`customfield_10028`)

## Step 4: Extract Comment Data

For each matching issue, scan its comments for entries containing `MID-SPRINT-ADD`. Only include comments whose creation timestamp is **on or after the sprint start date** — issues may carry tags from previous sprints.

For each qualifying comment, extract:
- **Reason**: Everything after `reason:` on the `MID-SPRINT-ADD` line (single line only)
- **Author**: The comment author (from Jira metadata)
- **Date added**: The comment creation timestamp
- **Days after sprint start**: Calculate from the sprint start date

**Deduplication:** If an issue has multiple qualifying `MID-SPRINT-ADD` comments (e.g. different reasons), count the issue only once for metrics (addition count, percentage, unplanned points). Use the earliest qualifying comment as the primary entry. List additional reasons in a comma-separated format in the Reason column (e.g. `customer escalation, scope change`).

**Calculate points:** After deduplication, compute:
- **Unplanned points**: sum of story points for mid-sprint addition issues only
- **Planned points**: total points (from Step 2) minus unplanned points
- **Percentage**: if total issue count is zero, report 0%

Note: Jira's API may strip square brackets from comment bodies. Match on `MID-SPRINT-ADD` rather than `[MID-SPRINT-ADD]`.

## Step 5: Generate the Report

Build a markdown report. Sanitize all Jira-derived values (summaries, authors, reasons) for markdown table rendering — escape pipe characters and strip newlines.

**Reason categorization:** Group free-text reasons into categories using keyword matching: reasons containing "escalation" or "customer" map to "Customer escalation"; "dependency" or "blocker" map to "Unplanned dependency"; "capacity" or "backlog" map to "Backlog pull"; anything else maps to "Other". Each issue's points count toward one category only.

```markdown
# Mid-Sprint Additions Report — {Sprint Name}

**Sprint:** {name} ({start_date} → {end_date})
**Goal:** {sprint goal}
**Total issues in sprint:** {n}
**Mid-sprint additions:** {n} ({percentage}%)

## Additions

| # | Issue | Summary | Added By | Date Added | Days In | Reason | Category | Points |
|---|-------|---------|----------|------------|---------|--------|----------|--------|
| 1 | EC-1234 | Fix the thing | Rob Nester | 2026-07-18 | +3 | customer escalation | Customer escalation | 2 |

(Escape `|` in cell values with `\|` and strip newlines so each row stays on one line.)

## Summary by Reason Category

Categories are assigned per the keyword mapping above. Use these exact category names:
- **Customer escalation** — reason contains "escalation" or "customer"
- **Unplanned dependency** — reason contains "dependency" or "blocker"
- **Backlog pull** — reason contains "capacity" or "backlog"
- **Other** — anything that doesn't match the above

| Category | Count | Total Points |
|----------|-------|--------------|
| Customer escalation | 2 | 5 |
| Unplanned dependency | 1 | 3 |
| Backlog pull | 1 | 1 |

## Observations

{Brief analysis: percentage of unplanned work, most common reason category, total unplanned story points vs total planned story points}
```

If no qualifying `[MID-SPRINT-ADD]` comments are found, distinguish between two cases:

**No tags at all** (JQL returned zero issues):

```markdown
# Mid-Sprint Additions Report — {Sprint Name}

No [MID-SPRINT-ADD] tags found in any sprint issues.

This could mean:
- No stories were added mid-sprint
- Stories were added but the [MID-SPRINT-ADD] comment convention wasn't used
- Use /mid-sprint-add <ISSUE-KEY> to tag future mid-sprint additions
```

**Tags exist but all predate the sprint start** (JQL found issues but all comments were filtered out by the date check):

```markdown
# Mid-Sprint Additions Report — {Sprint Name}

No qualifying [MID-SPRINT-ADD] tags found for this sprint.

{n} issue(s) have [MID-SPRINT-ADD] comments, but all predate the sprint start ({start_date}).
These may be from a previous sprint.
```

## Step 6: Save the Report

Slugify the sprint name for the filename: lowercase, replace spaces and special characters with hyphens, collapse multiple hyphens. For example, `Conforma 26-10` becomes `conforma-26-10`.

Write the report to `.claude/reports/sprint-report-{slugified-name}.md`.

```bash
mkdir -p .claude/reports
```

## Step 7: Post to Slack (if --post)

The report must be saved (Step 6) before posting.

If `--post` was specified and a Slack MCP server is available:

1. Determine target channels: use any channels listed after `--post`, or default to `#team-conforma` if none specified.
2. If the report exceeds Slack's message limit (~4000 chars), post a summary (header + additions table) and note the full report file path.
3. For each channel, look up the channel ID by name and post the report. Track success/failure per channel independently.
4. Confirm which channels received the report and note any failures.

If `--post` was specified but no Slack MCP is available, tell the user:
> No Slack MCP server is configured. Report saved locally — copy and paste it into the appropriate channel manually.

If `--post` was not specified, remind the user:
> Run `/sprint-report --post` to share this report in #team-conforma.

## Step 8: Present Summary

```text
Sprint report generated:
- Sprint: {name}
- Total issues: {n}
- Mid-sprint additions: {n} ({percentage}%)
- Unplanned points: {n}
- Report saved to: .claude/reports/sprint-report-{slugified-name}.md
{- Posted to: #channel1, #channel2 (if --post)}
```
