---
name: intake-tracking
description: Reports on intake ticket status across Asana/Jira/Salesforce — what's new, stuck, overdue, or approaching launch — and estimates intake-to-launch time. Use when asked for a status update, a list of open/stuck tickets, or how a campaign is tracking toward launch.
---

# Intake Tracking / Status Reporting

Gives a status view across the intake-to-launch pipeline without MOPS manually checking Asana, Jira, and Salesforce separately.

## Process

1. Query the Asana MCP (and Jira MCP, if the ticket lives there) for intake-related tasks: open, in-progress, and recently completed.
2. Cross-reference against Salesforce campaign records where relevant (has a campaign been created yet? is it live?).
3. Group and report:
   - **New / unclassified** — intake received, not yet turned into an Asana task
   - **In progress** — classified, task created, work underway
   - **Stuck / overdue** — no movement past an expected checkpoint, or past target date
   - **Launched** — Salesforce campaign live
4. When asked "how long is intake-to-launch taking," compute the gap between intake submission date and the Salesforce campaign's live date for completed tickets, and report the average/range.
5. Present as precise bullet points (per @.claude/rules/communication-style.md), not a wall of text.

## Notes
- If Asana/Jira task fields don't cleanly indicate "intake date" or "launch date" yet, flag that gap instead of guessing at dates — this skill's accuracy depends on those fields existing and being populated.
- This skill reads status; it doesn't move tickets forward on its own. Use `intake-classification`, `campaign-naming`, and `sf-campaign-spec` for that.
