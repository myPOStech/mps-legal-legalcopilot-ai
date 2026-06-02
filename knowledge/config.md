# Configuration

What the team has configured for the Legal Copilot. This file ships with the plugin and documents the canonical settings; the live shared state lives on SharePoint at `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md`.

## Jira board

- **Cloud ID:** `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- **Projects watched:** `LEGAL`, `AIRD`
- **To-Do status:** `To Do`
- **Default JQL:** `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`

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

The Microsoft 365 MCP is still used for SharePoint **reads** (the read path i