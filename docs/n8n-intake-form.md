# n8n intake forms (beta)

**Status: built as an unpublished draft (22 Oct 2026), not tested yet.** n8n workflow `Legal Intake` (`FicMGfmDaJxaXBym`), with raw submissions logged to the data table `legal_intake_submissions`. It needs a basic-auth credential ("Legal Intake form login") and a Jira Cloud credential selected before the first test. Until it is published and its URLs are in `knowledge/config.md` (section "Intake forms (beta)"), the Copilot never sends a form link; `/triage` just asks its (max 3) questions.

## Why

Contract review and regulatory question requests arrive by email with too little context. Across the LEGAL tickets triaged between late August and October 2026, the triage skills listed a median of 5 missing-information questions per ticket (up to 10). Each question round adds a touch with the requester and breaks the two-touch target (`knowledge/two-touch-rule.md`). The forms collect the facts up front.

## Scope of the beta

- Two forms: **Contract review** and **Regulatory / compliance question**.
- Everything else keeps arriving by email at `legal@mypos.com`.
- Run for 6-8 weeks, then compare in `/legal-report` (section 1d): two-touch rate and time to close for form tickets against email tickets of the same matter types.

## Workflow outline

One n8n workflow, `Legal Intake`, with two form triggers:

```
[n8n Form Trigger: Contract review]  --\
                                        >-- [Code: validate + build description] --> [Jira: create issue] --> [Jira: add attachments] --> [Form ending page]
[n8n Form Trigger: Regulatory question] --/
```

1. **Form Trigger** (one per form). Fields below. Turn on the trigger's basic authentication (a shared team credential), or only publish the URL on an internal SharePoint page. Do not leave an unauthenticated public URL that creates Jira tickets.
2. **Code / validate.** Reject submissions whose requester email is not `@mypos.com` (show an error page). Trim free text. Build the description block below.
3. **Jira: create issue.** Project `LEGAL`, issue type `Task`, summary `[{form}] {short title}`, description = the block below, labels `intake:form` and `intake:{form-id}` (e.g. `intake:contract-review-v1`). Use a service-account Jira credential, the same way the `Legal Copilot` filing workflow uses a service account for SharePoint.
4. **Jira: add attachments.** Every uploaded file goes onto the new ticket. The Copilot's normal flow (`documents_from_jira`) then files them to SharePoint at triage.
5. **Ending page.** "Thanks. Your request is {LEGAL-key}. Legal will send you a first response within 2 business days." (the `standard` tier first-response target in `knowledge/sla-policy.md`).

The Copilot does not need a new trigger: the 08:00 `/triage-board` sweep picks up the new To-Do ticket like any other.

## Description block (the contract with `/triage`)

`/triage` Step 2 recognises a form ticket by the first line. Keep this format stable; bump the version (`v2`) if fields change.

```
Intake-Form: contract-review v1
Submitted-By: {name} <{email}>
Submitted-At: {ISO timestamp}
Related-Ticket: {LEGAL-key or "none"}
---
{Field label}: {answer}
{Field label}: {answer}
...
```

An unanswered optional field is written as `Not sure`, never left out, so `/triage` can tell "not asked" from "unknown".

## Fields

### Contract review (`contract-review v1`)

| Field | Type | Required |
|---|---|---|
| Your name, your myPOS email, your team | text | yes |
| myPOS entity signing | dropdown (myPOS AD, myPOS Limited, myPOS Payments Ltd, other / not sure) | yes |
| Counterparty name | text | yes |
| Our role | dropdown (customer / supplier / partner / distributor / other) | yes |
| Contract type | dropdown (NDA, MSA, SOW, SaaS, partnership, amendment / renewal, other) | yes |
| Whose template | dropdown (ours / theirs / not sure) | yes |
| Value band (annual) | dropdown (<EUR 10k, 10-50k, 50-100k, >100k, not sure) | yes |
| Signing deadline | date | no |
| What happens if we miss it | text | only if a deadline is given |
| What you need from Legal | dropdown (full review + redline / specific clauses / sign-off only / negotiation support) | yes |
| Commercial terms already agreed? | yes / no / partly | yes |
| Personal data shared or processed? | yes / no / not sure | yes |
| Business approver | text | no |
| Existing ticket (if any) | text | no |
| Contract file and any earlier version | file upload (multiple) | yes |
| Anything else we should know | long text | no |

### Regulatory / compliance question (`regulatory-question v1`)

| Field | Type | Required |
|---|---|---|
| Your name, your myPOS email, your team | text | yes |
| The question, in one sentence | text | yes |
| Entity and licence | dropdown (myPOS AD (BNB), myPOS Limited (CBI EMI), myPOS Payments Ltd (FCA), other / not sure) | yes |
| Markets affected | multi-select (EEA countries, UK, other) | yes |
| Product or feature | text | yes |
| What decision depends on the answer | long text | yes |
| Deadline | date | no |
| What drives that deadline | text | only if a deadline is given |
| Has a regulator asked about this? | yes / no | yes |
| What Compliance has already checked | long text | no |
| Existing ticket (if any) | text | no |
| Supporting documents | file upload (multiple) | no |

"Has a regulator asked about this? = yes" must lead to the `regulator` risk flag at triage, which makes the ticket `critical` and exempt from the two-touch rule.

## Open items before go-live

- Build the workflow in n8n and test it end to end on a test ticket.
- Agree the access model (basic auth vs. internal SharePoint link only).
- Paste the two production form URLs into `knowledge/config.md` and re-publish the plugin.
- Check the n8n Form Trigger upload size limit against typical contract PDFs.
