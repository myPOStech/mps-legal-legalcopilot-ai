---
name: jira-fields-and-flags
description: Set Jira-native Priority, Due date, Labels and the Flag/Impediment field directly on the ticket via mcp__atlassian__editJiraIssue. Use after a legal-triage-* skill has produced priority + SLA + risk flags, and after jira-auto-assign has set the assignee. Replaces the previous behaviour of posting risk + SLA + deadline as Jira comments.
tools: Read, mcp__atlassian__editJiraIssue, mcp__atlassian__getJiraIssueTypeMetaWithFields, mcp__atlassian__getJiraIssue
---

# Jira fields and flags

Push deadlines, priority, labels and impediment flags into the Jira ticket itself, where the team can actually filter and sort on them. No comments.

## Inputs

```json
{
  "ticket_key": "LEGAL-4321",
  "priority": "Highest" | "High" | "Medium" | "Low" | "Lowest",
  "sla_days": 1-90,
  "matter_type": "nda" | "contract_review" | ...,
  "risk_flags": ["regulator", "claim", "tight_deadline", "inspection"],
  "human_review_required": true | false,
  "today_iso": "2026-06-02"
}
```

## Step 1: Compute the due date

`due_date = today_iso + sla_days calendar days` formatted `YYYY-MM-DD`.

If the resulting date falls on Saturday or Sunday, roll back to the previous Friday -- internal deadlines should land on workdays so they trigger morning sweeps.

## Step 2: Compute labels

Build a stable, alphabetically sorted label set:

- `matter:{matter_type}` (e.g., `matter:nda`)
- `auto-triaged-{today_iso}` (e.g., `auto-triaged-2026-06-02`)
- For each risk flag: `risk:{flag}` (e.g., `risk:regulator`, `risk:claim`)
- If `human_review_required`: add `needs-lawyer-review`

Never replace existing labels. Always merge: read the current label set via `mcp__atlassian__getJiraIssue` (`fields: ["labels"]`) and combine.

## Step 3: Compute the flag field

myPOS Jira uses the system **Flagged** field (`customfield_10021`, value `Impediment`) to mark a ticket as blocked / needs attention. Set it ON if any of:

- `risk_flags` includes `regulator`, `inspection`, or `claim`
- `human_review_required == true`
- `sla_days <= 2`

Otherwise clear it.

> **Field discovery.** If the `customfield_10021` ID is wrong on a particular Jira instance, look up the right ID once via `mcp__atlassian__getJiraIssueTypeMetaWithFields` and cache it in `knowledge/config.md` under "Jira field IDs". The skill always reads from there first; the customfield ID above is the documented fallback.

## Step 4: Push the edit

A single `editJiraIssue` call with all fields. Reasons:

- Atomic: either all fields land or none do; partial state is the worst outcome.
- Cheap: one HTTP round trip.

```
mcp__atlassian__editJiraIssue(
  cloudId = "fb47470f-f5c2-44bc-8182-f2a22f059adb",
  issueIdOrKey = ticket_key,
  fields = {
    "priority":  {"name": priority},
    "duedate":   due_date,
    "labels":    merged_labels,
    "customfield_10021": flagged_value   // [{"value": "Impediment"}] OR []
  }
)
```

Where `flagged_value` is `[{"value": "Impediment"}]` when on, and `[]` when off (Jira represents "no flag" as an empty array, not `null`).

If the edit returns an error mentioning an unknown field, peel off that field and retry. Order of importance: priority > duedate > flagged > labels. Never silently drop priority or duedate. If those two cannot be set, return:

```json
{"applied": false, "reason": "priority/duedate edit rejected: {error}", "fields_attempted": {...}}
```

## Step 5: Return

```json
{
  "applied": true,
  "priority": "High",
  "due_date": "2026-06-09",
  "labels_added": ["matter:nda", "auto-triaged-2026-06-02", "risk:claim"],
  "flagged": true,
  "fields_skipped": []
}
```

The calling command surfaces these in the AI Triage comment under "Fields set" -- not as a duplicate of what's already on the ticket, but as a one-line confirmation:

```
Fields set: priority=High, due=2026-06-09, flagged=yes, labels=[matter:nda, risk:claim, ...]
```

---

## Hard rules

- NEVER set priority via a comment. The priority field exists -- use it.
- NEVER set deadlines via a comment. The duedate field exists -- use it.
- NEVER blow away existing labels. Always merge.
- NEVER set a duedate in the past. If sla_days computes to a past date (clock skew, retroactive triage of an old ticket), set duedate to today_iso instead and add the label `sla-already-elapsed`.
- NEVER turn the Flagged field on without a real reason. Spurious flags train the team to ignore them.
