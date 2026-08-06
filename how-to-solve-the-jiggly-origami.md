# Daily Lead-Routing QA — plan

## Context

Leads have been routing to the wrong reps for weeks, with no way to see it happening. Today the only detection is a rep noticing a bad lead in their queue and telling someone.

The ask: a daily agent that takes the day's form fills / MQLs and buckets them into **routed properly**, **routed through the rules but to the wrong rep**, and **didn't route at all** — landing in one live Google Sheet that Forkan and Lucio (RevOps) review each morning.

**This is buildable, and better than expected.** Recon against the live org (all read-only) found that Acquia's lead routing is **100% custom and fully declarative**: standard Salesforce `AssignmentRule` returns 0 rows, but `Lead_Routing_Rule__c` is a queryable 76-row table (43 active), and **every routed lead is stamped with the rule that fired** (`Lead_Routing_Rule__c` lookup + `Lead_Routed_Date__c`). So expected owner can be independently recomputed and diffed against actual — this is a real audit, not a heuristic.

Volume is small and very reviewable: **~11 leads/day** in scope (verified: 46 routed + 28 unrouted MQLs per 7 days).

### What is verifiable — and what is not

Being straight about this up front is the difference between a sheet they trust and one they stop opening.

| Claim | Status |
|---|---|
| Did the lead route at all | **Verifiable** — direct field states, zero inference |
| Did the *correct rule* fire | **Verifiable** — recompute all 5 criteria, lowest `Order_by__c` wins (`Order_by__c` is platform-unique, so no ties) |
| Correct owner for `Direct To User` / `Account Owner` / `Account Team Member` | **Verifiable** — deterministic (36% of routed MQLs) |
| Correct rep within a **Round Robin** pool | **NOT verifiable** — 64% of routed MQLs. No rotation-pointer object exists, `Queue_Name__c` is null on all 43 active rules, and the rule→pool link isn't in the data. We assert *pool/region consistency only* |
| A human reassigned it right after routing | **Verifiable** via `LeadHistory` — directional evidence, not proof |

### Three config bugs already found (fix-once, not daily noise)

- **4 active rules route to deactivated users** — Yoshi Han (orders 46, 525, 1590), Noriyuki Ishii (order 1800). **14,636 open leads** currently carry `Assigned_Inactive_User__c = true`.
- **Rule `T1 Unmatched - EMEA - CC - RR DAM Team` (order 783) has `Region__c = "Americas"`** — duplicates order 780, so EMEA Content Cloud leads never reach an EMEA rule. Order 783 is also fully shadowed by 780 (unreachable), as is order 1850 by 1812.
- **Coverage gap:** continental EMEA, MEA, and most of APJ have no all-other-products catch-all — the EMEA rules (1401/2200) are country-restricted to `DK;GB;IE;NO;SE`. This is almost certainly why **order 1001 (`RR AMER BDRs`) acts as the de-facto global fallback**: 3 of 40 sampled routed leads fired it while the lead's own region was APJ or EMEA.

That last one is the likely root cause of "leads routing to the wrong people," and it's a config fix, not a per-lead triage item.

---

## Deliverables

1. `.claude/skills/lead-routing-audit/SKILL.md` (~130 lines) — the skill
2. `.claude/skills/lead-routing-audit/reference.md` — full recompute predicate table + 11 linter checks
3. One Google Sheet, 4 tabs, appended daily
4. A **local** launchd schedule (per the chosen host — `gws` credentials are local-only)
5. `routines/lead-routing-audit/README.md` — documenting the local-host decision and the cloud-migration path
6. Registration + contract updates (below)

Model on `.claude/skills/campaign-hygiene-audit/SKILL.md` — same shape (read-only SF auditor → grouped findings → spec-plus-handoff, never a completion claim). Frontmatter is `name` / `description` / `argument-hint` only.

---

## Bucket logic

Labels match the ask; **evaluation order is 4 → 3 → 2 → 1, first match wins**, so each lead appears exactly once. Sub-signals are boolean columns, never extra rows.

**Scope per run:** leads where `Lead_Routed_Date__c` OR `Marketing_Qualified_Date__c` falls in the report day (explicit datetime bounds, not `TODAY`, so a past day can be re-run and the timezone is explicit).

### Bucket 4 — Bypassed the rules by design *(evaluated first so it can't leak into 1 or 2)*
`MQL_Reason__c = 'Partner - Deal Registration'` OR owner is `Leads - Partner Deal Registration` (`00G6g000003ESwrEAG`). **High confidence** — 25 of 28 unrouted MQLs in 7 days. Counted separately, which is what keeps bucket 3 a small high-signal list instead of ~91% false positives.

### Bucket 3 — Didn't route at all
Any of: `Lead_Routed_Date__c` null · `Lead_Routing_Rule__c` null · owner in a failure queue (`Marketing Leads Routing Fallback` `00GPb00000QVSx7MAH`, `Leads Queue - Team Member Not Found` `00GPb00000Po3dqMAB`, `Leads - 72 hour lead re-route` `00GPb00000RiAddMAF`) · an MQL still sitting in `Leads - Marketing Queue`. **High confidence** — direct field states.

> `Leads - Marketing Queue` holds **1,252,779** leads — it is the pre-routing resting state and nurture pool, **not** a failure signal unless an MQL is stuck there.

### Bucket 2 — Routed through the rules, wrong rep
Routed and rule-stamped, plus ≥1 sub-signal. **The load-bearing mechanic: compare against the *router-set* owner, not current `OwnerId`.** Current owner is contaminated — 8 of 40 sampled routed leads had since been DQ'd into `Leads - DQ/Recycle Queue` by the rep they were correctly routed to.

> **Router-set owner** = `NewValue` of the earliest `LeadHistory` row where `Field='Owner'` AND `CreatedBy.Name='B2BMA Integration'` AND `OldValue='00G6g000003ESwpEAG'`. Verified 3/3. Owner changes are **double-logged** (a names row and an 18-char-Id row) — dedupe by keeping the Id row, display from the names row.

| Sub-signal | Precision | Role |
|---|---|---|
| Owner ≠ expected, deterministic modes | **High** | drives the bucket |
| Owner is an inactive user | **High** | drives the bucket; links to the 4 broken rules |
| RR pool role inconsistency | Medium | drives the bucket, Medium severity |
| Human reassignment within **4 business hours** | Medium | Medium severity |
| Wrong rule fired vs. recompute | **Low** | column only, never the sole driver |

- **Why rule-divergence is demoted:** lead `00QPb00001kS1PcMAK` fired rule 780 where the recompute says rule 52 — **both resolve to the same person**. Rule-identity divergence with an identical owner is not a misroute.
- **Why N = 4 business hours, not 24:** lead `00QPb00001ldfhFMAQ` was routed *correctly*, then DQ'd 19h later by that same rep. A 24h window calls that a misroute. Also exclude reassignments to DQ/Recycle carrying a `Disqualified_Reason__c`, self-reassignments, and anything by `B2BMA Integration` / `MA Qualified Integration` / `MA Saleswings Integration`.

### Bucket 1 — Routed properly
No sub-flag fired. **Keep these rows** — at n≈11 the denominators and week-over-week trend are half the value.

Add a **`Severity`** column (High / Medium / Low) so the morning review triages instead of reading every row.

### Expected-owner recompute
Null rule criteria = **wildcard** (confirmed: the "All Regions"/"All Products" rules are literally null). All five must match; lowest `Order_by__c` wins.

| Dimension | Predicate |
|---|---|
| Tier | `rule.Tier__c == tier(lead)` — never wildcard (non-null on all 43) |
| Region | `rule.Region__c` null OR `== lead.Region__c` (read per-lead; **never** GROUP BY / WHERE it) |
| Country | `rule.Country__c` null OR `lead.CountryCode ∈ split(rule.Country__c, ';')` |
| Product | `rule.Product__c` null OR `== lead.Product_Cloud_Interest__c` |
| Account type | `rule.Account_Type__c` null OR (`lead.Account__c != null` AND `Account.Type ∈ split(...)`) |

**There is no `Matched__c` field** — the Matched/Unmatched grid *is* `Account_Type__c` (non-null ⇒ lead must carry a matching account), and the Matched twins simply sit at lower `Order_by__c`. Words in `Name` are descriptive only.

Then per `Route_to__c`: `Direct To User` → `rule.User__c` · `Account Owner` → `Account.OwnerId` · `Account Team Member` → any `AccountTeamMember` with the matching `TeamMemberRole` (zero rows ⇒ expect the Team-Member-Not-Found queue) · `Round Robin` → **pool-level assertion only**, comparing the owner's `UserRole.Name` against the pool token parsed from `rule.Name`. `Queue` is unused on all 43 active rules; its appearance is a linter error.

The RR pool→`UserRole` map (4 entries) is the **only** static table needed — **put it on the Sheet's Reference tab, not in the skill folder**, so Lucio can correct it without a code change.

---

## Sheet schema (4 tabs)

1. **`Daily Summary`** — append, one row/day. What they actually read at 9am: date, leads in scope, count per bucket, count per severity, top divergence class, rows appended, `PARTIAL` marker. Never a percentage without its denominator beside it.
2. **`Detail`** — append-only, ~11 rows/day (~230/month):
   `Run_Date | Bucket | Severity | Lead_Id | Lead_URL | Lead_Name | Company | Country | Region | Tier | MQL_Reason | Product | Account_Id | Account_Type | Rule_Fired | Route_Mode | Router_Set_Owner | Owner_Role | Expected_Owner | Expected_Basis | Current_Owner | Status | Flags | Action ‖ Reviewer_Note | Resolved`
   - `Bucket` as a sortable string (`1 Routed OK` / `2 Wrong rep` / `3 Not routed` / `4 Bypassed`) + conditional formatting and one saved filter view per bucket. Not emoji — they break sort and search.
   - `Expected_Basis` is the "why" column (`rule 30 → Account Owner (001Pb…)`) — it's what stops the standup re-deriving the logic.
   - `Flags` = semicolon-joined codes (`OWNER_MISMATCH;RULE_USER_INACTIVE;REASSIGNED_2.1H`).
   - **Minimum to act on a misroute:** `Lead_URL`, `Rule_Fired`, `Router_Set_Owner`, `Expected_Owner`, `Expected_Basis`, `Flags`.
   - Last two columns are **human-owned** — the skill writes only `A:X` so annotations survive every append.
3. **`Config Linter`** — *not* appended. A state snapshot of the 43 rules, written with `values.update` to a fixed range, with `First_Seen` / `Last_Checked` so a bug open three weeks is obvious.
4. **`Reference`** — human-owned and **read** by the skill: RR pool→role map, bucket definitions, calibration baselines, the confirm-with-Lucio list.

**Idempotency** (natural key = `Run_Date` + `Lead_Id`): read `Detail!A:D` first and skip existing keys; if today's date already exists on `Daily Summary`, the run is a no-op. A deliberate re-run is delete-and-rewrite of that date's block — and because it can destroy reviewer annotations, that is **always confirmed, never automatic**. Use `valueInputOption=RAW` + `insertDataOption=INSERT_ROWS` so `00Q…` Ids and dates aren't coerced. Introspect exact request shapes with `gws schema sheets.spreadsheets.values.append` rather than guessing.

I'll create the Sheet via `gws sheets spreadsheets create`; **Forkan shares it with Lucio manually** (a permission change isn't mine to make).

---

## Query plan — 8 calls, timeout-safe

| # | Query | Rows |
|---|---|---|
| Q1 | 43 active rules, 13 explicit fields incl. `User__r.IsActive`, `User__r.UserRole.Name` | 43 |
| Q2/Q3 | The day's leads by `Lead_Routed_Date__c` / `Marketing_Qualified_Date__c`, ~22 explicit fields; union by Id | ~11–15 |
| Q4 | `Account WHERE Id IN (…)` → `Type, OwnerId, Owner.Name, Owner.IsActive, Owner.UserRole.Name` | ≤10 |
| Q5 | `AccountTeamMember WHERE AccountId IN (…) AND TeamMemberRole IN (…)` — only if an ATM rule is expected | ≤30 |
| Q6 | `LeadHistory WHERE Field IN ('Owner','Status') AND LeadId IN (SELECT Id FROM Lead WHERE <date predicate>)` — **semi-join, verified instant** | ~40–60 |
| Q7 | `User WHERE Id IN (<router-set owner ids>)` for anything unresolved | ≤15 |
| Q8 | Linter (weekly): all 76 rules + their `User__c` users | ~91 |

**Hard rules for the skill:**
- **Never** `getObjectSchema('Lead')` — 300 fields / 132KB, blew the token cap. Field lists are always explicit; use `FieldDefinition` for lookups (~1KB).
- **Never** filter `LeadHistory` by `CreatedDate` — times out even at `LAST_N_DAYS:1 LIMIT 10`. Always `LeadId`-scoped.
- **Never** GROUP BY `Region__c`, `AVP_Geography__c`, `Business_Segment__c`, `Account_Id__c` — 1300-char formula fields, confirmed error. Aggregate on `CountryCode`.
- **Never** use `Routing_MQL_Age__c` / `MQL_Age__c` — 0 on most routed leads, non-zero with no pattern on others.

---

## Phase 0 — calibrate before the first append

Backtest the recompute across 30 days of rule-stamped MQLs (~225 leads). **Any sub-signal firing on >20% of leads with a systematic pattern is a config finding for the Linter tab, not per-lead rows** — the order-1001 region divergence will almost certainly trip this. Write the measured baseline rates onto the Reference tab so the reviewers know what normal looks like.

Skipping this is how the sheet ends up crying wolf on day one and getting ignored by week two.

---

## Prerequisites — the write contract

Google Sheets is **not in the `.claude/rules/write-actions.md` §1 verb registry at all**, and §6 authorizes unattended writes only for the weekly `#mops-team` post. Before the first automated run:

1. Add a **Google Sheets** block to §1 — `values.append` / `values.update` via `gws` = **Class A** (reversible, nobody notified).
2. Add a §4 idempotency row (the natural key above).
3. Add a §6 row authorizing the unattended append to **this sheet only**.
4. Log the decision in `decisions/log.md`; one `decisions/actions.md` line per run.
5. Update `tools/available-tools.md` — it predates the `gws` discovery and has no Sheets entry.

Until §6 is logged, the skill produces the rows for a human to paste.

**Salesforce stays query-only** — every row is a finding plus exact fix values, never a completion claim (§9).

## Scheduling

`gws` is authed against Forkan's local keyring, so this runs **locally**: a launchd agent invoking `claude -p "/lead-routing-audit"` each weekday morning. `StartCalendarInterval` re-fires a missed job once on wake, so a closed laptop delays rather than skips — but a multi-day gap will show as missing `Run_Date` rows, which the Daily Summary tab makes visible. The cloud-migration path (install `gws` in a routine env + a Google OAuth refresh token as a routine secret) goes in the routine README for when this proves out.

Register the skill in `CLAUDE.md` → `## Skills` → **Built:**, and the routine under `## Routines`.

---

## Verification

1. **Recompute correctness** — hand-check 10 leads across all four `Route_to__c` modes, including the verified cases: `00QPb00001kS1PcMAK` (rule divergence, same owner → must land bucket 1, `RULE_DIVERGENCE` flag only), `00QPb00001ldfhFMAQ` (correctly routed then DQ'd 19h later → bucket 1, **not** bucket 2), `00QPb00001l3aGLMAY` (Algeria → AMER BDR → bucket 2 or the calibrated region class).
2. **Router-set owner extraction** — confirm it reconstructs the routed owner on leads whose current owner is now DQ/Recycle. This is the single most breakable piece.
3. **Linter** — must independently rediscover all three known bugs (4 inactive-user rules, order 783's region contradiction, orders 783/1850 shadowed).
4. **Idempotency** — run twice for the same day; second run appends zero rows and leaves `Reviewer_Note` intact.
5. **Timeout safety** — every query returns; no full describe, no date-filtered `LeadHistory`.
6. **Human check** — run 3–5 days manually and walk the sheet with Lucio before enabling the schedule.

## Confirm with Lucio / the routing owner

1. Is **order 1001 intentionally the global fallback**, or is the EMEA/APJ coverage gap a bug? *(Biggest question — it likely explains the whole complaint.)*
2. `Account_Type__c` non-null ⇒ requires a matched account — is that the real Matched/Unmatched mechanism?
3. Is `Lead.Region__c` the field the engine compares, and `Product_Cloud_Interest__c` (not `Product_Interest__c`) the product input?
4. `Marketing_Tier__c` `1`/`2` → `Tier 1`/`Tier 2`; are tiers 3/4/No Tier outside routing entirely?
5. The RR pool→`UserRole` map, and N = 4 business hours.
6. Is `Leads - Partner Deal Registration` the complete bucket-4 definition?
7. Confirm the **14,636** inactive-owner leads stay **out** of daily scope — that's a backlog cleanup project, not a daily QA signal.
8. Who is Lucio, and what's the team name? ("Reve Hopkins" in the request reads like dictation — I've assumed RevOps.)

## Explicitly out of scope

- **Which rep was next in a Round Robin rotation** — unverifiable, and the skill will say so rather than imply it.
- **Any Salesforce write**, including fixing the 4 broken rules.
- **Time-to-route SLA metrics** — if wanted later, it's `Lead_Routed_Date__c − Marketing_Qualified_Date__c` shown as a raw interval; one record has an MQL date **7 seconds before** `CreatedDate`, so negatives are real and print as-is.
- **Segment breakdown** — `Business_Segment__c` is `"Incomplete Data"` on most sampled MQLs.
