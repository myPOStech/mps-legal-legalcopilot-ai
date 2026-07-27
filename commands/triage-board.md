---
description: Sweep the Jira board -- triage every To-Do legal ticket end-to-end, auto-assign, set fields, and re-assign + transition any ticket where a legal team member already replied. Files outputs to SharePoint via n8n. Runs daily at 08:00 Europe/Sofia on weekdays.
argument-hint: (no args)
---

# /triage-board

Process every To-Do ticket on the legal board: dedupe, classify, draft, three-pass review (legal -> Devil's advocate -> business), set Jira fields, auto-assign, file via the n8n workflow, comment.

**Designed to run autonomously every weekday at 08:00 Europe/Sofia** (registered by `/setup-copilot`), and on demand whenever you want a fresh sweep.

## Constants

- Atlassian Cloud ID: `fb47470f-f5c2-44bc-8182-f2a22f059adb`
- Default JQL: `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC`
- n8n workflow ID: `VAKq9Bra0RA0SdCO`
- Shared memory file: `myPOS Legal/Claude skills memory/Copilot/_knowledge/legal_copilot_memory.md` (**the shared Legal team library -- an old private copy exists under `myPOS Legal 1/` and must never be read**)
- Team routing config: `${CLAUDE_PLUGIN_ROOT}/knowledge/team-routing.md`

> **Tool naming:** Microsoft 365 and Atlassian tools may be mounted under a session-specific server prefix. Match tools by suffix. If a tool named here does not exist under any prefix, follow the step's fallback -- do not invent a tool name.

## Phase 0: Context budget (hard rules for the whole sweep)

Scheduled sweeps have died mid-run from oversized payloads (4 of 12 runs in June 2026). These rules are not optional:

- The board-scan JQL MUST request only: `key, summary, status, assignee, priority, duedate, updated, labels`. Never fetch descriptions or comments in the board scan.
- Fetch full ticket content ONLY for tickets selected for new triage, ONE at a time, with `responseContentFormat: markdown`. If a single payload still exceeds ~20K tokens, work from summary + the first 2000 characters of the description.
- Run each per-ticket triage inside the `triage-reviewer` / `business-reviewer` subagent pipeline; never hold more than one full ticket in the main context.
- Read ONLY the live memory file under `myPOS Legal/`. Never open the old private `myPOS Legal 1/` copy even if search returns it.
- Cap new triages at 3 per sweep. List deferred tickets in the Phase 6 summary.
- **Checkpoint after every phase:** append one line to a local run-log file (`run-log-{date}.md` in the working folder): phase name, tickets written to, timestamp. If the run dies, the next sweep (or a human) can see exactly what was already applied to Jira.

## Phase 1: Refresh shared knowledge

### 1a. Read shared memory

The Microsoft 365 connector has no direct path-read tool. Use: `sharepoint_search` for `legal_copilot_memory` -> take the hit whose `webUrl` contains `/myPOS Legal/` -> `read_resource` on its `uri`.

Plus the bundled seeds in `${CLAUDE_PLUGIN_ROOT}/knowledge/`:

- `patterns.md`
- `feedback-log.md`
- `sharepoint-map.md`
- `team-routing.md` (consumed by `jira-auto-assign`)

**Fallbacks (in order):** SharePoint memory read fails -> seeds only, surface a warning. Bundled seeds unreadable (app-internal path; common in scheduled runs) -> fetch them from the SharePoint `_knowledge` folder with the same search + read pattern. Missing there too -> conservative defaults, flag in Phase 6.

### 1b. Detect lawyer corrections

JQL: `project in (LEGAL, AIRD) AND status = Done AND resolved >= -7d ORDER BY resolved DESC` with `fields: ["summary", "assignee", "resolutiondate"]`. For each closed ticket with an "AI Triage" comment, diff the AI draft against the sent reply. Each `/reply-and-close` run should have appended a `### Lawyer feedback` block to the memory file -- if a recent close-out is missing it, list the omission in Phase 6.

### 1c. Detect patterns

3+ corrections sharing category + matter type + jurisdiction -> propose a new pattern (PROPOSED -- awaits lawyer approval; surfaced in Phase 6). The sweep never auto-edits a skill.

---

## Phase 2: Last-replier reconciliation

**Before** scanning To-Do tickets, reconcile any ticket where a Legal team member has already responded.

### 2a. Find candidates

JQL: `project in (LEGAL, AIRD) AND status = "To Do" AND updated >= -7d`

For each ticket, call `mcp__atlassian__getJiraIssue` with `fields: ["comment", "assignee", "summary", "status"]` and `responseContentFormat: markdown`.

Walk the comment thread in reverse chronological order. For each comment, the author's `accountId` is what counts.

Skip any comment authored by:

- The Copilot itself (look for the `## AI Triage` heading or the `SLA Warning` heading -- those are bot comments)
- The reporter (the ticket originator -- they aren't "answering")
- Anyone NOT in the active legal roster from `team-routing.md`

The first matching comment is the **last legal-team reply**. Its author is the **last_replier**.

### 2b. Transition, then reassign, then verify

If a `last_replier` exists for a ticket:

1. If `status == "To Do"`:
   - `mcp__atlassian__getTransitionsForJiraIssue` to find the transition ID for `In Progress`.
   - `mcp__atlassian__transitionJiraIssue` to move the ticket from To Do to In Progress.
2. If `current_assignee != last_replier` (re-check AFTER the transition):
   - Invoke `jira-auto-assign` with `force_reassign: true` and `override_account_id: {last_replier.account_id}` (the skill respects the override and skips its normal expertise-map logic). Record the rule applied as `last_replier`.
3. **Verify:** re-read the assignee field after the transition + assignment. The LEGAL project's "Start Progress" post-function reassigns the ticket to the acting account (observed overwriting correct owners 6 times in June 2026). If the assignee is not the intended one, set it again and note the correction in Phase 6.
4. Skip this ticket in Phase 3 -- the lawyer is on it already; the Copilot should not redraft over their work.

Record the moved tickets in the Phase 6 summary under `Last-replier reassigned and transitioned`.

**Guardrails:**

- NEVER reassign a ticket whose last legal reply is older than 14 days. After 14 days assume the lawyer has dropped it; let Phase 3 triage from scratch.
- NEVER transition a ticket out of To Do for any reason other than a legitimate last-replier match in this phase. Phase 3's normal triage NEVER auto-transitions To-Do tickets.
- NEVER post a comment as part of Phase 2. The assignee field + status field are the channel.

---

## Phase 3: Scan and process

JQL: `project in (LEGAL, AIRD) AND status = "To Do" ORDER BY created ASC` (compact fields only -- see Phase 0)

Exclude any tickets handled in Phase 2.

**Re-triage cap:** exclude any ticket that already carries an `## AI Triage` comment with NO human comment after it. Those tickets are awaiting the lawyer, not another triage (LEGAL-4990 was AI-triaged 4 times in 21 days under the old behaviour). If such a ticket's SLA is breached, Phase 4 handles the single escalation line.

For each remaining To-Do ticket (max 3 per sweep), follow `/triage`'s Steps 2-10:

1. Fetch ticket (one at a time, compact format)
2. Dedupe
3. Classify with the matching legal-triage skill
4. Apply risk gates
5. Devil's advocate review (`triage-reviewer` subagent)
6. Business reviewer (`business-reviewer` subagent)
7. Auto-assign (`jira-auto-assign`) + set fields (`jira-fields-and-flags`)
8. File outputs to SharePoint via `sharepoint-filer` (n8n)
9. Deliver the draft (Jira comment + SharePoint .docx; plus a native Outlook draft when a tool with suffix `outlook_create_draft` is present, else the n8n draft workflow if deployed -- see `/triage` Step 9.)
10. Post the AI Triage Jira comment

### Ticket disposition

| Result | Status |
|---|---|
| Successfully triaged, no risk, both reviewers approve | Stay in To-Do (the lawyer reviews and runs `/reply-and-close`) |
| Risk flags / `human_review_required` | Stay in To-Do, marked clearly |
| Duplicate of an open ticket | Transition to Done with merge comment |
| Conflict | Stay in To-Do; both tickets flagged |
| Doesn't fit any of the 10 matter types | Stay in To-Do; clarification comment posted |
| n8n filing failed | Stay in To-Do; error noted in Phase 6; NO AI Triage Jira comment posted |
| Auto-assign failed | Triage continues; assignee left unset; Phase 6 calls it out as a routing gap |

The board sweep never auto-closes a triaged ticket. Closing is always lawyer-gated via `/reply-and-close`. Only confirmed duplicates auto-close.

---

## Phase 4: SLA watchdog

```
JQL: project in (LEGAL, AIRD) AND statusCategory != Done ORDER BY created ASC
```

(compact fields only)

For each open ticket, compare `now() - created` against the SLA. If >75% elapsed AND no lawyer response since the AI triage comment AND no SLA Warning already posted for the current SLA window, post:

```markdown
**SLA Warning**

This ticket was created {N} days ago. SLA for {priority} priority is {sla_days} days.
{remaining_days} day(s) remaining before SLA breach.

*Please review the AI draft in the AI Triage comment above (and the SharePoint case folder), then run `/reply-and-close` or escalate.*
```

> Note: the SLA *deadline* is already on the ticket's `duedate` field (set by `jira-fields-and-flags`). This Phase 4 comment is the only acceptable case where the Copilot posts deadline-like content -- the warning is the action, not the metadata.

---

## Phase 5: Overdue check (lightweight)

```
JQL: project in (LEGAL, AIRD) AND duedate < now() AND statusCategory != Done
```

Count overdue tickets per assignee. If any assignee has >= 3 overdue tickets, surface in Phase 6 under `Overload signals`. Do NOT post per-ticket Jira comments here; the productivity report (`/legal-report`) is the long-form view.

---

## Phase 6: Print summary

```
Board sweep complete ({timestamp})

Last-replier reassigned and transitioned: {count}
  {list of {ticket_key, from -> to, reason: "last reply by {name}"}}
  {list of post-function assignee corrections, if any}

Processed (new triage): {total} (cap 3/sweep; deferred: {list})
  Triaged + draft ready: {count}
  Awaiting lawyer review: {count}
  Merged duplicates: {count}
  Conflicts escalated: {count}
  Clarification requested: {count}
  SLA warnings posted: {count}
  n8n filing failures (need re-run): {count}
    {list of {ticket_key, error}}
  Auto-assign failures (routing gap): {count}
    {list of {ticket_key, reason}}

Overload signals:
  {list of {assignee, overdue_count}} (>= 3 overdue tickets each)

SharePoint case folders created:
  {list of URLs}

Memory file updated:
  https://mypos0.sharepoint.com/sites/legal/Shared%20Documents/myPOS%20Legal/Claude%20skills%20memory/Copilot/_knowledge/legal_copilot_memory.md

{if pattern proposals: "PROPOSED PATTERNS (review and approve):"}
{pattern proposals}

Next scheduled run: {next weekday 08:00 Europe/Sofia}
Run /legal-report for the full productivity view.
```

File the run summary via the n8n workflow under `_runs/triage-board/RUN-{date}-board-sweep`. If the workflow is unavailable, say so in the summary -- do not silently skip.

---

## Hard rules

- NEVER send emails. The Copilot only creates drafts (Jira comment, SharePoint .docx, and a native Outlook draft when `outlook_create_draft` is available); the lawyer sends from Outlook (see `/triage` Step 9).
- NEVER auto-close a triaged ticket. Closing is lawyer-gated via `/reply-and-close`. Only confirmed duplicates auto-close.
- NEVER auto-edit any skill. Propose patterns, wait for human approval.
- NEVER bypass the n8n workflow for SharePoint writes.
- NEVER post the AI Triage Jira comment for a ticket whose n8n filing failed.
- NEVER fall back to local-Desktop saving on workflow failure.
- NEVER transition a ticket out of To Do except via the Phase 2 last-replier rule or a duplicate-merge close. Phase 3's normal triage NEVER auto-transitions To-Do tickets.
- NEVER reassign in Phase 2 if the last legal reply is older than 14 days.
- NEVER post the priority/due/flag/assignee as a comment -- those are fields now (set by `jira-fields-and-flags` and `jira-auto-assign`).
- NEVER violate the Phase 0 context budget: compact JQL fields, one full ticket at a time, subagent isolation, 3-triage cap, per-phase checkpoints.
- NEVER re-triage a ticket whose AI Triage comment has no human reply after it.
- NEVER read the old private memory file under `myPOS Legal 1/` (trailing " 1").
