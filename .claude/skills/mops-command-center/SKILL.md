---
name: mops-command-center
description: Builds the MOps manager briefing — one structured snapshot of new/at-risk/overdue tickets, on-time SLA %, workload by person, Salesforce email-campaign performance, send-calendar look-ahead, and threshold alerts — pulled live from Asana/Salesforce/Jira. This is the shared brain behind the manager dashboard and the daily brief + alerts routine. Use when asked for the manager's command-center view, to (re)generate the dashboard data, or to run the daily/alert check.
argument-hint: [optional mode, e.g. "full" (default) | "alerts-only"]
---

# MOps Command Center — Manager Briefing

The single source of truth for the MOps manager's operating picture. Runs read-only across Asana, Salesforce, and Jira, computes the manager KPIs **once**, and returns a structured `briefing` object plus a short narrative. The live dashboard and the daily-brief/alerts routine both consume this exact object — compute the numbers here, never re-derive them downstream.

**Who this serves:** Jennifer Schroback, Sr. Manager of Agentic Marketing Operations (manager + senior-agentic-marketing role are the same person). She cares about: new and at-risk tickets, on-time SLA performance, workload balance across the team, the chronically-null Priority field, and email-campaign performance.

**Non-negotiables:**
- Never fabricate a number — every figure traces to a tool query. If a connector fails or a field is empty, put the gap in the briefing (`dataGaps[]`), don't guess.
- **Read-only.** This skill only reads. It creates/edits nothing. Salesforce MCP is query-only anyway (`tools/available-tools.md`); Asana/Slack/Jira actions belong to the separate action layer, invoked conversationally with confirmation — not here.
  - The **dashboard's** two write actions (Asana Assign, Jira Close) are part of that action layer and are governed by `.claude/rules/write-actions.md` — Assign is Class B (notifies the assignee), Close is Class C (this Jira workflow is forward-only and Done is terminal, so it's one-way). Both must confirm before firing, never auto-retry, and be logged in `decisions/actions.md`. The dashboard is where that contract was first implemented; the contract is now the source of truth, not the button code.
- **No local-vault dependency.** This skill must run unattended in a cloud routine that cannot see the local Obsidian vault (`~/Desktop/MOps vault`). Every constant it needs (SLA table, team roster, thresholds) is inlined below or in this skill folder — never read the vault at runtime.

## Modes

- `full` (default) — compute the entire briefing.
- `alerts-only` — compute just the threshold checks (Step 4) for the daytime alert cadence; skip the heavier email-performance pull (Step 3.C) unless an email threshold needs it. Faster, low-noise.

## Step 1 — Resolve windows

Compute from the run date (today):
- **now** — the run timestamp (record it; the dashboard shows freshness from it).
- **retro** — trailing 7 days, for throughput/on-time.
- **email window** — trailing 30 days, for email performance (and the prior 30 for deltas).
- **look-ahead** — next ~10 days, for the send calendar.

State exact dates in the output so the analyzed window is never ambiguous.

## Step 2 — Pull intake + workload (Asana, and Jira where relevant)

Source of record: the Asana **`[MOps] Intake`** project.

- **New tickets** — tasks whose `Project Type` custom field (gid `1206591746930193`) is unset. This is the current best proxy for "newly submitted, not yet classified" until the live intake form exists — label it as a proxy in the output, not as ground truth.
- **Open / in-progress** — incomplete tasks with a `Project Type` set; read assignee, due date, `MOPS- Status` (Triage → Assigned → In Progress → Waiting for Feedback → Blocked → Deprioritized → Incomplete → Completed), and Priority.
- **Completed (retro)** — tasks completed in the trailing 7 days, with assignee and completion date (for throughput + on-time).
- **Workload by person** — count of open/incomplete tasks per assignee. Canonical names: **Aayushi Sharma (lead) · Harish Pandey · Jennifer Schroback · Felipe Tencio** (+ Forkane Lebdi). Include zero counts; sort descending.
- **Null-priority count** — open tasks with Priority unset (a known, recurring data-quality problem the manager tracks).
- **Blocked** — tasks in `Blocked`/`Waiting for Feedback`; capture the specific missing info from the task notes, don't guess.

Reuse the exact grouping logic from the `intake-tracking` skill (New/unclassified · In progress · Stuck/overdue · Launched) rather than re-deriving it. For the email send calendar, read `Email`-type sub-tasks converted to **Milestones** (see `email-send-calendar`), querying **both** completion states explicitly — the default calendar view hides completed ones.

## Step 3 — Pull the KPIs

### A. On-time SLA % (real turnaround, not the flag)
Do **not** use the Asana `Out of SLA/Rush` flag — it's populated on almost no tickets (a dry run showed 34/35 blank), so it reads a false ~97%. Compute on-time from the actual dates instead.

**Method.** For each ticket **completed** in the retro window (last 30 days), compute the **business-day turnaround** from `created_at` (submission proxy) to `completed_at`, and mark it on-time if `turnaround ≤ SLA(Project Type)`. Report `onTimePct` (share within SLA), `avgCycleDays` (mean business-day turnaround), and a per-type breakdown. Business days = weekdays only (holidays ignored — note as a caveat).

**Turnaround SLA table** — from the official *MOps Requests: SLA Timeline Requirements (2025)* doc (`references/sops/mops-sla-timeline.md`), keyed to the live Asana `Project Type` enum. Business days from submission:

| Project Type | SLA (business days) | Source line |
|---|---|---|
| `SFDC Campaign only (single)` | 1 | SFDC Campaign up to 5 = 1 bd |
| `SFDC Campaigns only (multiple)` | 2 | SFDC Campaign >5 = 2 bd (see note) |
| `UTM(s) (+ SFDC Campaign)` | 2 | UTMs w/ net-new SFDC = 2 bd |
| `Email(s) only \| Nurture Sequences` | 5 | Single Promo Email = 5 bd (see note) |
| `Form Request` | 5 | Form (standalone) = 5 bd |
| `Reporting` | 5 | Salesforce Reports = 5 bd |
| `List Upload` | 7 | 3–7 bd (upper bound used) |
| `IT/Integration` | 15 | 7–15 bd (upper bound used) |

**Approximation notes** (the Project Type enum is coarser than the SLA doc):
- **SFDC multiple**: the doc's breakpoint is >5 *campaigns* (≤5 = 1 bd, >5 = 2 bd). The enum can't tell the count, so `multiple` → 2 bd as a default. (If a campaign-count field is ever added, key off it.)
- **Email single vs multiple**: single promo = 5 bd, **multiple/series = 8 bd** in the doc. The enum can't distinguish, so 5 bd is applied to both (stricter than reality for multi-send). Nurture sequences have no discrete turnaround SLA — folded in here as a known imperfection.
- **UTMs-only** (SFDC already exists) = 1 bd in the doc, but there's no UTM-only Project Type — only `UTM(s) (+ SFDC Campaign)` (= 2 bd), which we map.

**Exclude from the on-time %** (count separately, keep out of numerator and denominator):
- **`Webinar Request` and `Event (+ SFDC Campaign)`** — the doc explicitly anchors "2 weeks / 4 weeks out" to the **live webinar/event date** (a requestor lead-time), with the MOps deliverable chain counted *from the 1st promotional send date*, not task creation. So a created→completed turnaround is the wrong measure (it produced spurious 0% / 28-day cycles). Excluded until an event-date/send-date-anchored SLA is modeled separately.
- No-SLA types: `Audiences`, `Automations | Martech`, `Other`, `UAT`, `Issues`, `6Sense CE`, `Team OOO`, `Routing` (7–15 bd in the doc but no matching enum value), and unset.

Caveats to surface: `created_at` is the submission proxy; business days ignore holidays. A validation run (2026-07-21) over 61 scored completions returned **~34% on-time, avg ~17.5 business days** (Email 23%, Reporting 42%, SFDC single 11%, List Upload 80%).

### B. Workload balance
From Step 2 counts. Flag imbalance when the top assignee's open-task count is ≥ 2× the team median (Harish/Aayushi are the historically loaded pair — surface it, don't editorialize).

### C. Email campaign performance (Salesforce — system of record for Pardot rollups)
Salesforce has no Pardot connector; Pardot pushes engagement rollups onto Campaign records. Query Salesforce read-only (`soqlQuery`, `getObjectSchema`, `find`).

- **Core metrics** (current 30d + prior 30d for deltas):
  `SELECT COUNT(Id), SUM(TotalEmailsDelivered), SUM(UniqueEmailOpens), SUM(UniqueEmailTrackedLinkClicks) FROM Campaign WHERE TotalEmailsDelivered > 0 AND StartDate >= <start> AND StartDate <= <end> AND Type != 'Operational'`
- **Top emails** (row-level, same filters, `ORDER BY TotalEmailsDelivered DESC`): `Name, StartDate, TotalEmailsDelivered, UniqueEmailOpens, UniqueEmailTrackedLinkClicks`. Rank top 5 by CTR; flag any "top" email sent to a tiny audience (<200) so it doesn't skew.
- Metrics: open rate = unique opens / delivered; CTR = unique clicks / delivered.

**Known sync limits — report as unavailable, never estimate around them** (re-check each run, don't assume last run's gaps hold):
- `NumberSent` is 0 org-wide → delivery rate can't be computed.
- Subject lines aren't synced → use campaign name as a labeled proxy.
- `CampaignMember` is responder-biased (~7% sync) → any persona split is within-sample, not true per-persona rates.

**Pipeline attribution — influenced vs sourced (do not conflate).** Wherever per-campaign opps or pipeline appear (the report generator's top-campaigns tables, any pipeline tile), the Campaign rollups (`NumberOfOpportunities` / `AmountAllOpportunities`) are **influenced** (multi-touch, default Campaign Influence model), not sourced. Label them **"influenced opps" / "influenced pipeline"** and show the **sourced** figure (`Opportunity.CampaignId` primary source) alongside so influence is never read as sourcing. Both inherit upstream campaign-member gaps (Terminus/paid-ad leads, in-flight Pardot→SF Flow). Full logic: `pipeline-influenced-email` + memory `sf-attribution-model`.

For deeper email analysis (subject-line patterns, persona engagement, a full deck), hand off to `parbot-email-performance` — this skill carries only the headline email tile.

### D. Triage speed (assignment lag)
How fast a submitted ticket gets an owner — the front-half of cycle time, and the metric that most directly exposes the intake bottleneck (a ticket with no owner has no clock running). Two parts: a **lag distribution** over recently-completed tickets (needs Asana activity history), and a live **unowned-now** watchlist (needs only `get_tasks`).

**Lag over completed tickets.** Reuse the same completed set as on-time SLA (§3.A). For each **top-level** (`parent` is null) completed ticket, pull its activity feed with `get_task_stories` and read:
- `task_created_at` = `created_at` of the earliest story (stories are oldest-first).
- **first assignment** = earliest story with `resource_subtype == "assigned"`; capture its `created_at` and its `created_by` (the *assigner*).
- Lag = business-hours from `task_created_at` → first assignment (weekdays only, holidays ignored — same convention as §3.A).

Report `medianLagBizHours`, the distribution buckets (≤1h immediate / 8–24h ~1bd / 1–3bd / 3bd+), and `neverAssignedCount` (completed with no assignment story).

**Two honesty adjustments — do not skip, or the number lies low:**
- **Self-assignment at creation.** ~28% of tickets are created *and* assigned by the same person (template intake self-assigns). These have ~0 lag but represent no triage decision. Report `selfAssignedSharePct` (share where assigner == creator) alongside the headline so a near-zero median isn't read as "triage is instant."
- **True-triage subset.** Compute `triagedMedianBizHours` over only the tickets where **assigner ≠ creator** (someone routed it to someone else). This is the real triage-speed signal.

Also surface **triage-owner concentration** (`assignmentsByPerson` — who does the assigning); triage is currently concentrated on the lead.

**Unowned-now watchlist** (live, no stories needed — from Step 2's open set). Open, incomplete tickets with **no assignee**, excluding auto-generated event scaffold (empty names, `[Event Name]`, `[NAME]`/`[Name]` prefixes). For each: `daysWaiting` = business days from `created_at` to now. Sort descending — the top of this list is work actively rotting.

**Validation baseline (2026-07-25, n=30 completed top-level, last ~30d):** overall median lag ~0.4 business-h (compressed by 28% self-assignment + heavy template intake); **true-triage median ~11.8 business-h (~1.5 bd)**, p90 ~46h, one ~88-bd outlier; `MOPS- Status → Assigned` median ~14 business-h; 1/30 never got an assignment story; assignments concentrated on Aayushi (11) and Harish (6). A prior open-work snapshot (2026-07-24) found **~33 real unowned tickets**, oldest ~365 days.

**Caveats to surface:** `created_at` is the submission proxy (§3.A); the lag pull is the heaviest query in the briefing (one `get_task_stories` call per completed ticket) — scope it to the completed retro set, not the whole project. The `MOPS- Status → Assigned` signal misses transitions phrased "changed … from X to Assigned" (substring match on "MOPS- Status to Assigned").

## Step 4 — Threshold checks → `alerts[]`

Each tripped threshold becomes one alert `{ type, severity, message, entities[] }`. These drive the routine's daytime alerts and the dashboard's alert strip.

- **sla-risk** — open ticket due within its SLA window and not yet In Progress/Completed.
- **urgent-unassigned** — Priority = Rush/Urgent with no assignee.
- **on-time-below-target** — `onTimePct` below target (default target 80%; the documented recent baseline is ~47%, so expect this to fire until it improves).
- **send-clash** — two email sends sharing a date and audience/region in the look-ahead window (reuse `email-send-calendar`'s clash logic).
- **null-priority** — `nullPriorityCount` > 0.
- **stale-unowned** — an unowned-now ticket (§3.D) waiting beyond its `Project Type` SLA (§3.A table) with still no assignee; escalate the oldest. `warn` (or `critical` if `daysWaiting` > 2× the SLA). Entities = the offending tickets.

Give each a severity (`info` | `warn` | `critical`) so the routine can decide what's worth interrupting the day for (critical/warn) vs. what only belongs in the morning brief (info).

## Step 5 — Emit the `briefing` object

Return this exact shape (the dashboard and routine depend on the keys). Use `null` + a `dataGaps[]` entry for anything unavailable — never omit a key or fake a value.

```json
{
  "generatedAt": "<ISO timestamp>",
  "windows": { "retro": "<start>..<end>", "email": "<start>..<end>", "lookAhead": "<start>..<end>" },
  "newTickets": [ { "name": "", "asanaGid": "", "submitted": "", "note": "Project Type unset (proxy for new)" } ],
  "atRisk": [ { "name": "", "assignee": "", "due": "", "reason": "" } ],
  "overdue": [ { "name": "", "assignee": "", "due": "", "daysLate": 0 } ],
  "onTimePct": 0,
  "avgCycleDays": 0,
  "workloadByPerson": [ { "name": "", "openTasks": 0 } ],
  "nullPriorityCount": 0,
  "blocked": [ { "name": "", "assignee": "", "missingInfo": "" } ],
  "emailPerf": {
    "sends": 0, "openRate": 0, "ctr": 0,
    "openRateDelta": 0, "ctrDelta": 0,
    "topEmails": [ { "name": "", "sent": 0, "openRate": 0, "ctr": 0, "smallAudience": false } ]
  },
  "triageSpeed": {
    "medianLagBizHours": 0, "triagedMedianBizHours": 0, "selfAssignedSharePct": 0,
    "neverAssignedCount": 0,
    "lagDistribution": { "immediate_le1h": 0, "within_1bd": 0, "d1_3bd": 0, "gt3bd": 0 },
    "assignmentsByPerson": [ { "name": "", "count": 0 } ],
    "unownedNow": [ { "name": "", "asanaGid": "", "daysWaiting": 0, "projectType": "" } ]
  },
  "lookAheadSends": [ { "name": "", "sendDate": "", "audience": "", "clash": false } ],
  "alerts": [ { "type": "", "severity": "info|warn|critical", "message": "", "entities": [] } ],
  "dataGaps": [ "" ]
}
```

Then a **narrative** — max 5 bullets, plain sentences, most important first (e.g. "7 new tickets, 3 unassigned"; "On-time 51% — below the 80% target, driven by 4 overdue SFDC-campaign requests"; "Harish carries 12 open tasks vs. a team median of 5"). Internal tone: casual (per `.claude/rules/communication-style.md`).

## Dashboards (two forms, this folder)
- **`dashboard-live.html`** — the **self-refreshing** surface (shipped). Declares the `mcp` runtime capability and calls the viewer's own Salesforce (`soqlQuery`) + Asana (`get_tasks`) connectors on every visit, paginating and computing the briefing **in-browser** (compute logic mirrors this skill, keyed off custom-field **gids**, not names). Acquia-branded (navy header + wordmark + droplet motif, Acquia-Blue titles, stat tiles, status colors). Publish with `capabilities:{mcp:{servers:[{server:"claude_ai_Salesforce",tools:["soqlQuery"]},{server:"claude_ai_Asana",tools:["get_tasks","get_task_stories","update_tasks"]},{server:"claude_ai_Atlassian",tools:["searchJiraIssuesUsingJql","getTransitionsForJiraIssue","transitionJiraIssue"]}]},downloads:true}` — the `claude_ai_Atlassian` server powers the **Jira intake channel** section (below), including the per-row **Close** write action; `downloads` powers the report's **Save .md file** button and must be restated on every publish (see the Email Performance Report note below; `capabilities` is a full-set declaration, so omitting `downloads` silently breaks the button). Because it declares `mcp` it is **private per-viewer and cannot be shared by public link** — each viewer opens it with their own connectors. The Acquia wordmark is inlined as a data URI at build (`__LOGO_DATA_URI__`).
- **`dashboard-template.html`** — the **snapshot** render target for the routine: same brand, but data is injected server-side (routine fills `__BRIEFING_JSON__` from this skill's `briefing` object and re-publishes). Use this where a shareable, no-connector-needed URL is wanted, or where the routine (not the viewer) does the fetch.

**Email Performance Report generator — lives on the LIVE surface only.** A dedicated card in the Email Performance section that produces the **full email-performance brief** (the same structure/format as `outputs/email-performance/Email_Performance_Brief_*.md` and the `parbot-email-performance` deck) for **any quarter, month, or week** the viewer picks — a granularity `<select>` (quarter/month/week) drives a period `<select>` (last 8 quarters / 12 months / 12 weeks; the most recent *complete* period is preselected, in-progress periods are badged). On **Generate**, it runs one `soqlQuery` over `Campaign` for `[priorStart..currentEnd]` (fields incl. `Type`, the rollup pipeline fields, and `Product_Cloud_Interest__c`) plus a best-effort `CampaignMember` persona query, then computes everything **client-side from that single row set** so the on-screen report and the Markdown export reconcile to the exact same source rows. Sections mirror the brief: reliability notes → at-a-glance metrics vs. the prior same-length period (CTR/CTOR deltas are real; the pipeline-$ delta is labelled maturation-lag, never "decline") → **what makes the best campaigns win** (Driver #1 audience-size cohorts, #2 product line, #3 scale-vs-precision concentration) → top campaigns by pipeline and by CTR (≥1k delivered) → deliverability red flags (large sends under 8% open) → data-driven recommendations → directional persona read. Pipeline uses the Campaign rollup `AmountAllOpportunities` / `NumberOfOpportunities` — the `pipeline-influenced-email` single-source metric the written brief used. **⚠️ Correcting a prior error in this skill:** these rollups are **influenced** (multi-touch, fed by the org's **default Campaign Influence model**), **not** primary-source. Confirmed empirically (2026-07-30): a campaign showing `NumberOfOpportunities = 93` mapped exactly to the default influence model's 93 distinct opps, while its true primary-source opps (`Opportunity.CampaignId`) were only **2**. Consequences: (a) label these columns **"Influenced opps" / "Influenced pipeline,"** never bare "opps"; (b) summing `NumberOfOpportunities` across campaigns can **double-count** the same opp (influenced counts aren't deduped by Opp ID across campaigns), so a windowed sum is an **upper bound**, not additive. For the **sourced** companion (smaller, under-counts due to the manual-entry dependency), run a separate primary-source query: `SELECT COUNT(Id), SUM(Amount) FROM Opportunity WHERE CampaignId IN (<campaign ids>)`. Show sourced alongside influenced so influence is never read as sourcing. The live report does this with **one** `sfSourcedByCampaign` pull over all campaigns in both periods, feeding **both** a per-campaign "Sourced opps" column in the top-campaigns table **and** period-wide "Sourced pipeline / Sourced opportunities" rows in the at-a-glance (degrades to "—" if that query fails). See `pipeline-influenced-email` and memory `sf-attribution-model`. Three export controls: **Copy Markdown** (clipboard), **Show Markdown** (reveals a read-only textarea), and **Save .md file** — the last uses the `downloads` runtime capability (`window.claude.downloads.save({filename, data})`, `.md` is in the base allowlist) and, if `downloads` isn't granted in the view, degrades to revealing the Markdown for manual copy. All export the report in the exact written-brief format (structure/subject/format). **Publishing note:** enabling `Save .md file` requires the artifact be published with `capabilities: {mcp:{…}, downloads:true}`; because `capabilities` is a full-set declaration, the `mcp` manifest (servers `claude_ai_Salesforce` / `claude_ai_Asana`) must be restated alongside `downloads`, which only validates from a session where those **claude.ai** connectors are connected (a local-MCP Claude Code session can't restate them, so it can only carry `mcp` forward by omitting `capabilities`, leaving `Save .md file` in its fallback state). Read-only; degrades to a connector-prompt if Salesforce isn't authorized. The compute/render/markdown functions were unit-checked in node against the live Q2 2026 pull (reconciled to 78 campaigns · 1,089,367 delivered · $7,386,728 pipeline).

**Triage-speed card — lives on the LIVE surface only.** The live dashboard renders the full triage card itself: it declares `get_task_stories` in its `mcp` capability and, each refresh, pulls activity feeds for recently-completed top-level tickets (capped at `TRIAGE_CAP` = 40, `TRIAGE_CONC` = 6 concurrent) to compute lag in-browser, mirroring the §3.D method. It degrades gracefully — if the tool isn't granted or a feed fails, the card shows a note and the rest of the page still renders. This is the heaviest work on the page and re-runs on the 10-minute auto-refresh; trim `TRIAGE_CAP` or gate it to manual refresh if the cost bites. The **snapshot** `dashboard-template.html` deliberately does **not** carry a triage tile. The unowned-now watchlist is already a standalone card on the live surface (Unassigned tickets + aging), so the triage card focuses on lag / median / self-assignment / who-assigns. The `triageSpeed` briefing keys (§5) remain for the daily-brief/alerts routine to report in Slack — they're not consumed by the snapshot template.

**Jira intake channel card — lives on the LIVE surface only.** The MOPS Jira project (`project = MOPS`) is a **second request front-door** into the team, parallel to the Asana `[MOps] Intake` project — other teams file requests here too, so the dashboard's Asana-only view was under-reporting real intake. The live surface adds a "Intake — Jira channel" section that declares `searchJiraIssuesUsingJql` in its `mcp` capability and, each refresh, pages the full open backlog (`statusCategory != Done`, ~409 issues, `nextPageToken` loop, capped at 12 pages) and computes three things in-browser (`computeJira`): (1) **real-vs-noise split** — ~120 of 409 are an automated privacy/DSAR stream (reporter = the single automation account `5e163dff9af3650e9e40a3e4`, null email, or a summary matching the privacy/DSAR regex), split out so the ~289 real ops tickets are visible; (2) **orphaned-real alert** — real tickets whose assignee is a **deactivated** user (`assignee.active === false`), which can't move without reassignment (~188 currently → critical alert); (3) **aging** — buckets + oldest table by **created age**, deliberately **not** `updated`: a project-wide bulk edit reset `updated` to ~today on most issues, so it's a dead staleness signal (as is `duedate` — 0/409 populated). Read-only for the metrics; degrades to a connector prompt if Atlassian isn't authorized. The section loads independently (its own `loadJira()` + degrade path) so a Jira failure never blanks the Asana/Salesforce sections.

**Close action (write).** The orphaned and oldest tables carry a per-row **Close** button (`closeJiraIssue`), mirroring the Asana Assign action's confirm pattern. This project's workflow is **forward-only and linear** — `To Do → In Progress → In Review → Done` — and **Done is terminal (no reopen transition exists**, verified against closed Ticket + Sub-task issues, both returning zero transitions). So Close **chains** transitions live (`getTransitionsForJiraIssue` at each hop, taking the Done-category transition when present, else the single forward one; bails on an ambiguous multi-transition status rather than guessing) until it reaches Done, and sets the **required** `resolution` on the Done step (`transitionJiraIssue`, `fields.resolution.id`; the confirm offers **Done / Won't Do / Outdated** = ids 10000 / 11010 / 6). Writes are **not** auto-retried (a rejected write may have applied). Because Done is terminal, the confirm warns it's **one-way** — there's no in-page undo. The chaining logic was unit-checked (In Progress → In Review → Done applies transition 4 then 5, resolution only on the Done hop). The `transitionJiraIssue` write itself was **not fired in-session** (any real close is irreversible here), so its first live use is its verification — by design, the first real close is made knowingly by the manager. Validated against a full 409-issue pull (2026-07-28): reconciles to 409 total / 289 real / 120 noise / 207 orphaned. The snapshot `dashboard-template.html` does not carry this section.

## Notes
- For the live surface, the skill's job is the compute *contract* the in-browser code mirrors; for the snapshot surface, this skill produces the `briefing` object and the `mops-command-center` routine handles delivery and re-publish.
- On first live run, confirm the org's Campaign email fields via `getObjectSchema('Campaign')` and the current persona field (as `parbot-email-performance` does) before trusting the SOQL above; cache what works.
- "New ticket" via unset `Project Type` is a **proxy** until the live intake form ships — always label it as such, and revisit once the form exists (tracked in `projects/intake-pipeline-automation/`).
- If a connector needs re-auth, record it in `dataGaps[]` and finish the rest of the briefing rather than aborting.
