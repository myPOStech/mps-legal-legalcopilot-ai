---
description: After the lawyer has sent the reply from Outlook -- verify the sent email, file it + new documents to SharePoint via the n8n Legal Copilot workflow, capture lawyer edits as feedback, and transition the ticket to Done.
argument-hint: <TICKET-KEY> [--dry-run]
---

# /reply-and-close

**The Copilot never sends email itself.** `/triage` may have left an AI draft in the lawyer's Outlook Drafts folder (subject starts `[AI DRAFT {ticket_key}]`), but sending is always the lawyer's act. (The one automatic email, the intake-form invite, is sent by the n8n workflow at triage; see `/triage` Step 4b item 4. It counts as a touch below.) The flow is therefore lawyer-sends-first:

1. The lawyer reviews the AI draft (Outlook draft, the ticket's AI Triage comment, or the SharePoint .docx), edits it, and **sends the reply themselves from Outlook**.
2. This command then: verifies the sent email exists, diffs it against the AI draft, files the sent email + any new documents to SharePoint via the n8n workflow, appends a feedback entry to the shared memory file, and transitions the Jira ticket to Done.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md` (never the old private `myPOS Legal 1/` copy)

## Input

`$ARGUMENTS`:
- `<TICKET-KEY>` (required) -- e.g., `LEGAL-4321`
- `--dry-run` (optional) -- show what would happen without filing or transitioning

---

## Step 1: Recover the AI draft

`mcp__atlassian__getJiraIssue` for the ticket with comments. Locate the most recent `## AI Triage` comment and extract:

- The `Draft Response` section (this is the AI draft baseline for the edit diff)
- The matter type, skill used, risk flags, `human_review_required` state, and the SharePoint folder URL

If no AI Triage comment exists, tell the user to run `/triage {key}` first and stop.

---

## Step 2: Confirm the reply was sent

Ask (skip if the user already said so in the invocation):

```
Has the reply for {ticket_key} been sent from Outlook?
  (a) Yes -- sent from my mailbox
  (b) Yes -- sent from the shared legal mailbox
  (c) Not yet
```

If (c): stop. This command only runs after the reply is sent.

---

## Step 3: Locate the sent email

`outlook_email_search` with:
- `folderName`: `Sent Items` (plus `mailboxOwnerEmail` if (b) in Step 2)
- `query`: the ticket key, else the ticket summary keywords
- `afterDateTime`: `last 7 days`

Then `read_resource` on the best match to get the full body, subject, recipients, and sent timestamp.

If nothing is found: ask the user to either paste the sent text (accepted as the source of truth, noted as `manual paste` in the filing) or correct the mailbox/search terms. Do not proceed on guesswork.

If multiple matches: pick the most recent, show its subject + first line + timestamp, and confirm with the user.

---

## Step 4: Detect lawyer edits

Compare the sent body against the AI draft from Step 1:

| Comparison | Disposition |
|---|---|
| Identical | No edits captured. Continue. |
| Whitespace/formatting only | No edits captured. Continue. |
| Material changes | Capture as feedback in Step 6. |

---

## Step 4b: Count touches with the requester

Count the **touches** on this matter (definition in `knowledge/two-touch-rule.md`: a message from Legal to the requester that delivers the answer or asks them for something; acknowledgements and Copilot-only Jira comments do not count):

- Sent emails to the requester since the ticket was created: `outlook_email_search` on `Sent Items` (and the shared legal mailbox if (b) in Step 2) with the ticket key or summary keywords, `afterDateTime` = the ticket's `created` date. Count each distinct message once, including the reply found in Step 3. - The automatic intake invite: if the ticket carries the label `intake:form-sent` or an `Intake-Invite: sent` comment, count exactly one touch for it (the email went out from the Legal mailbox, not from the lawyer's Sent Items). The form answers then arrive as a comment on this same ticket, so both touches are counted here.
- Jira comments by legal-team members (roster in `knowledge/team-routing.md`) that ask the requester something or give them the answer. `Intake-Invite:` and `Intake-Form:` comments are system comments, not lawyer comments, whoever authored them.

Read `two_touch_eligible` from the ticket's `path:*` label (`path:full_track` = not eligible; no `path:*` label = triaged before 0.5.0, treat as not eligible). Record `touches = N`. If the ticket is eligible and `N > 2`, set `two_touch_breached = true` and note in one line why (e.g. "requester sent the contract only after the second ask"). The lawyer can correct the count in Step 5.

---

## Step 5: Confirm before filing and closing

Show the lawyer:

```
About to file and close {ticket_key}.

Sent email found: {subject} -- {sent timestamp}
To: {recipient list}
Body preview: {first 300 chars}
New documents to file: {count}
Material edits vs AI draft: {yes/no}
Touches with requester: {N}{if two_touch_eligible: " (two-touch ticket" + (if two_touch_breached: ", BREACHED: " + reason) + ")"}

Confirm? (y / n / show-full-body / touches=<N> to correct the count)
```

If `--dry-run`, print the above and stop. Otherwise wait for explicit `y`.

**Risk gate:** if the AI Triage comment says `human_review_required` or shows risk flags, additionally require an existing Jira comment from the lawyer containing `approved for send` or `risk reviewed`. If absent, refuse to close and say exactly which comment to add.

---

## Step 6: File the final email + new documents to SharePoint

Invoke the `sharepoint-filer` skill (n8n-backed) with:

- `ticket_key`, `ticket_summary`, `matter_type` (from the AI Triage comment)
- `files`:
  - The sent email, role `final_email`. Raw MIME is not retrievable through the connector, so render what `read_resource` returned (headers: from/to/cc/date/subject, then the body) as a single `.html` string and pass it in `text_documents` (`content_text`, `mime_type: text/html`); no base64 needed. Name it `{TICKET-KEY}_{tag}_{YYYY-MM-DD}_final.html`.
  - Any new Jira attachments since `/triage` ran, handled per `sharepoint-filer` Step 3 item 4: signature / inline images skipped (image files only), real attachments up to 6 MB raw inline in `documents`, `documents_from_jira` only above 6 MB, at most 10 documents per call. Skip ones already listed in the case folder (check via `sharepoint_search` scoped to the case folder name).
- `memory_notes` -- a brief markdown block:
  - That the ticket was closed, the recipient list, the sent timestamp
  - Whether material edits were captured (and the category if so)
  - `touches: {N}`, and if `two_touch_breached`, `two_touch_breached: {reason}` (the board sweep's pattern detection reads these)
  - The `### Lawyer feedback` block from Step 7 if edits were material
- `triage_metadata` -- passed through from the AI Triage comment

If the case folder ends up with 0 files although documents were sent, say so; the filer retries once with `documents` only (`sharepoint-filer` Step 6). If the workflow still returns `success: false`, HALT before transitioning the ticket. Surface the error and let the lawyer retry. The audit trail must be complete on SharePoint before the ticket closes. An attachment listed in `attachments_not_filed` does not block the close.

Capture the returned `sharepoint_folder_url`, `documents_filed[].sharepoint_url`, `memory_file_url` and `attachments_not_filed` for Step 8.

---

## Step 7: Capture lawyer edits as feedback (memory file)

If material edits were detected in Step 4, make sure the `memory_notes` block includes:

```markdown
### Lawyer feedback ({ticket_key})
**Skill used:** {legal-triage-* skill, from AI Triage comment}
**AI draft excerpt:** "{first material-changed sentence as written by AI}"
**Lawyer changed to:** "{first material-changed sentence as sent}"
**Category:** {best guess: factual correction | tone | scope | jurisdiction-specific | other}
**Captured by:** /reply-and-close
```

The n8n workflow appends this to the shared memory file under the case entry. The next `/triage-board` run reads it back and surfaces emerging patterns. This gap (AI draft vs. what the lawyer actually sent) is the highest-signal training data we have.

---

## Step 8: Update Jira

Post a final comment on the ticket:

```markdown
**Reply sent (by lawyer) and ticket closed**

Sent to: {recipient list} on {sent timestamp}
Filed to SharePoint: [{matter_type}/{ticket_key}/]({sharepoint_folder_url})
- Final email: [{final.html}]({sharepoint_url_of_final_email})
- {N} new attachments (if any)
{if attachments_not_filed: "- Attachments not filed: {filename} ({reason}), ..."}

{if material edits: "Lawyer-edited draft -- {summary of edits} -- captured to memory file."}

Memory file: [legal_copilot_memory.md]({memory_file_url})

*Closed by Legal Copilot via n8n workflow VAKq9Bra0RA0SdCO.*
```

Then set the touch labels: read the current labels, merge in `touches:{N}` (plus `two-touch-breached` when `two_touch_breached`), and write them back with one call to the Atlassian tool whose suffix is `editJiraIssue`. Never drop existing labels.

Then transition to Done (match Atlassian tools by suffix; the newer connector has no `getTransitionsForJiraIssue`):
1. If a tool with suffix `getTransitionsForJiraIssue` exists, use it to find the Done transition ID. Otherwise pass the transition by name (`Done`) to the tool in step 2.
2. The tool with suffix `transitionJiraIssue` to apply it.
3. Re-read the assignee afterwards; if a workflow post-function changed it, restore the correct assignee.

---

## Step 9: Summarise

```
Filed and closed {ticket_key}:
  Sent by lawyer: {timestamp} to {recipient list}
  Filed to SharePoint: {sharepoint_folder_url}  (final email + {N} new attachments)
  Touches with requester: {touches}{if two_touch_breached: " -- two-touch target missed (" + reason + ")"}
  Memory file updated: {memory_file_url}
  Jira: transitioned to Done
  {if edits captured: "Lawyer edits captured ({category}) -- will improve future drafts after the next /triage-board run"}
```

---

## Hard rules

- NEVER send email from the Copilot's own tools. Only the lawyer sends; the automatic intake invite is sent by the n8n workflow at triage, never here. This command verifies and files what was sent.
- NEVER call any tool whose suffix is `outlook_send_mail`, `outlook_send_draft` or `outlook_forward_mail`, even though the connector exposes them.
- NEVER file or close without explicit `y` confirmation in Step 5 (unless `--dry-run`, which changes nothing).
- NEVER close a `human_review_required` or risk-flagged ticket without the lawyer's `approved for send` / `risk reviewed` Jira comment.
- ALWAYS file the final email to SharePoint via the n8n workflow BEFORE transitioning to Done. On `success: false`, do not transition.
- NEVER bypass the n8n workflow with a direct M365 SharePoint write.
- NEVER fall back to local-Desktop saving on workflow failure. Surface the error and let the lawyer retry.
- NEVER fabricate the recipient list -- it must come from the sent email located in Step 3 (or the lawyer's pasted copy, marked as such).
- NEVER read the old private memory file under `myPOS Legal 1/` (trailing " 1").
