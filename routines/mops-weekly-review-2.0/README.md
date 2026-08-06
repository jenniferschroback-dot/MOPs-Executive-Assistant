# MOps Weekly Review 2.0

Scheduled cloud agent (Claude Code routine) that runs the `MOps-weekly-report` skill end-to-end, unattended, ahead of the Thursday Project Review Meeting.

- **Routine ID:** `trig_01Ssw1JXB8TKzF6dd2dt1kJW`
- **Link:** https://claude.ai/code/routines/trig_01Ssw1JXB8TKzF6dd2dt1kJW
- **Schedule:** every Thursday, 7:00 AM Pacific (`0 14 * * 4` UTC). Fixed UTC cron — drifts to ~6:00 AM PT once DST ends (~Nov); needs a manual nudge back at that point.
- **Repo:** [github.com/forkanelebdi-ACQ/MOps-Executive-Assistant](https://github.com/forkanelebdi-ACQ/MOps-Executive-Assistant) (`main`) — created 2026-07-14 so the cloud agent had something to clone, and **renamed 2026-08-06 from `MOPs-Weekly-Review`**. It is no longer a weekly-review-only mirror: as of 2026-08-06 it holds this entire project, so the skill the routine clones is the same file this repo edits, not a copy that can drift.
  - ⚠️ **GitHub redirects the old URL, so the routine keeps working** — but the redirect is a courtesy, not a guarantee. If anyone creates a new repo named `MOPs-Weekly-Review` under this account, the redirect breaks and the routine starts cloning the wrong thing. Update the routine's repo config to the new name rather than relying on it.
- **Environment:** "Weekly Review Report" (`env_0135gKmDW697XShahAoE82wx`)
- **Model:** claude-sonnet-5
- **MCP connections:** Salesforce, Asana, Atlassian (Jira), Google Drive, Slack

## What it does
Follows `.claude/skills/MOps-weekly-report/SKILL.md` in the routine's repo: pulls live Asana/Salesforce/Jira data, builds the 19-slide deck, converts it to Google Slides (Drive folder `0ACqafLRVUxJzUk9PVA`), and posts the link + headline summary to **#mops-team** in Slack.

## Write contract (required dependency)
This routine performs a write (the Slack post), so its repo **must ship a copy of `.claude/rules/write-actions.md`** — cloud routines clone skill folders and do **not** inherit `.claude/rules/` from this project. Without it the routine runs unguarded.

The Slack post to `#mops-team` is the contract's **one standing unattended authorization** (§6). It covers the weekly review post to that channel and nothing else. Dedupe on the run date before sending so a retried run doesn't double-post, and append the send to `decisions/actions.md`.

## Keeping in sync
The routine's repo is a standalone push target, not a live mirror — if `MOps-weekly-report`, `.claude/rules/write-actions.md`, or its context dependencies change in this project, re-push to `MOps-Weekly-Review` or the routine will run against a stale version.

## Created
2026-07-14, per Forkan's request to automate the weekly deck + Slack delivery instead of running it manually each week.
