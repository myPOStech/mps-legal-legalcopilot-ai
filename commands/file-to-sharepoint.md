---
description: File a ticket's attachments + latest AI draft directly to SharePoint via the n8n Legal Copilot workflow, and append a memory entry. Returns the live SharePoint URLs for the case folder and every uploaded file.
argument-hint: <TICKET-KEY> [--include-comments]
---

# /file-to-sharepoint

Pick up the attachments + latest AI draft for a Jira ticket and file them to SharePoint by calling the n8n `Legal Copilot` workflow. The workflow creates `myPOS Legal/{matter_folder}/{TICKET-KEY}/`, uploads each document, appends a memory entry to `legal_copilot_memory.md`, and returns the SharePoint URL of every file.

> **Filing is fully programmatic now.** Every write goes through the n8n workflow (webhook `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`, workflow ID `VAKq9Bra0RA0SdCO`). The old local-Desktop-mirror flow has been retired.

Useful when:
- A ticket has new documents added mid-flight that weren't there during initial triage
- You want to manually trigger filing without re-running the full `/triage` flow
- A lawyer wants to attach their own redlined version to the matter folder

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- n8n production webhook: `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`
- SharePoint site: `https://mypos0.sharepoint.com/sites/legal`
- SharePoint root path: `myPOS Legal/`

## Input

`$ARGUMENTS`:
- `<TICKET-KEY>` (required) -- e.g., `LEGAL-4321`
- `--include-comments` (optional) -- also file the Jira comment thread as a `comments.docx` document in the case folder

## Step 1: Fetch the ticket

`mcp__atlassian__getJiraIssue` for the given key with `responseContentFormat: markdown`. Extract:
- Summary (used to build counterparty tag if not otherwise derivable)
- Description
- Attachments (URLs + filenames)
- All comments (only if `--include-comments`)
- Any AI Triage comment (parse the matter type out of it)

If the ticket doesn't have an AI Triage comment yet, ask the user:

```
This ticket hasn't been triaged yet. Should I:
  (a) Run /triage first, then file (recommended)
  (b) File anyway -- I'll ask you for the matter type
```

---

## Step 2: Resolve the matter folder

Read `knowledge/sharepoint-map.md` (bundled with the plugin) and use the matter type from Step 1 to determine the matter-type folder. Default mapping:

| Matter type | Top-level folder (`case_folder` value) |
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

The team can override this in `knowledge/sharepoint-map.md` -- read that file first and let it win.

The workflow places everything under:

```
myPOS Legal/{case_folder}/{TICKET-KEY}/
```

There is no per-ticket subfolder by summary -- the ticket key IS the case-folder identifier.

---

## Step 3: List existing files in the case folder

Before staging new files, list what's already in `myPOS Legal/{case_folder}/{TICKET-KEY}/` via `mcp__microsoft-365__sharepoint_list_items` (read-only, reliable). This lets us:

- Pick the next free `v{N}` for a new draft
- Skip attachments that are already uploaded (filename match)

If the folder doesn't exist yet, the listing returns empty -- the workflow will create the folder on the first call.

---

## Step 4: Stage filenames

Per the team filing rule:

```
{TICKET-KEY}_{counterparty-or-subject-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

Where:
- `counterparty-or-subject-tag` is derived from the ticket (look for company name in description; fall back to a 3-word slug of the summary)
- `YYYY-MM-DD` is the current date
- `role-marker` is `v{N}` for drafts, `attachment_{stem}` for attachments, `comments` for the comment thread, `final` for sent emails

The Devil's advocate review is **not** a separate file. It is embedded as anchored Word comments inside the draft `.docx`:

```
LEGAL-4321_AcmeCorp_2026-04-27_v1.docx          ← original draft + embedded review comments
```

One Word comment per finding (anchored to the relevant text range), plus a final verdict summary comment on the document title.

---

## Step 5: Build the documents array

For each file:

1. **AI draft** -- if a recent draft exists in the most recent AI Triage Jira comment, render it to `.docx` (use the `docx` skill -- never `.md`). If review findings are present in the AI Triage comment, embed them as Word comments inside the draft `.docx`.
2. **Jira attachments** -- download via `mcp__atlassian__fetch` (or the equivalent Atlassian MCP fetch). Skip any whose target filename already exists in the case folder.
3. **Jira comment thread** (only if `--include-comments`) -- render the full thread as `comments.docx` (one heading per comment, author + timestamp as sub-heading, body as paragraphs).

For each file, base64-encode the bytes and pair with the IANA `mime_type`:

| File type | mime_type |
|---|---|
| `.docx` | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| `.pdf` | `application/pdf` |
| `.eml` | `message/rfc822` |
| `.png` / `.jpg` | `image/png` / `image/jpeg` |
| anything else | use Python `mimetypes.guess_type` or default to `application/octet-stream` |

---

## Step 6: Build `memory_instructions`

Append a markdown block summarising what this filing run added:

```markdown
**Trigger:** /file-to-sharepoint
**Matter type:** {matter_type}
**Documents added:** {count}
{list of filenames + roles}
{if --include-comments: "**Jira comment thread captured.**"}
```

If the ticket already has an AI Triage classification, fold the priority / SLA / jurisdictions / risk flags into the memory block too.

---

## Step 7: Call the n8n workflow

Invoke the `sharepoint-filer` skill (which wraps the n8n call) with:

- `ticket_key` -- the Jira key
- `matter_type` -- from Step 1/2
- `files` -- the staged document array from Step 5
- `review_findings` -- if present in the AI Triage comment
- `memory_notes` -- the markdown block from Step 6

The skill calls `mcp__n8n__execute_workflow` (workflow ID `VAKq9Bra0RA0SdCO`, type `webhook`) with the payload and returns:

- `success` -- boolean
- `sharepoint_folder_url` -- e.g., `https://mypos0.sharepoint.com/sites/legal/Shared Documents/myPOS Legal/NDAs/LEGAL-4321/`
- `documents_filed` -- list of `{filename, sharepoint_url, status}`
- `memory_file_url` -- URL of the shared memory file
- `chat_summary` -- multi-line summary ready to print

If `success: false`, surface the error and stop. Do NOT post the Jira comment in Step 8.

---

## Step 8: Update the Jira ticket

Add a comment to the ticket:

```markdown
**Files filed to SharePoint**

- Case folder: [{case_folder}/{TICKET-KEY}/]({sharepoint_folder_url})
- Memory file updated: [legal_copilot_memory.md]({memory_file_url})

Files filed:
- [{filename1}]({sharepoint_url1}) -- {status}
- [{filename2}]({sharepoint_url2}) -- {status}

*Filed by Legal Copilot via n8n workflow VAKq9Bra0RA0SdCO.*
```

---

## Step 9: Summarise

```
Filed {N} files for {ticket_key}:

  Case folder: {sharepoint_folder_url}

  Original draft: {filename}  (Devil's advocate review embedded as Word comments)
  Attachments: {count}
  {if --include-comments: "Comments: comments.docx"}

  Memory file updated: {memory_file_url}

Jira comment posted.
```

---

## Hard rules

- NEVER overwrite an existing draft `v{N}` -- always pick the next free version by listing the case folder first.
- NEVER strip the ticket key from filenames -- it's the cross-reference back to Jira.
- NEVER write the Devil's advocate review as a separate file.