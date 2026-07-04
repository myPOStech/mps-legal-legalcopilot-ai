# Changelog

All notable changes to the myPOS Legal Copilot plugin.

## [0.4.5] - 2026-07-04

SharePoint filing relocation. Legal triage outputs and the shared memory file now land in the shared Legal team library `myPOS Legal/` instead of the private `myPOS Legal 1/` library that only the workflow owner could reach.

### Changed
- **Filing destination moved from the private `myPOS Legal 1/` library to the shared `myPOS Legal/` team library.** Target root is now `/sites/legal/Shared Documents/myPOS Legal/Claude skills memory/Copilot`. The `myPOS Legal 1/` copy was reachable only by the workflow owner, so the rest of the Legal team could not open filed documents or the shared memory file.
- **n8n `Legal Copilot` workflow (`VAKq9Bra0RA0SdCO`) repointed and republished.** Three references updated: the `Prepare State` node's `docs_root` and `memory_file_path`, and the hard-coded folder URL in `Upload Updated Memory File`. Every other path (folder creation, per-document upload, memory download) is derived from these, so the production webhook now writes to the shared library.
- **All plugin references repointed to `myPOS Legal/`.** `sharepoint-filer` and `jira-auto-assign` skills plus the `/triage`, `/triage-board`, `/triage-inbox` and `/reply-and-close` commands. The "never read the stale copy" guardrails are inverted: the shared `myPOS Legal/` file is now canonical and the old private `myPOS Legal 1/` copy is the one never to read or cite. This reverses the 0.4.3 and 0.4.4 guidance, which had treated the private library as canonical.

### Note
- The shared `myPOS Legal/` memory file goes live again on the next filing. Entries written to the private `myPOS Legal 1/` copy since the two diverged are not carried over automatically; migrate that history manually if the shared file needs it.

### Plugin version
- Bumped to **0.4.5**.

---

## [0.4.4] - 2026-07-03

Reality-alignment release, driven by the first weekly usage review (30 days of session transcripts, n8n execution logs, and a live tool inventory). Removes every reference to Microsoft 365 tools that do not exist, hardens scheduled sweeps against mid-run death, and reworks `/reply-and-close` to a lawyer-sends-first flow.

### Fixed
- **Nonexistent M365 tools removed everywhere.** The Microsoft 365 connector is read-only (search + read_resource). `outlook_email_create_draft`, `outlook_email_send`, `sharepoint_read_file`, `sharepoint_list_items`, `sharepoint_upload_file` and `sharepoint_create_folder` have never existed in it; 0 Outlook drafts were created in 15 sessions. All commands and skills now use the real read pattern (`sharepoint_search` -> `read_resource`) and deliver drafts via the AI Triage Jira comment + the SharePoint `.docx`. `/triage` Step 9 optionally calls the (not yet deployed) n8n `legal-copilot-draft` workflow and degrades silently when absent.
- **`/reply-and-close` reworked to manual-send.** The old flow depended on finding an Outlook draft (never created) and a send tool (does not exist). New flow: the lawyer sends from Outlook; the command locates the sent email (`outlook_email_search` in Sent Items), diffs it against the AI draft from the Jira comment, files it via n8n as a rendered `.html` (raw `.eml` MIME is not retrievable), captures edit feedback to the memory file, and transitions the ticket. Risk-gate confirmation rules preserved.
- **Stale `myPOS Legal/` path purged from `/triage-board`, `/triage-inbox`, `/reply-and-close`, `jira-auto-assign`.** These four still pointed at the pre-0.4.3 base path; a stale duplicate memory file at that path (last write 2026-05-19) confused three June sessions. All references now use `myPOS Legal 1/` and explicitly forbid reading the stale copy.
- **`/triage-board` Phase 2 order: transition first, then assign, then verify.** The LEGAL project's "Start Progress" post-function reassigns tickets to the acting account; it overwrote correct owners 6 times in June (5 on 2026-06-26, 1 on 2026-07-02). The assignee is now set after the transition and re-verified.
- **`sharepoint-filer`: curl fallback removed.** Sandbox egress blocks the webhook (403 on CONNECT, observed 2026-06-02); the n8n MCP `execute_workflow` tool is the only transport. New error code `n8n_mcp_unavailable`. Also: `success_but_empty` uploads now count as failures (that is how the 12/13-byte stubs slipped through), and `memory_instructions` is documented as a markdown string, never a JSON object (two raw JSON blobs were pasted into the memory file on 2026-05-27).

### Added
- **`/triage-board` Phase 0: context budget.** 4 of 12 June sweeps died mid-run with no report, consistent with context exhaustion from 30-50K-token `getJiraIssue` payloads. Hard rules: compact-field JQL for all scans, one full ticket at a time, per-ticket subagent isolation, max 3 new triages per sweep, and a per-phase checkpoint line to a local run log so an interrupted run leaves evidence of what was already written to Jira.
- **Re-triage cap** in `/triage` Step 3 and `/triage-board` Phase 3: a ticket with an AI Triage comment and no human reply after it is never re-triaged (LEGAL-4990 was triaged 4 times in 21 days); SLA breach produces one escalation line instead.
- **Knowledge-seed fallback** in all commands and `jira-auto-assign`: when `${CLAUDE_PLUGIN_ROOT}/knowledge/` is unreadable (app-internal path; killed the 2026-07-01 sweep), fetch `team-routing.md` / `patterns.md` / `sharepoint-map.md` from the SharePoint `_knowledge` folder; degrade to conservative defaults with a flag if missing there too.
- **`/triage` Step 2 attachment gate:** if the ticket's substance is an unreadable attachment, stop and ask for it in chat instead of classifying from the summary (the 2026-06-03 MVR redraft stalled on exactly this).
- **`/triage-inbox` idempotency without mail writes:** the connector cannot mark emails read or tag them. Tickets now embed `Source-Message-Id`, processed IDs are logged to the memory file and skipped on the next sweep. The unread-filter and category-tagging instructions are gone.

### Not in this release (queued for 0.5.0, tracked as n8n proposals)
- Workflow-side `documents_from_jira` + `text_documents` fields (server-side attachment fetch; removes the 40-60 KB inline-base64 ceiling that kept real drafts out of SharePoint all June).
- New `legal-copilot-draft` n8n workflow (real Outlook drafts via Microsoft Graph under the service account; needs Mail.ReadWrite consent).
- Archiving the inactive duplicate workflow `Legal Copilot copy - NS`.

### Plugin version
- Bumped to **0.4.4**.

---

## [0.4.3] - 2026-06-03

Filing reliability fix -- stop the "receipt-only" silent failure pattern, correct the live SharePoint base path, and document the strict n8n payload contract.

### Fixed
- **`sharepoint-filer` SKILL.md** -- expanded Hard rules. Three new prohibitions:
  - NEVER substitute a `.txt` "receipt" for the actual document. If the real `.docx` cannot be inlined as `content_base64` inside one MCP tool call, the skill MUST return `success: false` with `error: "document_too_large_to_inline"` -- not silently degrade to a receipt stub. The lawyer never re-runs `/file-to-sharepoint`, so a receipt-only filing leaves the case folder permanently empty.
  - NEVER invent alternative payload shapes (`file_manifest`, `payload_path`, `documents_from_url`). The workflow accepts exactly one shape: `documents: [{filename, content_base64, mime_type}]`. Anything else throws `Document item missing filename or content_base64` (see execution 6727 on 2026-05-27 -- LEGAL-4912 hard-failed because the caller sent `file_manifest` + `payload_path` instead).
  - NEVER include the ticket key in `case_folder`. The workflow concatenates `case_id` itself. Sending `Claims/LEGAL-4912` produces `.../Claims/LEGAL-4912/LEGAL-4912/` (see execution 6727).
- **`sharepoint-filer` SKILL.md** -- documented the correct live SharePoint base path: `myPOS Legal 1/...` (note trailing ` 1`), not `myPOS Legal/...`. The `Prepare State` node in workflow `VAKq9Bra0RA0SdCO` hard-codes the former; previous SKILL docs were stale.
- **`sharepoint-filer` SKILL.md** -- added a "Known-good caller pattern" Python snippet so future agents reuse the proven shape.
- **`/triage` Step 8** -- explicit and prominent: on `success: false` HALT, surface the workflow error, and do NOT proceed to Step 9 (Outlook draft) or Step 10 (AI Triage Jira comment). A failed filing with a clear error is better than a "successful" filing of a receipt stub that points the lawyer at an empty SharePoint folder.

### Diagnosed (not fixed in this release -- planned for 0.5.0)
- Workflow `VAKq9Bra0RA0SdCO` has no server-side document-fetch path. The caller must inline ~50 KB of base64 per docx into one MCP tool call, which agents intermittently punt on. 0.5.0 will add a `documents_from_jira: [{filename, jira_attachment_url, mime_type}]` field to the workflow plus an Atlassian Bearer Token credential, so n8n fetches Jira attachments server-side and the caller only ever inlines the (sub-100 KB) AI-generated draft.

### Plugin version
- Bumped to **0.4.3**.

---

## [0.4.2] - 2026-06-02

Repo moved to the myPOStech GitHub org.

### Changed
- `plugin.json` `homepage` and `repository` now point to `https://github.com/myPOStech/mps-legal-legalcopilot-ai`.
- README install instruction updated to `/plugin marketplace add myPOStech/mps-legal-legalcopilot-ai`.
- Plugin version bumped to **0.4.2**.

---

## [0.4.1] - 2026-06-02

Expertise map confirmed by the head of legal.

### Added
- **Daniela Chavdarova** added to the active roster in `knowledge/team-routing.md`.

### Changed
- Replaced the placeholder expertise map with the head-of-legal-confirmed routing:
  - Ivan Troyanov -- Corporate changes, GTCs, Regulatory, Contract Review, Claims
  - Atanas Rusenov -- Regulatory, Claims, Projects, Inspection Support, Contract Review
  - Daniela Chavdarova -- GTCs, Claims, Regulatory, Contract Review
  - Denitsa Dimitrova -- Contract Review, NDA, KYC
  - Jay Manjdadria -- NDA, KYC
  - Nikolay Saragerov -- Projects, GTCs, Inspection Support, Materials Review, Contract Review
- Plugin version bumped to **0.4.1**.

---

## [0.4.0] - 2026-06-02

Auto-routing, native Jira fields, three-pass review, and an on-demand productivity report.

### Added
- **`jira-auto-assign` skill** -- picks a Legal team member based on the matter type, severity, and a soft capacity cap, then sets the Jira `assignee` field directly. Reads routing rules from `knowledge/team-routing.md` so the team can change the map without code changes.
- **`jira-fields-and-flags` skill** -- writes `priority`, `duedate`, merged `labels`, and the system `Flagged` field directly to the Jira ticket. Deadlines, risk markers and SLA dates are no longer hidden inside comment threads; they're filterable.
- **`business-reviewer` subagent** -- third review pass that runs after the Devil's advocate. Reconciles the legal-triage draft with the Devil's advocate findings from a commercial / partner-relationship angle and produces a final verdict (`approve` / `revise` / `escalate`). Surfaces a one-line summary into the Jira comment alongside the Devil's advocate verdict.
- **`legal-productivity-report` skill + `/legal-report` command** -- on-demand report covering overdue tickets, per-lawyer load, median time-to-close, and a per-lawyer off-Jira-hours prompt (manual entry; not scraped from calendars). Files a `.docx` to `myPOS Legal/Reports/Productivity/`.
- **`knowledge/team-routing.md`** -- single editable source of truth for team roster, expertise map, severity overrides, senior reviewer, and the soft capacity threshold.

### Changed
- `/triage` and `/triage-board` now call `jira-auto-assign`, `jira-fields-and-flags`, and `business-reviewer` in addition to the existing pipeline. Priority, due date, assignee and the flag field are no longer posted as Jira comments -- they live on the ticket fields.
- `/triage-board` runs a new **Phase 2: last-replier reconciliation** before the To-Do scan. If a Legal team member has already replied to a To-Do ticket, the ticket is reassigned to that lawyer and transitioned from To Do to In Progress. The 14-day-old reply rule prevents stale reassignments.
- `/triage-board` adds an overload signal section in the summary (any assignee with >= 3 overdue tickets).
- The plugin's GitHub homepage and repository fields point to `https://github.com/AtanasRRusenov/mps-legal-legalcopilot-ai` to conform to myPOS GitHub naming conventions.
- Plugin version bumped to **0.4.0**.

### Hard rules (new)
- The Copilot never reassigns an already-assigned ticket from `/triage`. Reassignment is exclusive to `/triage-board`'s Phase 2 last-replier rule.
- The business reviewer cannot approve a draft on a risk-flagged matter (regulator / inspection / claim / tight deadline). Commercial reasoning does not overrule risk gates.
- The productivity report never publishes an "under-utilised" label in the shareable body of the report. That signal is private to the head of legal and goes in a separate section of the docx.

---

## [0.3.0] - 2026-05-07

Filing layer rewritten on top of the team's n8n `Legal Copilot` workflow.

### Changed
- **SharePoint filing now goes through n8n** (workflow ID `VAKq9Bra0RA0SdCO`, webhook `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`). The Microsoft 365 MCP cannot reliably write to SharePoint, so every write -- documents and the shared memory file -- is routed through the n8n workflow, which uses the SharePoint REST API directly via a service-account OAuth credential.
- The local Desktop mirror has been retired. There is no longer a `~/Desktop/Legal Copilot/` tree, no per-lawyer `_knowledge/` cache, and no manual drag-and-drop step. Lawyers no longer need to mirror anything by hand.
- The team's shared knowledge collapses into a single SharePoint markdown file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`. Every n8n filing run appends a `## Case: {key}` block to it.
- Case folders are now keyed on the Jira ticket key (`myPOS Legal/{matter-folder}/{TICKET-KEY}/`) instead of the sanitised ticket summary. Re-filing a ticket is idempotent.
- `sharepoint-filer` skill rewritten to call the n8n workflow.
- `/triage`, `/triage-board`, `/triage-inbox`, `/reply-and-close`, `/file-to-sharepoint` updated to use the n8n flow end-to-end.
- `/setup-copilot` simplified -- it no longer creates a local Desktop tree or seeds local knowledge files. It now verifies Atlassian, Microsoft 365 (read), and n8n (write) connectivity.
