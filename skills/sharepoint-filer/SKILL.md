---
name: sharepoint-filer
description: File triage outputs (drafts, attachments, sent emails, comment threads) directly into SharePoint by calling the team's n8n "Legal Copilot" workflow. The workflow creates the case folder, uploads every document, and appends an entry to the shared memory file at `myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`. Returns the SharePoint URL of every uploaded file plus the memory-file URL so callers can cite them in Jira and chat. Replaces the old local-Desktop-mirroring flow -- no manual drag-and-drop required.
---

# SharePoint filer (n8n-backed)

Push every triage output to SharePoint in one round trip by calling the team's n8n `Legal Copilot` workflow. The workflow creates the case subfolder, uploads each document, downloads-appends-uploads the shared memory file, and returns a JSON summary with the live SharePoint URL of every uploaded file.

> **Why n8n:** the Microsoft 365 MCP cannot reliably write to SharePoint. The n8n workflow authenticates via a service-account OAuth credential and uses the SharePoint REST API directly, which is reliable. The workflow is the single source of truth for "where do legal triage outputs end up." When this skill calls the workflow, it does **not** need to know SharePoint paths, folder IDs, or auth -- the workflow owns all of that.

## The n8n workflow

| Property | Value |
|---|---|
| Name | `Legal Copilot` |
| Workflow ID | `VAKq9Bra0RA0SdCO` |
| Production webhook | `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing` |
| Method | `POST` |
| Available in MCP | yes -- callable via `mcp__n8n__execute_workflow` |
| What it does | Creates `/sites/legal/Shared Documents/myPOS Legal 1/Claude skills memory/Copilot/{case_folder}/{case_id}/`, uploads each document, downloads the memory file, appends a markdown entry, re-uploads it, returns a JSON summary |
| Memory file location | `myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md` |
| SharePoint root | `https://mypos0.sharepoint.com/sites/legal/Shared Documents/myPOS Legal 1/Claude skills memory/Copilot/` |

> The base path is `myPOS Legal 1/` (note the trailing ` 1`), not `myPOS Legal/`. Earlier docs were stale. Confirmed against the live workflow's `Prepare State` node.

## Inputs

The caller (`/triage`, `/file-to-sharepoint`, `/reply-and-close`, `/triage-board`, `/triage-inbox`) provides:

| Input | Required | Description |
|---|---|---|
| `ticket_key` | yes | Jira key (e.g., `LEGAL-4321`). Becomes `case_id` in the n8n payload. |
| `ticket_summary` | yes | Ticket title. Used in filenames and as part of the memory entry. |
| `matter_type` | yes | One of the 10 matter types (NDA, contract review, regulatory question, corporate change, project, KYC support, GTCs, materials review, claims, inspection support). Maps to the matter-type folder name. |
| `files` | yes | List of `{name, source, content_or_url, role, mime_type}` where `role` ∈ `{draft, attachment, final_email, comments}` and `mime_type` is the IANA type (e.g., `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `application/pdf`, `message/rfc822`). |
| `review_findings` | optional | Devil's advocate findings. Embedded as anchored Word comments inside the draft `.docx` BEFORE base64-encoding. List of `{severity, location, issue, suggested_edit}`. |
| `counterparty` | optional | If known, used in filenames; otherwise derived from the ticket. |
| `memory_notes` | optional | Free-text notes the caller wants appended to the shared memory file. The skill builds the final `memory_instructions` markdown block from these + the structured triage metadata. |
| `triage_metadata` | optional | Dict with priority, SLA, jurisdictions, risk flags, devil's-advocate verdict. Folded into `memory_instructions`. |

## Output

Returns:

| Field | Description |
|---|---|
| `success` | Boolean. True when every document upload succeeded AND the memory-file write succeeded. |
| `case_folder` | The matter-type folder path the workflow used (e.g., `NDAs`). |
| `case_id` | The ticket key (echo). |
| `sharepoint_folder_url` | The SharePoint URL of the case folder (browser-friendly). |
| `documents_filed` | List of `{filename, sharepoint_url, status}` for each uploaded file. |
| `memory_file_url` | The SharePoint URL of the shared memory file. |
| `memory_file_updated` | Boolean. True when the workflow successfully re-uploaded the memory file. |
| `chat_summary` | A multi-line string ready to print to chat (and to embed in the Jira comment) summarising what was filed and where. |
| `error` | When `success: false`, a short machine-readable error code (e.g., `document_too_large_to_inline`, `workflow_http_error`, `workflow_rejected_payload`). |

---

## Step 1: Resolve the matter-folder name

Map the input `matter_type` to one of the 10 top-level folder names:

| Matter type | `case_folder` value |
|---|---|
| NDA | `NDAs` |
| Contract review | `Contract Reviews` |
| Regulatory question | `Regulatory Questions` |
| Corporate change | `Corporate Changes` |
| Project | `Projects` |
| KYC support | `KYC Support` |
| GTCs | `GTCs` |
| Materials review | `Materials Reviews` |
| Claims | `Claims` |
| Inspection support | `Inspection Support` |

The team can override these in `knowledge/sharepoint-map.md`. If the file has a non-default mapping for this matter type, use that instead.

`case_folder` is the **matter-type folder name only**. Do NOT include the ticket key -- the workflow concatenates it. Passing `Claims/LEGAL-4912` produces `.../Claims/LEGAL-4912/LEGAL-4912/` (confirmed in execution 6727 on 2026-05-27).

The workflow builds the final SharePoint path as `myPOS Legal 1/Claude skills memory/Copilot/{case_folder}/{case_id}/`. This skill does NOT need to know the SharePoint base URL -- the workflow owns it.

---

## Step 2: Compute filenames

Filenames follow the team convention:

```
{TICKET-KEY}_{counterparty-or-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

| Variable | How to derive |
|---|---|
| `TICKET-KEY` | Input `ticket_key` |
| `counterparty-or-tag` | Input `counterparty` if provided. Otherwise extract a 2-3 word slug from the ticket summary (capitalise, no spaces). |
| `YYYY-MM-DD` | Today's date (lawyer's local date). |
| `role-marker` | Per the table below. |
| `ext` | Per the role. |

| Role | Marker | Extension | Example |
|---|---|---|---|
| draft | `v{N}` | `.docx` | `LEGAL-4321_AcmeCorp_2026-04-27_v1.docx` |
| attachment | `attachment_{original-stem}` | original | `LEGAL-4321_AcmeCorp_2026-04-27_attachment_NDA-Acme.pdf` |
| final_email | `final` | `.eml` | `LEGAL-4321_AcmeCorp_2026-04-27_final.eml` |
| comments | `comments` | `.docx` | `LEGAL-4321_AcmeCorp_2026-04-27_comments.docx` |

**Versioning for drafts:** the workflow uploads with `overwrite=true`, so the skill is responsible for picking the right `v{N}`. Before calling the workflow, list the existing files in the case folder via `mcp__microsoft-365__sharepoint_list_items` (read-only, reliable). Pick the next free `v{N}`. If the read fails, default to `v1` and surface a warning in `chat_summary`.

**Attachment collisions:** if two attachments share an original-stem in the same call, append a counter: `attachment_NDA-Acme.pdf` → `attachment_NDA-Acme-2.pdf`.

---

## Step 3: Build each document

For each file in the input list:

1. **draft** -- convert to `.docx` (use the `docx` skill so the output is a real Word document). If `review_findings` is non-empty, embed each finding as a Word comment anchored to the `location` text range. Comment author = `Devil's advocate review`. Comment body =
   ```
   [{severity}] {issue}
   Suggested edit: {suggested_edit}
   ```
   If `location` cannot be matched, attach to the first paragraph and prefix the body with `(general)`. Add a final summary comment on the document title with the verdict line: `Devil's advocate verdict: {approve | revise | escalate} -- {N} blockers, {M} concerns, {K} nits`.

2. **comments** -- render the Jira comment thread as a `.docx` (one heading per comment, author + timestamp as a sub-heading, body as paragraphs). Never `.md`.

3. **final_email** -- fetch the `.eml` from Outlook by message ID (use `mcp__microsoft-365__outlook_email_search` to confirm the message and its raw MIME, or save the message body + headers as `.eml` directly).

4. **attachment** -- download from the source (Jira attachment URL via `mcp__atlassian__fetch` or the equivalent Atlassian MCP fetch).

5. Read the resulting bytes from disk, base64-encode, and pair with the IANA `mime_type`.

> The real document bytes MUST go into `content_base64`. Substituting a `.txt` "receipt" describing the document is a hard violation of the rules below -- see "Hard rules" at the bottom of this file.

---

## Step 4: Build `memory_instructions`

The shared memory file lives at `myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`. The workflow appends the `memory_instructions` string to it under a new `## Case: {case_id}` heading (the workflow adds the heading and timestamp).

The skill builds the markdown body using the triage metadata + the caller's `memory_notes`:

```markdown
**Matter type:** {matter_type}
**Priority:** {triage_metadata.priority} | **SLA:** {triage_metadata.sla_days}d
**Jurisdictions:** {triage_metadata.jurisdictions or "unspecified"}
**Skill used:** {triage_metadata.legal_triage_skill}
**Devil's advocate verdict:** {triage_metadata.da_verdict}
{if triage_metadata.risk_flags: "**Risk flags:** {flags}"}
{if triage_metadata.human_review_required: "**Lawyer review required before send.**"}

{caller's memory_notes -- typically: action taken, lawyer feedback summary, any pattern updates}
```

If `memory_notes` is empty and `triage_metadata` is empty, the body is just `Filed by Legal Copilot.` -- the workflow still appends a timestamped entry so the audit trail is complete.

---

## Step 5: Call the n8n workflow

Use `mcp__n8n__execute_workflow` (or POST the webhook directly with `mcp__workspace__bash` curl as a fallback). The MCP route is preferred -- it's auth-free for workflows in our project and gives us the execution ID.

**Payload shape (the ONLY shape the workflow accepts):**

```json
{
  "type": "webhook",
  "webhookData": {
    "method": "POST",
    "body": {
      "case_id": "LEGAL-4321",
      "case_folder": "NDAs",
      "documents": [
        {
          "filename": "LEGAL-4321_AcmeCorp_2026-04-27_v1.docx",
          "content_base64": "<base64 string>",
          "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
        },
        {
          "filename": "LEGAL-4321_AcmeCorp_2026-04-27_attachment_NDA-Acme.pdf",
          "content_base64": "<base64 string>",
          "mime_type": "application/pdf"
        }
      ],
      "memory_instructions": "**Matter type:** NDA\n**Priority:** Medium | **SLA:** 5d\n..."
    }
  }
}
```

Call:

```
mcp__n8n__execute_workflow(
  workflowId = "VAKq9Bra0RA0SdCO",
  executionMode = "production",
  inputs = <payload above>
)
```

Wait for the response. Production runs respond synchronously through the workflow's `Respond to Webhook` node.

**Curl fallback (if the MCP is unavailable):**

```bash
curl -sS -X POST https://myposai.app.n8n.cloud/webhook/legal-copilot-filing \
  -H 'Content-Type: application/json' \
  -d @payload.json
```

Pipe the JSON response into the same parsing logic.

### Known-good caller pattern

```python
import base64

with open(docx_path, "rb") as f:
    b64 = base64.b64encode(f.read()).decode("ascii")

payload = {
    "type": "webhook",
    "webhookData": {
        "method": "POST",
        "body": {
            "case_id": ticket_key,                       # "LEGAL-5261"
            "case_folder": matter_folder,                # "Regulatory Questions" -- NO ticket suffix
            "documents": [{
                "filename": filename,                    # team naming convention
                "content_base64": b64,                   # the REAL bytes, base64-encoded
                "mime_type": mime_type
            }],
            "memory_instructions": memory_md             # markdown block from Step 4
        }
    }
}
# Then: mcp__n8n__execute_workflow(workflowId="VAKq9Bra0RA0SdCO",
#                                   executionMode="production",
#                                   inputs=payload)
```

---

## Step 6: Parse the response

The workflow returns:

```json
{
  "success": true,
  "case_id": "LEGAL-4321",
  "message": "Case LEGAL-4321 filed successfully...",
  "documents_filed": [
    { "filename": "...", "sharepoint_url": "https://mypos0.sharepoint.com/...", "status": "success" }
  ],
  "memory_file_updated": true,
  "memory_file_path": "myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md"
}
```

Build the skill's return object from this:

```json
{
  "success": true,
  "case_folder": "NDAs",
  "case_id": "LEGAL-4321",
  "sharepoint_folder_url": "https://mypos0.sharepoint.com/sites/legal/Shared Documents/myPOS Legal 1/Claude skills memory/Copilot/NDAs/LEGAL-4321/",
  "documents_filed": ["...passthrough..."],
  "memory_file_url": "https://mypos0.sharepoint.com/sites/legal/Shared Documents/myPOS Legal 1/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md",
  "memory_file_updated": true,
  "chat_summary": "<the workflow's `message` field, lightly reformatted>"
}
```

If the workflow returns `success: false` OR HTTP non-200, surface the full error in `chat_summary` and return `success: false`. The caller MUST NOT proceed to "post triage comment to Jira" or "transition to Done" if `success: false`.

---

## Step 7: Return result

The caller pipes `chat_summary` straight into:
- The `/triage` chat output to the lawyer
- The Jira AI Triage comment (the SharePoint URLs are clickable in Jira's markdown)

---

## Hard rules

- **NEVER bypass the n8n workflow.** Programmatic SharePoint writes via the M365 MCP are unreliable in this deployment; this skill exists precisely to centralise filing through n8n.
- **NEVER substitute a `.txt` "receipt" for the actual document.** If the caller is tempted to upload a small text file describing what the real document is (because inlining ~40 KB+ of base64 feels awkward) -- STOP. That defeats the entire purpose of the workflow. The lawyer never re-runs `/file-to-sharepoint` manually; the case folder ends up with a receipt and no real draft.
- **If `content_base64` for the real document cannot be assembled inside one tool call**, the skill MUST return `success: false` with `error: "document_too_large_to_inline"` and surface the issue in `chat_summary`. The caller (`/triage`, `/triage-board`, `/triage-inbox`) MUST then halt -- DO NOT post the AI Triage Jira comment as if the filing succeeded. A failed filing with a clear error is better than a "successful" filing of a receipt stub.
- **NEVER send `file_manifest`, `payload_path`, `documents_from_url`, or any other payload shape that is not the literal `documents: [{filename, content_base64, mime_type}]` array.** The workflow rejects everything else with `Document item missing filename or content_base64` (see execution 6727 on 2026-05-27 -- failed because the caller sent `file_manifest` + `payload_path` instead of inlining).
- **`case_folder` MUST be the matter-type folder name only** (e.g., `Regulatory Questions`, `Claims`, `NDAs`) -- NOT `Regulatory Questions/LEGAL-5261`. The workflow concatenates `case_id` itself; passing the case id twice produces an ugly `.../Claims/LEGAL-4912/LEGAL-4912/...` path (see execution 6727).
- **Memory file path is `myPOS Legal 1/...`, not `myPOS Legal/...`** -- confirmed by the live workflow's `Prepare State` node. Earlier docs in this skill used `myPOS Legal/`; the workflow's actual base path is `myPOS Legal 1/` (note the trailing space-1). Any documentation referencing the old path is stale.
