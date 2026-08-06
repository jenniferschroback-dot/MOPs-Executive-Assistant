# MOps Intake Pipeline

Scheduled cloud agent (Claude Code routine) that chains `campaign-gate-check` → `intake-classification` → `campaign-naming` → `sf-campaign-spec`, plus `intake-tracking` for a status snapshot, into one run.

- **Routine ID:** `trig_011hNPAvXmW4jJsZPzSYddgb`
- **Link:** https://claude.ai/code/routines/trig_011hNPAvXmW4jJsZPzSYddgb
- **Schedule:** `0 15 * * 1-5` UTC (8:00 AM Pacific, weekdays) — **currently disabled.** Run manually ("run now") until the live intake form lands and the "new ticket" proxy (see below) is trustworthy; then flip `enabled: true`.
- **Repo:** [github.com/forkanelebdi-ACQ/mops-intake-pipeline](https://github.com/forkanelebdi-ACQ/mops-intake-pipeline) (`main`) — dedicated repo created 2026-07-15 so the cloud agent has something to clone; mirrors this project.
- **Environment:** "Default" (`env_012geB3Ka1PNQ2CQxHgTrW8p`) — no dedicated environment exists for this routine; no tool is available to create one.
- **Model:** claude-sonnet-5
- **MCP connections:** Asana, Salesforce (read-only), Atlassian (Jira), Slack

## Why manual-only for now
Two hard constraints, both surfaced during setup (2026-07-15):
- **No live intake form yet.** `projects/intake-pipeline-automation/README.md` and `decisions/log.md` document a deliberate sequencing call: stabilize the intake form first, then automate on top of it. There's no single landing point to poll for genuinely "new" submissions today.
- **Salesforce MCP is read/query-only.** `tools/available-tools.md` confirms there's no create/update tool — this routine can only *propose* the Salesforce Campaign spec, never create it.

## What it does, per run
1. Reads `.claude/skills/campaign-gate-check/SKILL.md`, `intake-classification/SKILL.md`, `campaign-naming/SKILL.md`, `sf-campaign-spec/SKILL.md`, `intake-tracking/SKILL.md` (plus `CLAUDE.md`/`context/*.md`/`decisions/log.md`) as the source of truth — the routine prompt doesn't reimplement their logic.
2. Queries Asana `[MOps] Intake` for tasks with the `Project Type` custom field (gid `1206591746930193`) unset — the proxy for "new."
3. For each new ticket: classifies it, then runs `campaign-gate-check`:
   - **`no`** — sets `Project Type` only (so it isn't re-picked-up), no sub-tasks, no ticket comment. Noted in the Slack summary only.
   - **`needs-human-input`** — doesn't touch the ticket; flagged in the Slack summary for a human call.
   - **`yes`** — creates sub-tasks (`intake-classification`), proposes a name (`campaign-naming`), then proposes the full Salesforce Campaign spec **as a comment on that Asana ticket** (`sf-campaign-spec`) — since there's no SF write path, a human executes it manually.
4. Runs `intake-tracking`'s status logic across Asana/Jira/Salesforce and posts one consolidated summary to **#mops-team** in Slack: tickets processed this run + overall pipeline health.

## Write contract (required dependency)
This routine creates Asana tasks/sub-tasks and posts to Slack, so its repo **must ship a copy of `.claude/rules/write-actions.md`** — cloud routines clone skill folders and do **not** inherit `.claude/rules/`. Without it the routine runs unguarded.

Two contract items to settle **before** this is re-enabled:
- The Slack summary to `#mops-team` is **not** covered by the contract's standing authorization (§6), which is scoped to the weekly review post only. This routine needs its own §6 row, approved and logged in `decisions/log.md`, or it can't send unattended.
- Unattended task creation must run the idempotency check (§4) — match existing sub-tasks by name before creating — or a re-fired run duplicates the whole set. This is the most likely failure mode for a scheduled pipeline.

## Keeping in sync
Same caveat as the weekly-review routine: this repo is a standalone push target, not a live mirror. Re-push after any change to the five skills above, `.claude/rules/write-actions.md`, or their supporting context/decision files, or the routine runs against a stale version.

## Created
2026-07-15, per Forkan's request to chain the four intake-to-launch skills into one pipeline, with a dedicated `campaign-gate-check` skill split out per his request so the Salesforce-Campaign gate has one home instead of being duplicated across skills.
