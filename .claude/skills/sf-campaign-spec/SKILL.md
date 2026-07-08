---
name: sf-campaign-spec
description: Builds the Salesforce campaign record spec — campaign type, dates, and campaign member status scaffolding — from classified, named intake data, and creates it via the Salesforce MCP. Use after a request has been classified and named, when it's time to actually create or configure the Salesforce campaign.
---

# Salesforce Campaign Spec + Status Scaffolding

Creates the Salesforce side of a request: the campaign record and its member status scaffolding, matched to campaign type — so statuses aren't configured by hand per campaign and the Pardot ↔ SF sync doesn't break.

**The exact schema below is a placeholder.** Acquia's real Salesforce campaign record types, required fields, and status picklist values have not been confirmed yet. Do not treat the table below as production truth — verify against the actual org (or ask Forkan/MOPS) before relying on it, and update this file once confirmed.

## Process

1. Take the classified + named request (from `intake-classification` and `campaign-naming`).
2. Determine the campaign type (Email Campaign, Event/Webinar, etc.).
3. Look up the required member statuses for that type in the table below.
4. **If the campaign type isn't in the table, or you're not certain the statuses are current, say so and ask** rather than defaulting to a generic status list.
5. Propose the full spec (name, type, dates, statuses) and confirm before creating anything.
6. Use the Salesforce MCP to create the campaign record and configure member statuses.

## Campaign type → status scaffolding (PLACEHOLDER — VERIFY)

| Campaign type | Draft member statuses |
|---|---|
| Email Campaign | Sent → Opened → Clicked → Responded _(typical Pardot-style flow — not yet confirmed against this org)_ |
| Event / Webinar | Invited → Registered → Attended → No Show _(typical event flow — not yet confirmed)_ |

## Notes
- Getting this wrong silently is worse than asking — a bad status scaffold is exactly the "wrong status scaffolding" pain point this skill exists to fix, not repeat.
- Once real record types/statuses are confirmed, replace the table above and remove this warning.
