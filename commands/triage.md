---
description: Triage a single Jira ticket or pasted email -- classify, dedupe, draft, three-pass review (legal -> Devil's advocate -> business), set Jira fields, auto-assign, file to SharePoint via n8n, post.
argument-hint: <TICKET-KEY | jira URL | pasted email>
---

# /triage

Triage a single legal request end-to-end. The ticket does **not** move to Done -- the lawyer reviews the draft (in the AI Triage Jira comment and the SharePoint .docx), then runs `/reply-and-close` to file the sent reply and close.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Projects watched: `LEGAL`, `AIRD`
- n8n filing workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`
- Team routing config: `${CLAUDE_PLUGIN_ROOT}/knowledge/team-routing.md`

> **Tool naming:** Microsoft 365 and Atlassian tools may be mounted under a session-specific server prefix. Match tools by suffix (`sharepoint_search`, `read_resource`, `outlook_email_search`, `getJiraIssue`, ...). If a tool named in this file does not exist under any prefix, treat that step per its own fallback instructions -- do not invent a tool name.

## Input

`$ARGUMENTS` is one of:
- A Jira ticket key (e.g., `LEGAL-4321`, `AIRD-44`)
- A Jira URL (extract the issue key from the path)
- Raw email content (subject + body + sender)

If `$ARGUMENTS` is empty, ask the user what to triage.

---

## Step 1: Load shared knowledge from SharePoint

The Microsoft 365 connector has no direct path-read tool. Read the shared memory file with this exact pattern:

1. `sharepoint_search` with query `legal_copilot_memory`.
2. Take the result whose `webUrl` contains `/myPOS Legal/`. **IGNORE any result under `/myPOS Legal 1/` (trailing " 1") -- that is the old private copy and must never be read or cited.**
3. `read_resource` with the returned `uri`.

Plus the bundled seed knowledge in `${CLAUDE_PLUGIN_ROOT}/knowledge/`:

- `patterns.md` -- learned rules from lawyer feedback
- `feedback-log.md` -- correction history (seed)
- `team-routing.md` -- assignment rules (read by the auto-assign skill in Step 7)

**Fallbacks (in order):**
- SharePoint memory read fails -> continue with the seeds and surface a one-line warning.
- `${CLAUDE_PLUGIN_ROOT}/knowledge/` unreadable (app-internal path; common in scheduled runs) -> fetch `team-routing.md`, `patterns.md` and `sharepoint-map.md` from the SharePoint `_knowledge` folder with the same search + read_resource pattern.
- Missing there too -> continue with conservative defaults, flag it in the final summary, and suggest `/setup-copilot` to seed them.

---

## Step 2: Fetch the request content

**Jira key provided:** `mcp__atlassian__getJiraIssue` with `responseContentFormat: markdown`. Extract summary, description, reporter, attachments list, existing comments.

**Email pasted:** use the subject, body, and sender directly.

**Attachment gate:** if the ticket's substance lives in an attachment (scanned letter, PDF) and its text cannot be extracted from Jira in this session, STOP and tell the lawyer exactly which file to attach in chat. Do not classify from the summary alone.

---

## Step 3: Deduplication check

JQL search for open near-duplicates:

```
project in (LEGAL, AIRD) AND statusCategory != Done AND summary ~ "{key terms}"
```

Scan recent `## Case:` entries inside `legal_copilot_memory.md` for >90% similarity too.

| Finding | Action |
|---|---|
| Duplicate of an open ticket | Link as "is duplicated by", merge content, transition new ticket to Done. Stop. |
| >90% match to a closed ticket | Note for Step 4 (adapt the previous response). Continue. |
| Conflict (same matter, contradictory asks) | Link both as "relates to", flag, escalate. Stop. |
| New request | Continue. |

**Re-triage cap:** if the ticket already carries an `## AI Triage` comment and there is NO human comment after it, do NOT run a fresh triage. If the SLA is breached, post one single escalation line mentioning the assignee; otherwise report "already triaged, awaiting lawyer" and stop.

---

## Step 4: Classify using the matching legal-triage skill

Pick exactly one of the 10 published legal-triage skills based on the matter (mapping table unchanged from prior version -- see `commands/triage-board.md` Phase 3 for the full table).

The chosen skill produces structured output: matter type, priority, SLA, jurisdiction, risk flags, missing-info questions, recommended action, draft response.

**Risk gates (hard rules, never override):**
- Any risk flag (regulator / inspection / claim / tight deadline) -> `human_review_required = true`
- Skill type is `claims` or `inspection_support` -> `human_review_required = true`
- Confidence < 0.7 OR ambiguous classification -> `human_review_required = true`

---

## Step 5: Devil's advocate review (review pass 1 of 2)

Pass the draft from Step 4 to the `triage-reviewer` subagent. The subagent runs the `devils-advocate-review` skill and returns `verdict`, `summary`, `findings`, optional `revised_draft`.

If verdict = `revise`: apply the suggested edits, re-run review. Cap at 2 revision passes; if still `revise`, set `human_review_required = true` and pass the latest draft into Step 6.

If verdict = `escalate`: set `human_review_required = true`. Continue to Step 6 anyway -- the business reviewer can still produce a reconciled draft and a clear summary.

---

## Step 6: Business reviewer (review pass 2 of 2)

Pass the latest draft AND the Devil's advocate output AND the triage metadata to the `business-reviewer` subagent (`agents/business-reviewer.md`).

Also include `context`:

```json
{
  "counterparty_name": "{from triage metadata or ticket}",
  "is_strategic_partner": true | false | unknown,
  "deal_size_estimate": "small" | "medium" | "large" | "unknown",
  "open_relationship_tickets": "{count from JQL: counterparty same, statusCategory != Done}"
}
```

If you do not know whether the counterparty is strategic, pass `unknown` -- the agent will surface a `context_gap`.

The agent returns: `verdict`, `business_override`, `summary`, `reconciliations`, `additional_findings`, `final_draft`, `context_gaps`.

**Verdict reconciliation (Devil's advocate + business):**

- Both `approve` -> final verdict `approve`.
- Devil's advocate `approve`, business `revise` -> apply the business `final_draft`. Final verdict `approve` (we are within review budget).
- Devil's advocate `revise/escalate`, business `approve` (with `business_override: true`) -> only valid if NO risk flag set. Final verdict `approve`. Surface the override prominently in the Jira comment.
- Either subagent escalates AND risk flag set -> `escalate`. `human_review_required = true`.
- Otherwise default to the more conservative verdict.

The `final_draft` from the business reviewer is what gets filed and posted.

---

## Step 7: Auto-assign and set Jira fields

Two parallel skill invocations (no comments, just field edits):

### 7a. Auto-assign

Invoke `jira-auto-assign` with:

```json
{
  "ticket_key": "{key}",
  "matter_type": "{from Step 4}",
  "confidence": {from Step 4},
  "human_review_required": {bool},
  "risk_flags": [...],
  "priority": "{Jira priority}",
  "sla_days": {N}
}
```

The skill returns the assignee chosen and the rule applied. Reassignment of an already-assigned ticket is NOT done by /triage -- only `/triage-board` with last-replier logic does that.

### 7b. Set fields and flag

Invoke `jira-fields-and-flags` with:

```json
{
  "ticket_key": "{key}",
  "priority": "{priority}",
  "sla_days": {N},
  "matter_type": "{...}",
  "risk_flags": [...],
  "human_review_required": {bool},
  "today_iso": "{YYYY-MM-DD}"
}
```

The skill sets `priority`, `duedate`, `labels`, and the `Flagged` field directly on the ticket. No comment.

If either skill returns `applied: false` / `assigned: false`, capture the reason and surface it in the Step 10 comment under a `Field-set warnings:` line, then continue.

---

## Step 8: File outputs to SharePoint via n8n

Invoke `sharepoint-filer` with the business reviewer's `final_draft`, all Jira attachments, and findings from both reviewers (Devil's advocate + business). The filer embeds them as Word comments inside the `.docx`:

- One anchored comment per Devil's advocate finding.
- One anchored comment per business reviewer reconciliation, prefixed `[Business review]`.
- One anchored comment per business reviewer additional finding.
- A verdict summary comment on the document title that includes BOTH the Devil's advocate verdict and the business reviewer verdict.

The filer base64-encodes the `.docx` (and every Jira attachment) and inlines them in the workflow's `documents` field. **Do NOT pass a `file_manifest`, `payload_path`, or `documents_from_url` shape -- the workflow only accepts the literal `documents: [{filename, content_base64, mime_type}]` array.** Do NOT substitute a `.txt` "receipt" for the real document if base64-inlining feels awkward; the workflow will accept it and "succeed", but the case folder ends up with a receipt and no draft -- this is the failure mode we are trying to eliminate.

**On `success: false` -- HALT.** Surface the workflow error verbatim to the user. DO NOT proceed to Step 9 (draft delivery) or Step 10 (Jira AI Triage comment). A failed filing with a clear error is better than a "successful" filing of a stub that points the lawyer at an empty SharePoint folder. The lawyer can re-run `/file-to-sharepoint LEGAL-XXXX` once the underlying issue is resolved.

---

## Step 9: Deliver the draft

Draft delivery has three surfaces, in order of preference:

1. The full draft ALWAYS goes into the "Draft Response" section of the AI Triage Jira comment (Step 10). That comment is the lawyer's primary review surface, and the guaranteed fallback.
2. The annotated `.docx` filed to SharePoint in Step 8 is the editable copy.
3. Native Outlook draft (best effort). Match a draft-creation tool by SUFFIX, not by a hard-coded name: look for a tool whose suffix is `outlook_create_draft` (observed working on LEGAL-5438, 15 Jul 2026). If present, call it with:
   ```json
   {"subject": "Re: {ticket summary}", "body": "{final_draft as HTML with a DRAFT banner}", "bodyType": "html"}
   ```
   Include the returned `webLink` in Steps 10 and 11. The draft lands in the lawyer's own Outlook Drafts folder; it is never sent. If no draft tool is present under any prefix this session, skip silently -- the Jira comment and SharePoint `.docx` are sufficient. Do NOT invent a tool name.
4. If the connector draft tool is absent but the n8n `legal-copilot-draft` workflow is deployed (check once per session with the n8n `search_workflows` tool), call it via `execute_workflow` with `{"ticket_key": "{key}", "subject": "Re: {ticket summary}", "body_html": "{final_draft as HTML}"}` and include the returned `webLink`. If neither exists, the draft lives in the Jira comment -- designed behaviour, no warning needed.

---

## Step 10: Post the AI Triage Jira comment

`mcp__atlassian__addCommentToJiraIssue` with `contentFormat: markdown`:

```markdown
---
## AI Triage

**Matter type:** {type} | **AI confidence:** {confidence}%
**Skill used:** {legal-triage-* skill name}
**Jurisdictions:** {jurisdictions or "unspecified"}

**Auto-assigned to:** {assignee_name} ({rule_applied})
**Fields set:** priority={priority}, due={due_date}, flagged={yes|no}, labels=[{labels_added}]

{if risk flags: "**Risk flags:** {flags}"}
{if human_review_required: "**NEEDS HUMAN REVIEW BEFORE SEND**"}
{if similar past ticket: "**Similar past ticket:** {key} -- response adapted from previous matter"}
{if missing info: "**Missing information:**\n{questions}"}

**Recommended action:** {action}

**Devil's advocate verdict:** {da_verdict} -- {da_summary}
**Business reviewer verdict:** {br_verdict}{if business_override: ' (OVERRIDE applied)'} -- {br_summary}

**Filed to SharePoint:** [{matter_type}/{ticket_key}/]({sharepoint_folder_url})
{list of {filename -> sharepoint_url} for each filed document}
**Memory file updated:** [legal_copilot_memory.md]({memory_file_url})

---
## Draft Response (for lawyer to review, edit, and send)

{business reviewer final draft}

---
*Review the Draft Response above (or the SharePoint .docx){if draft webLink: ", or open the [Outlook draft]({webLink})"}. Send your reply from Outlook, then run `/reply-and-close {ticket_key}`.*
---
```

Note: priority, due date, assignee, and flag are NOT repeated here as bullet items above the draft -- those live on the ticket fields now. The "Fields set" line is a one-line confirmation for the audit trail only.

---

## Step 11: Summarise to the user

```
Triage complete for {ticket_key}:
  Matter: {type} | Priority: {priority} | SLA: {sla_days}d | Due: {due_date}
  Assigned to: {assignee_name} ({rule_applied})
  Skill: {legal-triage-skill-name}
  Risk flags: {flags or "none"}
  Devil's advocate: {da_verdict} | Business reviewer: {br_verdict}{if override: ' (override)'}
  Filed: {sharepoint_folder_url}
  Memory: {memory_file_url}
  Outlook draft: {webLink | "n/a -- draft is in the Jira comment"}
  Jira comment posted; fields set: priority, due, labels, flag.
  {if human_review_required: "Lawyer review required before /reply-and-close"}
```

---

## Hard rules

- NEVER send emails. The Copilot only creates drafts; the lawyer reviews and sends from Outlook. The draft lives in the Jira comment, the SharePoint .docx, and (when a draft tool is available) the lawyer's Outlook Drafts folder.
- Match Microsoft 365 tools by SUFFIX. The connected connector exposes `outlook_create_draft` (create-draft) and `outlook_email_search` (read); it does NOT expose a tool named `outlook_email_create_draft`. Never invent a tool name; if a needed suffix is absent this session, follow the step's documented fallback.
- NEVER move the ticket to Done. That's `/reply-and-close`'s job.
- NEVER post priority, due date, deadline, flag, or assignee as a Jira comment. Use the fields.
- NEVER skip the business reviewer pass. Three opinions, every time.
- NEVER let `business_override: true` ride if any risk flag is set.
- NEVER fabricate `context.is_strategic_partner` -- pass `unknown` if you do not know.
- NEVER override risk gates.
- NEVER reassign an already-assigned ticket from /triage. That is `/triage-board`'s job.
- NEVER re-triage a ticket whose last AI Triage comment has no human reply after it (see Step 3 re-triage cap).
- NEVER post the AI Triage comment if the n8n filing returned `success: false`.
- NEVER substitute a `.txt` "receipt" for the real `.docx` when calling the filer. If base64-inlining is impossible, the filer must return `success: false` with `error: "document_too_large_to_inline"` and `/triage` must halt.
- NEVER bypass the n8n workflow with a direct M365 SharePoint write.
- NEVER read or cite the old private memory file under `myPOS Legal 1/` (trailing " 1").
- NEVER fall back to local-Desktop saving when n8n fails.
