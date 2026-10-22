# SLA policy

The single source of truth for how long the Legal team has to close a ticket. `/triage`, `/triage-board` and `/triage-inbox` read this file after the `legal-triage-*` skill runs and use it to set `sla_days` (and therefore the Jira due date). The published skills still print their own "SLA Suggestion" (4h / 24h / 3 / 5 business days); treat that only as an urgency signal for picking the priority. It is **never** the `sla_days` written to Jira.

To change a threshold: edit the tables below, bump the plugin version, re-publish. No skill needs to change.

---

## Units

- `sla_days` is in **calendar days**. `jira-fields-and-flags` adds them to today and rolls a Saturday/Sunday due date back to Friday.
- The SLA is the **resolution** deadline: the due date by which the ticket should be Done.
- The **first response** target is separate and is not written to a Jira field. It is how quickly a lawyer should send the requester something substantive (the reviewed AI draft, or the one question round), so a 14-day SLA never means two weeks of silence. It is shown in the AI Triage comment.

---

## Critical flags

These risk flags make a ticket critical, whatever its priority:

```yaml
critical_flags:
  - regulator
  - inspection
  - claim
  - claim_or_dispute
  - tight_deadline
  - statutory_deadline
  - short_response_window
```

Other flags the skills raise (`counterparty_template`, `data_processing`, `high_value`, `auto_renewal`, `cross_border`, `ip_transfer`, `missing_scope`, `translation_required`, ...) are review notes. They do not change the tier.

---

## Tiers

Pick the first row that matches, top to bottom.

| Tier | When | `sla_days` | First response |
|---|---|---|---|
| `critical` | Priority P1, OR any critical flag | 1 | 4 hours |
| `high` | Priority P2, OR matter type `claims` / `inspection_support` without a critical flag | 7 | 1 business day |
| `project-pressing` | Matter type `project` with a committed go-live or business deadline | 90 | Scoping call within 5 business days |
| `project-standard` | Matter type `project`, no hard timeline | 180 | Scoping call within 10 business days |
| `standard` | Everything else (P3 / P4, no critical flag): the non-critical tier | 14 | 2 business days |

> `claims` and `inspection_support` never drop below `high`, even when the skill says P3 / P4.

---

## External deadlines win

If the triage found a hard external date (signing deadline, statutory filing date, regulator response date, launch / effective date, notice-period start), the due date must land before it:

```
policy_due   = today + tier sla_days
external_due = external_date - buffer
due          = min(policy_due, external_due)
sla_days     = max(1, due - today)
```

| Tier | Buffer before the external date |
|---|---|
| `critical`, `high`, `standard` | 2 business days |
| `project-pressing`, `project-standard` | 10 business days |

If `external_due` is already in the past, set `sla_days = 1` and let `jira-fields-and-flags` add `sla-already-elapsed`. Record the external date and which value won in the triage output (`sla_basis: policy | external_deadline`).

A stated wish ("ASAP", "by end of week if possible") is not an external deadline. Only a date with a real consequence counts.

---

## Output of this step

The calling command passes these on to `jira-fields-and-flags`, the reviewers and the AI Triage comment:

```json
{
  "sla_tier": "critical" | "high" | "standard" | "project-pressing" | "project-standard",
  "sla_days": 1-200,
  "sla_basis": "policy" | "external_deadline",
  "external_deadline": "YYYY-MM-DD" | null,
  "first_response_target": "4 hours" | "1 business day" | "2 business days" | "scoping call within 5 business days" | "scoping call within 10 business days"
}
```

---

## Hard rules

- NEVER write the skill's own "SLA Suggestion" into `sla_days`.
- NEVER let `sla_days` exceed 200 or drop below 1.
- NEVER downgrade a ticket with a critical flag below `critical` because the requester said it isn't urgent. Lawyers can do that by hand; the Copilot cannot.
- A lawyer may change the due date on the ticket by hand at any time. If a ticket already has a `duedate` when the Copilot triages it, `jira-fields-and-flags` keeps it and only reports what the policy would have set.
