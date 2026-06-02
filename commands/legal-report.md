---
description: Generate an on-demand legal team productivity and overdue-tickets report. Pulls from Jira, prompts for off-Jira hours per lawyer, files a .docx to SharePoint.
argument-hint: [last_7_days | last_30_days | this_quarter | YYYY-MM-DD..YYYY-MM-DD]
---

# /legal-report

Generate a report of overdue tickets and team productivity over a date range. Designed for the head of legal -- prompts each lawyer's off-Jira hours manually so the report accounts for work that never made it into Jira.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Projects watched: `LEGAL`, `AIRD`
- Default range: last 30 calendar days
- n8n filing workflow ID: `VAKq9Bra0RA0SdCO`
- Report folder: `myPOS Legal/Reports/Productivity/`

## Input

`$ARGUMENTS` is one of:
- `last_7_days`
- `last_30_days` (default)
- `this_quarter`
- An explicit range `YYYY-MM-DD..YYYY-MM-DD`

If `$ARGUMENTS` is empty, default to `last_30_days`.

## Step 1: Resolve the date range

Compute `range_start` and `range_end` (both ISO dates). `range_end` is always today.

## Step 2: Invoke the report skill

```
Skill(skill: "legal-productivity-report")
```

Pass the resolved range. The skill handles Jira pulls, the per-lawyer off-Jira prompts, doc rendering, and SharePoint filing.

The skill writes the docx, asks you for off-Jira hours for each active lawyer one at a time, and returns the SharePoint URL when done.

## Step 3: Print the chat summary

See `skills/legal-productivity-report/SKILL.md` Step 6 for the format. Always include:

- Overdue total + percentage
- Top three slippers (key, assignee, days overdue)
- AT RISK / OVERLOADED / OK roll-up
- Count of unassigned tickets (routing gap)
- SharePoint URL of the filed docx
- SharePoint URL of the updated memory file

## Hard rules

- NEVER skip the off-Jira prompt for any active lawyer. Better to record `null` than to silently treat them as having zero hours.
- NEVER auto-publish the UNDER-UTILISED section. That section is private to the head of legal.
- NEVER run this on a schedule. It is on demand only, by design.
