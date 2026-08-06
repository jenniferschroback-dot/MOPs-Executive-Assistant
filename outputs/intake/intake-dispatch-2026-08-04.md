# Intake Dispatch — 2026-08-04

**Window:** 2026-08-04T05:03:59Z → 2026-08-04T13:00:31Z
**4 new submissions · 2 triaged already (skipped) · 1 proposed · 1 blocked**

🚨 **Escalation:** `Source Subscription Notification - All Active customers` arrived with **Priority = Urgent** and **Out of SLA/Rush = Yes**, unassigned, and with **no Project Type** — so the routing engine can't touch it. Highest-attention item in this window.
⚠️ **Load imbalance:** 1 of 1 proposal → Aayushi (100%). n=1, so this is not a trend — reported for consistency. Current open in `[MOPs] Intake`: Felipe 24 · Aayushi 21 · Harish 21.
⚠️ **2 need classification** — null `Project Type`, both created today. This is the recurring new-submission pattern, not backlog rot: **100% of today's real top-level arrivals are unclassified.**
🧹 **0 `[Event Name]` placeholder sub-tasks created this window** — the template automation's substitution bug did not fire today. (11 such rows still sit on the board from earlier runs.)

## Proposals — new top-level submissions

### SFDC CID - Reachdesk Alice Programs - Coffee Voucher  ·  *Project Type: null*
`1217159184786251` · created 2026-08-04 07:17Z · unassigned · 0 sub-tasks
- **Owner →** **Aayushi**  (rule 4: creator `Samantha Wilding` is on Aayushi's stakeholder list; requester inferred from `created_by`, `Requestor` is null)  ·  their load: 21 open
  - ⚠️ **Rule-3 near-match, flagged not applied.** "Alice" is on rule 3's integration list (Zoom / CVent / Alice → Harish), and rule 3 outranks rule 4. Not fired because this reads as a **campaign-ID request for** an Alice program, not integration work **on** Alice. If that reading is wrong, the owner is Harish. Easy human override.
- **Priority →** *cannot derive.* No `Project Type` ⇒ no SLA base clock. `Out of SLA/Rush` is null, no target date set.
- **Sub-tasks →** *cannot propose.* The tier lookup keys off `Project Type`. Name suggests `SFDC Campaign only (single)`, but per the skill's own rule the type is never inferred — classify first, then the tier is likely **none** (leaf type).
- **Writes this would need:** classify `Project Type` (Class A) · assign owner (Class B, notifies Aayushi)

## Secondary — new standalone sub-task requests
None to propose. Both new sub-tasks in this window were already assigned on creation (contract §4 — silent skip):
- `Atlanta Tech Week 2026 Pre-Event Email Build` → Aayushi · parent: *Atlanta Tech Week 2026 Pre Event Marketing Email* · MOPS-Status `Require Information/Description`
- `AI Advisor Mode - Email Build 02` → Aayushi · parent: *Secondary Emails Send for AI Advisor Mode* · MOPS-Status `Completed`

Neither is template-generated (no placeholder, no ≥3-sibling burst) — both are hand-created by Aayushi, so they're real work already owned.

## Excluded
- 1 test row — `[TEST RUN - MOps Automation] SFDC Campaigns (multiple) - DrupalCon Rotterdam` (ours, created by Forkan)
- 0 template-generated sub-tasks
- 0 blank orphans

## Blocked
- **`Source Subscription Notification - All Active customers`** (`1217151738800971`, created 06:22Z) — **needs classification.** Null `Project Type`, and `created_by` is **null** too, so rule 4 has nothing to match on either. No rule fires. Already carries `Priority = Urgent` and `Out of SLA/Rush = Yes`, so someone has triaged the urgency but not the type. **Recommend a human sets `Project Type` today** — it's flagged rush and nothing can route it.

## n8n boundary — daily QA read
- **No divergence to report.** Neither real arrival has a `Project Type`, so neither could have entered an n8n-covered branch — n8n gates on the submission's type, and there is nothing to gate on.
- **Worth noting:** `SFDC CID - Reachdesk Alice Programs - Coffee Voucher` looks like it *should* be `SFDC Campaign only (single)` — the one type n8n covers unconditionally. Because the requester left the type blank, **n8n silently skipped a ticket it would otherwise have named**. That's the practical cost of the unclassified problem: it doesn't just block this skill, it blocks the production automation too.
- n8n coverage remains **inferred from Project Type**, not read back from Asana — the two open questions (does n8n write back? does it hold SF write credentials?) are still unanswered.

## Limits in force this run
- **No Region field exists** on the board — rules 2 and 5 cannot fire. Not applicable today (nothing reached them), but it caps coverage at ~64% in general.
- **`created_by` is a proxy for the stakeholder**, not the stakeholder itself. The one owner proposal here rests entirely on that inference.
- **Priority derivation is uncalibrated** — no proposal made today, so nothing to compare.
