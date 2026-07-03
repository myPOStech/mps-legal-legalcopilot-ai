---
description: Sweep recent emails in the legal inbox -- create Jira tickets for new matters and triage them. Files outputs to SharePoint via n8n.
argument-hint: [mailbox] [count] [filter]
---

# /triage-inbox

Read recent emails in the legal mailbox, dedupe against existing Jira tickets, create new tickets for new matters, and triage each one.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Default mailbox: `legal@mypos.com`
- Default count: 10 emails
- Process order: oldest first
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md` (never the stale `myPOS Legal/` copy)

> **Capability note:** the Microsoft 365 connector is READ-ONLY for mail. There is no tool to mark an email read, tag it, move it, or send. Idempotency comes from the Jira dedupe (Phase 3b) plus the processed-message log in the memory file -- NOT from read-state.

## Input

`$ARGUMENTS` may contain:
- A mailbox identifier (default: `legal@mypos.com`)
- A count (default: 10)
- A filter (e.g., `from:compliance@mypos.com`, `last 24 hours`)

---

## Phase 1: Load shared knowledge from SharePoint

Use `sharepoint_search` for `legal_copilot_memory` -> take the hit whose `webUrl` contains `/myPOS Legal 1/` -> `read_resource` on its `uri`.

Plus the bundled seed `${CLAUDE_PLUGIN_ROOT}/knowledge/patterns.md`.

**Fallbacks:** SharePoint read fails -> seed only, one-line warning. Seed unreadable (app-internal path) -> fetch `patterns.md` from the SharePoint `_knowledge` folder; missing there too -> continue without patterns and flag it.

From the memory file, collect the `processed_message_ids` mentioned in recent `## Case:` entries (see Phase 3f) -- these emails are already handled.

---

## Phase 2: Search the inbox

`outlook_email_search` with:
- `mailboxOwnerEmail`: the configured legal mailbox (requires delegated access)
- `folderName`: `Inbox`
- `afterDateTime`: the user-provided window, default `last 3 days`
- limit: configured count, oldest first where supported

There is NO unread-only filter in the connector. Pull the recent window and rely on dedupe: skip any message whose ID is in `processed_message_ids` (Phase 1) or whose thread already has a Jira ticket (Phase 3b).

If no candidate emails remain after dedupe, print "Inbox clear -- nothing new to triage" and stop.

---

## Phase 3: For each email

### 3a. Read full content

`read_resource` with the message URI from the search result. Extract: subject (after stripping `Re:`/`Fwd:`/`AW:`), body, sender, sent date, attachments list.

### 3b. Dedupe against Jira

```
JQL: project in (LEGAL, AIRD) AND statusCategory != Done AND summary ~ "{normalised subject}"
```

Also check sender domain matches and the recent `## Case:` entries inside `legal_copilot_memory.md` for >90% similar closed tickets.

**Decisions:**

| Finding | Action |
|---|---|
| Open ticket exists, same thread | Add the email content as a comment on that ticket. Log the message ID (3f). Skip new-ticket creation. |
| Closed ticket >90% match | Reopen? No -- create a new ticket linked as "is follow-up of" the closed one. Continue to 3c. |
| Conflict | Create a new ticket linked to the conflicting ones, flag the conflict, escalate. Stop processing this email. |
| New matter | Continue to 3c. |

### 3c. Create Jira ticket

`mcp__atlassian__createJiraIssue`:
- project: `LEGAL` (default) or `AIRD` (only if email is clearly an AI/data-rights matter)
- summary: the email subject (stripped)
- description: the email body, with sender + date header AND the Outlook message ID on the last line (`Source-Message-Id: {id}`)
- issue type: Task
- reporter: lookup by email via `mcp__atlassian__lookupJiraAccountId` -- if not found, leave default and note in description

### 3d. Run the full triage flow on the new ticket

Same as `commands/triage.md` Steps 2-10: classify with the matching legal-triage skill, Devil's advocate + business review, auto-assign + set fields, file outputs to SharePoint via the n8n workflow (`sharepoint-filer` skill), deliver the draft (Jira comment + SharePoint .docx; there is no Outlook draft tool), post the AI Triage Jira comment.

If the n8n workflow fails for this ticket, capture the error and continue to the next email -- the new Jira ticket stays in To-Do for manual re-triage with `/triage <KEY>`. Do NOT post the AI Triage comment for that ticket.

### 3e. Read-state (do not attempt)

There is no mail-write tool: the Copilot cannot mark the email read or apply an Outlook category. Do not try. The audit trail lives in Jira (`Source-Message-Id` in the description) and the memory file (3f).

### 3f. Log the processed message

Include the Outlook message ID in the `memory_notes` passed to `sharepoint-filer`, as a line `processed_message_id: {id}`. The next inbox sweep reads these back in Phase 1 and skips the email.

---

## Phase 4: Summary table

```
Inbox triage complete: {N} emails processed

| # | From | Subject | Type | Priority | Risk | Jira | Filed |
|---|------|---------|------|----------|------|------|-------|
| 1 | acme@... | NDA review request | NDA | Medium | none | LEGAL-4321 | OK |
| 2 | regulator@... | Information request | Inspection support | Critical | regulator | LEGAL-4322 | OK |
| 3 | partner@... | Contract redline | Contract review | Low | none | LEGAL-4323 | FAILED (n8n) |

{count} new tickets created
{count} added to existing tickets
{count} conflicts escalated
{count} flagged for human review
{count} n8n filing failures (need re-run)
{count} skipped as already processed
```

---

## Hard rules

- NEVER send emails or reply to the source email. No send tool exists; the lawyer replies after review.
- NEVER attempt to mark an email read, tag it, or move it -- no mail-write tool exists. Idempotency = Jira dedupe + `processed_message_id` log in the memory file.
- NEVER create a ticket without embedding the `Source-Message-Id` in the description.
- NEVER post the AI Triage comment for a ticket whose n8n filing failed.
- NEVER fabricate facts -- if information is missing, ask via the Jira ticket.
- NEVER mention individuals by name in ticket descriptions of sensitive matters -- use roles.
- NEVER override risk gates.
- NEVER bypass the n8n workflow for SharePoint writes.
- NEVER read the stale memory file under `myPOS Legal/` (no trailing " 1").
