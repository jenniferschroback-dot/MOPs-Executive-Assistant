---
name: parbot-email-performance
description: Generates the Pardot Email Performance deck for any time range or period a user specifies (a quarter, a month, a half, a custom date range, a trailing window like "last 30 days") — a 6-slide Acquia-branded .pptx covering core email metrics, top performing emails, subject-line patterns, and persona engagement — sourced from Salesforce (system of record for Pardot rollups, since Pardot itself has no MCP connector). Use when asked to run a Pardot/email performance report for a given period.
argument-hint: [period, e.g. "Q3 2026", "June 2026", "last 30 days", "April 1 - June 1 2026"]
---

# ParBot: Pardot Email Performance

Analyzes a specified period of Acquia's Pardot (Account Engagement) email sends and delivers the findings as a 6-slide Acquia-branded PowerPoint deck. Be rigorous: never fabricate a metric — if a data point is unavailable, say so on the slide (e.g., "prior-period comparison unavailable — data not accessible") instead of estimating silently.

## Step 1: Resolve the reporting window

Take the reporting period as the argument — it can be a named period (`Q3 2026`, `H1 2026`, `June 2026`, `FY2026`), an explicit date range (`April 1 – June 30 2026`), or a relative trailing window (`last 30 days`, `trailing 6 weeks`, computed back from today). If no period is given, ask what period to run for before doing anything else.

Parse it into an explicit **current window** (start date → end date, inclusive) and a comparison **prior window** — the immediately preceding period of the same length (e.g. Q1 for a Q2 request, May for a June request, the 30 days before a "last 30 days" window). Use the prior window for period-over-period comparison (QoQ/MoM/etc. as appropriate to the period type — call it "prior-period comparison" generically on the deck).

**Quarter-specific caveat:** the original spec this skill is based on defined a quarter's window as first-day-of-quarter → first-day-of-third-month (e.g. Q2 2026 = April 1 – June 1), covering only ~2 of the 3 months — never confirmed as intentional vs. a mistake in that spec. When the requested period is a calendar quarter, **ask Forkan which definition to use** (full 3-calendar-month quarter vs. this 2-month convention) on first use, then remember the answer for future quarterly runs. Non-quarter periods (months, halves, custom ranges) aren't affected by this ambiguity — use the literal dates given/computed.

State the resolved current and prior windows explicitly on the deck's methodology note so the period actually analyzed is never ambiguous to the reader.

## Step 2: Pull the data — Salesforce is the system of record

There is **no direct Pardot connector**. Pardot pushes email engagement rollups onto connected SFDC Campaign records, so query Salesforce (`soqlQuery`, `getObjectSchema`, `find`, `getRelatedRecords`) instead — it runs under the authenticated user's permissions.

1. **Core period metrics** — aggregate on `Campaign`:
   `SELECT COUNT(Id), SUM(TotalEmailsDelivered), SUM(UniqueEmailOpens), SUM(UniqueEmailTrackedLinkClicks) FROM Campaign WHERE TotalEmailsDelivered > 0 AND StartDate >= <window start> AND StartDate <= <window end> AND Type != 'Operational'`
   Repeat for the prior window.
2. **Per-campaign detail** (top emails, subject-line proxy, product/GTM lenses) — same filters, row-level:
   `SELECT Name, StartDate, Type, TotalEmailsDelivered, UniqueEmailOpens, UniqueEmailTrackedLinkClicks, GTM_Playbook__c, Product_Cloud_Interest__c FROM Campaign WHERE ... ORDER BY TotalEmailsDelivered DESC`
3. **Persona field discovery** — query metadata, don't dump full schemas:
   `SELECT QualifiedApiName, Label FROM FieldDefinition WHERE EntityDefinition.QualifiedApiName = '<Lead|Contact>' AND (QualifiedApiName LIKE '%ersona%' OR QualifiedApiName LIKE '%Function%')`
   As of the 2026-07-18 run, the persona field is **`Job_Function__c`** (present on both Lead and Contact). `Partner_Persona__c` and `DAM_*_Persona*__c` are segment-specific — don't use them for the portfolio view. Re-run this discovery query each cycle rather than assuming the field hasn't changed; state on the slide which field was used.
4. **Persona / audience engagement** — `CampaignMember` joined to the campaign window:
   `SELECT <Lead|Contact>.Job_Function__c, HasResponded, COUNT(Id) FROM CampaignMember WHERE Campaign.TotalEmailsDelivered > 0 AND Campaign.StartDate >= ... AND Campaign.StartDate <= ... AND Campaign.Type != 'Operational' AND Type = '<Lead|Contact>' GROUP BY <Lead|Contact>.Job_Function__c, HasResponded`
   Lead members = prospects; Contact members = customers/known accounts.

Definitions: unique opens → open rate; unique clicks / delivered → CTR; delivered / sent → delivery rate. Define every metric in a footnote where it first appears. Deduplicate resends and A/B variants into one send for totals (compare variants only for subject-line analysis). Timestamp every query in the deck's source footnotes.

### Known sync limitations — report as unavailable, never estimate around them
- **`NumberSent` is not synced** (0 org-wide) → delivery rate cannot be computed.
- **Subject lines are not synced** → use the campaign name as a labeled proxy; no character-length/personalization-token analysis until subject lines are added to the sync.
- **`CampaignMember` is responder-biased**: only a small fraction of recipients sync as members (~7% in the 2026-07-18 run), mostly those who engaged. Persona metrics are *within-sample* (share of synced members who responded), not true per-persona open/click rates.
- **`GTM_Playbook__c`** may be unpopulated — if blank for most/all campaigns in the window, note the gap and fall back to `Product_Cloud_Interest__c` as the product lens.

Re-check these limitations each run rather than assuming last quarter's gaps still hold — sync coverage can change.

## Step 3: Analyze

1. **Core email metrics** — total sends, delivery rate, open rate (unique), CTR for the period; prior-period deltas with directional indicators (or labeled external B2B SaaS benchmarks if prior-period data is unavailable, citing the source); briefly note data-quality caveats (e.g., Apple Mail Privacy Protection inflating opens).
2. **Top performing emails** — rank top 5–10 by CTR primarily, open rate secondary. Show campaign/subject-line proxy, send date, audience size, open rate, CTR. Flag any "top" email sent to a small audience (e.g. under 200) instead of letting it skew the ranking. Analyze winning names/subject lines for length, personalization tokens, phrasing, urgency language, question vs. statement framing, and theme — mapped to Acquia products/GTM plays where identifiable. Turn patterns into 3–5 concrete, testable recommendations.
3. **Persona engagement** — break down opens/clicks by the confirmed persona field. Identify most-engaged personas; flag underengaged ones using a stated quantitative bar (e.g. engagement rate below 50% of the portfolio median, or high volume + bottom-quartile CTR). Segment prospect (Lead) vs. customer (Contact) engagement and note content differences where the data supports it.

**Cross-cutting, apply throughout:** tie top content and persona engagement back to Acquia product lines; attribute emails to GTM Play where identifiable and note which plays drove engagement; distinguish prospect vs. customer wherever supported.

Sanity-check before reporting: delivered ≤ sent, unique clicks ≤ unique opens is typical (investigate anomalies, don't just report them). If persona field, GTM tagging, or prior-period comparison data is missing or unusable, still complete the rest of the deck and add a clearly labeled gap note describing what instrumentation change would fix it next time. No prospect-level PII (names, emails) — aggregate only.

## Step 4: Build the deck (max 6 slides, fixed structure)

1. Title — period label, date range, "Pardot Email Performance — [Period]"
2. Core email metrics + prior-period/benchmark comparison
3. Top performing emails (ranked table/chart)
4. Subject-line analysis + recommendations
5. Persona engagement (prospect vs. customer split, underengaged personas flagged)
6. Takeaways & next-period recommendations — **mandatory**, specific and prioritized (what to do, for which audience/persona, expected impact), not generic advice

Acquia-branded: Acquia primary blue palette, clean white layouts, consistent title bar (match brand template if one is available in Drive; ask before assuming a specific template file). Charts over tables where possible. Every slide has one clear headline takeaway written as a full sentence, not just a label. Keep text scannable — no dense paragraphs, ~5 bullets per slide max.

## Step 5: Output & delivery

1. Save the deck locally as `outputs/email-performance/Pardot_Email_Performance_<Period>_<YYYY-MM-DD>.pptx`, where `<Period>` is a filename-safe label for the resolved window (e.g. `Q3-2026`, `June-2026`, `Last-30-Days`, `2026-04-01_to_2026-06-30`). Chart PNGs built for the deck go in `outputs/email-performance/assets/`, and a written brief companion (`Email_Performance_Brief_<Period>_<YYYY-MM-DD>.md`) alongside the deck. `mkdir -p` as needed; never write to `outputs/` root — see `.claude/rules/output-files.md`.
2. Upload it via Google Drive `create_file` (`contentMimeType: application/vnd.openxmlformats-officedocument.presentationml.presentation`, base64 content) into the same shared Drive folder the MOps Weekly Review uses (folder id `0ACqafLRVUxJzUk9PVA`), leaving `disableConversionToGoogleType` unset so Drive converts it to native Google Slides.
3. In your final message: the Google Slides link, plus a one-paragraph summary of the top 3 findings.

If Drive is unavailable, still finish the deck locally and flag the delivery gap rather than silently skipping it.

## Notes
- This skill only reads Salesforce/Pardot-rollup data — it doesn't create or edit campaigns, so it isn't blocked by the read-only Salesforce MCP limitation noted in `tools/available-tools.md`.
- The quarter-window ambiguity and the current persona field (`Job_Function__c`) should both be reconfirmed on first use and whenever a schema shift seems possible — don't treat either as permanently settled.
- For very short or unusual periods (e.g. a single week, or a range with few/no synced campaigns), the ~7% CampaignMember responder-bias and small-sample skew described above get worse, not better — call out low sample sizes even more explicitly than in a full-quarter run.
- Don't silently drop a required slide if a section has no data — keep the slide and state the gap plainly (matches the "no fabrication" standard the whole routine is built on).
