# Two-touch rule

Policy version: 0.5.0 (ships with plugin 0.5.0; the copy in the SharePoint `_knowledge` folder must carry the same version line, `/setup-copilot` checks it)

Non-critical tickets should close in **no more than 2 interactions with the requester**. The triage skills tend to produce long missing-information lists (median 5 questions, up to 10, across the LEGAL tickets triaged between late August and October 2026) and multi-step paths ("confirm X, then send Y, then we will review"). Each of those is another round trip. This file sets the risk-based limit that `/triage`, the Devil's advocate pass and the business reviewer all apply.

---

## What counts as a touch

A **touch** is a message from Legal to the requester that either delivers the answer or asks them for something.

Not a touch:
- An automatic acknowledgement, or a Jira comment the requester never sees (AI Triage, SLA Warning).
- An internal message to another myPOS team that the requester is not waiting on.
- The requester writing to us unprompted.

---

## Who it applies to

| SLA tier (`knowledge/sla-policy.md`) | Two-touch rule |
|---|---|
| `standard` | Applies |
| `high` without a critical flag | Applies |
| `critical` | Exempt (`full_track`) |
| `project-pressing`, `project-standard` | Exempt (`full_track`) |
| Matter type `claims` or `inspection_support` | Exempt (`full_track`) |
| Any ticket a lawyer has labelled `full-track` | Exempt |

---

## Resolution paths

Every triage picks exactly one `resolution_path`. Prefer the paths higher in the table.

| Path | Use when | Touch 1 | Touch 2 |
|---|---|---|---|
| `answer_now` | Enough information to answer | The answer | (none) |
| `answer_on_assumptions` | Gaps exist, but a reasonable assumption fills each one | The answer, with an **Assumptions** block: "We've assumed X. If that's wrong, tell us and we'll adjust." | Only if the requester corrects an assumption |
| `one_question_round` | No useful answer is possible without facts only the requester has | At most 3 questions, all in one message (or the intake form link, see below) | The answer |
| `full_track` | Exempt tickets only | No limit | No limit |

### Building touch 1

1. Sort the skill's missing-information list into **blocking** (no useful answer without it) and **assumable** (a sensible default exists, or it only changes a detail).
2. Turn every assumable item into a line in the draft's Assumptions block. Do not ask it.
3. Ask at most **3** blocking questions. If more than 3 seem blocking, keep the 3 that change the answer most; assume the rest.
4. If a document is missing, ask for every missing document in the same message.
5. No sequential gating. Phrases like "once you confirm X we will review Y and then revert" are not allowed. Say what we will do as soon as we have the answers.
6. End with one clear ask, or "no action needed from you".

### Intake form redirect (beta)

For `contract_review` and `regulatory_question` tickets on the `one_question_round` path, touch 1 may be the matching intake form link from `knowledge/config.md` (section "Intake forms (beta)") instead of the questions. Use it only when:
- the form URL in `config.md` is not `TBD`, and
- 3 or more blocking items are missing (fewer than that: just ask them).

The form submission comes back as a new ticket with the `intake:form` label. Link it to the original ticket as "relates to"; the answer on the new ticket is touch 2.

Never send the form link on a ticket that already came from the form.

---

## What the reviewers do with it

- **Devil's advocate (`triage-reviewer`):** on a two-touch ticket, a gap it finds becomes an assumption or a caveat in the draft, not a new question to the requester. It may only add a question if it is blocking and the 3-question cap still has room.
- **Business reviewer:** scores "interaction budget" as a fourth axis and returns `revise` if the final draft breaks the cap (see `agents/business-reviewer.md`).

---

## Tracking

- `jira-fields-and-flags` adds `path:{resolution_path}` (e.g. `path:answer_on_assumptions`) at triage.
- `/reply-and-close` counts the touches on the ticket and adds `touches:{N}`. If the ticket was two-touch eligible and `N > 2`, it also adds `two-touch-breached` and records why in the memory file.
- `/legal-report` reports the share of eligible tickets closed within 2 touches, by matter type.

---

## Hard rules

- NEVER put more than 3 questions to the requester in one message on a two-touch ticket.
- NEVER spread questions across messages to stay under the cap.
- NEVER apply the rule to a `full_track` ticket. Risk beats speed.
- NEVER turn a legal position into an assumption. Assumptions cover facts about the request (entity, role, volumes, dates). The legal view itself is never assumed.
