---
name: pipeline-influenced-email
description: Measures the sales pipeline (and opportunities) tied to email campaigns from Salesforce — both influenced (multi-touch Campaign Influence) and sourced (Primary Campaign Source) — the "did email make money" view. Use for email→pipeline/revenue impact, to add a pipeline metric to the command-center dashboard, or to report email-attributed pipeline for a period.
argument-hint: [period, e.g. "Q3 2026", "last 30 days", "April 1 - June 30 2026"]
---

# Pipeline-Influenced $ on Email

Quantifies how much **pipeline** email marketing is tied to, from Salesforce (Pardot's engagement metrics aren't monetary — pipeline lives in Salesforce). Single source of truth for the metric wherever it appears: the `mops-command-center` dashboard's email section, the `parbot-email-performance` deck, or a standalone answer.

**Rigor rule:** never fabricate a dollar figure. If a field is empty or unreliable, report it as such (see "What NOT to use") rather than estimating around it.

## Two attribution layers — never conflate them

Acquia's SF has **two distinct** campaign→pipeline lenses. They answer different questions and produce very different numbers. Always label which one you're showing.

| Layer | Question it answers | Mechanism | Typical size |
|---|---|---|---|
| **Influenced** (multi-touch) | "What pipeline did this email *touch*?" | `CampaignInfluence` object / the Campaign rollup fields | **Larger** |
| **Sourced** (first-touch) | "What pipeline did this email *originate*?" | `Opportunity.CampaignId` (Primary Campaign Source) | **Smaller** |

**⚠️ The Campaign rollup fields (`NumberOfOpportunities`, `AmountAllOpportunities`) are INFLUENCED, not sourced.** They're fed by the **default Campaign Influence model** — multi-touch. This was confirmed empirically (drill-down 2026-07-30): a campaign showing `NumberOfOpportunities = 93` mapped exactly to the default influence model's 93 distinct opps, while its true primary-source opps (`Opportunity.CampaignId`) numbered only **2**. Reading the rollup as "opps this email sourced" is wrong by ~40×. See memory `sf-attribution-model` for the full logic.

### Influenced (multi-touch) — the headline the command center shows
- **Pipeline influenced ($)** = `SUM(AmountAllOpportunities)` across the email campaigns in the window.
- **Opportunities influenced (#)** = `SUM(NumberOfOpportunities)`.
- **Won deals (#)** = `SUM(NumberOfWonOpportunities)` — the *count* is trustworthy.
- **Responses (#)** = `SUM(NumberOfResponses)`.
- Powers influenced-opp / influenced-amount reporting and the Engage event dashboards (Denver / Paris / London).

### Sourced (first-touch) — smaller, leakage-prone, but the "originated" truth
- **Sourced pipeline ($)** = `SUM(Opportunity.Amount)` for opps whose `CampaignId` is one of the window's email campaigns.
- **Sourced opps (#)** = `COUNT` of those opps.
- This is the field that actually drives "pipeline by campaign" reports and trial-sourced reporting.

## Metrics — derived (make it comparable, not just a big number)
- **Pipeline per 1k delivered** = `pipeline ÷ (delivered / 1000)` — normalizes for send size; preferred when ranking sends.
- **Pipeline per campaign** = `pipeline ÷ #campaigns`.
- **Response → opp rate** = `opportunities ÷ responses`.
- **Per-campaign pipeline** — rank individual campaigns by `AmountAllOpportunities` for the "top emails by pipeline" view.

## What NOT to use (state the gap on the surface, don't work around it)
- **`AmountWonOpportunities` (won $)** — unreliable in this org (mostly `0`; observed a **−$2,502** on one campaign). Report **won count**, not won dollars, until it's cleaned up.
- **`ExpectedRevenue`, `ActualCost`, budget/ROI fields** — `null` org-wide → **no ROI or cost-per-pipeline** is computable. Do not synthesize one.

## Data source — SOQL (read-only; SF is system of record, no Pardot connector)

**Influenced (Campaign rollups):**
```sql
SELECT Name, StartDate, TotalEmailsDelivered, UniqueEmailOpens, UniqueEmailTrackedLinkClicks,
       NumberOfResponses, NumberOfOpportunities, AmountAllOpportunities, NumberOfWonOpportunities
FROM Campaign
WHERE TotalEmailsDelivered > 0 AND StartDate >= <window start> AND StartDate <= <window end> AND Type != 'Operational'
ORDER BY AmountAllOpportunities DESC NULLS LAST
```

**Sourced (Primary Campaign Source) — for one campaign or a set:**
```sql
SELECT COUNT(Id) sourced_opps, SUM(Amount) sourced_pipeline
FROM Opportunity
WHERE CampaignId IN (<campaign ids from the query above>)
```

**Per-campaign influenced, deduped by Opp ID (when you need the honest single-campaign count, not the rollup):**
```sql
SELECT COUNT_DISTINCT(OpportunityId) opps, SUM(RevenueShare) rev_share
FROM CampaignInfluence
WHERE CampaignId = '<id>' AND ModelId = '<default model id, IsDefaultModel = true>'
```
Confirm fields via `getObjectSchema('Campaign')` / `getObjectSchema('CampaignInfluence')` on first run if a schema shift is suspected. Find the default model with `SELECT Id FROM CampaignInfluenceModel WHERE IsDefaultModel = true`.

## Attribution & caveats — keep the number honest

1. **Influenced double-counts across campaigns.** Influenced-opp counts were **not confirmed deduplicated by unique Opp ID**, so cross-campaign roll-ups (and `SUM(NumberOfOpportunities)` over a window) can count the same opp for multiple campaigns. A *single-campaign* `COUNT_DISTINCT(OpportunityId)` is safe; a windowed sum is an upper bound — say so.
2. **Sourced under-counts.** `Opportunity.CampaignId` doesn't populate reliably: SF can auto-associate it at lead conversion, but in practice it often depends on the rep setting it manually (hence the "set Primary Campaign Source" critical-action callout on the SET enablement slides). Sourced pipeline is a floor, not the full picture.
3. **Both inherit upstream campaign-member gaps.** Neither layer works unless the record is a `CampaignMember` on the right campaign first. Known gap: extended Terminus (paid-ad) URLs bypass the redirect that creates the association, and no automation reliably stamps those leads onto campaigns. The hybrid Pardot→SF Flow (Campaign lookup by 18-char ID with `utm_campaign` fallback, dedup on member creation) is designed to close this — **in-flight via an admin-team Jira ticket, not live as of 2026-07-30**. Member creation works today via the Pardot connector (B2BMA Integration) off form fills and Engagement Studio actions.
4. **Pipeline lags the send.** Opportunities form *after* the email, so a recent window shows less pipeline because deals haven't matured — not because the email underperformed. Compare like-for-like windows (quarter vs prior quarter) and note maturation; avoid a period-over-period "delta" that reads as a decline.
5. **Currency.** Long decimals suggest currency conversion. Confirm a single reporting currency (`CurrencyIsoCode` / converted fields) before summing across regions.
6. **Small-audience skew.** A tiny send with one big opp can dominate per-1k ratios — flag sends under ~200 delivered, consistent with `parbot-email-performance`.

## Output

- **As a dashboard section** (command-center): an **"Influenced pipeline"** tile (headline $) with **influenced opps**, **won count**, **$ per 1k sent**, **response→opp rate**, plus a **sourced opps / sourced $** pair shown alongside so influence is never read as sourcing, and a **top-emails-by-pipeline** mini-table (Name · Sent · Influenced opps · Pipeline). Footnote: *"Influenced = multi-touch (default Campaign Influence model), may double-count across campaigns; sourced = Primary Campaign Source (under-counts, manual-entry dependency); won $ and cost not shown (SFDC gaps); pipeline lags the send."*
- **As a standalone answer / deck slide:** influenced + sourced side by side, the derived metrics, the top 5 campaigns by pipeline, and the caveats stated plainly.

## Notes
- Read-only — reads Salesforce and computes; never writes.
- Reconcile the email population with `parbot-email-performance` (same `TotalEmailsDelivered > 0`, `Type != 'Operational'` filter) so pipeline and engagement numbers describe the same set of sends.
- Business logic single source of truth: memory `sf-attribution-model` and (once created) the vault `concepts/campaign-attribution` page.
