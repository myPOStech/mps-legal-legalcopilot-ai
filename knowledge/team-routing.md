# Legal team routing

This file is the source of truth for which lawyer a triaged Jira ticket gets assigned to.

Edit this file and re-publish the plugin to update routing. No code changes needed.

---

## Team roster

The Legal team members the Copilot can route to. `account_id` is the Jira `accountId` used by `mcp__atlassian__editJiraIssue` and `mcp__atlassian__lookupJiraAccountId`.

| Name | Email | Jira accountId | Active |
|---|---|---|---|
| Atanas Rusenov | atanas.rusenov@mypos.com | 712020:f4e75142-4a80-4634-885d-239661e80849 | yes |
| Daniela Chavdarova | daniela.chavdarova@mypos.com | 712020:e8eab1e8-c4c7-4b50-a933-9f1552383611 | yes |
| Denitsa Dimitrova | denitsa.dimitrova@mypos.com | 712020:4636a3aa-01db-4f29-aa5b-529d08e5870a | yes |
| Ivan Troyanov | ivan.troyanov@mypos.com | 712020:32870eda-fc09-4ff4-a69f-f43481017599 | yes |
| Jay Manjdadria | jay.manjdadria@mypos.com | 62855af6222d36006fb76bdd | yes |
| Nikolay Saragerov | nikolay.saragerov@mypos.com | 712020:c6192173-4c4a-400f-b3a0-b198dcf3a407 | yes |

To deactivate a lawyer (vacation, leaving): set `Active = no`. The router will skip them and rebalance.

---

## Expertise map

Confirmed by the head of legal on 2026-06-02.

Each lawyer's full areas of expertise:

- **Ivan Troyanov** -- Corporate changes, GTCs, Regulatory, Contract Review, Claims
- **Atanas Rusenov** -- Regulatory, Claims, Projects, Inspection Support, Contract Review
- **Daniela Chavdarova** -- GTCs, Claims, Regulatory, Contract Review
- **Denitsa Dimitrova** -- Contract Review, NDA, KYC
- **Jay Manjdadria** -- NDA, KYC
- **Nikolay Saragerov** -- Projects, GTCs, Inspection Support, Materials Review, Contract Review

Routing by matter type. **Primary** is tried first; **secondaries** are tried in order if the primary is inactive or over capacity (see soft capacity rule below).

| Matter type | Primary | Secondaries |
|---|---|---|
| nda | Jay Manjdadria | Denitsa Dimitrova |
| contract_review | Denitsa Dimitrova | Ivan Troyanov, Nikolay Saragerov, Atanas Rusenov, Daniela Chavdarova |
| regulatory_question | Ivan Troyanov | Atanas Rusenov, Daniela Chavdarova |
| corporate_change | Ivan Troyanov | Atanas Rusenov |
| project | Nikolay Saragerov | Atanas Rusenov |
| kyc | Denitsa Dimitrova | Jay Manjdadria |
| gtcs | Ivan Troyanov | Daniela Chavdarova, Nikolay Saragerov |
| materials_review | Nikolay Saragerov | Atanas Rusenov |
| claims | Ivan Troyanov | Atanas Rusenov, Daniela Chavdarova |
| inspection_support | Atanas Rusenov | Nikolay Saragerov |

If the matter type is missing or the triage skill returns a confidence below 0.7, the router falls back to the **senior reviewer** (see below).

> **Notes on the map.** `corporate_change` has only Ivan as a domain owner; Atanas is the secondary as senior reviewer + overflow. `materials_review` has only Nikolay; Atanas is the secondary for the same reason. `claims` and `inspection_support` are always treated as `severity = high`, so the high-severity pool catches them before the expertise map -- the rows above are the fallback when severity has been manually downgraded.

---

## Severity override

Anything with `severity = high` skips the expertise map and goes to one of two senior reviewers, oscillating between them so neither gets overloaded.

```yaml
high_severity_pool:
  - Ivan Troyanov
  - Atanas Rusenov
```

Triggers that count as `severity = high`:

- Matter type is `claims` or `inspection_support`
- Any risk flag set (regulator, inspection, claim, tight deadline)
- Priority field on the source ticket is `Highest` or the Jira priority maps to P1
- SLA <= 2 calendar days
- The triage skill returns `human_review_required = true` AND `confidence < 0.7`

### Oscillation rule

Keep a running counter on the SharePoint memory file. Each new high-severity assignment increments it. Even counts go to the first in the pool, odd counts to the second. To peek/update the counter without re-reading the whole memory file, the auto-assign skill uses the marker line:

```
<!-- legal-copilot:high-severity-counter:N -->
```

inside `legal_copilot_memory.md`. The skill increments `N` (via the n8n filing workflow, since the memory file is the only canonical state) when it assigns a high-severity ticket.

If the counter line is absent, treat it as `N = 0`.

---

## Senior reviewer

The fallback when expertise mapping is ambiguous, when the matter type is missing, or when the head of legal needs to weigh in.

```yaml
senior_reviewer:
  name: Atanas Rusenov
  email: atanas.rusenov@mypos.com
  account_id: "712020:f4e75142-4a80-4634-885d-239661e80849"
```

---

## Capacity rule (soft)

If the candidate primary already has 8 or more open To-Do or In Progress tickets assigned to them, the router skips to the first available secondary. This avoids piling work on one person during quiet periods of others. Threshold is intentionally generous; tighten it in this file if the team grows.

```yaml
soft_capacity_threshold: 8
```

---

## How to change routing

1. Edit the relevant table above.
2. Bump the `version` in `.claude-plugin/plugin.json`.
3. Re-publish the plugin and ask each lawyer to run `/plugin update`.

The auto-assign skill always reads this file fresh on each run, so once the plugin is updated locally the new routing kicks in on the next triage.
