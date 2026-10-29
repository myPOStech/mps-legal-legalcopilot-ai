# Permission hook for unattended runs

`hooks.json` registers one `PreToolUse` hook. When Claude Code is about to call a tool whose name matches the regex in `matcher`, the hook prints a JSON `allow` decision and the call goes ahead without a permission prompt. That is what lets the 08:00 `/triage-board` sweep run unattended.

Why a hook and not `settings.json`: a plugin's `settings.json` keeps only the `agent` and `subagentStatusLine` keys when it is installed; a `permissions.allow` list in it is dropped, so the allow-list the plugin shipped up to 0.4.9 never reached a lawyer's machine (Claude Code docs, plugins reference, "settings").

## What the matcher covers

Server prefixes seen so far, any of which may front the same connector:

| Prefix | Where it appears |
|---|---|
| `mcp__atlassian__`, `mcp__microsoft-365__`, `mcp__n8n__` | user-configured servers |
| `mcp__Atlassian_MCP__`, `mcp__Microsoft_365__` | October 2026 desktop runs |
| `mcp__claude_ai_Atlassian__`, `mcp__claude_ai_Microsoft_365__` | connectors Claude Code fetches from claude.ai |
| `mcp__plugin_mypos-legal-copilot_<server>__` | the servers bundled in this plugin's `.mcp.json` |
| `mcp__<uuid>__` | claude.ai / Cowork connectors |

Tool suffixes: Jira read and write (issue, comment, link, transition, create), Outlook read and draft-create, SharePoint read and search, n8n workflow execute and inspect.

## What it never allows

- `outlook_send_mail`, `outlook_send_draft`, `outlook_forward_mail`: the Copilot only creates drafts.
- `sharepoint_upload_file`, `sharepoint_update_file`, `sharepoint_delete_item`, `sharepoint_move_item`, `sharepoint_rename_item`, `sharepoint_copy_item`: every write goes through the n8n workflow.
- The generic Atlassian `executeRead` / `executeWrite` / `executeDestructive` tools.
- Built-in tools (`Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`). If a scheduled run still prompts for those, add them to your own `.claude/settings.local.json` (see `docs/permissions-snippet.json`).

A user or organisation `deny` or `ask` rule for any of these tools still wins over the hook.

## Changing the list

Edit the `matcher` regex, keep the `^mcp__(...)__(...)$` shape, bump the plugin version, re-publish. Test with one `/triage` on a test ticket and confirm no permission prompt appears for the listed tools.
