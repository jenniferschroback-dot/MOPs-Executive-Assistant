---
name: sla-watchdog
description: Watches open MOPS intake tickets against their per-type SLA clock and flags at-risk/breached items before they slip, with the SOP escalation path for Rush/Urgent. Use for an SLA check, an at-risk/overdue sweep, or as the daily watchdog routine. Also the single source for the per-type SLA table.
argument-hint: [scope — e.g. "all open" | a person | a Project Type]
---

# SLA Watchdog

The flagship gap: documented per-type SLAs exist, but observed on-time performance runs **~47%**. `mops-command-center` computes at-risk/SLA% for the dashboard; this skill **operationalizes** it — watches each open ticket against its type's clock and escalates *before* breach, instead of noticing after. It's also the **single source of the SLA table** (other skills, e.g. `intake-routing`'s priority step, read it from here).

## Inputs
- Scope from `$ARGUMENTS` (default: all open `[MOPs] Intake` tasks).
- Per ticket: `Project Type`, submission date, target/live date, `MOPS-Status`, assignee, stated urgency.
- SLA source of truth: Confluence **MOPAS** "Marketing Operations SLAs" + `references/sops/mops-sla-timeline.md` (twin copies — if they've drifted, flag it; don't silently pick one).

## SLA table (business days unless noted)

| Submission type | SLA |
|---|---|
| SFDC Campaign creation (≤5) | 1 day |
| SFDC Campaign creation (>5) | 2 days |
| UTMs only (campaigns exist) | 1 day |
| UTMs + net-new SFDC Campaigns | 2 days |
| Single Promotional Email | submit 5 days out → scheduled 1 day out |
| Multiple Promotional Emails | submit 8 days out → scheduled 1 day out |
| Webinar Request | 2 weeks out from live webinar |
| Event Request | 4 weeks out from live event |
| Form (standalone) Request | 5 days out → published 1 day out |
| Salesforce Report (per report) | 5 days out → final approval 1 day out |
| IT Integration Request | 7–15 days (complexity-dependent) |
| List Upload | 3–7 days (complexity-dependent) |
| Routing Request | 7–15 days (complexity-dependent) |

Also enforced: **max 3 marketing emails per record per week** (shared send calendar); **fixed weekly send-day assignments**. Rush/Urgent → escalated to Marketing VP/Director for approval.

## Logic

1. **Compute the clock** per open ticket: days remaining vs the SLA target (anchor on submission date, or on the live/target date for event/webinar/scheduled-send types).
2. **Bucket:**
   - 🔴 **Breached** — past the SLA target and not Completed.
   - 🟠 **At-risk** — inside the final portion of the window (e.g. ≤1 day of buffer, or a scheduled send inside its "1 day out" gate) and not Completed.
   - 🟢 **On track** — omit from alerts (don't add noise).
3. **Escalation:**
   - Rush/Urgent tickets → route to the VP/Director approval path per SOP (name it; don't self-approve).
   - Unassigned + at-risk/breached → escalate to triage (pair with `intake-routing`).
4. **Blocked/Waiting** tickets aren't auto-breaches — surface separately (the clock may be legitimately paused).

## Output — an alert digest (breached first, then at-risk)

Digest in chat by default. If the run is the scheduled daily sweep or the caller wants a file, write it to `outputs/sla/sla-watchdog-YYYY-MM-DD.md` — never `outputs/` root (`.claude/rules/output-files.md`).

```
## SLA Watchdog — [scope] — [date]

### 🔴 Breached ([n])
- [ticket] · [type] · owner [name] · SLA [x] · [k days over] · status [MOPS-Status]

### 🟠 At-risk ([n])
- [ticket] · [type] · owner [name] · [hours/days of buffer left]

### ⏸ Blocked / Waiting ([n]) — clock may be paused
### 🚨 Rush/Urgent needing VP/Director approval ([n])
```

## Writes — follow the contract
- **Slack alert** (`slack_send_message`) = **Class C** — confirm before sending. As a **daily routine**, an unattended post is **not** authorized (only the weekly review is, contract §6) → default to producing the digest for a human to post unless authorization is explicitly added and logged.
- **Dedupe** on a run key (date + ticket + alert-bucket) against the last run so the same breach isn't re-alerted daily (contract §4).
- **Optional "threshold → tracked issue":** the recurring ask is to *open/update a tracked issue when a metric crosses threshold* rather than only reporting weekly. Creating a Jira issue (`createJiraIssue`) = Class B and a transition = Class C — confirm per instance; never auto-transition.

## Notes
- Single source of the SLA table — `intake-routing` (priority) and `mops-command-center` read it from here; don't duplicate it elsewhere.
- The ~47% figure is an observed proxy, not a live metric — this skill's job is to close that gap ticket-by-ticket, not to reproduce the number.
- If the two SLA copies (MOPAS vs the raw doc) have drifted, flag it as a policy-sync risk rather than picking one silently.
