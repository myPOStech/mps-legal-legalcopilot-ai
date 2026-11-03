# Changelog

All notable changes to the myPOS Legal Copilot plugin.

## [0.5.2] - 2026-10-30

The intake-form invitation goes out automatically; forms are open links; SharePoint upload failures answer a clear error.

### Changed
- **Automatic intake invite (the one agreed exception to "the Copilot never sends").** `/triage` Step 4b item 4 no longer puts the form link in a draft. When a `contract_review` / `regulatory_question` ticket is on the `one_question_round` path with 3+ blocking gaps, is two-touch eligible, has no risk flag, did not come from the form and was not invited before, `/triage` calls the n8n `Legal Intake` workflow's `Invite webhook` trigger. The workflow sends a fixed, clearly labelled AI-generated invitation from the Legal mailbox with the link and ticket key prefilled, posts an `Intake-Invite: sent` comment and adds the label `intake:form-sent`. The Copilot cannot change the text and never calls a send tool. On any failure it falls back to asking its (max 3) questions in the draft. Step 9 creates no Outlook draft when the invite was sent; the Step 10 comment says the ticket is waiting for the answers. Hard rules in `/triage`, `/triage-board`, `/triage-inbox`, `/reply-and-close`, the README safety rails and `hooks/README.md` state the exception.
- **`/triage` Step 3:** `Intake-Invite: sent` and `Intake-Form:` comments are system comments whoever authored them; the first does not lift the re-triage cap, the second does.
- **`/reply-and-close` Step 4b:** the invite counts as one touch, read from the `intake:form-sent` label.
- **Forms are open links and ask no identity.** The shared basic-auth login is dropped (both form triggers: authentication none). The fields "Your name", "Your myPOS email", "Your team" and "What happens if we miss the deadline" are removed: the invitation link carries the ticket number (now required) and the requester is the person on that ticket. A missing or unknown ticket number ends on an error page with nothing saved; the old new-ticket mode is gone. The contract form gains "If we are the customer: what are we buying, in a sentence" (team feedback), written into the block as `What we are procuring (if we are the customer):`. `/triage` Step 2 reads the form block from the comment only (the 29 Oct test tickets with the block in the description are still read). `knowledge/config.md` (policy version 0.5.2: copy it to SharePoint again) and `docs/n8n-intake-form.md` updated, including the invite text and the trigger contract.
- **`sharepoint-filer`:** new error code `sharepoint_upload_failed` (HTTP 502 from the filing workflow once change 1 of the 30 Oct review is published): a document upload failed after 3 retries, nothing was recorded, retry the whole filing. `docs/n8n-legal-copilot-workflow.md` documents the change and the guard added to it.

### n8n side (outside this repo, both drafts, publishing is manual)
- `Legal Intake` (`FicMGfmDaJxaXBym`): invite trigger branch (10 nodes), forms without login or identity fields, new-ticket path removed. Needs a Microsoft Outlook credential on `Send intake invite` before publishing.
- `Legal Copilot` (`VAKq9Bra0RA0SdCO`): change 1 of the 30 Oct weekly review, with a guard so a partial upload failure cannot answer 200.

### Plugin version
- Bumped to **0.5.2**. Run `/plugin update`; copy `config.md` to the SharePoint `_knowledge` folder; run `/setup-copilot`.

---

## [0.5.1] - 2026-10-29

Intake forms go live: production URLs in `knowledge/config.md`, prefill parameter corrected, workflow built and tested.

### Changed
- **`knowledge/config.md`**: the two intake form URLs replace `TBD` (`https://myposai.app.n8n.cloud/form/legal-contract-review` and `.../legal-regulatory-question`), plus the prefill parameter `?existing_ticket=LEGAL-NNNN` and the access model (shared basic-auth login published on the internal Legal SharePoint page). From this version `/triage` Step 4b sends the form link on the `one_question_round` path for contract review and regulatory question tickets with 3+ blocking gaps. Policy version line bumped to 0.5.1: copy the file to the SharePoint `_knowledge` folder again.
- **`docs/n8n-intake-form.md`**: prefill is keyed on the field **name** (`existing_ticket`), not the label, and works only on the production URL; status, access model, go-live record of the 29 Oct live test (LEGAL-6014, LEGAL-6015) and the remaining open items.

### n8n side (outside this repo)
- `Legal Intake` workflow `FicMGfmDaJxaXBym`: comment-mode branch built (ticket lookup, comment as ADF, additive labels via REST, not-found fallback), credentials attached, three live runs passed. Publishing is a manual step in n8n.

### Plugin version
- Bumped to **0.5.1**. Run `/plugin update`; copy `config.md` to the SharePoint `_knowledge` folder; run `/setup-copilot`.

---

## [0.5.0] - 2026-10-22

Longer, risk-based SLAs; a two-interaction target for non-critical tickets; intake form beta (spec only).

### Added
- **`knowledge/sla-policy.md`** -- one SLA policy for every triage. Tiers: `critical` 1 day, `high` 7 days, `standard` (non-critical) 14 days, `project-pressing` 90 days, `project-standard` 180 days, all in calendar days, with a separate first-response target per tier. A hard external deadline pulls the due date earlier (2 business days before it; 10 for projects). The legal-triage skills' own SLA suggestions (4h / 24h / 3 / 5 business days) are no longer used for due dates.
- **`knowledge/two-touch-rule.md`** -- non-critical tickets (`standard`, and `high` without a critical flag) should close in at most 2 interactions with the requester. Each triage picks a resolution path (`answer_now`, `answer_on_assumptions`, `one_question_round`, `full_track`), asks at most 3 blocking questions, and turns the rest into stated assumptions. Critical tickets, projects, claims and inspections are exempt.
- **`docs/n8n-intake-form.md`** -- spec for the `Legal Intake` n8n workflow (contract review + regulatory question forms), built in n8n as an unpublished draft (`FicMGfmDaJxaXBym`). Form URLs in `knowledge/config.md` are `TBD`, so the Copilot never sends a link until they are filled in.
- `/triage` Step 4b: applies the SLA policy and the two-touch rule after classification. The AI Triage comment now shows the SLA tier, first-response target, resolution path, the questions asked and what was assumed.
- `/triage` Step 2 reads `Intake-Form:` blocks (ticket description or comment) as given facts.

### Changed
- `jira-fields-and-flags`: `sla_days` range 1-180 (was 1-90); adds `sla:{tier}` and `path:{resolution_path}` labels; keeps a due date that is already on the ticket.
- `jira-auto-assign`: `sla_days` range 1-180.
- `triage-reviewer` (Devil's advocate): on two-touch tickets, gaps become assumptions or caveats, not new questions.
- `business-reviewer`: fourth axis "interaction budget"; fixes over-long question lists itself and returns `revise` if the draft still needs more than 3 questions or a multi-step path.
- `/triage-board` Phase 4 SLA watchdog: SLA window is now `duedate - created`, tier read from the `sla:*` label. Tickets with no `duedate` (triaged before the Copilot set Jira fields) are skipped by design; a lawyer can set a due date by hand to bring one back under the watchdog.
- `/reply-and-close`: counts touches with the requester, labels `touches:{N}` and `two-touch-breached`.
- `/legal-report`: reports the two-touch rate by matter type and the intake form share.

### Fixed (weekly usage review of 23 Oct 2026, FIX-A to FIX-E)
- **Jira attachments no longer abort SharePoint filing (FIX-A, FIX-B).** n8n Cloud's SSRF guard blocks `documents_from_jira` fetches (execs 25235, 26726), which left the `LEGAL-5886` and `LEGAL-5915` case folders empty. `sharepoint-filer` now skips signature / inline images (image files only: `image001.png` etc., or an image under 20 KB; a non-image is never skipped on size), sends real attachments up to 6 MB raw (about 8 MB base64) inline in `documents`, uses `documents_from_jira` only above 6 MB, and caps a call at the workflow's 10 documents. Anything not filed is returned in `attachments_not_filed` and shown as "Attachments not filed: ..." in the Jira comment. If a case folder ends up with 0 files, the filer says so and retries once with `documents` only. Same rules in `/triage` Step 8, `/file-to-sharepoint` and `/reply-and-close`. Server side: a draft version of the n8n `Legal Copilot` workflow (`8773470a`) reports blocked fetches as `fetch_failed` instead of stopping (`docs/n8n-legal-copilot-workflow.md`); **it is live only once published**.
- **Outlook drafts are verified and always explained (FIX-C).** `/triage` Step 9: subject must be exactly `[AI DRAFT {key}] Re: {original subject}`; after creating, search Drafts for it, retry once, then the n8n fallback only if `zMZJM8RvzluqLIrl` is active. Every triage, and the `/triage-board` summary, prints `Outlook draft: created (...)` or `Outlook draft: NOT created, reason: ...`. No prefixed draft had appeared since 14 Oct.
- **Unattended runs no longer auto-decline connector calls (FIX-D, reworked in review on 29 Oct).** The original fix added the `mcp__Microsoft_365__*` and `mcp__Atlassian_MCP__*` names to `settings.json`. Review found that a plugin's `settings.json` keeps only `agent` and `subagentStatusLine` on install, so no permission list the plugin ever shipped reached a lawyer's machine. `settings.json` is removed. The plugin now ships a `PreToolUse` hook (`hooks/hooks.json`, see `hooks/README.md`) that allows the Copilot's read, draft-create, Jira and n8n tools by name under any server prefix (`mcp__atlassian__`, `mcp__Atlassian_MCP__`, `mcp__claude_ai_...__`, `mcp__plugin_mypos-legal-copilot_...__`, UUID-prefixed Cowork connectors). Send tools, direct SharePoint writes and the generic Atlassian `execute*` tools are never allowed; user and organisation `deny` / `ask` rules still win. `docs/permissions-snippet.json` is the optional explicit list for a user's own `.claude/settings.local.json`. `/setup-copilot` Step 4 checks that the smoke test ran without prompts.
- **Prompt templates (FIX-E).** `docs/prompt-templates.md` adds 1b (attachments + NDA guard + Outlook draft line) and 6 (weekly health check). `/triage` Step 4 also says to use `anthropic-skills:legal-triage-nda`, never `legal:triage-nda`.

### Review fixes (29 Oct 2026, before merge)
- **Business reviewer `revise` on a two-touch ticket is no longer reported as `approve`.** `/triage` Step 6 reconciled "Devil's advocate approve + business revise" as `approve` in every case, which let an over-budget draft through. On a two-touch-eligible ticket that verdict now stays `revise`, sets `human_review_required`, and the AI Triage comment says the two-touch budget could not be met.
- **Size rule for skipped attachments is image-only.** The 20 KB rule now applies to image files only (`image001.png` etc., `.png` / `.jpg` / `.jpeg` / `.gif`, `image/*`); a short `.docx` or `.pdf` is never skipped because of its size. Same wording in `/triage` Step 8, `/file-to-sharepoint` and `/reply-and-close`.
- **Inline limit is 6 MB raw.** The workflow's ~8 MB limit is on the base64-encoded document; base64 adds a third. The filer also caps a call at the workflow's 10 documents (draft first, then final email / comments, then attachments smallest first) and reports the rest in `attachments_not_filed` instead of getting a 422.
- **Policy files are checked, not just copied.** `sla-policy.md`, `two-touch-rule.md` and `config.md` carry a `Policy version:` line; `/setup-copilot` Step 3 item 8 reads the SharePoint `_knowledge` copies and reports missing or stale ones; prompt template 6 (weekly health check) compares them too.
- **Intake-form answers land on the same ticket.** The spec had the form create a second ticket, which would have undercounted touches by one (touch 1 on the original, touch 2 on the new ticket) and left the original open. `docs/n8n-intake-form.md` now has a comment-mode branch: the Copilot's link prefills the ticket key, the answers are posted as an `Intake-Form:` comment on that ticket with the `intake:form` label, and `/triage` Step 3 treats that comment as the requester's reply and re-triages. A new ticket is created only when no ticket is named. Same wording in `knowledge/two-touch-rule.md`, `knowledge/config.md`, `/triage` Steps 2 and 4b and `/reply-and-close` Step 4b.
- **Kept due dates are explained.** `/triage` Step 10 and 11 print "existing date kept; policy would set ..." when `jira-fields-and-flags` returns `due_date_kept`.
- **Jira tool names by suffix.** `/triage` Step 10, `/reply-and-close` Step 8, `/triage-board` 2b and `/setup-copilot` no longer hard-code `addCommentToJiraIssue` / `getTransitionsForJiraIssue`; the newer Atlassian connector exposes `addOrEditJiraIssueComment` and no transitions lookup (pass the transition by name).
- Smaller: `sla-policy.md` states the P1-P4 to Highest/High/Medium/Low/Lowest mapping and the `sla_days` cap is 180 (the longest tier) in the policy, `jira-fields-and-flags` and `jira-auto-assign`; `config.md` no longer ends mid-sentence; `triage-reviewer` and `/triage` Step 4b say that on `full_track` tickets `questions_to_ask` is the uncapped missing-information list; `/legal-report` always prints the two-touch line; `/triage-inbox` names Step 4b; `commands/setup-copilot.md` had 2 KB of NUL padding at the end since 14 Oct, removed.

### Not in this release
- The published `anthropic-skills:legal-triage-*` skills still print their old SLA suggestion, and `legal-triage-project` still defines SLA as time to the scoping call. They are outside this repo; the policy file overrides them for due dates.

### Plugin version
- Bumped to **0.5.0**. Run `/plugin update`, copy `sla-policy.md`, `two-touch-rule.md` and `config.md` (each with its `Policy version: 0.5.0` line) to the SharePoint `_knowledge` folder so scheduled runs that cannot read the plugin folder pick them up, then run `/setup-copilot` to confirm the copies and that the permission hook is active.

---

## [0.4.9] - 2026-10-13

AIRD project removed from scope.

### Changed
- **The Copilot now watches the `LEGAL` Jira project only.** Every JQL that used `project in (LEGAL, AIRD)` now uses `project = LEGAL`: `/triage-board` (board scan, last-replier reconciliation, lawyer-corrections check, SLA watchdog, overdue check), `/triage` and `/triage-inbox` dedupe searches, `jira-auto-assign` capacity check, and the three `legal-productivity-report` queries. AIRD tickets are AI/R&D product work, not legal matters, and were taking the oldest slots under the 3-per-sweep triage cap.
- `/triage-inbox` always creates new tickets in `LEGAL` (the AIRD option is gone).
- `knowledge/config.md`, `/triage`, `/legal-report` and `docs/pilot-runbook.md` list `LEGAL` as the only watched project.

### Plugin version
- Bumped to **0.4.9**. Run `/plugin update` so scheduled runs pick it up.

---

## [0.4.8] - 2026-10-07

Applied from the weekly usage review of 2 Oct 2026 (FIX-01 to FIX-06).

### Fixed
- **Outlook drafts (FIX-01).** `/triage` Step 9 now prefers `outlook_create_reply_draft` when the original Outlook message id is known (keeps the draft in the thread), else `outlook_create_draft`. Subject `[AI DRAFT {key}] Re: ...`, AI DRAFT banner, requester only. The n8n fallback now searches for the real workflow name `Legal Copilot Draft` (`zMZJM8RvzluqLIrl`) and uses it only when active; it was searching for `legal-copilot-draft`, which never matched. A missing draft never halts a triage; the reason is recorded. `/triage-inbox` and `/reply-and-close` no longer claim that no draft tool exists. Send tools (`outlook_send_mail`, `outlook_send_draft`, `outlook_forward_mail`) are explicitly forbidden. `settings.json` allows `outlook_create_reply_draft`.
- **Devil's advocate review ran against a placeholder (FIX-02).** `agents/triage-reviewer.md` called `devils-advocate-review`, which resolves to this plugin's own stub. It now calls `anthropic-skills:devils-advocate` and treats any loaded text containing "STUB" as not resolved. `/triage` Step 4 and `/triage-board` invoke the published `anthropic-skills:legal-triage-*` skills by full name. All 11 stub skills now have the description `DO NOT INVOKE` so they no longer trigger on their own.
- **sharepoint-filer contract (FIX-03).** Removed the stale "Planned 0.5.0" note: `text_documents` and `documents_from_jira` are live. Jira attachments are now passed by URL in `documents_from_jira` (no download, no size problem); the sent email goes in `text_documents`. Every `.docx` is zip-validated before upload (`docx_invalid` on failure); without a sandbox shell the draft is filed as `.md` instead of hand-assembled base64. Same changes in `/file-to-sharepoint`, `/reply-and-close` and `/triage` Step 8.
- **n8n MCP endpoint (FIX-05).** `.mcp.json` pointed at `https://myposai.app.n8n.cloud/mcp`, which returns 404 (ENDPOINT_NOT_FOUND since 24 Aug). Now `https://myposai.app.n8n.cloud/mcp-server/http` (verified 7 Oct 2026: returns 401 without auth, i.e. the endpoint exists).

### Changed
- **n8n `Legal Copilot` workflow (FIX-04, server side, published 7 Oct 2026).** .docx ZIP-header check, `case_folder` validation, HTTP 422 JSON errors, retries with backoff, automatic memory-file archiving above 250,000 characters, and a guard that stops the run instead of overwriting the memory file when it cannot be read. Details and rollback: `docs/n8n-legal-copilot-workflow.md`.
- **`settings.json` (FIX-05).** Allow-listed read-only tools used by scheduled runs: `sharepoint_search`, `read_resource`, `get_granted_scopes`, n8n `search_workflows`, `search_workflow_executions`, `get_workflow_execution`, `get_workflow_history`.

### Added
- `docs/prompt-templates.md` (FIX-06): templates for single-ticket triage, contract review, regulatory questions, translation, and a connector preflight line.
- `docs/n8n-legal-copilot-workflow.md`: accepted body, responses, and change log of the filing workflow.

### Plugin version
- Bumped to **0.4.8**. Cowork installs were still on 0.4.5 on 2 Oct 2026; run `/plugin update` (or reinstall via the plugin customizer) so 0.4.6 to 0.4.8 actually reach the scheduled runs.

---

## [0.4.7] - 2026-10-07

Routing change.

### Changed
- **`project` matter type now routes to Atanas Rusenov first.** In `knowledge/team-routing.md` the primary owner for `project` moved from Nikolay Saragerov to Atanas Rusenov; Nikolay Saragerov is now the secondary. Severity override, capacity rule and senior-reviewer fallback are unchanged. The SharePoint `_knowledge/KNOWLEDGE-SEEDS/team-routing.md` live copy was updated to match.

### Plugin version
- Bumped to **0.4.7**.

---

## [0.4.6] - 2026-07-27

Applied from the weekly usage review. Corrects the Outlook draft tool name and aligns the filing input contract.

### Fixed
- **Outlook draft tool name.** `/triage` Step 9 and `/triage-board` no longer assert that no draft tool exists. They now match a create-draft tool by suffix `outlook_create_draft` (the connected Microsoft 365 connector's real name, verified on LEGAL-5438, 15 Jul 2026) and create a native draft in the lawyer's Outlook Drafts folder when present, falling back to the Jira comment + SharePoint .docx. The prior wrong name `outlook_email_create_draft` is removed from Step 9, the hard rules, and the `settings.json` allow-list. The phantom `outlook_email_send` entry is removed (the Copilot never sends).
- **sharepoint-filer input contract.** The Inputs table previously documented a `files: [{name, source, content_or_url, role}]` shape that did not match the n8n workflow. It now documents the real contract the workflow accepts: `documents` (base64), `text_documents` (plain text; `content_text` or `content`), and `documents_from_jira` (server-side fetch). Removes the three-vocabulary drift.

### Plugin version
- Bumped to **0.4.6**.

---

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
