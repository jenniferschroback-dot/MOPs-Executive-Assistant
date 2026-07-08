---
name: intake-classification
description: Classifies a raw MOPS intake form submission into structured ticket data (request type, requester, priority, dates, deliverables) and maps it to the Asana task + sub-tasks it needs. Use when a new intake form or ticket comes in and needs to be read, interpreted, and turned into Asana work.
---

# Intake Classification

Turns an unstructured intake form submission into structured data ready to become an Asana task (and its sub-tasks), instead of MOPS reading and re-typing it by hand.

## Process

1. **Read the raw submission** (pasted text, form export, or linked ticket).
2. **Extract fields:**
   - Requester (name, team)
   - Request type (see table below)
   - Target date / deadline
   - Priority/urgency (if stated; otherwise ask)
   - Any product, region, or segment mentioned
   - Free-text goal/description
3. **Classify the request type** against the known types below. If it doesn't match a known type, say so explicitly and ask the user to confirm the type rather than guessing.
4. **Expand into sub-tasks** using the mapping below.
5. **Output a structured summary** (bullet points) before creating anything in Asana — get confirmation first, then use the Asana MCP to create the parent task and sub-tasks.

## Known request types → sub-tasks

_Only two types are confirmed from actual MOPS workflow so far — extend this table as more request types come up in real intake forms._

| Request type | Confirmed sub-tasks |
|---|---|
| Email Campaign | (sub-task list not yet specified — ask MOPS/Forkan for the standard breakdown before assuming one) |
| Event / Webinar | Landing page (confirmed example). Likely also needs confirmation email, calendar invite, promo — **verify before assuming** |

For any other request type, do NOT invent a sub-task list — ask what breakdown MOPS uses, then add it to this table for next time.

## Output format

```
**Request type:** ...
**Requester:** ...
**Priority:** ...
**Target date:** ...
**Summary:** ...
**Proposed sub-tasks:** ...
```

## Notes
- Campaign naming is a separate step — see the `campaign-naming` skill. Don't generate a campaign name here.
- If the intake volume or fields don't match what's described here, treat this as a signal to update the skill, not to guess silently.
