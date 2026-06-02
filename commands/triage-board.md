---
description: Sweep the Jira board -- triage every To-Do legal ticket end-to-end, auto-assign, set fields, and (NEW) re-assign + transition any ticket where a legal team member already replied. Files outputs to SharePoint via n8n. Runs daily at 08:00 Europe/Sofia on weekdays.
argument-hint: (no args)
---

# /triage-board

Process every To-Do ticket on the legal board: dedupe, classify, draft, three-pass review (legal -> Devil's advocate -> business), set Jira fields, auto-assign, file via the n8n workflow, comment.

**Designed to run autonomously every weekday at 08:00 Europe/Sofia** (registered by `/setup-copilot`), and on demand whenever you want a fresh sweep.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Default JQL: `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`
- Team routing config: `${CLAUDE_PLUGIN_ROOT}/knowledge/team-routing.md`

## Phase 1: Refresh shared knowledge

### 1a. Read shared memory

`mcp__microsoft-365__sharepoint_read_file` for the memory file. Plus the bundled seeds in `${CLAUDE_PLUGIN_ROOT}/knowledge/`:

- `patterns.md`
- `feedback-log.md`
- `sharepoint-map.md`
- `team-routing.md` (consumed by `jira-auto-assign`)

If the SharePoint read fails, fall back to the seeds and surface a warning.

### 1b. Detect lawyer corrections

JQL: `project in (LEGAL, AIRD) AND status = Done AND resolved >= -7d ORDER BY resolved DESC`. For each closed ticket with an "AI Triage" comment, diff the AI draft against the sent reply. Each `/reply-and-close` run should have appended a `### Lawyer feedback` block to the memory file -- if a recent close-out is missing it, list the omission in Phase 6.

### 1c. Detect patterns

3+ corrections sharing category + matter type + jurisdiction -> propose a new pattern (PROPOSED -- awaits lawyer approval; surfaced in Phase 6). The sweep never auto-edits a skill.

---

## Phase 2: Last-replier reconciliation (NEW)

**Before** scanning To-Do tickets, reconcile any ticket where a Legal team member has already responded.

### 2a. Find candidates

JQL: `project in (LEGAL, AIRD) AND status = "To Do" AND updated >= -7d`

For each ticket, call `mcp__atlassian__getJiraIssue` with `fields: ["comment", "assignee", "summary", "status"]` and `responseContentFormat: markdown`.

Walk the comment thread in reverse chronological order. For each comment, the author's `accountId` is what counts.

Skip any comment authored by:

- The Copilot itself (look for the `## AI Triage` heading or the `SLA Warning` heading -- those are bot comments)
- The reporter (the ticket originator -- they aren't "answering")
- Anyone NOT in the active legal roster from `team-routing.md`

The first matching comment is the **last legal-team reply**. Its author is the **last_replier**.

### 2b. Reassign + transition

If a `last_replier` exists for a ticket:

1. If `current_assignee != last_replier`:
   - Invoke `jira-auto-assign` with `force_reassign: true` and `override_account_id: {last_replier.account_id}` (the skill respects the override and skips its normal expertise-map logic). Record the rule applied as `last_replier`.
2. If `status == "To Do"`:
   - `mcp__atlassian__getTransitionsForJiraIssue` to find the transition ID for `In Progress`.
   - `mcp__atlassian__transitionJiraIssue` to move the ticket from To Do to In Progress.
3. Skip this ticket in Phase 3 -- the lawyer is on it already; the Copilot should not redraft over their work.

Record the moved tickets in the Phase 6 summary under `Last-replier reassigned and transitioned`.

**Guardrails:**

- NEVER reassign a ticket whose last legal reply is older than 14 days. After 14 days assume the lawyer has dropped it; let Phase 3 triage from scratch.
- NEVER transition a ticket out of To Do for any reason other than a legitimate last-replier match in this phase. Phase 3's normal triage NEVER auto-transitions To-Do tickets.
- NEVER post a comment as part of Phase 2. The assignee field + status field are the channel.

---

## Phase 3: Scan and process

JQL: `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`

Exclude any tickets handled in Phase 2.

For each To-Do ticket, follow `/triage`'s Steps 2-10:

1. Fetch ticket
2. Dedupe
3. Classify with the matching legal-triage skill
4. Apply risk gates
5. Devil's advocate review (`triage-reviewer` subagent)
6. Business reviewer (`business-reviewer` subagent)
7. Auto-assign (`jira-auto-assign`) + set fields (`jira-fields-and-flags`)
8. File outputs to SharePoint via `sharepoint-filer` (n8n)
9. Create Outlook draft
10. Post the AI Triage Jira comment

### Ticket disposition

| Result | Status |
|---|---|
| Successfully triaged, no risk, both reviewers approve | Stay in To-Do (the lawyer reviews and runs `/reply-and-close`) |
| Risk flags / `human_review_required` | Stay in To-Do, marked clearly |
| Duplicate of an open ticket | Transition to Done with merge comment |
| Conflict | Stay in To-Do; both tickets flagged |
| Doesn't fit any of the 10 matter types | Stay in To-Do; clarification comment posted |
| n8n filing failed | Stay in To-Do; error noted in Phase 6; NO AI Triage Jira comment posted |
| Auto-assign failed | Triage continues; assignee left unset; Phase 6 calls it out as a routing gap |

The board sweep never auto-closes a triaged ticket. Closing is always lawyer-gated via `/reply-and-close`. Only confirmed duplicates auto-close.

---

## Phase 4: SLA watchdog

```
JQL: project in (LEGAL, AIRD) AND statusCategory != Done ORDER BY created ASC
```

For each open ticket, compare `now() - created` against the SLA. If >75% elapsed AND no lawyer response since the AI triage comment, post:

```markdown
**SLA Warning**

This ticket was created {N} days ago. SLA for {priority} priority is {sla_days} days.
{remaining_days} day(s) remaining before SLA breach.

*Please review the AI draft (Outlook AI Drafts folder) and run `/reply-and-close` or escalate.*
```

> Note: the SLA *deadline* is already on the ticket's `duedate` field (set by `jira-fields-and-flags`). This Phase 4 comment is the only acceptable case where the Copilot posts deadline-like content -- the warning is the action, not the metadata.

---

## Phase 5: Overdue check (NEW, lightweight)

```
JQL: project in (LEGAL, AIRD) AND duedate < now() AND statusCategory != Done
```

Count overdue tickets per assignee. If any assignee has >= 3 overdue tickets, surface in Phase 6 under `Overload signals`. Do NOT post per-ticket Jira comments here; the productivity report (`/legal-report`) is the long-form view.

---

## Phase 6: Print summary

```
Board sweep complete ({timestamp})

Last-replier reassigned and transitioned: {count}
  {list of {ticket_key, from -> to, reason: "last reply by {name}"}}

Processed (new triage): {total}
  Triaged + draft ready: {count}
  Awaiting lawyer review: {count}
  Merged duplicates: {count}
  Conflicts escalated: {count}
  Clarification requested: {count}
  SLA warnings posted: {count}
  n8n filing failures (need re-run): {count}
    {list of {ticket_key, error}}
  Auto-assign failures (routing gap): {count}
    {list of {ticket_key, reason}}

Overload signals:
  {list of {assignee, overdue_count}} (>= 3 overdue tickets each)

SharePoint case folders created:
  {list of URLs}

Memory file updated:
  https://mypos0.sharepoint.com/sites/legal/Shared%20Documents/myPOS%20Legal/Claude%20skills%20memory/Copilot/_knowledge/legal_copilot_memory.md

{if pattern proposals: "PROPOSED PATTERNS (review and approve):"}
{pattern proposals}

Next scheduled run: {next weekday 08:00 Europe/Sofia}
Run /legal-report for the full productivity view.
```

---

## Hard rules

- NEVER send emails -- only create drafts in AI Drafts folder.
- NEVER auto-close a triaged ticket. Closing is lawyer-gated via `/reply-and-close`. Only confirmed duplicates auto-close.
- NEVER auto-edit any skill. Propose patterns, wait for human approval.
- NEVER bypass the n8n workflow for SharePoint writes.
- NEVER post the AI Triage Jira comment for a ticket whose n8n filing failed.
- NEVER fall back to local-Desktop saving on workflow failure.
- NEVER transition a ticket out of To Do except via the Phase 2 last-replier rule or a duplicate-merge close. Phase 3's normal triage NEVER auto-transitions To-Do tickets.
- NEVER reassign in Phase 2 if the last legal reply is older than 14 days.
- NEVER post the priority/due/flag/assignee as a comment -- those are fields now (set by `jira-fields-and-flags` and `jira-auto-assign`).
