# myPOS Legal Copilot

AI triage and drafting assistant for the myPOS Legal team.

You give it a Jira ticket, an email, or point it at the board. It classifies the request, checks for duplicates, drafts a response in myPOS legal house style, runs a Devil's advocate review on the draft, files attachments to SharePoint via the team's n8n workflow, puts the draft in the lawyer's Outlook Drafts folder when the Microsoft 365 connector allows it (and always in the Jira comment and a SharePoint .docx), and updates Jira. It never sends: the lawyer does. It learns from lawyer feedback and gets better over time.

Runs entirely inside **Claude Code** -- no backend, no API keys, no server to maintain. SharePoint writes are routed through the team's n8n `Legal Copilot` workflow because the Microsoft 365 MCP cannot reliably write to SharePoint.

---

## Setup (one-time, ~5 minutes)

### Step 1: Install Claude Code

Download the desktop app from [claude.ai/download](https://claude.ai/download) and sign in with your myPOS account.

### Step 2: Install the Legal Copilot plugin

Open Claude Code and run, in this exact order:

```
/plugin marketplace add myPOStech/mypos-legal-marketplace-main
/plugin install mypos-legal-copilot@mypos-legal
```

You'll be asked to confirm. Say yes.

### Step 3: Connect Atlassian, Microsoft 365, and n8n

The first time you run a command, Claude Code will ask for permission to connect to **Atlassian** (Jira), **Microsoft 365** (Outlook + SharePoint read), and the **myPOS n8n** instance (SharePoint writes via the `Legal Copilot` workflow). Click through the OAuth prompts -- one window each, log in with your myPOS account, done. You only do this once.

### Step 4: Verify everything works

Run:

```
/setup-copilot
```

This will:
- Verify connectivity to Atlassian, Microsoft 365 (Outlook read + draft-create + SharePoint read), and n8n
- Sanity-check the SharePoint structure under `myPOS Legal/`
- Run a one-shot smoke test of the n8n filing workflow (creates and uploads a tiny test file under `myPOS Legal/_smoke-test/`)
- Register the daily 08:00 Europe/Sofia scheduled run for `/triage-board`

You should only need to do this once on each machine you install on.

---

## What it can do for you

### Triage a single ticket on demand

> **New in 0.5.2.** The intake-form invitation is now sent automatically. When a contract review or regulatory question is missing three or more facts, the n8n workflow emails the requester a fixed, clearly labelled AI-generated invitation with the form link and their ticket number, posts "Intake-Invite: sent" on the ticket, and the Copilot waits for the answers before drafting. The forms are open links now (no shared login) and no longer ask who you are: the ticket number from the invitation identifies the requester. Filing failures from SharePoint come back as a clear "sharepoint_upload_failed" error instead of a timeout.

> **New in 0.5.1.** The intake forms are live for the beta. When a contract review or regulatory question needs three or more facts only the requester has, the Copilot's draft sends them a short form link with their ticket number already filled in (`knowledge/config.md`, "Intake forms (beta)"). Their answers land as a comment on the same ticket and the Copilot re-triages it; no second ticket is created. The forms asked for a shared Legal login (dropped in 0.5.2).

> **New in 0.5.0.** Due dates come from one risk-based SLA policy (`knowledge/sla-policy.md`): 14 days for non-critical tickets, 3 to 6 months for projects, tighter only for critical matters or hard external deadlines. Non-critical drafts aim to close in **two interactions** with the requester (`knowledge/two-touch-rule.md`): at most 3 questions, everything else stated as an assumption. Intake forms for contract and regulatory requests are specified in `docs/n8n-intake-form.md` (beta, not live yet). Unattended runs are no longer blocked by permission prompts: the plugin ships a permission hook (`hooks/hooks.json`) that allows its read, draft-create, Jira and n8n tools, and `/setup-copilot` now checks that the policy files scheduled runs fall back to are on SharePoint at the current version.

> **New in 0.4.0.** Every triage now runs a **three-pass review** -- the legal-triage skill drafts, the Devil's advocate stress-tests, and the business reviewer reconciles the two from a commercial / partner-relationship angle. The Copilot also **auto-assigns the ticket** to the right lawyer (rules in `knowledge/team-routing.md`) and **sets Jira priority, due date, labels and the Flagged field directly on the ticket** -- those facts are no longer buried inside the comment thread.

### Triage a single ticket on demand

```
/triage LEGAL-4321
```

Claude reads the ticket, classifies it, runs the matching legal-triage skill, drafts a response, has Devil's advocate review the draft, files documents to SharePoint via n8n, creates a draft reply in your Outlook Drafts folder when the connector allows it (the draft is always in the Jira comment and the SharePoint .docx too), and posts the triage summary as a Jira comment. **The ticket does not move to Done unless the lawyer approves.**

### Review and send

Review the AI draft (in your Outlook Drafts folder, the AI Triage Jira comment or the SharePoint .docx), edit it, and **send it yourself from Outlook**. The Copilot never sends. Then run:

```
/reply-and-close LEGAL-4321
```

Finds the reply you sent (Sent Items), files it + any new documents to SharePoint via n8n, appends a close-out entry (with your edits captured as lawyer feedback) to the shared memory file, and transitions the ticket to Done.

### File documents to the right SharePoint folder

```
/file-to-sharepoint LEGAL-4321
```

Picks up Jira attachments + the latest draft and files them under `myPOS Legal/{matter-folder}/{TICKET-KEY}/` via the n8n workflow. Useful when a ticket has new documents added mid-flight.

### Triage the inbox

```
/triage-inbox
```

Scans recent emails in the legal inbox (skipping ones already processed), dedupes against existing Jira tickets, creates new tickets for new matters, drafts replies (Outlook Drafts when available), and files everything to SharePoint via n8n.

### Sweep the whole board

```
/triage-board
```

Processes every To-Do ticket on the board, end-to-end. Runs automatically every weekday at 08:00 Europe/Sofia once `/setup-copilot` has registered the schedule. Run it manually any time you want a fresh pass.

### Generate a team productivity report

```
/legal-report
```

On-demand report. Pulls every overdue ticket, per-lawyer load and median time-to-close from Jira, then asks you for each lawyer's off-Jira hours over the same window (manual entry -- the report only counts what you confirm). Files a `.docx` to `myPOS Legal/Reports/Productivity/` and surfaces who is AT RISK, OVERLOADED, and where the routing gaps are.

Args (optional): `last_7_days`, `last_30_days` (default), `this_quarter`, or an explicit `YYYY-MM-DD..YYYY-MM-DD`.

---

## How filing works

The Copilot routes every SharePoint write through the team's n8n workflow `Legal Copilot` (workflow ID `VAKq9Bra0RA0SdCO`, webhook `https://myposai.app.n8n.cloud/webhook/legal-copilot-filing`). The workflow:

1. Creates the case folder `myPOS Legal/{matter-folder}/{TICKET-KEY}/`
2. Uploads each document (draft `.docx`, attachments, sent email (rendered `.html`), comment threads) into that folder
3. Downloads the shared memory file, appends a new `## Case: {TICKET-KEY}` block with timestamp + filed documents + the caller's notes, and re-uploads it
4. Returns a JSON summary with the SharePoint URL of every uploaded file

> **Why n8n:** the Microsoft 365 MCP's SharePoint write path is unreliable. The n8n workflow uses the SharePoint REST API directly via a service-account OAuth credential, which is reliable. Reads still go through the Microsoft 365 MCP -- only writes are routed through n8n.

```
https://mypos0.sharepoint.com/sites/legal/Shared Documents/myPOS Legal/
  ├── NDAs/
  │   └── LEGAL-4321/
  │       ├── LEGAL-4321_AcmeCorp_2026-04-27_v1.docx     ← draft + embedded review comments
  │       ├── LEGAL-4321_AcmeCorp_2026-04-27_attachment_NDA-Acme.pdf
  │       └── LEGAL-4321_AcmeCorp_2026-04-27_final.html
  ├── Contract Reviews/
  ├── Regulatory Questions/
  ├── Corporate Changes/
  ├── Projects/
  ├── KYC Support/
  ├── GTCs/
  ├── Materials Reviews/
  ├── Claims/
  ├── Inspection Support/
  └── Claude skills memory/Copilot/_knowledge/
      └── legal_copilot_memory.md   ← the team's shared brain (patterns + ticket log + feedback)
```

**File names** follow this pattern:

```
LEGAL-4321_AcmeCorp_2026-04-27_v1.docx        ← original draft (Devil's advocate review embedded as Word comments)
LEGAL-4321_AcmeCorp_2026-04-27_final.html      ← the email actually sent
```

The Devil's advocate review is embedded directly inside the draft `.docx` as anchored Word comments -- one comment per finding, plus a verdict summary comment on the document title. All triage outputs are filed as `.docx`; nothing is saved as `.md` (except the shared memory file, which is markdown by design).

After each `/triage`, `/triage-board`, `/file-to-sharepoint`, or `/reply-and-close` run, the chat output and the Jira comment include direct SharePoint URLs for the case folder and every uploaded file.

### The shared memory file

The team's brain (case log, lawyer feedback, emerging patterns) lives in a single SharePoint markdown file:

```
myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md
```

Each filing run appends a new `## Case: {TICKET-KEY}` block to it. Every triage starts by reading this file from SharePoint via the Microsoft 365 MCP (read), so all lawyers share the same context. There is no longer a local Desktop cache.

You can change folder names by editing one file in the plugin: `knowledge/sharepoint-map.md`. Re-publish the plugin and run `/plugin update` -- no developer needed.

---

## Safety rails (hard rules, never overridden)

- **Never sends emails itself** -- drafts only (Jira comment, SharePoint .docx, and your Outlook Drafts folder when available); the lawyer always sends. One agreed exception since 30 Oct 2026: when a contract or regulatory request is missing three or more facts, the n8n workflow emails the requester a fixed intake-form invitation (text in `docs/n8n-intake-form.md`); the Copilot cannot change its wording and never sends anything else
- **Never closes a risk-flagged ticket** -- regulator, inspection, claim, or tight-deadline tickets always require a lawyer to approve before close
- **Never auto-edits its own rules** -- when it spots a recurring lawyer correction, it proposes a change and waits for sign-off
- **Never invents facts** -- if information is missing, it asks
- **Never names individuals** -- uses roles ("the Compliance team", not "Maria from Compliance")
- **Drafts are always marked** `[DRAFT - FOR LAWYER REVIEW BEFORE SENDING]`
- **Never bypasses the n8n workflow** -- direct SharePoint writes via the Microsoft 365 MCP are not authorised here because they are unreliable
- **Never falls back to local-Desktop saving** when the n8n workflow fails -- the lawyer is told what failed and decides

---

## Updates

Run `/plugin update` whenever you want the latest version. New skills, fixes, and improved drafting style ship through the same channel. No reinstall needed.

If something looks broken after an update, you can pin to an earlier version or roll back -- ask Claude.

---

## Asking for help

Open Claude Code and say "ask the legal copilot maintainers" -- it'll prepare a bug report including your last command, the relevant Jira ticket, and the error, then post it to the maintainer channel. Or message the AI Transformation team directly.

## For pilot users

If you're the first lawyer installing this on a fresh machine, follow [docs/pilot-runbook.md](docs/pilot-runbook.md) — a 30-minute step-by-step guide that walks you through install, the OAuth dance, three smoke tests on a controlled test ticket, and verifying the next morning's automatic 08:00 sweep.

## Project structure (for the curious)

```
.claude-plugin/plugin.json   # plugin manifest
commands/                    # the slash commands you type
skills/                      # the legal triage playbooks + Devi
