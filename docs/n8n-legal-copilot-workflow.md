# n8n "Legal Copilot" workflow: change log and contract

Workflow `Legal Copilot`, id `VAKq9Bra0RA0SdCO`, webhook `POST https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`.

## Accepted body

| Key | Required | Notes |
|---|---|---|
| `case_id` | yes | Jira key, e.g. `LEGAL-4321` |
| `case_folder` | yes | Matter folder only, e.g. `NDAs`. Must not contain the `case_id`. Sub-paths such as `_runs/triage-board` are allowed. |
| `documents` | one of | `[{filename, content_base64, mime_type}]`. A `.docx` must start with the ZIP header `PK\x03\x04`. |
| `text_documents` | one of | `[{filename, content_text, mime_type}]` (or `content`). Base64-encoded server-side. |
| `documents_from_jira` | one of | `[{filename, jira_attachment_url, mime_type}]`. Fetched server-side with the Jira credential. |
| `memory_instructions` | yes | Markdown string appended to the memory file. |

Max 10 documents per call; each base64 document max ~8 MB. No documents = memory-only update.

## Responses

| HTTP | Body | Meaning |
|---|---|---|
| 200 | `{success, case_id, message, documents_filed[], memory_file_updated, memory_file_path}` | Filed. Check each `documents_filed[].status`; `success_but_empty` is a failure. |
| 400 | `{success:false, error}` | `case_id` / `case_folder` missing or `case_folder` contains the `case_id`. |
| 422 | `{success:false, error}` | Payload rejected in `Resume State`: `docx_invalid: ...`, too many documents, oversize document, malformed `text_documents` / `documents_from_jira` item. |
| error / timeout | n/a | Upload or memory step failed after 3 retries, or `memory_download_failed` (the memory file could not be read, so it was not overwritten). Treat as `success:false`. |

## 2026-10-07: FIX-04 (weekly usage review of 2 Oct 2026), published version `bccb4e14`

Rollback target: version `9dbf837c` (27 Jul 2026).

- **N1** `Resume State`: rejects any `.docx` without a ZIP header (`docx_invalid`). Prevents the silent corrupt drafts of 4 Sep (exec 13321, 13322) and 24 Sep (exec 14897, 14905, 14914).
- **N2** `Validate Input`: requires non-empty `case_folder` and rejects a `case_folder` that contains the `case_id` (the `Claims/LEGAL-4912` double-path of exec 6727). `Respond Validation Failed` now says why.
- **N3** `Resume State` error output -> new `Respond Payload Rejected` node, HTTP 422 JSON, instead of a hanging webhook.
- **N4** `Upload Document to SharePoint`, `Upload Updated Memory File`, `Upload Memory Archive`: retry 3 times, 3 s apart.
- **N5** `Append to Memory Content`: when the memory file exceeds 250,000 characters, the oldest half of the `## Case:` entries is written to `_knowledge/legal_copilot_memory_archive_{date}_{time}.md` (new nodes `Archive Needed?`, `Upload Memory Archive`, `Restore Memory Payload`) before the live file is rewritten. The preamble (patterns, team notes) always stays in the live file. If the archive upload fails, the live file is not touched.
- **Guard** `Append to Memory Content`: if downloading the memory file fails with anything other than "not found", the run stops with `memory_download_failed` instead of overwriting the memory file with an empty one.

Tested 7 Oct 2026 with pinned data: exec 21567 (invalid .docx -> 422), 21569 (valid .docx + text_documents -> 200, no archive), 21570 (`case_folder` containing `case_id` -> 400).

## Not yet changed

`Legal Copilot Draft` (`zMZJM8RvzluqLIrl`) remains inactive. `/triage` uses the Microsoft 365 connector's `outlook_create_reply_draft` / `outlook_create_draft` first and only falls back to this workflow when it is active.
