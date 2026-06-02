---
description: Sweep unread emails in the legal inbox -- create Jira tickets for new matters and triage them. Files outputs to SharePoint via n8n.
argument-hint: [mailbox] [count] [filter]
---

# /triage-inbox

Read unread emails in the legal mailbox, dedupe against existing Jira tickets, create new tickets for new matters, and triage each one.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Default mailbox: `legal@mypos.com`
- Default count: 10 emails
- Process order: oldest first
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`

## Input

`$ARGUMENTS` may contain:
- A mailbox identifier (default: `legal@mypos.com`)
- A count (default: 10)
- A filter (e.g., `from:compliance@mypos.com`, `last 24 hours`)

---

## Phase 1: Load shared knowledge from SharePoint

Use `mcp__microsoft-365__sharepoint_read_file` to read:

```
myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

Plus the bundled seed `${CLAUDE_PLUGIN_ROOT}/knowledge/patterns.md`. If the SharePoint read fails, fall back to the seed and surface a one-line warning.

---

## Phase 2: Search the inbox

`mcp__microsoft-365__outlook_email_search` with:
- folder: Inbox of the configured legal mailbox
- filter: `isRead eq false` (plus any user-provided filter)
- top: configured count
- order: oldest first

If no unread emails, print "Inbox clear -- no unread emails" and stop.

---

## Phase 3: For each email

### 3a. Read full content

`mcp__microsoft-365__read_resource` (or the M365 MCP equivalent) for the message ID. Extract: subject (after stripping `Re:`/`Fwd:`/`AW:`), body, sender, sent date, attachments.

### 3b. Dedupe against Jira

```
JQL: project in (LEGAL, AIRD) AND statusCategory != Done AND summary ~ "{normalised subject}"
```

Also check sender domain matches and the recent `## Case:` entries inside `legal_copilot_memory.md` for >90% similar closed tickets.

**Decisions:**

| Finding | Action |
|---|---|
| Open ticket exists, same thread | Add the email content as a comment on that ticket. Mark email as read. Skip new-ticket creation. |
| Closed ticket >90% match | Reopen? No -- create a new ticket linked as "is follow-up of" the closed one. Continue to 3c. |
| Conflict | Create a new ticket linked to the conflicting ones, flag the conflict, escalate. Stop processing this email. |
| New matter | Continue to 3c. |

### 3c. Create Jira ticket

`mcp__atlassian__createJiraIssue`:
- project: `LEGAL` (default) or `AIRD` (only if email is clearly an AI/data-rights matter)
- summary: the email subject (stripped)
- description: the email body, with sender + date header
- issue type: Task
- reporter: lookup by email via `mcp__atlassian__lookupJiraAccountId` -- if not found, leave default and note in description

### 3d. Run the full triage flow on the new ticket

Same as `commands/triage.md` Steps 2-9: classify with matching legal-triage skill, Devil's advocate review, file outputs to SharePoint via the n8n workflow (`sharepoint-filer` skill), create Outlook draft, post Jira comment.

If the n8n workflow fails for this ticket, capture the error and continue to the next email -- the new Jira ticket stays in To-Do for manual re-triage with `/triage <KEY>`.

### 3e. Mark email as read

After successful triage **and successful filing**, mark the source email as read and tag it (Outlook category `Triaged`) so it doesn't re-enter the queue. If filing failed, leave the email unread so the next sweep retries it.

---

## Phase 4: Summary table

```
Inbox triage complete: {N} emails processed

| # | From | Subject | Type | Priority | Risk | Jira | Filed |
|---|------|---------|------|----------|------|------|-------|
| 1 | acme@... | NDA review request | NDA | Medium | none | LEGAL-4321 | ✓ |
| 2 | regulator@... | Information request | Inspection support | Critical | regulator | LEGAL-4322 | ✓ |
| 3 | partner@... | Contract redline | Contract review | Low | none | LEGAL-4323 | ✗ (n8n failure) |

{count} new tickets created
{count} added to existing tickets
{count} conflicts escalated
{count} flagged for human review
{count} n8n filing failures (need re-run)
```

---

## Hard rules

- NEVER send emails -- only create drafts.
- NEVER reply to the source email automatically -- the lawyer reviews the AI Drafts entry first.
- NEVER mark an email as read until the corresponding Jira ticket has been triaged successfully **and** the n8n filing workflow returned `success: true`.
- NEVER fabricate facts -- if information is missing, ask via the Jira ticket.
- NEVER mention individuals by name -- use roles.
- NEVER override risk gates.
- NEVER bypass the n