---
name: jira-auto-assign
description: Pick the right Legal team member for a Jira ticket and set the Jira assignee directly via mcp__atlassian__editJiraIssue. Use after a legal-triage-* skill has produced a matter type and severity, and before /triage posts the AI Triage comment. Reads team and expertise rules from knowledge/team-routing.md so the team can re-route by editing one file.
tools: Read, Skill, mcp__atlassian__editJiraIssue, mcp__atlassian__searchJiraIssuesUsingJql, mcp__atlassian__lookupJiraAccountId, mcp__microsoft-365__sharepoint_search, mcp__microsoft-365__read_resource, mcp__n8n__execute_workflow
---

# Jira auto-assign

Pick the right lawyer for a Jira ticket and assign them. No comments, no narration -- the assignee field is the channel.

## Inputs

Caller passes a structured dict:

```json
{
  "ticket_key": "LEGAL-4321",
  "matter_type": "nda" | "contract_review" | "regulatory_question" | "corporate_change" | "project" | "kyc" | "gtcs" | "materials_review" | "claims" | "inspection_support" | null,
  "confidence": 0.0-1.0,
  "human_review_required": true | false,
  "risk_flags": ["regulator", "claim", ...],
  "priority": "Highest" | "High" | "Medium" | "Low" | "Lowest",
  "sla_days": 1-90
}
```

Optional (used only by `/triage-board` Phase 2): `force_reassign: true`, `override_account_id: "{accountId}"`.

## Step 1: Load routing rules

Read `${CLAUDE_PLUGIN_ROOT}/knowledge/team-routing.md`. If that path is unreadable (app-internal path; common in scheduled runs), fetch `team-routing.md` from the SharePoint `_knowledge` folder instead: `sharepoint_search` for `team-routing` -> take the hit under `/myPOS Legal/` -> `read_resource` on its `uri`.

Parse:

- The **Team roster** table -- skip any row with `Active = no`.
- The **Expertise map** table -- one primary plus secondaries per matter type.
- `high_severity_pool` -- the YAML list under "Severity override".
- `senior_reviewer` -- the YAML block under "Senior reviewer".
- `soft_capacity_threshold` -- integer.

If neither source is readable or any required section is unparseable, abort and return:

```json
{"assigned": false, "reason": "team-routing.md missing or malformed"}
```

Do NOT pick a random lawyer. A bad assignment is worse than no assignment.

## Step 2: Compute severity

`severity = "high"` if any of these is true:

- `matter_type` in `["claims", "inspection_support"]`
- `risk_flags` is non-empty
- `priority` in `["Highest", "High"]` (Jira "High" maps to P1 in myPOS practice)
- `sla_days <= 2`
- `human_review_required == true` AND `confidence < 0.7`

Otherwise `severity = "normal"`.

## Step 3: Pick the candidate

### If severity = high

1. Read the SharePoint memory file: `sharepoint_search` for `legal_copilot_memory` -> take the hit whose `webUrl` contains `/myPOS Legal/` (NEVER the old private `/myPOS Legal 1/` copy) -> `read_resource` on its `uri`.
2. Search for the line `<!-- legal-copilot:high-severity-counter:N -->`. Extract `N` (integer). If missing, `N = 0`.
3. If `N % 2 == 0`: candidate is `high_severity_pool[0]`. Else `high_severity_pool[1]`.
4. After the assignment succeeds (Step 5), increment `N` via the n8n filing workflow (Step 6).

### If severity = normal

1. Look up `matter_type` in the **Expertise map**. If matter_type is `null` or not in the map: skip straight to **senior_reviewer**.
2. Candidate = the **primary** for that matter type, if `Active = yes`.
3. If the primary is inactive OR over capacity (Step 4), walk the **secondaries** in order. First active + under-capacity lawyer wins.
4. If no secondary is available either: candidate = senior_reviewer.

## Step 4: Soft capacity check

For the candidate, run JQL via `mcp__atlassian__searchJiraIssuesUsingJql`:

```
project in (LEGAL, AIRD) AND assignee = "{candidate.account_id}" AND statusCategory != Done
```

Use `fields: ["summary"]` and `maxResults: 50` (enough to count, we don't need the bodies).

If the result count is >= `soft_capacity_threshold`, treat the candidate as over capacity and pick the next from the chain (Step 3 secondaries, then senior_reviewer).

The senior_reviewer is exempt from the capacity check -- they are always assignable as the last resort.

## Step 5: Assign

Call:

```
mcp__atlassian__editJiraIssue(
  cloudId = "fb47470f-f5c2-44bc-8182-f2a22f059adb",
  issueIdOrKey = ticket_key,
  fields = {"assignee": {"accountId": candidate.account_id}}
)
```

If the call returns an error (account does not exist, permission denied), retry once with the senior_reviewer. If that also fails, abort and return:

```json
{"assigned": false, "reason": "edit-issue failed: {error}"}
```

**Transition interaction:** if the caller transitions the ticket in the same flow, the transition MUST happen before this assignment, and the caller MUST re-read the assignee afterwards -- the LEGAL project's "Start Progress" post-function reassigns the ticket to the acting account (observed overwriting correct owners 6 times in June 2026). If the verify shows the wrong assignee, call editJiraIssue once more.

## Step 6: Update the high-severity counter (high severity only)

Call the n8n filing workflow `VAKq9Bra0RA0SdCO` with a memory-only update payload:

```json
{
  "case_id": "{ticket_key}",
  "case_folder": null,
  "documents": [],
  "memory_instructions": "increment_high_severity_counter"
}
```

The workflow side handles the actual `N -> N+1` rewrite. If the workflow returns `success: false`, log a warning to the chat summary but do NOT roll back the assignment -- a slightly stale counter is preferable to leaving the ticket unassigned.

## Step 7: Return

```json
{
  "assigned": true,
  "assignee_name": "Jay Manjdadria",
  "assignee_account_id": "62855af6222d36006fb76bdd",
  "severity": "normal" | "high",
  "rule_applied": "matter_type_primary" | "matter_type_secondary[i]" | "high_severity_oscillation" | "senior_reviewer_fallback" | "capacity_override" | "last_replier",
  "open_load_at_assignment": 3
}
```

The calling command embeds this verbatim into the AI Triage Jira comment under a new "Auto-assigned" line. No separate Jira comment is posted by this skill.

---

## Hard rules

- NEVER post a Jira comment. The assignee field IS the signal -- comments would pollute the audit trail.
- NEVER assign on the wrong cloudId. Always use `fb47470f-f5c2-44bc-8182-f2a22f059adb`.
- NEVER guess an account_id. If lookup fails, fall back to the senior_reviewer.
- NEVER reassign a ticket that already has an assignee unless the caller passes `force_reassign: true` (used only by `/triage-board` last-replier logic). Default is single-shot.
- NEVER edit `team-routing.md` from this skill. The team owns that file.
- NEVER read the old private memory file under `myPOS Legal 1/` (trailing " 1").
- ALWAYS assign AFTER any status transition in the same flow, and verify the assignee stuck.
