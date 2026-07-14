# Work / Business Context

## Company
Acquia

## Team
MOPS (Marketing Operations) — 5 people. See @context/team.md for who's on it.

**Core team responsibilities:**
- Keep Asana up to date
- Create Salesforce campaigns
- Break big tasks (e.g. "Email Campaign", "Event") into required sub-tasks (e.g. landing page for a webinar)

**Forkan's personal focus within the team:**
- Intake form classification
- Campaign naming and Salesforce campaign spec creation
- Building automations that facilitate MOPS tasks

## Volume
Roughly 10–20 intake forms come in per week.

## Tools used daily
Asana, Salesforce, Pardot, Jira, Claude Code, Slack

## MCP servers connected
Asana, Salesforce, Jira, Slack (all connected as of this setup). Pardot is used daily but not yet MCP-connected.

## Known pain points
- **Manual intake** — requestors fill a form; MOPS reads, interprets, and re-enters the data by hand into Asana tasks and SF campaign records.
- **No naming enforcement** — campaign names drift from convention; SF records get created inconsistently, breaking reporting and attribution. _(Convention now defined: `Region_Channel_Product_Description_YYYY-Qn` — see `.claude/skills/campaign-naming/SKILL.md`.)_
- **Wrong status scaffolding** — campaign member statuses must be manually configured per campaign type; missed steps break the Pardot ↔ SF sync.
- **Slow time to launch** — the gap between intake submission and a fully configured, live SF campaign is measured in days, not minutes.
