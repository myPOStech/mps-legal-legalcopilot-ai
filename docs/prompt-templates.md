# Prompt templates

Reusable prompts for the most common Legal Copilot tasks, built from the weekly usage reviews of 21 Sep, 25 Sep and 2 Oct 2026. Fill the braces, delete lines you do not need. The weekly review re-checks these against real sessions and proposes updates here.

## 1. Single ticket triage

```
/triage LEGAL-{NNNN}
Context Claude cannot see: {e.g. counterparty is a strategic partner / deal size ~EUR X / we already agreed Y by phone}
Output: Jira comment + SharePoint .docx + Outlook reply draft to {requester}.
If filing fails, stop and tell me; do not post the comment.
```

Why: business-reviewer otherwise passes `is_strategic_partner: unknown` and produces a context_gap you then answer in a second round.

## 1b. Single ticket triage (attachments and NDA guard)

```
/triage LEGAL-{NNNN}
Ignore signature images (image001.png etc.). File only: the draft .docx and these attachments: {names or "none"}.
For NDAs use anthropic-skills:legal-triage-nda, not legal:triage-nda.
End the summary with the line "Outlook draft: created / NOT created, reason".
```

Why (weekly review of 23 Oct 2026): signature images sent as Jira attachments made two filings fail on the n8n SSRF guard and left empty case folders; two NDA skills can trigger on "NDA"; Outlook drafts stopped after 14 Oct without any run saying why.

## 2. Contract review / mark-up

```
Review the attached {agreement type} for {myPOS entity} as {our role: customer/supplier/distributor}.
Governing law: {X}. Our must-haves: {liability cap >= 12 months fees, no exclusivity, DPA}.
Deliver: tracked-changes .docx + a 1-page issues list ranked by risk. English only. Do not change commercial terms.
```

Why: contract sessions in the window (Dominaite, Finimid, Alcineo) all hinge on role and entity; stating them up front saves a clarification round.

## 3. Regulatory question

```
Question: {one sentence}.
Entity and licence: {myPOS Limited (CBI EMI) / myPOS Payments Ltd (FCA) / myPOS AD (BNB)}.
Jurisdiction(s): {list}. Date the answer must hold for: {date}.
Answer in chat, max 1 page, cite primary sources with links. Flag anything you could not verify.
```

Why: regulatory sessions in the window (MIA licence, CCA registration, payfac guidance) depend on which licensed entity is asking.

## 4. Translation

```
/translate-contract {file} into {language}. Keep numbering and defined terms.
Bilingual table: {yes/no}. Glossary to follow: {file or "none"}.
```

## 5. Quick check before a long task

```
Before starting, confirm: M365 connected? n8n Legal Copilot reachable? Jira reachable? If any is not, tell me and stop.
```

Use as the first line of any multi-step copilot run; it turns 20-minute failures into 1-minute ones.

## 6. Weekly health check

```
Run the Legal Copilot health check: plugin version vs GitHub, M365 get_granted_scopes, n8n last 7 days of Legal Copilot executions (errors only), newest [AI DRAFT LEGAL-] draft date, Jira read. Read-only. Report only what changed since last week.
```

Why: the 23 Oct review found version drift (0.4.8 installed, 0.4.9 on GitHub) and a 9-day gap in Outlook drafts that a quick weekly check would have caught.
