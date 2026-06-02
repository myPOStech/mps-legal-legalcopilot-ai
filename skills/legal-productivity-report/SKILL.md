---
name: legal-productivity-report
description: Generate an overdue-tickets and team-productivity report for the myPOS Legal team. Pulls from Jira (open tickets, due dates, recent transitions, worklogs), prompts each lawyer for off-Jira hours, and files a .docx report to SharePoint via the n8n filing workflow. Triggered on demand by /legal-report. Use when the user asks for an overdue report, productivity report, who is behind, who is overloaded, or any "how is the team doing" question.
tools: Read, Skill, mcp__atlassian__searchJiraIssuesUsingJql, mcp__atlassian__getJiraIssue, mcp__microsoft-365__sharepoint_read_file, mcp__n8n__execute_workflow
---

# Legal productivity report

Tell the head of legal who is overloaded, who is behind, and what slipped past its deadline. Account for the fact that most of the team also does work that never makes it into Jira.

This skill is **on demand only**. It is not scheduled. The lawyer runs `/legal-report` whenever they want a snapshot.

## Inputs

`$ARGUMENTS` may include a date range like `last_7_days`, `last_30_days` (default), `this_quarter`, or an explicit `YYYY-MM-DD..YYYY-MM-DD`. If absent, use the last 30 calendar days.

## Step 1: Pull from Jira

### 1a. Overdue tickets

```
JQL: project in (LEGAL, AIRD) AND duedate < now() AND statusCategory != Done ORDER BY duedate ASC
```

Fields: `summary`, `assignee`, `duedate`, `priority`, `labels`, `created`, `status`. Maximum 100 results.

For each overdue ticket compute:

- `days_overdue = today - duedate`
- `assignee_name` (display name; if unassigned, mark "UNASSIGNED" -- this is a routing failure, not a productivity issue, and gets called out separately in Step 5)

### 1b. Closed-in-range tickets per lawyer

```
JQL: project in (LEGAL, AIRD) AND resolved >= "{range_start}" AND assignee is not EMPTY ORDER BY resolved DESC
```

For each lawyer in the roster (from `knowledge/team-routing.md`), count:

- `tickets_closed_in_range`
- `avg_time_to_close_days` -- median is more useful than mean here; use the 50th percentile of `(resolved - created)`
- `tickets_currently_open` (statusCategory != Done assigned to them)
- `tickets_in_progress` (status = "In Progress")
- `tickets_overdue_assigned` (subset of 1a where they are the assignee)

### 1c. Recent transitions (a productivity proxy)

```
JQL: project in (LEGAL, AIRD) AND status changed during ("{range_start}", "{range_end}") ORDER BY updated DESC
```

For each lawyer, count distinct tickets where they triggered a transition. This catches work that doesn't end in Done (e.g., bouncing a ticket to Blocked, asking for clarification) but is still real work.

## Step 2: Prompt for off-Jira hours

For each active lawyer in `team-routing.md`, ask the chat user (presumed to be the head of legal or whoever is running the report):

```
{lawyer_name} closed {tickets_closed_in_range} tickets in the last {range_days} days.
How many hours of off-Jira legal work did they put in over the same window?
(e.g., calls, drafting that didn't get filed, internal advice)
Enter a number, or "skip" to leave blank.
```

Wait for an answer per lawyer. If the user types `skip` or hits enter without a number, record `off_jira_hours = null` and tag the lawyer's section in the report with a `(off-Jira hours not recorded)` note.

This is intentionally a manual entry: per Atanas's preference, the team doesn't want timesheets or calendar parsing. The report's job is to make it easy to fill in, not to invent the data.

> **Tip for the head of legal**: Before running this report, do a 5-minute scan of the team's calendars (private 1:1s, calls labelled "advice", "drafting", "review") to estimate the off-Jira number. The report will only be as honest as that input.

## Step 3: Compute productivity signals

For each lawyer, mark:

- `load = tickets_currently_open` -- raw number
- `velocity = tickets_closed_in_range / range_days` -- per-day rate
- `overdue_share = tickets_overdue_assigned / max(1, tickets_currently_open)` -- 0.0 to 1.0
- `composite_status` (heuristic, see below)

### Composite status heuristic

| Condition | Status |
|---|---|
| `overdue_share >= 0.30` | **AT RISK** -- slipping on deadlines |
| `load >= 12` AND `overdue_share < 0.30` | **OVERLOADED** -- volume is high but they are still hitting dates |
| `load <= 3` AND `velocity < 0.1` (less than 1 close per 10 days) AND `off_jira_hours == null OR off_jira_hours < 10` | **UNDER-UTILISED** -- but flag for human check before raising it |
| Otherwise | **OK** |

Do NOT publish the **UNDER-UTILISED** label in the body of the report. Surface it only in a separate "for the head of legal -- private review section" near the end of the doc. Off-Jira work is invisible and we should not accuse anyone of being slow on the visible-Jira evidence alone.

## Step 4: Build the docx

The report is a `.docx` so it lands cleanly in SharePoint and the head of legal can mark it up.

Sections (in this order):

1. **Title block.** `Legal team report -- {range_start} to {range_end}`. Generated `{today}` by the Copilot.
2. **Headline numbers.**
   - Total overdue tickets: N (X% of all open)
   - Tickets closed this window: N
   - Median time to close: N days
   - Unassigned tickets: N (call this out separately -- it is a routing bug)
3. **Per-lawyer table.** One row each, columns: Name | Open | In Progress | Overdue | Closed (range) | Median TTC | Off-Jira hours | Status. Status is the composite from Step 3 (excluding UNDER-UTILISED).
4. **Overdue tickets list.** Grouped by assignee. For each ticket: key, summary, days overdue, original priority. Up to 30 entries; if more, link out to the JQL.
5. **Patterns.** One paragraph each:
   - "What kind of matter is slipping most?" (group overdue by `matter:*` label, top 3)
   - "Where is volume concentrated?" (top 2 assignees by `load`)
   - "What does the assigned-during-the-window-only-to-someone-who-didn't-reply pattern look like?" -- catches tickets that bounced silently. Use the recent-transitions data from 1c.
6. **For the head of legal -- private review section.** Anything UNDER-UTILISED. Heading marked with a clear "PRIVATE -- review before sharing".
7. **Methodology footer.** One short paragraph explaining the inputs (Jira + manually entered off-Jira hours), the date range, and the caveat that off-Jira hours are self-reported.

Use the `docx` skill (`Skill(skill: "docx")`) to render. Filename:

```
LEGAL-REPORT_{range_start}_{range_end}_v1.docx
```

## Step 5: File to SharePoint

Invoke `sharepoint-filer` with:

- `case_id`: `LEGAL-REPORT_{range_start}_{range_end}` (synthetic case id)
- `case_folder`: `Reports/Productivity`
- `files`: the rendered `.docx`
- `memory_notes`: a one-paragraph summary -- "report generated, overall load, top overdue assignees, any AT RISK lawyers". This goes into `legal_copilot_memory.md` so trends can be tracked across reports without re-opening the docx.

The report file lands at:

```
myPOS Legal/Reports/Productivity/LEGAL-REPORT_{range_start}_{range_end}/LEGAL-REPORT_{range_start}_{range_end}_v1.docx
```

## Step 6: Print summary in chat

A short text summary (no markdown headers, no separator lines):

```
Legal team report for {range_start} -> {range_end}

Overdue: {N} tickets ({X}% of all open).
Top three slippers (by days overdue):
  {key} -- {assignee} -- {days_overdue}d overdue
  ... up to three lines

Status:
  AT RISK: {names}
  OVERLOADED: {names}
  OK: {everyone else}

Unassigned tickets needing routing: {N}

Filed: {sharepoint_url_of_docx}
Memory updated: {memory_file_url}
```

---

## Hard rules

- NEVER label a lawyer UNDER-UTILISED in the shareable body of the report. That label is private and goes only in the head-of-legal section.
- NEVER fabricate off-Jira hours. If the user skips, record `null`.
- NEVER attribute overdue tickets to a lawyer who was assigned the ticket within the last 24 hours -- give them a day before counting against them.
- NEVER count Done tickets as overdue. `duedate < now()` AND `statusCategory != Done`.
- NEVER post the report contents into a Jira comment. The audit trail is for case work, not team metrics.
