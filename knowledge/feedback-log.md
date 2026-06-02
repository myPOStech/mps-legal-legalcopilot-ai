# Feedback log (seed bundled with the plugin)

This file ships with the plugin. It's the seed format and starting point. The live feedback log lives inside the shared SharePoint memory file at:

```
myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

Captures the diff between the AI draft and what the lawyer actually sent. The close-out command (`/reply-and-close`) appends a `### Lawyer feedback` block to the case entry inside the memory file. The board sweep (`/triage-board`, Phase 1b) reviews these blocks across the most recent week and proposes patterns when 3+ entries share a theme (same category + matter type + jurisdiction).

## Format

Inside `legal_copilot_memory.md`, each lawyer-feedback entry is nested under the case's `## Case: {ticket_key}` block:

```markdown
### Lawyer feedback ({ticket_key})
**Skill used:** <legal-triage-* skill>
**AI draft excerpt:** "<text the AI wrote>"
**