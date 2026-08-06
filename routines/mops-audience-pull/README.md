# MOPS Audience Pull — Daily (new submissions only)

Scheduled agent that runs the `audience-pull` skill every morning, reads **the audience-shaped tickets that arrived on the `[MOPs] Intake` board since the last run**, and either builds the list or says exactly what's blocking it.

> **Status: LIVE.** Loaded as a launchd agent running **07:00 America/Los_Angeles, every day**. Scope confirmed as new submissions only.

- **Host:** **local launchd agent** on Forkan's Mac — `com.acquia.mops.audience-pull`, plist in this folder.
  - Local is *required*: the deliverable is a file in `outputs/audiences/`, and a cloud routine's filesystem doesn't persist to this repo. It also needs the Salesforce MCP connection authenticated in Forkan's logged-in session.
- **Schedule:** **07:00 local, all seven days.** launchd uses system local time and this Mac is America/Los_Angeles, so it follows DST by itself. **If the machine's timezone changes, this time moves with it** — that's the one thing to re-check. 07:00 also keeps it clear of the 06:00 slot that `mops-intake-dispatch` occupies on weekdays.
- **Skill:** `.claude/skills/audience-pull/` → the **Routine mode — daily sweep** section. Same skill as the attended path, one behavioural change: an unattended run may never invent criteria.
- **Delivery:** `outputs/audiences/audience-sweep-YYYY-MM-DD.md`, one digest per run, written even when empty. Plus, for any ticket that's actually runnable, the usual `audiences/audience-<slug>-<date>.{csv,md}` pair.
- **Log:** `~/Library/Logs/audience-pull.log`
- **MCP connections:** Asana (read) + Salesforce (read). No Jira, Slack, or Sheets.
- **Writes:** **none to any live system.** Local repo files only.

## ⚠️ Two things to know before trusting this

### 1. It will be quiet, and that is the honest answer

Measured on the live board 2026-08-04:

| | |
|---|---|
| Open tickets with `Project Type = Audiences` | **1** (Atlanta Tech Week 2026, assigned to Felipe) |
| Audience-report tickets that are `[Event Name]` placeholders | **12** |

Genuine Audiences tickets arrive at roughly **one a month, not one a day**. Most days this routine will write an empty digest. That's expected — the empty file is the proof it ran, and the value is catching the real one *the morning it lands* rather than whenever someone notices.

**If you want a busy daily routine, this isn't the type to hang it on.** `intake-dispatch` already covers the whole board.

### 2. Morning slot ordering

`com.acquia.mops.lead-routing-audit` runs **05:30**, `com.acquia.mops.intake-dispatch` runs **06:00** weekdays, and this one runs **07:00**. Each gets its own hour, so no two `claude -p` processes contend for the Asana/Salesforce MCP connections.

Keep it that way when adding routines — schedule collisions are the failure mode here, not load.

## Scope — how "new submissions only" is enforced

`state.json` in this folder holds the watermark:

```json
{ "floor_date": "2026-08-04", "last_run_utc": null }
```

Each run sweeps `created_at_after = max(floor_date, last_run_utc − 1h)`.

- **`floor_date` is the backlog guard.** On the first run `last_run_utc` is null; without a floor the query would sweep the whole board history. **Never lower it** to pick up old tickets — that's a separate, explicit request.
- **The 1-hour overlap** absorbs Asana index lag so nothing falls between two windows.
- **The watermark advances to the run's START time, and only on success.** A failed run leaves it alone so the next run re-covers the window. Re-reporting is harmless; skipping isn't.
- A closed laptop widens the next window rather than losing a day. Missing days self-heal.

`state.json` changes every run — consider gitignoring it. The durable record is the digest series in `outputs/audiences/`.

## Detection — why `Project Type` isn't enough

`Project Type = Audiences` is set on **1** open ticket, while 12 audience-shaped tickets carry `Project Type = null`. Keying on the picklist alone would find nothing.

A new ticket matches if **either**:
- `Project Type` (field gid `1206591746930193`) = `Audiences`, **or**
- the name contains `Audience Report` · `Audience Pull` · `Target List` · `Contact List` · `Audience Request`

The digest reports which test matched. **A name-matched ticket with a null `Project Type` is itself a finding** — that's the classification gap showing up, not just an audience request.

## Per-ticket outcome — only one of four produces a CSV

| Outcome | When | What gets written |
|---|---|---|
| `READY` | Every filter dimension is explicit in the description or comments | Full pull: CSV + `.md` spec |
| `NEEDS_INPUT` | Any dimension missing or ambiguous | **Spec stub only, no CSV** — the exact questions, plus the SOQL that would run once answered |
| `BLOCKED` | `[Event Name]` placeholder, or no resolvable parent event | Nothing but the named blocker |
| `TOO_LARGE` | `COUNT(Id)` > 2,000 | Count + SOQL + funnel + handoff. **Never a truncated CSV** |

**`NEEDS_INPUT` is the expected outcome, not a failure.** An audience is a mailing list. A CSV built from a guessed reading of "our EMEA audience" is worse than no CSV, because it looks finished. The skill's standing rule — *never invent criteria* — is what makes unattended operation safe here, and it is the reason this routine can run without a `write-actions.md` §6 row.

Suppression and row screening are **not** relaxed for being unattended. They're the mandatory part (`audience-pull` Step 3), and the validated finding stands: **20% of rows carry defects every opt-out filter passes** — GDPR-erased records, job-changers, `.invalid` emails.

## What it does, per run

1. Reads `state.json`, computes the window.
2. Fetches new `[MOPs] Intake` tickets (project gid `1205660951274722`) since the watermark.
3. Matches audience-shaped tickets by the two-test rule above.
4. For each match: reads description **and** comments — audience criteria usually live in a comment or a linked doc.
5. Classifies into the four outcomes.
6. Runs the pull only for `READY` tickets, applying mandatory suppression + row screening.
7. Writes the digest. Advances the watermark only if the run succeeded.

## The `[Event Name]` bug is blocking here, not cosmetic

`intake-dispatch` reports the template automation's `[Event Name]` substitution bug as a count. For an audience pull it's **fatal to the ticket**: the request is literally `[Event Name] Pre-Event Email Audience Report - MOPs`, so there is no way to know which event's attendees to pull. The routine reports these as `BLOCKED` and counts them in the digest header.

12 of 13 audience-report tickets in the sampled window were these. **Fixing the template is worth more than this routine** — it would convert most `BLOCKED` rows into real work.

## ⚠️ Personal data accumulates unattended

Every `READY` pull writes a CSV of names, work emails, employers and job titles into `outputs/audiences/`, which sits in a git repo. Attended, a human decides each time. **Unattended, this accumulates on its own.**

Recommended: gitignore `outputs/**/*.csv`, and review the folder periodically. Do not commit these without a decision, and do not share them outside Acquia.

## Managing the schedule

```
launchctl print    gui/$(id -u)/com.acquia.mops.audience-pull   # status, run count, last exit
launchctl kickstart -p gui/$(id -u)/com.acquia.mops.audience-pull  # run now
launchctl bootout  gui/$(id -u)/com.acquia.mops.audience-pull   # turn it off
tail -f ~/Library/Logs/audience-pull.log
```

Re-copy the plist to `~/Library/LaunchAgents/` and `bootout` + `bootstrap` after editing it — launchd reads the installed copy, not the one in this repo.

## Still open

1. **Report the `[Event Name]` substitution bug** to whoever owns the template automation. It's the single change that would make this routine productive.
2. **No SLA exists for `Audiences`** (`audience-pull` known gap 1). The digest can't assert a due date, only a submission age.
3. **Confirm which of the three product-interest fields is authoritative** with Felipe (`Product_Cloud_Interest__c` vs `Product_Interest__c` vs `Product_Interest_Multi__c`). Affects every `READY` pull.
4. **Decide the gitignore question** on `outputs/**/*.csv` before the first few unattended pulls land.

## Not in scope

- Any write to Asana, Salesforce, Jira, Slack, or Sheets — including assigning the ticket to Felipe or commenting the result on it. That would need a §6 row.
- Creating a Salesforce Report, adding Campaign Members, or building a Pardot list. All writes; no path exists (`write-actions.md` §9). Permanently out of scope.
- The existing backlog, excluded by the floor date.

## Created

2026-08-04, from the request to run `audience-pull` daily scoped to new submissions only. Set to 07:00 PT.
