---
name: MOps-weekly-report
description: Generates the MOps Weekly Review — a 19-slide .pptx covering team throughput, email send calendar, Salesforce campaign performance, SLA/cycle time, and look-ahead priorities — pulled live from Asana/Salesforce/Jira. Use when asked to build or generate the weekly review deck ahead of the Project Review Meeting.
---

# MOps Weekly Review

Results-driven weekly deck generator, not just a status reporter: every slide should answer "what did this work *produce* for marketing?" Pull live data from every connector available. Never invent figures — every number traces to a tool query. If a connector fails or returns nothing, report the gap on the slide instead of guessing.

**Style/format reference:** `MOps_Weekly_Review_20260704.pptx` in Google Drive (folder id `0ACqafLRVUxJzUk9PVA`) is the example output to match for layout, tone, and level of detail. Its companion source doc `MOps_Weekly_Review_System_Prompt_v2.md` (same folder) is the origin of the spec below.

## Tools — use ALL of them each run
- **Asana** (`[MOps] Intake` project): completed/open tasks, assignees, due dates, blockers, sub-categories. Primary source for throughput and capacity.
  - **Email send calendar:** sub-tasks with `Project Type` = `Email` that have been converted to Milestones (see `email-send-calendar` skill) are the source for the two email-calendar slides below — completed ones due in the retro period = sent, incomplete ones due in the look-ahead period = upcoming. Query both completion states explicitly; don't rely on the default calendar-view filter, which only shows incomplete tasks (a gap Harish flagged 2026-07-13).
- **Salesforce (CRM)**: query Campaigns directly with SOQL (`soqlQuery`, `getObjectSchema`, `getRelatedRecords`, `find`). Primary source for campaign performance. Example baseline queries:
  - Campaigns created or launched in the retro period:
    `SELECT Id, Name, Type, Status, StartDate, NumberOfLeads, NumberOfContacts, NumberOfResponses, NumberOfConvertedLeads, NumberOfOpportunities, AmountAllOpportunities FROM Campaign WHERE CreatedDate = LAST_N_DAYS:7 OR StartDate = LAST_N_DAYS:7`
  - Active campaigns snapshot: same fields, `WHERE IsActive = true`.
  - Member engagement: `SELECT CampaignId, Status, COUNT(Id) FROM CampaignMember WHERE Campaign.CreatedDate = LAST_N_DAYS:7 GROUP BY CampaignId, Status`
  - Adjust field names to the org schema via `getObjectSchema('Campaign')` on first run; cache what works.
  - This skill only reads Salesforce data (no campaign creation/edit) so the read-only MCP limitation noted in `tools/available-tools.md` doesn't block it.
- **Jira**: any MOps-linked issues touched in the period (bugs, integration work); note status of anything blocking a campaign.
- If a connector needs re-auth, flag it on the Data Sources slide and continue with the rest.

## Periods
- **Retrospective:** trailing work week ending the day before the meeting.
- **Look-ahead:** next ~1.5 weeks of planned work.
Compute both dynamically from the run date; show exact dates on dividers and chart footers.

## Team (canonical names)
Aayushi Sharma (lead) · Harish Pandey · Jennifer Schroback ("Jennifer S." on cards) · Felipe Tencio. Full names on axes; charts sorted descending; include zero counts.

## Data logic
- **Completed (retro):** completed Asana tasks per assignee; team total = exec-summary number.
- **Upcoming (look-ahead):** open Asana tasks due per assignee; sum as "Total Capacity Required: N Tasks."
- **Sub-projects:** group by sub-category (Email Requests, SFDC & UTM, Webinar Requests, List Uploads); summarize activity; status badge each.
- **Blocked tickets:** ticket, assignee, the specific missing info (from task notes/fields — don't guess), priority badge.
- **SLA performance:** for each completed request, compare intake date → completion date against the MOps SLA for its request type. Report: % completed within SLA, average cycle time in business days, and any rush/late requests. If intake timestamps are missing, say so.
- **Campaign performance (Salesforce):** for campaigns launched or active in the retro period, report per campaign: members added, responses, response rate, leads/contacts, converted leads, opportunities and pipeline amount (where populated). Roll up a period total. Where a campaign maps to an Asana request, connect them ("MOps built it → here's what it produced").
- **Campaign data hygiene (Salesforce):** flag campaigns from the period with missing Type/Status/StartDate, zero members after 5+ business days, or names that don't follow the naming convention. Check against the actual convention in `.claude/skills/campaign-naming/SKILL.md` (`Region_Channel_Product_Description_YYYY-Qn`) — that skill is the source of truth for the format, not a hardcoded pattern here. Flag gently — naming guidance is advisory, not a hard fail.
- **Email send calendar (Asana):** *(added 2026-07-13, per Harish/Ayushi — mirrors what's manually walked through on the Thursday stakeholder call)*
  - **Sent last week:** completed `Email`-type milestone sub-tasks with due dates in the retro period — list campaign/send name, send date, region/audience.
  - **Upcoming next week:** incomplete `Email`-type milestone sub-tasks with due dates in the look-ahead period — same fields, plus flag any two sends sharing a date and audience/region (an audience clash Harish specifically calls out).
  - If no email milestone sub-tasks are found for a period, say so on the slide rather than leaving it blank or guessing.
- **Badges:** `ON TRACK` (green) · `IN PROGRESS` (blue) · `READY` (purple) · `BLOCKED` (red) · `AT RISK` (amber, for SLA/hygiene flags).

## Slides (19, in order)
1. Title: "MOps Weekly Review" / "Team Performance, Campaign Results & Strategic Roadmap" / `<date> | Project Review Meeting`
2. Divider — RETROSPECTIVE: "Reviewing Last Week" + period
3. Executive Summary: total completed + **3 headline results** (e.g., "12 tasks shipped · 4 campaigns launched reaching 3,240 contacts · 92% within SLA") + short narrative
4. Completed Tasks by Team Member: horizontal bar chart + source footer
5. **Emails Sent Last Week:** table — Send/Campaign Name · Send Date · Region/Audience — or "No emails sent this period" *(added 2026-07-13)*
6. **Campaign Performance (Salesforce):** table — Campaign · Members · Responses · Resp. Rate · Leads/Opps — with period roll-up line; footer citing SOQL source
7. **SLA & Cycle Time:** % within SLA, avg cycle time, rush/late list (or "No SLA breaches this period")
8. Lead Weekly Wins: categorized bullets + supporting visual
9. Team Performance Highlights: 3 cards (non-lead members)
10. Divider — LOOK AHEAD: "Planning Next Week" + period
11. Upcoming Tasks by Team Member: bar chart + "Total Capacity Required: N Tasks"
12. **Upcoming Email Sends Next Week:** table — Send/Campaign Name · Send Date · Region/Audience — flag same-day/same-audience clashes; or "No emails scheduled this period" *(added 2026-07-13)*
13. Priority Pipeline (lead): categorized priorities + visual
14. Upcoming Strategic Priorities: 3 cards (non-lead members)
15. Active Sub-Projects: table (Sub-Category · Core Activity · Status)
16. Incomplete Tickets: table (Ticket · Assignee · Missing Info · Priority)
17. **Data Hygiene & Recommendations:** flagged campaigns (missing fields, zero engagement, naming drift) + 2–3 concrete recommendations for next period, each tied to a number ("Fixing the 3 member-less campaigns unlocks response reporting for the Q3 webinar series")
18. Closing / Questions + `Next Review: <date> | Lead: <name>`
19. Data & Image Sources: every connector queried (with query date/time), any connector gaps, external image URLs + attribution

Keep slide count/order fixed; if a section has no data, keep the slide and note "No items this period." **The two email-calendar slides (5 and 12) are new as of 2026-07-13** — Forkan committed to adding them after Harish's feedback, but exact placement/format wasn't finalized in that meeting ("I'll check once more"). Treat their position as the working default, not locked, until confirmed with Jennifer/Ayushi.

## Style
- Deep navy bg (~#0d1626), off-white text; thin teal→green→orange→pink→purple gradient bar atop every slide.
- Rounded geometric sans headings; caps letter-spaced section eyebrows (teal = retro, orange = look-ahead) with short colored underline.
- Accents: teal #14b8a6, blue #3b82f6, orange #f97316. Card titles blue; icons teal or orange by section.
- Bar charts: horizontal, dark track, accent fill, value + name labels.
- Cards: rounded, bordered, icon → title → 1–3 lines. Tables: caps header row, status/priority as colored pills. Campaign performance numbers right-aligned.

## Output
Generate the deck locally as `.pptx` (named `MOps_Weekly_Review_<YYYY-MM-DD>.pptx`) to match the style/format reference above, matching its layout, tone, and level of detail. Tone: professional, concise, constructive — celebrate wins with numbers attached, surface blockers plainly, and end every retrospective claim with the result it drove. Attribute all images and cite every data source.

Then convert to Google Slides: upload the `.pptx` via Google Drive `create_file` (`contentMimeType: application/vnd.openxmlformats-officedocument.presentationml.presentation`, base64 content) into the same Drive folder as the style reference deck (folder id `0ACqafLRVUxJzUk9PVA`) — leave `disableConversionToGoogleType` unset so Drive converts it to a native Google Slides file (`application/vnd.google-apps.presentation`). If the target folder is ever wrong, confirm the right one rather than guessing.

## Delivery
Post a message to the **#mops-team** Slack channel with:
- The Google Slides link (from the `create_file` response)
- A short summary: headline stat from the Executive Summary slide (e.g., "12 tasks shipped · 4 campaigns launched · 92% within SLA") + a one-line callout of anything `BLOCKED`/`AT RISK`

There's no direct file-attachment upload to Slack via the connected MCP tools — the link is the deliverable, not an attached file. If Drive or Slack is unavailable, still finish producing the deck and flag the delivery gap rather than silently skipping it.
