# MOps Weekly Review 2.0

Scheduled cloud agent (Claude Code routine) that runs the `MOps-weekly-report` skill end-to-end, unattended, ahead of the Thursday Project Review Meeting.

- **Routine ID:** `trig_01Ssw1JXB8TKzF6dd2dt1kJW`
- **Link:** https://claude.ai/code/routines/trig_01Ssw1JXB8TKzF6dd2dt1kJW
- **Schedule:** every Thursday, 7:00 AM Pacific (`0 14 * * 4` UTC). Fixed UTC cron — drifts to ~6:00 AM PT once DST ends (~Nov); needs a manual nudge back at that point.
- **Repo:** [github.com/forkanelebdi-ACQ/MOPs-Weekly-Review](https://github.com/forkanelebdi-ACQ/MOPs-Weekly-Review) (`main`) — separate repo created 2026-07-14 specifically so the cloud agent has something to clone; mirrors this project's `.claude/skills/MOps-weekly-report/SKILL.md`.
- **Environment:** "Weekly Review Report" (`env_0135gKmDW697XShahAoE82wx`)
- **Model:** claude-sonnet-5
- **MCP connections:** Salesforce, Asana, Atlassian (Jira), Google Drive, Slack

## What it does
Follows `.claude/skills/MOps-weekly-report/SKILL.md` in the routine's repo: pulls live Asana/Salesforce/Jira data, builds the 19-slide deck, converts it to Google Slides (Drive folder `0ACqafLRVUxJzUk9PVA`), and posts the link + headline summary to **#mops-team** in Slack.

## Keeping in sync
The routine's repo is a standalone push target, not a live mirror — if `MOps-weekly-report` (or its context dependencies) changes in this project, re-push to `MOps-Weekly-Review` or the routine will run against a stale skill version.

## Created
2026-07-14, per Forkan's request to automate the weekly deck + Slack delivery instead of running it manually each week.
