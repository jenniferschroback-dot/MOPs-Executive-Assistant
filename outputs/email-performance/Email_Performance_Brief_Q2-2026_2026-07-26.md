# Email Campaign Performance Brief — Q2 2026

**Prepared for:** Acquia Marketing team
**Prepared by:** MOPS (Forkan) · **Date:** 2026-07-26
**Source:** Salesforce Campaign records (Pardot/Account Engagement engagement + opportunity rollups). Read-only, pulled live 2026-07-26.

---

## 0. How to read this (reliability first)

This brief is built to be trusted, so the ground rules are stated up front:

- **Window:** Q2 2026 = **April 1 – June 30, 2026** (full 3-calendar-month quarter), filtered to `StartDate` in window. *(Note: an older internal spec sometimes defined a "quarter" as a 2-month window — this brief uses the full 3 months. Flag if you need the 2-month definition to match another report.)*
- **Population:** every Salesforce Campaign with `TotalEmailsDelivered > 0` and `Type != 'Operational'` in the window → **78 campaigns**. Same definition used by the ParBot and pipeline skills, so numbers reconcile across MOPS reporting.
- **Reconciliation check:** the 78 row-level records sum exactly to the Salesforce aggregate totals (delivered 1,089,367 · opens 95,472 · clicks 5,555 · pipeline $7,386,728 · opps 3,263). **No estimated or filled-in numbers anywhere in this brief.**
- **Metric definitions:** open rate = unique opens ÷ delivered · CTR = unique clicks ÷ delivered · **CTOR (click-to-open) = unique clicks ÷ unique opens** (the cleanest engagement-quality metric) · pipeline = `AmountAllOpportunities` (single-touch, primary-source attribution).

**Known data limits — do not work around these (they shape every conclusion):**
- **Opens are inflated by Apple Mail Privacy Protection** (auto-opens). Treat open rate as directional; **CTR and CTOR are the more reliable engagement signals.**
- **Pipeline $ lags the send** — opportunities form *after* the email. A recent quarter always shows less matured pipeline. **Do not read a quarter-over-quarter pipeline $ drop as email underperforming.**
- **Pipeline is single-touch (primary-source only)** → it *undercounts* email's assist role. The $ means "pipeline whose primary source was this email," not "all email-influenced pipeline."
- **Won $ and cost/ROI fields are unreliable/null in this org** → we report **won opportunity *counts*, never won dollars or ROI.**
- **Subject lines and `NumberSent` are not synced** → no true delivery-rate and no subject-line length/token analysis; campaign name is used as a labeled proxy.
- **Persona data is within-sample only:** just **7.7%** of delivered recipients sync back as Campaign Members (responder-biased). Persona section is directional, not a true per-persona rate.

---

## 1. Q2 2026 at a glance

| Metric | Q2 2026 | Q1 2026 | QoQ | Reliable to compare? |
|---|---|---|---|---|
| Email campaigns | 78 | 71 | +7 | ✅ |
| Emails delivered | 1,089,367 | 1,849,724 | −41% | ✅ (volume down) |
| Open rate (unique) | 8.76% | 8.34% | +0.4 pt | ⚠️ MPP-inflated |
| **CTR (unique)** | **0.51%** | **1.00%** | **−49%** | ✅ **real signal** |
| **CTOR (click-to-open)** | **5.82%** | **12.04%** | **−52%** | ✅ **real signal** |
| Pipeline influenced | $7.39M | $47.64M | — | ❌ maturation lag — *not* a decline |
| Opportunities influenced | 3,263 | 16,725 | — | ❌ maturation lag |
| Won opportunities (count) | 516 | 2,944 | — | ❌ maturation lag |

**The one number to watch:** click-through collapsed by half QoQ (CTR 1.00%→0.51%, CTOR 12.0%→5.8%) while opens held flat. People are still opening; **far fewer are clicking.** This is a content/CTA/relevance problem, not a deliverability-to-inbox problem — and it is *not* explained by pipeline lag, so it is the most actionable finding in the quarter.

**Do not misread the pipeline drop.** Q2 pipeline ($7.4M) looks far below Q1 ($47.6M), but Q1's deals have had 3+ extra months to mature. This is expected lag, not underperformance. Compare $ only across fully-matured, like-aged quarters.

---

## 2. What makes one email campaign outperform another — the drivers

Ranked by strength of evidence in the data.

### Driver #1 — Audience precision (strongest, cleanest signal)
Every engagement and efficiency metric rises **monotonically** as the audience gets smaller and more targeted:

| Audience size | # sends | Delivered | Open rate | CTR | CTOR | Pipeline / 1k delivered |
|---|---|---|---|---|---|---|
| Mass (>50k) | 1 | 565,082 | 6.99% | 0.45% | 6.49% | $6,728 |
| Large (10k–50k) | 24 | 473,863 | 9.93% | 0.50% | 5.00% | $6,107 |
| Mid (1k–10k) | 20 | 42,105 | 15.14% | 0.84% | 5.52% | $8,405 |
| **Targeted (<1k)** | **33** | **8,317** | **30.55%** | **3.46%** | **11.33%** | **$40,487** |

- Targeted sends earn **~4.4× the open rate, ~7.6× the CTR, and ~6× the pipeline-per-1k** of the mass blast.
- **Takeaway:** tighter targeting is the single biggest controllable lever on email quality. This does **not** mean "stop doing big sends" (see Driver #3) — it means small, well-segmented sends are dramatically more efficient per email, and the portfolio is currently over-weighted to low-efficiency volume.

### Driver #2 — Send purpose & funnel stage
The highest-CTR reliable campaigns (≥1,000 delivered) cluster into two repeatable plays:

- **Account-targeted content / thought-leadership ("ATV" sends)** — top CTRs in the quarter: *engage london daniel* (2.89% CTR, 28% open), *figma* (2.57%), *source series* (2.34%), *dam explainer* (1.79%). Small, relevant, content-led → best click quality.
- **Late-funnel event nurture to warm/registered lists** — *paris countdown* (2.23% CTR), *invite9* reminders, "see you tomorrow / last chance" sequences. Engagement escalates as the audience self-selects deeper into the event funnel.
- **Takeaway:** relevance and intent beat reach for click quality. Registration-reminder sequences and account-targeted content are the reliable CTR winners — fund and templatize them.

### Driver #3 — Scale still wins on absolute pipeline (the strategic tension)
The quarter is dominated by **one** send — the DAM "governance, growth and stacks" blast:

- **51.9% of all Q2 delivered volume** (565k of 1.09M) and **51.5% of all Q2 pipeline** ($3.80M of $7.39M), 761 opps, 80 won.
- Its engagement rates are *below* portfolio average (6.99% open, 0.45% CTR), yet at massive scale it still produced the most pipeline of any single campaign.
- **Takeaway:** "better" depends on the goal. For **efficiency**, targeted wins overwhelmingly. For **absolute reach/pipeline volume**, a well-aimed mass send is still the workhorse. The two should be budgeted as different jobs, not compared head-to-head. *(Caveat: excluding this one send, portfolio open rate rises to 10.68% and CTR to 0.57% — one campaign is pulling the whole-quarter averages down.)*

### Driver #4 — Product line as a proxy for content strategy
| Product line | # | Delivered | Open rate | CTR | CTOR | Pipeline | Opps |
|---|---|---|---|---|---|---|---|
| Content Cloud (DAM) | 6 | 604,227 | 6.93% | 0.45% | 6.50% | $4.06M | 833 |
| DXP (Engage events) | 48 | 353,454 | 9.53% | 0.56% | 5.84% | $2.70M | 1,826 |
| Acquia Source | 19 | 87,715 | **18.05%** | **0.86%** | 4.74% | $0.44M | 459 |
| Drupal Cloud | 5 | 43,971 | 9.27% | 0.26% | 2.80% | $0.19M | 145 |

- **Acquia Source** content sends have by far the best open + CTR (small, targeted content — consistent with Drivers #1/#2).
- **DXP / Engage event series** is the **pipeline & opportunity workhorse** (1,826 opps — most of any line) via the invite→reminder→follow-up sequences.
- **Content Cloud** is pipeline-heavy but engagement-light — it *is* the mass blast.
- **Drupal Cloud** has the weakest CTOR (2.80%) — its clicks-per-open lag; worth a content/CTA review.

---

## 3. Top campaigns (reference tables)

**Top 5 by pipeline influenced ($):**
1. DAM governance growth and stacks — $3,802,010 · 565,082 delivered · 761 opps · 80 won
2. hounder webinar ada compliance — $433,350 · 22,913 delivered · 205 opps · 19 won · **$18,913/1k (very efficient)**
3. engage london cust last chance to register — $302,943 · 24,487 delivered · 129 opps · 24 won
4. dam workshop milwaukee — $220,896 · 14,846 delivered · 52 opps · $14,879/1k
5. engage paris cust pros invite9 — $208,982 · 11,866 delivered · 86 opps · $17,612/1k

**Top 5 by CTR (≥1,000 delivered, reliable sample):**
1. engage london daniel (ATV) — 2.89% CTR · 28.1% open · 1,140 delivered
2. figma (ATV) — 2.57% CTR · 30.6% open · 1,049 delivered
3. webinar ia seo es — 2.40% CTR · 20.5% open · 1,418 delivered
4. source series (ATV) — 2.34% CTR · 29.1% open · 1,069 delivered
5. engage paris cust countdown to paris — 2.23% CTR · 12.7% open · 11,566 delivered *(best CTR at scale)*

---

## 4. Red flags to fix (deliverability / targeting)
Several large sends had **open rates under 5%** — abnormally low even accounting for MPP, pointing to list-quality, inboxing, or targeting problems. These are dragging the whole-portfolio open rate down:

- *may finserv Digital Assets* — **2.70% open** on 21,615 delivered
- *engage denver cust pros invite8* — **3.06% open** on 46,498 delivered
- *engage denver cust pros invite9* — **4.47% open** on 46,192 delivered
- *invite8 denver partner engage* — 5.62% open on 12,128 delivered

**Action:** audit these lists/sender reputation/segmentation before the next send to those audiences. Fixing the sub-5% openers is low-effort, high-return.

---

## 5. Recommendations for the marketing team (prioritized)
1. **Diagnose the CTR collapse first.** CTR/CTOR halved QoQ with opens flat → the issue is content/CTA/relevance after the open. Audit Q2's mass and large sends for CTA clarity, link count, and offer fit vs. Q1's higher-clicking sends.
2. **Shift mix toward targeted sends.** They convert 4–8× better and produce 6× the pipeline per 1k. Set a floor on segmented/targeted volume rather than defaulting to broad blasts.
3. **Templatize the two proven winners** — account-targeted content (ATV) and event registration-reminder sequences — and reuse their structure across product lines.
4. **Keep mass sends, but aim them.** The DAM blast proves scale still drives absolute pipeline; pair scale with tighter segmentation to lift its 0.45% CTR.
5. **Fix the sub-5%-open lists** in Section 4 before re-sending to those audiences.
6. **Give Drupal Cloud a CTA/content review** — weakest CTOR (2.80%).
7. **Close the instrumentation gaps** so next quarter's brief is even more reliable: sync subject lines and `NumberSent` from Pardot, and improve Campaign Member sync coverage (currently 7.7%) to unlock true per-persona and delivery-rate analysis.

---

## 6. Persona read (directional only — 7.7% sample, do not quote as rates)
Among the ~7.7% of recipients who synced back as Campaign Members and engaged, the most-engaged job functions were **Marketing (3,921)**, **IT Architect/IT Operations (3,366)**, **Business Executive (1,739)**, and **Engineering/Development (1,482)**. *This reflects who engaged within a small responder-biased sample — not open/click rates by persona.* The prospect (Lead) vs. customer (Contact) split could not be reliably separated this run (queries returned identical distributions), so no split is presented. Improving Campaign Member sync coverage is the fix.

---
*Methodology, definitions, and all row-level data available on request. Prepared from live Salesforce data; re-runnable for any period via the MOPS `parbot-email-performance` and `pipeline-influenced-email` skills.*
