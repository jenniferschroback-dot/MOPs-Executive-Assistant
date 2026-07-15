---
name: sf-campaign-spec
description: Builds the Salesforce campaign record spec — campaign type, dates, and campaign member status scaffolding — from classified, named intake data, and creates it via the Salesforce MCP. Use after a request has been classified and named, when it's time to actually create or configure the Salesforce campaign.
---

# Salesforce Campaign Spec + Status Scaffolding

Creates the Salesforce side of a request: the campaign record and its member status scaffolding, matched to campaign type — so statuses aren't configured by hand per campaign and the Pardot ↔ SF sync doesn't break.

**The exact schema below is a placeholder.** Acquia's real Salesforce campaign record types, required fields, and status picklist values have not been confirmed yet. Do not treat the table below as production truth — verify against the actual org (or ask Forkan/MOPS) before relying on it, and update this file once confirmed.

**Known limitation:** the connected Salesforce MCP is currently read/query only (see `tools/available-tools.md`) — there is no create/update tool available yet. Propose the full spec as normal, but flag that the actual create step needs to happen manually (or via a write-capable connection) until that changes.

## Process

1. Take the classified + named request (from `intake-classification` and `campaign-naming`).
2. **Check the gate first, via `campaign-gate-check`** — don't re-derive the gate here. Only continue past this step if it returns `yes`.
   - **`no`:** skip silently — don't create a campaign, and don't post a "no campaign needed" note/comment either (`intake-classification` handles alerting the team via Slack for these).
   - **`needs-human-input`:** speak up rather than guessing — flag the ambiguous `Project Type`/gate result for a human call, then stop.
3. Determine the campaign type (Email, Event, Webinar, etc.) and whether this request needs one Campaign record or more than one. Webinar/Event requests resolve to a Webinar/Event-type Campaign *and* a separate Email-type Campaign **only when** `intake-classification` recorded the promotional-email flag as yes — if no promotional email was requested, create just the single Event/Webinar Campaign.
4. **When creating a paired Event/Webinar + Email Campaign,** set the Event/Webinar Campaign as the **Parent Campaign** of the Email Campaign (Salesforce `ParentId` field) — this is how Harish keeps the pair anchored to a single place instead of two disconnected records. Use the naming pattern from `campaign-naming/SKILL.md` step 6 (identical name, `Channel` segment swapped) for the child.
5. Look up the required member statuses for that type in the table below.
6. **If the campaign type isn't in the table, or you're not certain the statuses are current, say so and ask** rather than defaulting to a generic status list.
7. Propose the full spec (name, type, dates, statuses, parent/child linkage if applicable) and confirm before creating anything.
8. Use the Salesforce MCP to create the campaign record(s) and configure member statuses.

## Campaign type → status scaffolding (PLACEHOLDER — VERIFY)

| Campaign type | Draft member statuses |
|---|---|
| Email Campaign | Sent → Opened → Clicked → Responded _(typical Pardot-style flow — not yet confirmed against this org)_ |
| Event / Webinar | Invited → Registered → Attended → No Show _(typical event flow — not yet confirmed)_ |

## Notes
- Getting this wrong silently is worse than asking — a bad status scaffold is exactly the "wrong status scaffolding" pain point this skill exists to fix, not repeat.
- Once real record types/statuses are confirmed, replace the table above and remove this warning.
