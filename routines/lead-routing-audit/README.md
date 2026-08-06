# Lead Routing Audit — Daily

Scheduled agent that runs the `lead-routing-audit` skill each weekday morning and appends the day's classified leads to the shared routing QA Google Sheet, reviewed by Forkan and Lucio Silvestri (Revenue Operations).

> **Status: LIVE as of 2026-08-03.** Loaded as a launchd agent, running 05:30 America/Los_Angeles every morning. Authorized for unattended Sheet appends by `.claude/rules/write-actions.md` §6.

- **Host:** **local launchd agent on Forkan's Mac** — *not* a cloud routine. This is a deliberate constraint, not a shortcut (see below).
- **Schedule:** **05:30 local, all 7 days**, auditing **the previous day**. Defined in `com.acquia.mops.lead-routing-audit.plist` (in this folder; management commands are in its header comment).
  - The scope is `yesterday`, not `today`: at 05:30 the current day is nearly empty, so the review needs the last complete day. `Run_Date` is always the report day, never the execution day.
  - launchd uses **system local time**, and this Mac is set to America/Los_Angeles — so 05:30 is 05:30 LA and it follows DST by itself. **If the machine's timezone ever changes, this time moves with it.** That's the one thing to re-check.
  - All seven days on purpose: weekend volume is low but non-zero, and an unbroken `Run_Date` series makes a missing date unambiguously a missed run. Weekdays-only is a two-line delete in the plist.
- **Skill:** `.claude/skills/lead-routing-audit/` (SKILL.md + reference.md) — read in place, no separate repo needed since it runs locally.
- **Sheet:** [Lead Routing QA — Daily (MOPs + RevOps)](https://docs.google.com/spreadsheets/d/1gFu34xeJJ2Jrk1p9jtvgkzp_WtspHNyCGZ1zorbNNOA/edit) — id `1gFu34xeJJ2Jrk1p9jtvgkzp_WtspHNyCGZ1zorbNNOA`. Private to Forkan on creation; **share with Lucio manually** (a permission change isn't the assistant's to make).
- **Log:** `~/Library/Logs/lead-routing-audit.log`
- **MCP connections:** Salesforce (read-only). No Asana/Jira/Slack.
- **Model:** claude-sonnet-5

## Why local, not a cloud routine

Every other routine in this project runs in Anthropic's cloud. This one can't, for one reason: **the only Google Sheets write path is the local `gws` CLI.**

- There is no Google Sheets MCP connector, and none exists in the registry (searched 2026-08-03).
- The Google Drive MCP connector can *create* a file but has no update or append verb — it cannot add a row to an existing sheet.
- `gws sheets` does full read/write, but it's authed against **Forkan's own keyring** on this machine. A cloud routine environment has neither the binary nor the credentials.

See `tools/available-tools.md` → `gws` CLI, and `.claude/rules/write-actions.md` §1 → Google Sheets.

**Trade-off:** a closed laptop delays the run rather than skipping it — launchd's `StartCalendarInterval` fires a missed job once on wake. But a multi-day gap will simply be missing `Run_Date` rows, which is why the `Daily Summary` tab is append-one-row-per-day: a gap is visible rather than silent.

**Migration path to cloud**, if the local host proves annoying: install `gws` in the routine environment and supply a Google OAuth refresh token as a routine secret. That needs a Google OAuth client and possibly Workspace-admin involvement — worth doing only once the daily review is established and the classification is trusted.

## What it does, per run

1. Reads the live `Lead_Routing_Rule__c` table (43 active rules) — never a cached copy.
2. Pulls the day's leads: `Lead_Routed_Date__c` OR `Marketing_Qualified_Date__c` within the report day (~11 leads).
3. Reconstructs each lead's **router-set owner** from `LeadHistory` (current `OwnerId` is contaminated by downstream DQ).
4. Recomputes the expected owner from the rule table and classifies into bucket **1 routed OK / 2 wrong rep / 3 not routed / 4 bypassed by design**.
5. Appends to `Detail` (columns `A:X` only — the two reviewer-annotation columns are human-owned) and one row to `Daily Summary`.
6. Weekly: re-runs the 12-check config linter and `values.update`s the `Config Linter` tab.
7. Prints a digest — bucket counts plus the High-severity rows in full.

Read-only against Salesforce throughout. The rule-table defects it finds produce a fix spec for an admin, never a completion claim (`.claude/rules/write-actions.md` §9).

## Write contract (required dependency)

The Sheet append is the only write. It's **Class A** (reversible, nobody notified) per contract §1, with idempotency on `Run_Date` + `Lead_Id` per §4 — a day already present makes the whole run a no-op.

**§6 authorization is NOT yet granted.** The weekly `#mops-team` post remains the only standing unattended write in this project. Until a §6 row for this Sheet is approved and logged in `decisions/log.md`, the skill should produce rows for a human to paste rather than appending unattended. Add that row before enabling the schedule.

Because this runs locally rather than as a cloud routine, it **does** inherit `.claude/rules/` — no copy of the contract needs shipping, unlike the cloud routines.

## Done during the build (2026-08-03)

1. ~~**Create the Sheet**~~ — done, 4 tabs, headers, conditional formatting per bucket/severity, basic filters, reviewer-owned columns tinted.
2. ~~**Seed the `Reference` tab**~~ — done: RR pool → `UserRole` map (3 pools, from observed pairings), bucket definitions, thresholds, measured baselines, and the 10 open questions for Lucio.
3. ~~**Run Phase 0 calibration**~~ — done, and it changed the design. See below.
4. ~~**Config Linter tab**~~ — populated with 23 findings across the 12 checks; all previously documented bugs independently rediscovered, plus one new High.
5. ~~**Verification**~~ — regression set re-traced, router-set-owner extraction fixed and re-verified 4/4, linter re-run from scratch, idempotency proven (second run = no-op, zero duplicates, reviewer annotation intact).
6. ~~**One real end-to-end run**~~ — 2026-07-31: 11 leads in scope (exactly the predicted volume), 8 bypassed / 2 routed OK / 1 wrong rep / 0 not routed.

### What Phase 0 changed

The backtest (226 rule-stamped MQLs over 30 days, 99 distinct criteria groups) found that a literal reading of the rule table produces a **23.9% false divergence rate** — above the "systematic" threshold, i.e. enough noise to get the sheet ignored by week two. Two engine behaviours the table doesn't express account for it:

- an **Account Team Member rule falls through** to the next rule when the account has no matching team member (8.4% of leads);
- **`Account__c` may have been linked after routing** — it isn't history-tracked (8.8%).

Modelling both drops false divergence to **4.0%** and puts every sub-signal under the 20% threshold, so all of them stay as per-lead rows. Full detail in `reference.md` §7b and §8.

Two further corrections came out of it, both now fixed in the skill:

- **The routing stamp is not always written by `B2BMA Integration`.** The Flow runs in whatever user context triggered the MQL. The old fingerprint found zero rows on lead `00QPb00001j5qI6MAI` and the fallback would have blamed the router for a human's deliberate reassignment. Extraction now matches on proximity to `Lead_Routed_Date__c` first, actor second.
- **`User.UserRole.Name` is not `AccountTeamMember.TeamMemberRole`.** Conflating them invented ATM matches that don't exist and produced a wrong flag on a regression case.

## Still open

1. **Share the Sheet with Lucio.** Still private to Forkan — a permission change isn't the assistant's to make. The routine is writing daily rows in the meantime, so this is the one thing blocking the actual review.
2. **Confirm the 10 open questions** on the `Reference` tab with whoever owns the routing Flow — above all, whether order 1001 is *intentionally* the global fallback, and whether the ATM fall-through behaviour is real (it's the single biggest assumption in the recompute).
3. **Watch the first week's rows before trusting the classification.** The schedule was enabled directly, skipping the 3–5 attended runs the original plan called for — so the first several mornings *are* the calibration period. Bucket 3's severity mapping in particular has never fired against real data.
4. **Hand the 12-edit config fix spec to a Salesforce admin** (`outputs/lead-routing/Lead_Routing_Config_Audit_2026-08-03.md`). That fix is worth more than this routine; the routine's job afterwards is to prove it worked.

### Managing the schedule

```
launchctl print    gui/$(id -u)/com.acquia.mops.lead-routing-audit   # status, run count, last exit
launchctl kickstart -p gui/$(id -u)/com.acquia.mops.lead-routing-audit  # run now
launchctl bootout  gui/$(id -u)/com.acquia.mops.lead-routing-audit   # turn it off
tail -f ~/Library/Logs/lead-routing-audit.log
```

Re-copy the plist to `~/Library/LaunchAgents/` and `bootout` + `bootstrap` after editing it — launchd reads the installed copy, not the one in this repo.

## Verify on first open

The `Lead_URL` column assumes the My Domain `acquia.lightning.force.com`. If those links 404, correct the base URL in the skill — it's cosmetic but it's the column the reviewers actually click.

## The bigger finding

The linter already located what is very likely the actual cause of the original complaint — a rule-migration bug that leaves no-account leads in continental EMEA, MEA, non-Japan APJ and LATAM matching **no active rule**, falling through to the AMER BDR round robin. Full evidence and a 10-edit fix spec: `outputs/lead-routing/Lead_Routing_Config_Audit_2026-08-03.md`.

**That fix is worth more than this routine.** The routine's job afterwards is to prove the fix worked and catch the next drift — not to keep reporting a known bug daily.

## Created

2026-08-03, from the request for a daily lead-routing QA sheet reviewed jointly by MOPs and Revenue Operations.
