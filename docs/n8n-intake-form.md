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
[n8n Form Trigger: Contract review]  --\                                            /-- existing LEGAL ticket named --> [Jira: add comment + labels] --> [Jira: add attachments to it] --\
                                        >-- [Code: validate + build block] --> [IF] <                                                                                                   >-- [Form ending page]
[n8n Form Trigger: Regulatory question] --/                                            \-- no ticket named -----------> [Jira: create issue]          --> [Jira: add attachments]      --/
```

1. **Form Trigger** (one per form). Fields below. Turn on the trigger's basic authentication (a shared team credential), or only publish the URL on an internal SharePoint page. Do not leave an unauthenticated public URL that writes to Jira.
2. **Code / validate.** Reject submissions whose requester email is not `@mypos.com` (show an error page). Trim free text. Build the block below. Read the `Existing ticket (if any)` field; the Copilot's own link prefills it with the ticket key (see "How the Copilot sends the link"), so for a redirected requester it is already filled.
3. **IF an existing `LEGAL-*` ticket is named and exists: comment mode.** Post the block as a **comment on that ticket**, add the labels `intake:form` and `intake:{form-id}` (e.g. `intake:contract-review-v1`) to it, and add every uploaded file to that ticket. No new ticket. This keeps one ticket per matter, so the two-touch count and the SLA stay on the ticket the requester already knows.
4. **ELSE: new-ticket mode.** Project `LEGAL`, issue type `Task`, summary `[{form}] {short title}`, description = the block below, labels `intake:form` and `intake:{form-id}`, attachments on the new ticket. If the requester typed a ticket key that does not resolve, treat as new-ticket mode and write `Existing-Ticket-Given: {text} (not found)` in the block so the lawyer can link by hand. Use a service-account Jira credential in both modes, the same way the `Legal Copilot` filing workflow uses a service account for SharePoint.
5. **Ending page.** Comment mode: "Thanks. Your answers were added to {LEGAL-key}; Legal will come back to you on that ticket." New-ticket mode: "Thanks. Your request is {LEGAL-key}. Legal will send you a first response within 2 business days." (the `standard` tier first-response target in `knowledge/sla-policy.md`).

The Copilot does not need a new trigger. In comment mode the ticket is still `To Do`, so the 08:00 `/triage-board` sweep picks it up and `/triage` Step 3 treats the `Intake-Form:` comment as the requester's reply and re-triages with the new facts. In new-ticket mode the sweep picks up the new To-Do ticket like any other.

## How the Copilot sends the link

On the `one_question_round` path with 3+ blocking gaps (`knowledge/two-touch-rule.md`), touch 1 is the form link with the ticket key prefilled, e.g. `{form URL}?Existing%20ticket%20(if%20any)=LEGAL-6000` (n8n Form Trigger prefills a field from a query parameter whose name matches the field label). The draft also states the ticket key in words in case the prefill does not carry over. Never send the link on a ticket that already carries an `Intake-Form:` block.

## Description block (the contract with `/triage`)

`/triage` Step 2 recognises a form submission by the first line, whether it is a ticket description (new-ticket mode) or a comment (comment mode). Keep this format stable; bump the version (`v2`) if fields change.

```
Intake-Form: contract-review v1
Submitted-By: {name} <{email}>
Submitted-At: {ISO timestamp}
Mode: comment | new-ticket
Existing-Ticket-Given: {text the requester typed, or "none"; "(not found)" appended when it did not resolve}
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

- Build the workflow in n8n, including the comment-mode branch, and test both modes on a test ticket: one submission with the ticket key prefilled from the link, one with no ticket.
- Confirm the n8n version prefills the `Existing ticket (if any)` field from the URL query parameter. If it does not, the Copilot's draft must ask the requester to type the key, and the branch should also fall back to matching the requester's email against open LEGAL tickets that carry a sent form link.
- Agree the access model (basic auth vs. internal SharePoint link only).
- Paste the two production form URLs into `knowledge/config.md` and re-publish the plugin.
- Check the n8n Form Trigger upload size limit against typical contract PDFs.
