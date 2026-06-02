# Learned patterns (seed bundled with the plugin)

This file ships with the plugin. It's the seed knowledge every fresh install starts from. Once the team is running, the live patterns are recorded inside the shared SharePoint memory file at:

```
myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

The board sweep (`/triage-board`, Phase 1c) appends a `## Pattern: ...` block to that file when 3+ lawyer corrections share a theme. Patterns are NEVER auto-applied -- a senior lawyer reviews each proposed pattern in the memory file before the maintainer folds it into the next plugin release (i.e., back into this seed file).

## How to add a pattern manually

A senior lawyer can add a pattern at any time by editing the memory file on SharePoint. Use this format inside the file (after the active `## Case:` blocks):

```markdown
## Pattern: <one-line title>
*