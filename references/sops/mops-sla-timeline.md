# MOps Requests — SLA Timeline Requirements (2025)

_Source: "MOps Requests: SLA Timeline Requirements_2025.pdf" (official Acquia MOps doc, provided 2026-07-21). This is the authoritative SLA reference; the `mops-command-center` on-time metric and `intake-tracking` are keyed to it._

Purpose: ensure strategic/operational/executional work for customer/prospect-facing email messaging is completed on time.

- **Rush/Urgent** requests are raised to Marketing VP and Director level for approval.
- **Max 3 marketing emails per record per week** to any one audience.

## SLAs by submission type (business days from submission unless noted)

### SFDC Campaigns & UTM Requests
- SFDC Campaign, **up to 5** campaigns, creation only = **1 business day**
- SFDC Campaign, **more than 5** = **2 business days**
- **UTMs only** (SFDC campaigns already exist) = **1 business day**
- **UTMs with SFDC Campaigns** (net-new SFDC required) = **2 business days**
- Out-of-scope = MOps + stakeholder agree a timeline

### Single Promotional Email (one email, one version) — **5 business days out**
- 5 bd out: submit via Asana intake (details + banners/creative), leave unassigned; MOps assigns
- 3 bd out: proof due to stakeholder · 2 bd out: feedback due (copy/URL only, no template changes) · 1 bd out: final approval (email + send list) · 1 day out: MOps schedules

### Multiple Promotional Emails (series or multiple same-day) — **8 business days out**
- 8 bd out: submit · 4 bd: proof · 3 bd: feedback · 2 bd: final approval · 1 day out: schedule

### Webinar Requests — **2 weeks out from the live webinar** (lead-time)
- Submit ≥ 2 weeks before the webinar
- 7 bd out (from 1st promotional send): landing page + SFDC Campaign created & shared
- 4 bd: email proof · 3 bd: feedback · 2 bd: final approval (LP, emails, send list) · 1 bd: schedule
- Larger/complex = case-by-case

### Event Requests — **4 weeks out from the live event** (lead-time)
- Submit ≥ 4 weeks before the event
- 7 bd out (from 1st promotional send): event landing page + SFDC Campaign created & shared
- 5 bd: first email proof · 4 bd: feedback · 2 bd: final approval · 1 bd: schedule
- Larger/complex = case-by-case

### Form (standalone) Requests — **5 business days out**
- 5 bd: submit · 3 bd: first form proof · 2 bd: feedback + final approval · 1 bd: MOps publishes

### Salesforce Reports (per report) — **5 business days out**
- 5 bd: initial data validation · 3 bd: revisions · 1 bd: final approval

### IT Integration Requests — **7–15 business days** (by complexity)
### List Uploads — **3–7 business days** (by complexity)
### Routing Requests — **7–15 business days** (by complexity)

## Deliverables required at time of Asana request
Subject line & preheader · name of Email/Webinar/Event · stakeholder contact (name/email) · banners (or links to creative requests) · content copy (enough to start) · URLs · SFDC Campaign name · audience segmentation · date(s) · send time(s) · exclusion list(s).

## Send-calendar considerations
- Max 3 marketing emails per record per week; use the calendar to avoid same-audience overlap and the 3-email cap.
- **Tuesdays** are reserved for Welcome Nurture drips (do not book prospect emails Tuesday).
- Remove calendar HOLDs if you can't meet the 7-bd requirement; >3 on a day → consider another day (extra segmentation = extra analyst time).
- Attend the weekly Marketing Email meeting (calendar visibility, cross-team sends, process updates, special requests).

## Weekly send-day ownership
| Day | Type | Team owner |
|---|---|---|
| Monday | Webinar invites | Events, DG, Field Marketing |
| Tuesday | "Welcome" nurture emails | MOps, DG, Field Marketing |
| Wednesday | ABM · Customer comms · Multi-asset promo (tiered content) | DG, Customer Marketing, MOps |
| Thursday | Event invites · Partner comms | Events, Partner, Field Marketing |
| Friday | Rest day — no sends unless critical | MOps |

## How this maps into the dashboard (`mops-command-center`)
- **On-time %** scores turnaround SLAs only: SFDC single 1 / multiple 2, UTM(+SFDC) 2, Email 5, Form 5, Reporting 5, List Upload 7, IT/Integration 15. Webinar/Event are **excluded** (lead-time before the event, not turnaround). See the skill's Step 3.A for the approximation notes (enum can't see campaign count or single-vs-multi email).
- **Send calendar** clash flagging aligns with the "3/week per audience" + day-ownership rules — a future enhancement could flag day-type mismatches (e.g. a prospect email booked on Tuesday) and >3-per-audience-per-week directly.
