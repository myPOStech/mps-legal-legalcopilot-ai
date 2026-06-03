# Changelog

All notable changes to the myPOS Legal Copilot plugin.

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
