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
   - Once `Project Type` is known, run it through **`campaign-gate-check`** to get the yes/no/needs-human-input Salesforce Campaign gate — that skill is the source of truth for the gate table, not the "Creates SF Campaign?" column below.
4. **For `Webinar Request` / `Event (+ SFDC Campaign)` requests, check for a promotional-email flag** — a yes/no answer to "do you require a promotional email as part of this request?" (confirmed pattern from the existing webinar landing-page form, see `decisions/log.md` 2026-07-13). If the raw submission doesn't state this explicitly, ask rather than assuming. This flag is the gate `sf-campaign-spec` uses to decide whether to create the companion Email Campaign alongside the Event/Webinar Campaign — no promotional email means just the one Campaign.
5. **For any email-channel request (`Email(s) only | Nurture Sequences`, or the promotional-email add-on above), check whether it covers more than one send.** Stakeholders share multi-email requests inconsistently — as separate ticket comments per email, as a linked doc with one row per send, or as a single ticket — so read the whole submission (including comments/linked docs) before assuming there's only one email. For each send, extract: send date, subject line, pre-header, banner/creative link, and body/CTA copy. If any of these are missing for a send, flag the gap rather than inventing copy.
6. **Expand into sub-tasks** using the mapping below. For multi-send email requests, create one dated sub-task per email (see `email-send-calendar` skill for how these become milestone tasks on the shared email calendar).
7. **Output a structured summary** (bullet points) before creating anything in Asana — get confirmation first, then use the Asana MCP to create the parent task and sub-tasks.
8. **If `campaign-gate-check` returns `no`,** don't route the ticket through `campaign-naming`/`sf-campaign-spec` — instead post a short alert to the MOPS team Slack channel flagging the new task so a human decides what (if anything) to do with it. This is a proactive heads-up, not a note on the Asana ticket itself — per existing guidance, still don't comment "no campaign needed" on the ticket.
   - **If `campaign-gate-check` returns `needs-human-input`,** don't guess and don't create sub-tasks — post to Slack flagging the ambiguous `Project Type`/gate result for a human to resolve, then update `campaign-gate-check/SKILL.md`'s table once confirmed.

## Known request types → sub-tasks

Confirmed 2026-07-11 by cross-referencing live Asana `[MOPs] Intake` tasks (`Project Type` custom field, gid `1206591746930193`) against actual Salesforce Campaign records — see `decisions/log.md`. This field is the classification vocabulary; use its exact values, don't invent new ones.

**Whether the request needs `sf-campaign-spec` at all is decided by `campaign-gate-check`, not this table.** The "Creates SF Campaign?" column below is kept only as a quick-reference match to that skill's gate table — if the two ever disagree, `campaign-gate-check/SKILL.md` wins; update this column to match rather than the other way around.

| Project Type (Asana value) | Creates SF Campaign? | Notes / confirmed sub-tasks |
|---|---|---|
| `Webinar Request` | **Yes** — resolves to a pair of Campaigns (Webinar/Event type + Email type) | Observed cluster (from live tasks, not yet MOPS-confirmed as the fixed standard — sanity-check before relying on it as exhaustive): Landing page build + confirm live, Kickoff meeting, Registration emails (1–4), Banner/slide creative requests, Zoom setup, Dry run, Day-of hosting, Pre-event audience report, Post-webinar attendee + no-show emails, Recording upload to DAM, SFDC Campaign(s) & Form sub-task |
| `Event (+ SFDC Campaign)` | **Yes** — single or multiple Campaign records depending on event complexity | Same general shape as Webinar Request; physical/virtual event framing |
| `SFDC Campaign only (single)` | **Yes** — exactly one Campaign record | Usually the specific sub-task under a larger parent (name literally contains "SFDC Campaign") |
| `SFDC Campaigns only (multiple)` | **Yes** — more than one Campaign record for the same initiative | e.g. one Event-type + one Email-type Campaign for the same promo |
| `UTM(s) (+ SFDC Campaign)` | **Yes** (option currently disabled in Asana — confirm it's actually retired before assuming) | Tied to UTM-tracked promos |
| `Email(s) only \| Nurture Sequences` | **No** | Email build/send against a Campaign created elsewhere. Nurture Sequences subtype gets its own naming code — see `campaign-naming/SKILL.md`. |
| `Audiences` | **No** | Target list / audience pull for an existing Campaign |
| `Reporting` | **No** | Reads Campaign performance, doesn't create one |
| `List Upload` | **Usually not — check first** | Per Harish (2026-07-13): ~70% of the time this uses the Campaign already created for that event/webinar, no new one needed. The ~30% exception is a partner-hosted event/webinar (not hosted by Acquia/MOPS) with no existing Acquia Campaign — that case needs a new Campaign created just for the list import. Check whether a Campaign already exists for the named event before assuming either way. |
| `Form Request` | **Usually not — check first** | Per Harish (2026-07-13): ~70% of the time the Campaign already exists (created when the Event/Webinar Campaign was set up) and this ticket just needs the existing Campaign ID wired into the landing page form — no new Campaign. The ~30% exception is when the landing page was built by the web team ahead of any Campaign creation; then a new Campaign + ID is needed. Check for an existing Campaign tied to the named event/webinar before assuming either way. |
| `Other` | **No** | Misc execution/QA (push page live, test form, etc.) |
| `UAT`, `IT/Integration`, `Automations \| Martech`, `Issues`, `Team OOO` | **No** | Internal ops, not campaign-facing |

For any request that doesn't cleanly map to one of these `Project Type` values, do NOT invent a sub-task list or guess whether it needs an SF campaign — ask, then add the confirmed answer to this table.

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
- Turning a multi-send email request into dated Asana milestone sub-tasks (the shared email calendar) is a separate step — see the `email-send-calendar` skill.
- The Salesforce-Campaign gate (including the `List Upload`/`Form Request` existing-Campaign check) lives entirely in `campaign-gate-check` now — don't re-derive it here or in `sf-campaign-spec`.
- If the intake volume or fields don't match what's described here, treat this as a signal to update the skill, not to guess silently.
