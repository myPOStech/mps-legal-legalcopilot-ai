# SharePoint filing map

These are the filing rules the `sharepoint-filer` skill uses when it calls the n8n `Legal Copilot` workflow. They define what `case_folder` value the workflow receives for each matter type, which determines where uploaded documents land under `Shared Documents/myPOS Legal/`.

To rename a folder: change the right-hand column below AND rename the folder on SharePoint to match (the workflow uses the literal name from the payload). Re-publish the plugin and run `/plugin update`.

---

## SharePoint root

```
https://mypos0.sharepoint.com/sites/legal
└── Shared Documents/
    └── myPOS Legal/      ← all case folders + the shared memory file live here
```

---

## Top-level folders

| Matter type | `case_folder` value (used by the n8n workflow) |
|---|---|
| NDA | `NDAs` |
| Contract review | `Contract Reviews` |
| Regulatory question | `Regulatory Questions` |
| Corporate change | `Corporate Changes` |
| Project | `Projects` |
| KYC support | `KYC Support` |
| GTCs | `GTCs` |
| Materials review | `Materials Reviews` |
| Claims | `Claims` |
| Inspection support | `Inspection Support` |

The workflow places uploaded documents at:

```
myPOS Legal/{case_folder}/{case_id}/
```

where `case_id` = the Jira ticket key (e.g., `LEGAL-4321`). There is no per-ticket subfolder by summary -- the ticket key IS the case-folder identifier. This means re-filing a ticket is idempotent.

---

## Filename template

```
{TICKET-KEY}_{counterparty-or-tag}_{YYYY-MM-DD}_{role-marker}.{ext}
```

| Role | Marker | Extension |
|---|---|---|
| Original draft | `v{N}` | `.docx` |
| Attachment from Jira | `attachment_{original-stem}` | original |
| Final email (sent) | `final` | `.eml` |
| Comments thread | `comments` | `.docx` |

The Devil's advocate review is **not** filed as a separate file. Its findings are embedded as Word comments inside the draft `.docx` (one anchored comment per finding, plus a verdict summary comment). All triage outputs are `.docx` -- never `.md` (except the shared memory file, which is markdown by design).

The n8n workflow uploads with `overwrite=true`, so the `sharepoint-filer` skill picks the next free `v{N}` by listing the case folder via `mcp__microsoft-365__sharepoint_list_items` (read-only, reliable) before each upload.

---

## Per-matter-type overrides

By default, files for a ticket are filed under `{case_folder}/{ticket_key}/`. To group files differently for a specific matter type, add a section below. The skill respects forward slashes in `case_folder`, so the workflow will create intermediate folders as needed.

### Example overrides (not active by default)

```yaml
# Uncomment and edit to activate

# nda:
#   case_folder_pattern: "NDAs/{counterparty}"
#   reason: "Group all NDAs with the same counterparty under one parent folder"

# inspection_support:
#   case_folder_pattern: "Inspection Support/{regulator}"
#   reason: "Group all matters with one regulator u