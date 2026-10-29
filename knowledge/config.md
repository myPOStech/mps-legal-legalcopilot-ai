# Configuration

Policy version: 0.5.1 (ships with plugin 0.5.1; the copy in the SharePoint `_knowledge` folder must carry the same version line, `/setup-copilot` checks it)

What the team has configured for the Legal Copilot. This file ships with the plugin and documents the canonical settings; the live shared state lives on SharePoint at `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`.

## Jira board

- **Cloud ID:** `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- **Projects watched:** `LEGAL`
- **To-Do status:** `To Do`
- **Default JQL:** `project = LEGAL AND status = "To Do" ORDER BY created ASC`

## SharePoint

- **Site:** `https://mypos0.sharepoint.com/sites/legal`
- **Root path for filed cases:** `Shared Documents/myPOS Legal/`
- **Memory file:** `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`
- **Top-level matter folders:** see `knowledge/sharepoint-map.md`

## n8n filing workflow

All SharePoint writes are routed through the team's n8n `Legal Copilot` workflow (because the Microsoft 365 MCP cannot reliably write to SharePoint).

- **Workflow name:** `Legal Copilot`
- **Workflow ID:** `VAKq9Bra0RA0SdCO`
- **Production webhook:** `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`
- **HTTP method:** `POST`
- **Available in MCP:** yes -- callable via `mcp__n8n__execute_workflow` with `executionMode: "production"`
- **Required body fields:** `case_id`, `case_folder`, `documents` (array of `{filename, content_base64, mime_type}`), `memory_instructions`
- **What it does:** creates `myPOS Legal/{case_folder}/{case_id}/`, uploads each document, downloads + appends + re-uploads the memory file, returns a JSON summary

The Microsoft 365 MCP is still used for SharePoint **reads** (the read path is `sharepoint_search` followed by `read_resource`, as in `/triage` Step 1); every write goes through the n8n workflow above.

## SLA policy and two-touch rule

- **SLA tiers and due dates:** `knowledge/sla-policy.md` (single source; the legal-triage skills' own SLA suggestions are not used for due dates)
- **Two-touch rule and resolution paths:** `knowledge/two-touch-rule.md`

## Intake forms (beta)

Spec: `docs/n8n-intake-form.md`. `/triage` only sends a form link when the URL below is not `TBD`.

- **Contract review form URL:** `https://myposai.app.n8n.cloud/form/legal-contract-review`
- **Regulatory question form URL:** `https://myposai.app.n8n.cloud/form/legal-regulatory-question`
- **Prefill parameter:** append `?existing_ticket=LEGAL-NNNN` to either URL; n8n prefills the "Existing LEGAL ticket" field from the query parameter (keyed on the field name, production URL only). Example: `https://myposai.app.n8n.cloud/form/legal-contract-review?existing_ticket=LEGAL-6014`
- **Access:** both forms ask for the shared basic-auth login (n8n credential `Legal Intake form login`); the login is published on the internal Legal SharePoint page next to the links, never in a draft to a requester
- **n8n workflow:** `Legal Intake` (`FicMGfmDaJxaXBym`); comment mode built and live-tested on 29 Oct 2026 (LEGAL-6014 comment mode, LEGAL-6015 not-found fallback)
- **Mode:** when the form names an existing `LEGAL-*` ticket (the Copilot's link prefills it), the answers are posted as a comment on that ticket and the Copilot re-triages it; a new ticket is created only when no ticket is named or the named key does not exist
- **Labels added by the form:** `intake:form`, `intake:contract-review-v1` / `intake:regulatory-question-v1` (on the ticket the answers landed on)