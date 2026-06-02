# Default filing rules (shipped with the plugin)

These rules describe the SharePoint folder layout the n8n `Legal Copilot` workflow expects. The workflow creates `myPOS Legal/{case_folder}/{case_id}/` under `https://mypos0.sharepoint.com/sites/legal/Shared Documents/` and uploads documents into that case folder.

The customisable copy lives at `knowledge/sharepoint-map.md` and ships with the plugin. If the team renames a folder, change the right-hand column there AND rename the matching SharePoint folder by hand -- the workflow uses the literal name from the payload.

> Filing is now done programmatically through the n8n workflow `VAKq9Bra0RA0SdCO` (webhook `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`). The old local-Desktop-mirror flow has been retired. Lawyers no longer drag-and-drop folders into SharePoint.

## Top-level folders by matter type

| Matter type | Folder |
|---|---|
| NDA | `NDAs/` |
| Contract review | `Contract Reviews/` |
| Regulatory question | `Regulatory Questions/` |
| Corporate change | `Corporate Changes/` |
| Project | `Projects/` |
| KYC support | `KYC Support/` |
| GTCs | `GTCs/` |
| Materials review | `Materials Reviews/` |
| Claims | `Claims/` |
| Inspection support | `Inspection Support/` |

## Case-folder naming (what the workflow sees as `case_folder`)

The skill passes the matter-type folder name (right-hand column above) as `case_folder`. The workflow then creates the case folder at `myPOS Legal/{case_folder}/{case_id}/` where `case_id` is the Jira ticket key (e.g., `LEGAL-4321`).

Subfolder collisions are not possible -- the case folder is keyed on the unique Jira ticket key, not on the ticket summary.

## Filename template

```
{TICKET-KEY}_{counterparty-or-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

| Role | Marker | Extension |
|---|---|---|
| draft | `v{N}` | `.docx` |
| attachment | `attachment_{original-stem}` | original |
| final email (sent) | `final` | `.eml` |
| comments | `comments` | `.docx` |

The Devil's advocate review is **not** a separate file. Its findings are embedded as Word comments inside the draft `.docx` (one anchored comment per finding, plus a verdict summary comment). The legacy `_review.md` role has been retired.

## Per-matter-type overrides

None by default. To add overrides, edit `knowledge/sharepoint-map.md` in the plugin and re-publish. Example overrides the team might want:

- **NDAs** -- file under `NDAs/{counterparty}/{ticket_key}/` instead of `NDAs/{ticket_key}/`, so all NDAs with the same counterparty stay together. The skill would set `case_folder` to `NDAs/{counterparty}` and let the workflow append `{case_id}` as before.
- **Inspection support** -- file under `Inspection Support/{regulator}/{ticket_key}/` 