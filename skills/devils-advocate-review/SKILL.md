---
name: devils-advocate-review
description: DO NOT INVOKE. Pointer stub with no playbook. Always use the published skill `anthropic-skills:devils-advocate` instead.
---

# Devil's advocate review (stub reference)

This file is a placeholder. The plugin does NOT bundle the Devil's advocate skill body -- it lives in the published Anthropic skills plugin (or the myPOS-internal skills marketplace, depending on where it was published).

## How this works

When `/triage` calls the `triage-reviewer` subagent (`agents/triage-reviewer.md`), that subagent calls `Skill(skill: "anthropic-skills:devils-advocate")` directly. Calling `devils-advocate-review` does NOT reach the published skill: inside this plugin the name resolves to this stub (found in the 25 Sep and 2 Oct 2026 usage reviews). The plugin user must have `anthropic-skills:devils-advocate` available.

## To replace this stub with a bundled copy

If at any point we want the plugin to ship its own copy of the Devil's advocate skill (e.g., to pin a known-good version, or to operate without the published one):

1. Replace this `SKILL.md` with the full skill body (frontmatter + instructions).
2. Update `name:` to keep `devils-advocate-review` (so callers don't change).
3. Update `description:` to whatever the bundled version describes.
4. Bump the plugin version in `.claude-plugin/plugin.json`.

## Verifying availability

`/setup-copilot` runs a smoke test that confirms the skill resolves. If it doesn't, the user is told to either:
- Install the publisher plugin (e.g., `anthropic-skills`)
- Or ask for a bundled version of the legal copilot

## TODO before v0.1.0 ship

Confirm with the legal copilot maintainers (Atanas / Emil) where the Devil's advocate skill is published and whether plugin users will reliably have it available. If yes, this stub is fine. If publishing is uncertain, replace with a bundled copy.
