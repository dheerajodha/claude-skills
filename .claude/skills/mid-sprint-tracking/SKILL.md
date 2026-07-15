---
name: mid-sprint-tracking
description: Track and report on stories added to sprints after they've started. Use when adding issues to an active sprint mid-cycle, or when generating sprint reports that surface unplanned work.
---

# Mid-Sprint Tracking Skill

Track stories added to the Conforma team sprint after it has started. Provides structured tagging for mid-sprint additions and end-of-sprint reporting.

## Prerequisites

Requires a Jira MCP server (e.g. mcp-atlassian, plugin:atlassian:atlassian, or any MCP that provides Jira issue, sprint, and comment operations). The `/sprint-report --post` flag additionally requires a Slack MCP server.

If no Jira MCP is available, users can still follow the comment convention manually and search with JQL: `comment ~ "MID-SPRINT-ADD"`

## Comment Convention

When a story is added to a sprint after the sprint has started, the person adding it leaves a Jira comment with this machine-parseable tag:

```text
[MID-SPRINT-ADD] reason: <brief explanation>
```

Examples:
- `[MID-SPRINT-ADD] reason: customer escalation from KFLUXSPRT-7872`
- `[MID-SPRINT-ADD] reason: unplanned dependency for EC-1881`
- `[MID-SPRINT-ADD] reason: pulled from backlog, team has capacity`

The reason must be a single line — the report parser extracts everything after `reason:` on that line. The "who" and "when" are captured automatically by the Jira comment metadata (author + timestamp).

## Configuration

- **Board:** Conforma Team (ID: `10131`)
- **Project:** EC (Conforma)
- **Slack channel:** `#team-conforma`

## Commands

- `/mid-sprint-add` — Add a structured mid-sprint comment to a Jira issue and optionally move it into the active sprint
- `/sprint-report` — Generate a report of all mid-sprint additions for the current or most recent sprint
