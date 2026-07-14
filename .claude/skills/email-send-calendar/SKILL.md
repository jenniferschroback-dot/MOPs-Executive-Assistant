---
name: email-send-calendar
description: Turns a multi-send email request into dated Asana milestone sub-tasks that populate the shared MOPS email calendar, so sends stay visible and audience clashes get caught. Use after intake-classification extracts multiple email sends (standalone email request or a webinar/event's promotional-email add-on), when it's time to schedule those sends in Asana.
---

# Email Send Calendar

Turns each individual email send `intake-classification` extracted (date, subject, pre-header, banner/creative link, body/CTA) into a dated Asana milestone sub-task, matching the manual process Harish demoed live on 2026-07-13. This is what makes the "next week's emails" / "emails sent last week" view possible without anyone re-checking the calendar by hand.

## Why this exists
Harish's manual version of this (per the 2026-07-13 meeting): one sub-task per send date, `Project Type` = `Email`, converted to a **Milestone**, so it shows up in the Asana calendar view. He uses that calendar to (a) confirm the audience for one send doesn't collide with another send going to the same audience on the same day, and (b) walk stakeholders through what's shipping next week on the Thursday call. This skill exists so that scheduling doesn't depend on someone doing it by hand every time.

## Process

1. Take the per-send data from `intake-classification` (send date, subject, pre-header, banner link, body/CTA) — one send at a time.
2. Create one Asana sub-task per send under the parent request's task, named with the send's own campaign name (per `campaign-naming/SKILL.md` — same name as the parent Event/Webinar/Email Campaign, `Channel` = `Email`).
3. Set the sub-task's `Project Type` custom field to `Email`.
4. Set the sub-task's due date to the send date.
5. **Create the sub-task as a Milestone directly** — pass `resource_subtype: "milestone"` on task creation (confirmed working against the live `[MOPs] Intake` project on 2026-07-13, see `decisions/log.md`). When setting `Project Type` (or any custom field) in the same call, pass `custom_fields` as a JSON string (`"{\"<field_gid>\":\"<option_gid>\"}"`), not a nested object — the tool rejects an object.
6. Before finalizing, **check for audience clashes**: look up other `Email`-type milestone sub-tasks due the same date and flag it if any target the same audience/region as this send — don't silently schedule a same-day, same-audience collision.
7. If this send is the promotional-email add-on for a Webinar/Event request (see `intake-classification` step 4 and `sf-campaign-spec` step 4), link it to the parent Event/Webinar Campaign once that Campaign Id exists.
8. Confirm the full set of dated sub-tasks with the user before creating them in Asana, especially for requests with several sends (e.g. 3–4 emails for one webinar).

## Notes
- This skill only schedules the sends — it doesn't write the email copy or handle the Campaign record itself; those are `intake-classification` (data capture) and `sf-campaign-spec` (Campaign creation) respectively.
- Completed milestone sub-tasks (past send date, marked done) and incomplete ones (future send date) are exactly what `MOps-weekly-report` pulls for its "emails sent last week" / "upcoming email sends" sections — keep the `Project Type` = `Email` + due-date + completion-status pattern consistent so that reporting works.
- If a stakeholder's send date, subject, or copy is missing, ask rather than leaving a placeholder in the milestone task.
