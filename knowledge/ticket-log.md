# Ticket log (seed bundled with the plugin)

This file ships with the plugin. The live ticket log is **not** a separate file -- it lives as a sequence of `## Case: {ticket_key}` blocks inside the shared SharePoint memory file:

```
myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

Every n8n filing run (triggered by `/triage`, `/triage-board`, `/triage-inbox`, `/file-to-sharepoint`, or `/reply-and-close`) appends a new `## Case:` block to the memory file. The block includes a timestamp, the matter type, priority, SLA, jurisdictions, risk flags, the Devil's advocate verdict, the list of filed documents (with SharePoint URLs), and the action taken.

Older case entries remain in the memory file as the team's audit trail. They're not pruned -- the memory file is the canonical record of what the Copilot has done. (Older tickets remain independently searchable in Jira.)

## Format of a case block (inside t