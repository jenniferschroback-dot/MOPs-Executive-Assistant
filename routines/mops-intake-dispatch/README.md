# MOPS Intake Dispatch — Daily (propose-only, new arrivals only)

Scheduled agent that runs the `intake-dispatch` skill each weekday morning, reads **what arrived on the `[MOPs] Intake` board since the last run**, and writes a digest to `outputs/intake/` proposing an **owner** and a **sub-task set** for each new ticket.

> **Status: BUILT, NOT LOADED.** Skill, plist, and state file are in place and validated against the live board. The launchd agent is deliberately unloaded pending confirmation of the run time. One command enables it — see _Enabling_.

- **Stage:** **1 of 2 — propose only.** Writes nothing to any live system. Stage 2 (executing approved writes unattended) is a separate decision needing a `write-actions.md` §6 row, and shouldn't be considered until weeks of Stage 1 digests are trusted.
- **Scope:** **new arrivals only.** The ~158 open backlog tickets are deliberately excluded, enforced by a watermark + floor date rather than by judgment.
- **Host:** **local launchd agent** on Forkan's Mac — `com.acquia.mops.intake-dispatch`, plist in this folder.
  - Local is *required* by the delivery choice: the digest is a file in `outputs/intake/`, and a cloud routine's filesystem doesn't persist to this repo.
- **Schedule:** **07:00 local, Monday–Friday** (proposed — confirm before enabling). Weekdays only, unlike `lead-routing-audit`: intake submission is a business-hours activity and a weekend digest would almost always be empty. Monday's run covers the weekend automatically because the window is watermark-driven, not "yesterday"-driven.
- **Delivery:** `outputs/intake/intake-dispatch-YYYY-MM-DD.md`, one file per run, written even when empty.
- **Log:** `~/Library/Logs/intake-dispatch.log`
- **MCP connections:** **Asana only, read-only.** No Salesforce, Jira, Slack, or Sheets.
- **Writes:** **none.** Not to Asana, not anywhere. That's the point of Stage 1.

## Scope — how "new only" is enforced

`state.json` in this folder holds the watermark:

```json
{ "floor_date": "2026-08-04", "last_run_utc": null }
```

Each run fetches `created_at_after = max(floor_date, last_run_utc − 1h)`.

- **`floor_date` is the backlog guard.** On the first run `last_run_utc` is null; without a floor the query would sweep the entire board history. **Never lower it** to pick up old tickets — that's a separate, explicit request.
- **The 1-hour overlap** absorbs Asana index lag so nothing falls between two run windows.
- **The watermark advances to the run's START time, and only on success.** A failed run leaves it alone so the next run re-covers the window. Re-reporting is harmless; skipping isn't.
- A closed laptop widens the next window rather than losing a day. Missing days self-heal.

`state.json` changes every run — consider gitignoring it. The durable record is the digest series in `outputs/intake/`.

## The n8n boundary

An **n8n workflow already runs in production**, polling Asana every 15 minutes to detect new tickets, decide the campaign gate, and generate the campaign name. It has therefore already acted on every ticket in this routine's daily window.

**n8n covers:** `SFDC Campaign only (single)` · `UTM(s) (+ SFDC Campaign)` (dead — option disabled in Asana) · `Webinar Request` and `Event (+ SFDC Campaign)` **only when the submission explicitly states no companion promotional email.** Everything else, including `SFDC Campaigns only (multiple)`, is uncovered.

**Split:** n8n owns detect → gate → name. This routine owns **assign → prioritize → expand sub-tasks** — none of which n8n touches. The cadence gap is a feature, not a conflict: n8n is the real-time actor, this is the daily human-review layer. Nobody wants 96 digests a day.

The routine must **never propose a campaign name** and must **never re-derive the campaign gate** for an n8n-covered type. It should, however, report a **divergence** — a ticket n8n should have named that still has no name after a full day means n8n failed silently, which is otherwise invisible.

## What it does, per run

1. Two fetches against `[MOPs] Intake` (gid `1205660951274722`) at the watermark — top-level tickets, then sub-tasks.
2. Drops noise: blank orphans, `[Event Name]`-style placeholders, and our own `TESTING SKILL` / `[TEST RUN` rows — counted separately, never pooled.
3. Splits new sub-tasks into **template-generated** (excluded) vs **standalone requests** (secondary section).
4. Buckets survivors by `Project Type`.
5. Proposes an owner via the **Region-free** subset of `intake-routing`'s rules (**0**, 1, 3, 4, 6), using `created_by` in place of the absent `Requestor` field. Rule 0 sends the three no-pattern types (`IT/Integration`, `Automations | Martech`, `UAT`) to **Jennifer** with no sub-tasks.
6. Proposes a Priority from the SLA clock + target date, using the now-confirmed 5-value picklist.
7. Proposes the sub-task set from the catalog's evidence tier — **including proposing nothing** for the seven leaf types.
8. Runs the workload check; headlines any owner taking >50% of the run.
9. Writes the digest. Writes nothing else.

## What the live board actually looks like (measured 2026-08-03)

### Volume — expect quiet days

| | |
|---|---|
| New **top-level** tickets, last 7 days | **6** (~1/day) |
| New **sub-tasks**, same 7 days | **36** (~6:1) |

Many days will produce **zero** new top-level submissions. The digest is written anyway — a missing file is ambiguous, an empty one isn't.

⚠️ **6/week is below the 10–20/week in `context/work.md`.** Either intake volume has genuinely dropped, or a meaningful share arrives through the **Jira channel** (`mops-command-center` found ~289 real open ops tickets there, a second front door), or as sub-tasks under existing parents. Worth resolving — it determines whether this routine sees most of the team's inflow or a fraction of it.

### Triage gaps (from the 50 most recent open tickets, i.e. including backlog)

| Finding | Number | Consequence |
|---|---|---|
| Unassigned | **29 / 50 (58%)** | The routing gap is real — this is the value case |
| Null `Priority` | **47 / 50 (94%)** | Confirms `intake-routing`'s documented gap, quantified |
| Null `Project Type` | **15 / 50 (30%)** | But only **5** genuine — rest are orphans + template rows |
| Routable without Region | **32 / 50 (64%)** | Enough to be useful day one |
| → of those, to Aayushi | **24 / 32 (75%)** | ⚠️ See below |
| `Requesting Team` null | **49 / 50** | Not a usable Region proxy |
| `Requestor` null | **50 / 50** | Rule 4 needs `created_by` instead |
| `created_by` populated | **38 / 50** | Names map onto the stakeholder lists directly |

### Two findings that will show up in the first digest

**1. Unclassified is a new-submission problem, not backlog rot.** Of the 5 genuinely unclassified tickets, **4 were created in the previous 4 days**. A new-arrivals-only digest hits this immediately — it's likely the most common finding, not an edge case.

**2. The template automation has a live name-substitution bug.** Four of the ten sub-tasks that automation created on 2026-08-03 carry the literal string `[Event Name]` in their names. These aren't stale leftovers being cleaned up — they're being generated wrong right now. The digest reports the count every run.

### The one thing to watch

Rule 4 (named stakeholder — the routing table's *strongest* signal) sends **75% of routable tickets to Aayushi**, because her six named stakeholders file most of the board. The skill surfaces this as a headline rather than silently rebalancing, per `intake-routing`'s explicit instruction that the ownership model is not a suggestion.

So the first honest output of this routine may be *"the ownership table concentrates work on one person"* — a management finding for Jennifer, not something the routine should fix on its own.

## Enabling

Confirm the run time first (07:00 PT weekdays as written), then:

```
cp routines/mops-intake-dispatch/com.acquia.mops.intake-dispatch.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.acquia.mops.intake-dispatch.plist
launchctl kickstart -p gui/$(id -u)/com.acquia.mops.intake-dispatch   # smoke-test it now
```

Management:

```
launchctl print   gui/$(id -u)/com.acquia.mops.intake-dispatch   # status, run count, last exit
launchctl bootout gui/$(id -u)/com.acquia.mops.intake-dispatch   # turn it off
tail -f ~/Library/Logs/intake-dispatch.log
```

Re-copy the plist and `bootout` + `bootstrap` after editing it — launchd reads the installed copy, not the one in this repo.

## Still open

1. **Confirm the run time.** 07:00 PT weekdays is a proposal, not a decision.
2. **Resolve the volume discrepancy** — 6 new top-level tickets/week vs 10–20 expected. If most intake arrives via Jira, this routine is watching the smaller door.
3. **Confirm what auto-expands the event/workshop 5-set.** Asana project template? A rule? A Zap? This **partially answers the catalog's biggest open question**, and the answer may delete the sub-task-proposal branch for the whole Event family as redundant.
4. **Confirm the Project Type → Project Group bridge** with Harish/Aayushi. `intake-routing` flags its own bridge as unconfirmed; rule 6's type list is inference.
5. **Decide whether Region gets captured at intake.** 24% of tickets are unroutable without it, and no skill logic fixes an uncollected field. Highest-leverage fix available, and it's an Asana form change.
6. **Sanity-check one run by hand** against what Aayushi would actually have done — Priority especially, since 94% null means there's no baseline except human judgment.
7. **Report the `[Event Name]` substitution bug** to whoever owns the template automation.

## Not in scope

- Any write to Asana, Salesforce, Jira, Slack, or Sheets.
- The existing backlog. Excluded by design, per the floor date.
- Salesforce campaign creation — no write path exists (`write-actions.md` §9). Permanently out of scope, not just Stage 1.
- Pardot anything — no connector.
- Classifying the null-`Project Type` tickets. The digest *reports* them; `intake-classification` handles them attended.

## Relationship to `mops-intake-pipeline`

That routine (gate-check → classification → naming → sf-campaign-spec → tracking) is **disabled** pending a live intake form and Salesforce write access. This one deliberately avoids both dependencies: Asana only, read only. If `mops-intake-pipeline` ever goes live, the two overlap on classification and should be merged rather than run side by side.

## Created

2026-08-03, from the request for a daily routine that buckets intake by Project Type and proposes owners plus sub-task lists — scoped to propose-only, local + `outputs/intake/` delivery, and new arrivals only.
