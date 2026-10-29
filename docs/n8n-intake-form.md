# n8n intake forms (beta)

**Status: built, comment mode included, live-tested on 29 Oct 2026 against LEGAL.** n8n workflow `Legal Intake` (`FicMGfmDaJxaXBym`), with raw submissions logged to the data table `legal_intake_submissions`. The basic-auth credential (`Legal Intake form login`) sits on both form triggers and a Jira Cloud credential on the five Jira calls. The production URLs are in `knowledge/config.md` (section "Intake forms (beta)"), so from plugin 0.5.1 the Copilot sends the form link on the `one_question_round` path when the workflow is published; while the workflow is unpublished the link returns a 404 and `/triage` should fall back to asking its (max 3) questions.

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

1. **Form Trigger** (one per form). Fields below. Access model agreed on 29 Oct 2026: basic authentication with one shared team login (n8n credential `Legal Intake form login`), published on the internal Legal SharePoint page next to the links. An office / VPN IP allow-list (`ipWhitelist` option on the trigger) can be added later if IT provides the ranges. Never leave an unauthenticated public URL that writes to Jira.
2. **Code / validate.** Reject submissions whose requester email is not `@mypos.com` (show an error page). Trim free text. Build the block below. Read the `Existing ticket (if any)` field; the Copilot's own link prefills it with the ticket key (see "How the Copilot sends the link"), so for a redirected requester it is already filled.
3. **IF an existing `LEGAL-*` ticket is named and exists: comment mode.** Post the block as a **comment on that ticket**, add the labels `intake:form` and `intake:{form-id}` (e.g. `intake:contract-review-v1`) to it, and add every uploaded file to that ticket. No new ticket. This keeps one ticket per matter, so the two-touch count and the SLA stay on the ticket the requester already knows.
4. **ELSE: new-ticket mode.** Project `LEGAL`, issue type `Task`, summary `[{form}] {short title}`, description = the block below, labels `intake:form` and `intake:{form-id}`, attachments on the new ticket. If the requester typed a ticket key that does not resolve, treat as new-ticket mode and write `Existing-Ticket-Given: {text} (not found)` in the block so the lawyer can link by hand. Use a service-account Jira credential in both modes, the same way the `Legal Copilot` filing workflow uses a service account for SharePoint.
5. **Ending page.** Comment mode: "Thanks. Your answers were added to {LEGAL-key}; Legal will come back to you on that ticket." New-ticket mode: "Thanks. Your request is {LEGAL-key}. Legal will send you a first response within 2 business days." (the `standard` tier first-response target in `knowledge/sla-policy.md`).

The Copilot does not need a new trigger. In comment mode the ticket is still `To Do`, so the 08:00 `/triage-board` sweep picks it up and `/triage` Step 3 treats the `Intake-Form:` comment as the requester's reply and re-triages with the new facts. In new-ticket mode the sweep picks up the new To-Do ticket like any other.

## How the Copilot sends the link

On the `one_question_round` path with 3+ blocking gaps (`knowledge/two-touch-rule.md`), touch 1 is the form link with the ticket key prefilled: `{form URL}?existing_ticket=LEGAL-6000`. n8n prefills a field from a query parameter whose name matches the field **name** (`existing_ticket`), not the label, and only on the production `/form/` URL, never on the `/form-test/` URL (n8n docs, Form Trigger, "Prefill form fields with query parameters"). The draft also states the ticket key in words in case the requester retypes the address. Never send the link on a ticket that already carries an `Intake-Form:` block.

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

## Go-live record (29 Oct 2026)

- **Built and tested.** Comment-mode branch added (If "Ticket key given?", Jira get with the error output as fallback, Jira comment as ADF with one paragraph per line, additive label update through `PUT /rest/api/3/issue/{key}` with `update.labels.add`, "Mark not found" fallback). Live runs against LEGAL: no ticket named created LEGAL-6014; `LEGAL-999999` hit a real Jira "not found" and created LEGAL-6015 with `(not found)` in the block; naming LEGAL-6014 posted the block as comment 510831 on it and the label update returned HTTP 204. The comment reads back through the Atlassian connector with `Intake-Form:` as its first line.
- **Prefill confirmed** from the n8n docs: query parameter keyed on the field name, production URL only (see "How the Copilot sends the link"). No email-matching fallback needed.
- **Access model:** basic auth with the shared `Legal Intake form login` (see step 1 above).
- **Production URLs** are in `knowledge/config.md` since plugin 0.5.1.
- **Upload size:** n8n accepts files up to 200 MiB per field by default (`N8N_FORMDATA_FILE_SIZE_MAX`); the binding limit is Jira's own attachment limit, set by the Jira admin. Typical contract PDFs are well below either.

## Still open

- One real browser submission on the production URL after publishing, with a small PDF attached and the key prefilled from the link, to exercise basic auth, the prefill and the attachment path (the MCP test runs cannot carry files or the login prompt). Then close the test tickets.
- Before the forms go to the whole company: swap the Jira credential for a Legal service account, so automated comments do not carry a colleague's name and the token does not expire with a person. Set a reminder for the API token's expiry date.
- Rename the basic-auth credential in n8n from "Unnamed credential" to `Legal Intake form login`.
- Jira renders the `---` separator line of the description block as an em dash; harmless for `/triage`, which keys on the `Field: value` lines.
