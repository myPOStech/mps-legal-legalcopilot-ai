---
description: Triage a single Jira ticket or pasted email -- classify, dedupe, draft, three-pass review (legal -> Devil's advocate -> business), set Jira fields, auto-assign, file to SharePoint via n8n, post.
argument-hint: <TICKET-KEY | jira URL | pasted email>
---

# /triage

Triage a single legal request end-to-end. The ticket does **not** move to Done -- the lawyer reviews the draft in Outlook, then runs `/reply-and-close` to send and close.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Projects watched: `LEGAL`, `AIRD`
- n8n filing workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`
- Team routing config: `${CLAUDE_PLUGIN_ROOT}/knowledge/team-routing.md`

## Input

`$ARGUMENTS` is one of:
- A Jira ticket key (e.g., `LEGAL-4321`, `AIRD-44`)
- A Jira URL (extract the issue key from the path)
- Raw email content (subject + body + sender)

If `$ARGUMENTS` is empty, ask the user what to triage.

---

## Step 1: Load shared knowledge from SharePoint

Read the shared memory file from SharePoint via `mcp__microsoft-365__sharepoint_read_file`:

```
myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

Plus the bundled seed knowledge in `${CLAUDE_PLUGIN_ROOT}/knowledge/`:

- `patterns.md` -- learned rules from lawyer feedback
- `feedback-log.md` -- correction history (seed)
- `team-routing.md` -- assignment rules (read by the auto-assign skill in Step 7)

If the SharePoint memory read fails, fall back to the seeds and surface a one-line warning.

---

## Step 2: Fetch the request content

**Jira key provided:** `mcp__atlassian__getJiraIssue` with `responseContentFormat: markdown`. Extract summary, description, reporter, attachments list, existing comments.

**Email pasted:** use the subject, body, and sender directly.

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

The `final_draft` from the business reviewer is what gets filed and used for the Outlook draft.

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

**On `success: false` -- HALT.** Surface the workflow error verbatim to the user. DO NOT proceed to Step 9 (Outlook draft) or Step 10 (Jira AI Triage comment). A failed filing with a clear error is better than a "successful" filing of a stub that points the lawyer at an empty SharePoint folder. The lawyer can re-run `/file-to-sharepoint LEGAL-XXXX` once the underlying issue is resolved.

---

## Step 9: Create the Outlook draft

`mcp__microsoft-365__outlook_email_create_draft` in the `AI Drafts` folder:

- Body: the business reviewer's `final_draft` (with `[DRAFT - FOR LAWYER REVIEW BEFORE SENDING]` banner)
- Custom property: `{"x-mypos-legal-ticket": "<ticket_key>"}`

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
*Outlook draft saved to AI Drafts folder. Run `/reply-and-close {ticket_key}` after review.*
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
  Outlook draft: AI Drafts > "Re: {subject}"
  Jira comment posted; fields set: priority, due, labels, flag.
  {if human_review_required: "Lawyer review required before /reply-and-close"}
```

---

## Hard rules

- NEVER send emails -- only create drafts.
- NEVER move the ticket to Done. That's `/reply-and-close`'s job.
- NEVER post priority, due date, deadline, flag, or assignee as a Jira comment. Use the fields.
- NEVER skip the business reviewer pass. Three opinions, every time.
- NEVER let `business_override: true` ride if any risk flag is set.
- NEVER fabricate `context.is_strategic_partner` -- pass `unknown` if you do not know.
- NEVER override risk gates.
- NEVER reassign an already-assigned ticket from /triage. That is `/triage-board`'s job.
- NEVER post the AI Triage comment if the n8n filing returned `success: false`.
- NEVER substitute a `.txt` "receipt" for the real `.docx` when calling the filer. If base64-inlining is impossible, the filer must return `success: false` with `error: "document_too_large_to_inline"` and `/triage` must halt.
- NEVER bypass the n8n workflow with a direct M365 SharePoint write.
- NEVER fall back to local-Desktop saving when n8n fails.
