# MOps Command Center

Scheduled cloud agent(s) that run the `mops-command-center` skill unattended to (a) deliver the manager's morning brief and (b) fire real-time threshold alerts during the day. This is the proactive layer behind the manager dashboard.

> **Status: BLUEPRINT — not yet provisioned.** The skill (`.claude/skills/mops-command-center/SKILL.md`) and dashboard template exist; the cloud routine has not been created. Fill in the IDs/repo/env fields below when it is. Before enabling, complete the prerequisites in "Before this can run."

## Proposed shape — two triggers, one repo/skill

A single routine has one cron. This needs two cadences, so provision **two triggers** that share the same repo and skill, differing only by schedule and mode:

| Trigger | Schedule | Mode | Does |
|---|---|---|---|
| `mops-command-center-brief` | weekdays 7:00 AM PT (`0 14 * * 1-5` UTC) | `full` | Run the skill, post the manager brief to Slack, re-publish the dashboard snapshot. |
| `mops-command-center-alerts` | weekdays ~9–5 PT, every 2h (`0 16,18,20,22,0 * * 1-5` UTC) | `alerts-only` | Run only the threshold checks; post to Slack **only when a `warn`/`critical` alert trips**, deduped against last run. |

_(DST caveat, same as the weekly review: fixed UTC cron drifts ~1h once DST ends (~Nov) — nudge the cron then.)_

- **Routine IDs:** _TBD_
- **Repo:** _TBD_ — a separate repo the cloud agent clones (pattern: mirror this project's `.claude/skills/mops-command-center/` + the `context/` files the skill relies on). Standalone push target, not a live mirror — re-push after any skill change.
- **Environment:** _TBD_ (its own env; confirm which connectors are authorized there — separate from any local session).
- **Model:** claude-sonnet-5 (match the other routines).
- **MCP connections:** Asana, Salesforce (read-only), Atlassian (Jira), Slack, Google Drive.

## What it does
Follows `.claude/skills/mops-command-center/SKILL.md`: pulls live Asana/Salesforce/Jira data, computes the `briefing` object, then:

1. **Morning brief (`full`)** — posts the 5-bullet narrative + headline tiles to Slack (target TBD — DM to Jennifer Schroback, or a manager channel; confirm before first send), and re-publishes the dashboard.
2. **Dashboard re-publish** — fills `dashboard-template.html`'s `__BRIEFING_JSON__` token with the briefing object, writes the file, and publishes to a **stable Artifact path** so the URL never changes.
3. **Alerts (`alerts-only`)** — posts a short Slack alert for each new `warn`/`critical` threshold breach; suppresses anything already alerted (dedupe on ticket/alert identity vs. last-run state). Keep it low-noise: `info`-severity items ride the morning brief, they don't interrupt the day.

## Before this can run (prerequisites)
- **Salesforce stays read-only** — the brief/dashboard report on campaigns and email; they never create or edit SF records (`tools/available-tools.md`). No prerequisite, just a standing constraint.
- **Confirm the Slack delivery target** (DM vs. channel) and get standing authorization for the brief's own send. Concretely, per `.claude/rules/write-actions.md` §6: the only standing unattended authorization today is the weekly review post to `#mops-team`. **Both the morning brief and the daytime alerts need their own §6 rows** — approved by Jennifer, logged in `decisions/log.md`, and added to the contract — before either can send unattended. Until then, sends need per-instance confirmation. Note Slack sends are Class C (no unsend tool exists), so alert-target scoping is the highest-blast-radius decision here.
- **Ship the write contract into the routine repo** — cloud routines clone skill folders and do **not** inherit `.claude/rules/`, so the repo must carry a copy of `.claude/rules/write-actions.md`. The alert dedupe requirement is the contract's §4 idempotency rule, not an optional nicety: without a run-key check, a re-fired alert trigger re-sends every open breach.
- **Dashboard publish path from a routine** — confirm the routine environment can publish an Artifact (and to a stable, updatable URL). If Artifact publishing isn't available in the cloud env, fall back to posting the briefing as a Slack canvas and revisit. This is the main open technical dependency.
- **Live-dashboard (mcp) upgrade is separate** — the self-refreshing `mcp` version of the dashboard is built interactively after observing real connector tool shapes (see the skill + plan). The routine only ever publishes the **snapshot** form.

## Keeping in sync
Standalone push target, not a live mirror — if `mops-command-center/SKILL.md`, `dashboard-template.html`, or the `context/` dependencies change here, re-push or the routine runs against a stale version.

## Created
Blueprint added 2026-07-20, per Forkan's request to build a manager-facing agentic OS (live dashboard + daily brief + alerts) over the existing MOps skills.
