# n8n intake forms (beta)

**Status: built, comment mode live-tested on 29 Oct 2026; automatic invite trigger added 30 Oct 2026 (draft until published, Outlook credential pending).** n8n workflow `Legal Intake` (`FicMGfmDaJxaXBym`), with raw submissions logged to the data table `legal_intake_submissions`. The forms are open links (no login, decided 30 Oct 2026); a Jira Cloud credential sits on the Jira calls. The production URLs are in `knowledge/config.md` (section "Intake forms (beta)"). From plugin 0.5.2 the Copilot does not put the form link in a draft: on the `one_question_round` path it calls the workflow's invite trigger, which emails the requester the fixed invitation (see "How the invite is sent"). While the workflow version with the invite trigger is unpublished, the call fails and `/triage` falls back to asking its (max 3) questions.

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

1. **Form Trigger** (one per form). Fields below. Access model changed on 30 Oct 2026: open links, no login (the shared basic-auth login was judged too much friction). What limits abuse: the links are only ever sent by the automatic invite or published on the internal Legal SharePoint page, the workflow rejects any requester address that is not `@mypos.com`, and `ignoreBots` is on. An office / VPN IP allow-list (`ipWhitelist` option on the trigger) can be added later if IT provides the ranges.
2. **Code / validate.** Reject submissions whose requester email is not `@mypos.com` (show an error page). Trim free text. Build the block below. Read the `Existing ticket (if any)` field; the Copilot's own link prefills it with the ticket key (see "How the invite is sent"), so for a redirected requester it is already filled.
3. **IF an existing `LEGAL-*` ticket is named and exists: comment mode.** Post the block as a **comment on that ticket**, add the labels `intake:form` and `intake:{form-id}` (e.g. `intake:contract-review-v1`) to it, and add every uploaded file to that ticket. No new ticket. This keeps one ticket per matter, so the two-touch count and the SLA stay on the ticket the requester already knows.
4. **ELSE: new-ticket mode.** Project `LEGAL`, issue type `Task`, summary `[{form}] {short title}`, description = the block below, labels `intake:form` and `intake:{form-id}`, attachments on the new ticket. If the requester typed a ticket key that does not resolve, treat as new-ticket mode and write `Existing-Ticket-Given: {text} (not found)` in the block so the lawyer can link by hand. Use a service-account Jira credential in both modes, the same way the `Legal Copilot` filing workflow uses a service account for SharePoint.
5. **Ending page.** Comment mode: "Thanks. Your answers were added to {LEGAL-key}; Legal will come back to you on that ticket." New-ticket mode: "Thanks. Your request is {LEGAL-key}. Legal will send you a first response within 2 business days." (the `standard` tier first-response target in `knowledge/sla-policy.md`).

The Copilot does not need a new trigger. In comment mode the ticket is still `To Do`, so the 08:00 `/triage-board` sweep picks it up and `/triage` Step 3 treats the `Intake-Form:` comment as the requester's reply and re-triages with the new facts. In new-ticket mode the sweep picks up the new To-Do ticket like any other.

## How the invite is sent (automatic since 30 Oct 2026)

On the `one_question_round` path with 3+ blocking gaps (`knowledge/two-touch-rule.md`), touch 1 is an **automatic email**: `/triage` Step 4b item 4 calls the workflow's `Invite webhook` trigger (`POST /webhook/legal-intake-invite`, body `{ticket_key, requester_email, form}`) and the workflow does the rest. This is the one email the Copilot causes to be sent; the Copilot cannot change its text, which lives in the `Validate invite` node:

```
Subject: [LEGAL-NNNN] Legal intake form for your request

Dear sender,

Our AI triage bot has identified gaps in your request for legal review.
To this end, we kindly ask that you fill the intake form found here: {form URL}?existing_ticket=LEGAL-NNNN

*This is an AI-generated automatic email. It is sent based on pre-set rules on our legal copilot triage agent(s). If a lawyer has already reached out to you, please ignore this email.
```

Workflow steps on the invite branch: `Validate invite` (key `LEGAL-NNNN`, requester `@mypos.com`, form name; builds the link, subject and body) -> `Read ticket labels` (Jira) -> `Check invite guards` -> `Skip invite?` -> `Send intake invite` (Microsoft Outlook node, the Legal mailbox's credential) -> `Comment: invite sent` (an `Intake-Invite: sent` comment with Sent-To, Sent-At, Form, Link) -> `Label: form-sent` (adds `intake:form-sent`) -> `Respond invite sent`.

Guards (the workflow refuses, answering `{success:true, skipped}`): label `intake:form-sent` already on the ticket, label `intake:form` (the ticket came from the form), or the ticket is done. Errors: 400 `invite_rejected` (bad input), 404 `ticket_not_found`, 502 `invite_send_failed`. On any non-success answer `/triage` falls back to asking its questions in the draft.

The link prefills the form: n8n fills a field from a query parameter whose name matches the field **name** (`existing_ticket`), not the label, and only on the production `/form/` URL, never on the `/form-test/` URL (n8n docs, Form Trigger, "Prefill form fields with query parameters").

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

**"Other" follow-ups.** n8n forms cannot show a field only when "Other" is picked, so every dropdown that offers Other is followed by an always-visible optional text box ("If you chose Other: ..."). The Build nodes append that text to the chosen value on the same block line, so `/triage` reads one line: `Contract type: Other - franchise agreement`, `Our role: Other - licensor`, `myPOS entity signing: Other / not sure - probably the Dutch entity`. When the contract type is Other, the ticket summary uses the free text instead of the word "Other".

## Fields

### Contract review (`contract-review v1`)

| Field | Type | Required |
|---|---|---|
| Your name, your myPOS email, your team | text | yes |
| myPOS entity signing | dropdown: the group entity list below, then "Other / not sure" | yes |
| If you chose Other / not sure: which entity, or what you know | text (`entity_other`) | no |
| Counterparty name | text | yes |
| Our role | dropdown (customer / supplier / partner / distributor / other) | yes |
| If you chose Other: our role | text (`our_role_other`) | no |
| Contract type | dropdown (NDA, MSA, SOW, SaaS, partnership, amendment / renewal, other) | yes |
| If you chose Other: contract type | text (`contract_type_other`) | no |
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
| myPOS entity concerned | dropdown: the group entity list below, then "Other / not sure" (block line `myPOS entity:`) | yes |
| If you chose Other / not sure: which entity, or what you know | text (`entity_other`) | no |
| Markets affected | multi-select (EEA (all), specific EEA countries, UK, outside EEA / UK) | yes |
| Which countries (if specific or outside EEA / UK) | text (`markets_detail`) | no |
| Product or feature | text | yes |
| What decision depends on the answer | long text | yes |
| Deadline | date | no |
| What drives that deadline | text | only if a deadline is given |
| Has a regulator asked about this? | yes / no | yes |
| What Compliance has already checked | long text | no |
| Existing ticket (if any) | text | no |
| Supporting documents | file upload (multiple) | no |

"Has a regulator asked about this? = yes" must lead to the `regulator` risk flag at triage, which makes the ticket `critical` and exempt from the two-touch rule.

### Group entity list (both forms, as of 29 Oct 2026)

The same dropdown on both forms. `myPOS AD` no longer exists as such: it is the Bulgarian branch of myPOS Limited, code MPLB, listed first. Dormant entities sit at the end, before "Other / not sure". The list lives in the two Form Trigger nodes in n8n; when the group matrix changes, edit both triggers, re-publish the workflow, and update this list.

1. MPLB - myPOS Limited - Branch Bulgaria (formerly myPOS AD)
2. MPW - myPOS World
3. MPP - myPOS Payments
4. MPL - myPOS Ltd
5. MPS - myPOS Services
6. myPOS Limited - UK Branch
7. Cyris BG
8. Cyris AG
9. myPOS Partners
10. PT - Payment Technology France
11. MPI - myPOS Italy
12. MPNL - myPOS Netherlands
13. MPB - myPOS Belgium
14. myPOS Experience Centers B.V.
15. MPBG - myPOS BG
16. MPAT - myPOS AT
17. MPRO - myPOS EC RO
18. MPPGL - myPOS Portugal
19. MPECL - myPOS EC Ireland
20. MPHU - myPOS Hungary
21. TopOrder
22. myPOS UTP Group Limited
23. myPOS Merchant Services (Ireland) Limited
24. myPOS Merchant Services (Gibraltar) Limited
25. myPOS Merchant Services (UK) Limited
26. CRC Collections Ltd
27. myPOS Rentals Limited
28. Faster Processing International Limited
29. Faster Processing Limited
30. MYPOS CONSULTING SERVICES KFT
31. IQOM PAYMENT SOLUTION KFT
32. myPOS Technologies España, S.L.
33. myPOS Germany Holding GmbH
34. Lavego
35. MPSW - myPOS Swiss (dormant)
36. mPGB - myPOS GB (dormant)
37. Other / not sure

## Go-live record (29 Oct 2026)

- **Built and tested.** Comment-mode branch added (If "Ticket key given?", Jira get with the error output as fallback, Jira comment as ADF with one paragraph per line, additive label update through `PUT /rest/api/3/issue/{key}` with `update.labels.add`, "Mark not found" fallback). Live runs against LEGAL: no ticket named created LEGAL-6014; `LEGAL-999999` hit a real Jira "not found" and created LEGAL-6015 with `(not found)` in the block; naming LEGAL-6014 posted the block as comment 510831 on it and the label update returned HTTP 204. The comment reads back through the Atlassian connector with `Intake-Form:` as its first line.
- **Prefill confirmed** from the n8n docs: query parameter keyed on the field name, production URL only (see "How the invite is sent"). No email-matching fallback needed.
- **Access model:** basic auth on 29 Oct, replaced by open links on 30 Oct (see step 1 above).
- **Production URLs** are in `knowledge/config.md` since plugin 0.5.1.
- **Upload size:** n8n accepts files up to 200 MiB per field by default (`N8N_FORMDATA_FILE_SIZE_MAX`); the binding limit is Jira's own attachment limit, set by the Jira admin. Typical contract PDFs are well below either.
- **30 Oct 2026:** invite trigger built (10 nodes) and tested with simulated data: a normal call took the send path and answered `{success:true, sent_to, link, comment_id}`; a call on a ticket already labelled `intake:form-sent` answered `{success:true, skipped:'already_sent'}` without sending. Entity dropdowns updated to the 37-entry group list and "Other" follow-up boxes added (29 Oct).

## Still open

- **Outlook credential for the sending mailbox**: create a Microsoft Outlook OAuth2 credential in n8n (sign in as the mailbox the invitations should come from; for the beta your own account works, then set `from` = `legal@mypos.com` in the node's Additional Fields if that account has Send As rights on the shared mailbox) and select it on `Send intake invite`. Then publish the workflow.
- One real invite on a test ticket after publishing, then one real browser submission with a small PDF through the link in that email, to exercise the whole loop: invite email, prefill, attachment, `Intake-Form:` comment, re-triage by the next sweep.
- Before the forms go to the whole company: swap the Jira credential for a Legal service account, so automated comments do not carry a colleague's name and the token does not expire with a person. Set a reminder for the API token's expiry date.
- Delete the unused basic-auth credential ("Unnamed credential") in n8n; it is no longer referenced by the form triggers' authentication setting.
- Jira renders the `---` separator line of the description block as an em dash; harmless for `/triage`, which keys on the `Field: value` lines.
