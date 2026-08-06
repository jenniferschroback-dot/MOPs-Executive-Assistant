---
name: lead-routing-audit
description: Read-only daily QA on Salesforce lead routing — recomputes the expected owner from the live rule table and buckets each day's routed leads into routed-properly / wrong-rep / didn't-route / bypassed-by-design, then appends them to the shared routing QA Google Sheet. Use for a daily routing check, to investigate a misrouted lead, or to lint the routing rule table for config bugs. Also the single source for how Acquia's custom lead routing actually resolves.
argument-hint: [scope — e.g. "yesterday" | "today" | "2026-08-01" | "last 7 days" | a Lead Id | "linter only"]
---

# Lead Routing Audit

Acquia's lead routing is **100% custom** — standard Salesforce `AssignmentRule` for Lead returns zero rows. Routing lives in `Lead_Routing_Rule__c`, a declarative 76-row table (43 active), and **every routed lead is stamped with the rule that fired** (`Lead_Routing_Rule__c` + `Lead_Routed_Date__c`). That makes the expected owner independently recomputable, so this is a real audit rather than a heuristic.

Salesforce is **query-only** (no write path) — this skill finds misroutes and hands back exact fix values; a human applies them. It never claims to have fixed anything in SF (`.claude/rules/write-actions.md` §9). Its only write is appending rows to the QA Sheet.

Full recompute predicates, the 11 linter checks, and the verified evidence behind every claim are in `reference.md` — don't re-derive them from memory.

## Why this exists (confirmed defects, 2026-08-03)
- **Leads routing to the wrong reps** for weeks, with no detection except a rep noticing a bad lead and telling someone.
- **The root cause is a rule-migration bug, not random drift.** The old rule generation (the 33 now-inactive `1-`…`32-` rules) had proper region catch-alls with `Country__c` = wildcard. Their active replacements lost it: `1401`/`2200` are country-restricted to `DK;GB;IE;NO;SE`, and `525`/`601`/`1825` kept `Account_Type__c` populated so they only fire for leads that *have* an account — the opposite of an "Unmatched" rule's purpose. **No-account leads in continental EMEA, MEA, non-Japan APJ, and LATAM match no active rule** and fall through to order 1001 (`RR AMER BDRs`). Verified on 7 live leads: Korea, Indonesia ×2, New Zealand, Philippines, Australia and Brazil all stamped with the AMER rule.
- **5 active rules are unreachable** — 525, 601, 783, 1825, 1850 are each criteria-identical to a lower-`Order_by__c` twin.
- **4 active rules route to deactivated users** — Yoshi Han (orders 46, 525, 1590), Noriyuki Ishii (order 1800). **14,636 open leads** carry `Assigned_Inactive_User__c = true`.
- **Order 783 is tagged `Region__c = "Americas"` despite being the EMEA Content Cloud rule** — so EMEA CC leads never reach an EMEA rule.

Full evidence, the fix spec, and the two-generation history are in `reference.md` §6. These are **fix-once config bugs** — the skill reports them on the `Config Linter` tab and counts affected leads as a class, never as N individual daily misroutes.

## Inputs
- Scope from `$ARGUMENTS`. Default: **`yesterday`** — leads where `Lead_Routed_Date__c` OR `Marketing_Qualified_Date__c` falls in the report day. Use explicit datetime bounds in **Pacific** (`>= <day>T00:00:00-07:00 AND < <day+1>T00:00:00-07:00`), never `TODAY`/`YESTERDAY`, so a past day is re-runnable and the timezone is unambiguous. ~11 leads/day (verified: 46 routed + 28 unrouted MQLs per 7 days).
  - **`yesterday` is the right default because the scheduled run fires at 05:30 PT** — the current day is nearly empty at that hour, and a morning review needs the previous *complete* day. `Run_Date` is always the report day, never the execution day.
  - A day with genuinely zero leads in scope is still a `Daily Summary` row (`Leads_In_Scope = 0`, `Status = COMPLETE`). Writing the row is what keeps a missing `Run_Date` an unambiguous signal of a missed run.
- The rule table itself — read live every run (`Lead_Routing_Rule__c`), never cached in this folder. It changes without notice; a stale copy would silently invert the audit.
- The **RR pool → `UserRole` map**, bucket definitions, and calibration baselines: the Sheet's **`Reference` tab**. Human-owned so Lucio can correct it without a code change — read it, don't hardcode it here. Three pools exist across the 16 active RR rules (`RR AMER BDR` → `BDR NA`, `RR EMEA BDR` → `BDR EMEA`, `RR DAM Team` → `DAM Sales Advisor`/`DAM Sales Manager`); normalise a trailing `s` off the token.
- **Sheet:** `1gFu34xeJJ2Jrk1p9jtvgkzp_WtspHNyCGZ1zorbNNOA` — "Lead Routing QA — Daily (MOPs + RevOps)". Also in `routines/lead-routing-audit/README.md`.

## Buckets

Labels match how the team asks for them. **Evaluate 4 → 3 → 2 → 1, first match wins**, so each lead appears exactly once; sub-signals are columns, never extra rows. Assign `Severity` (High / Medium / Low) so the morning review triages instead of reading every row.

### 4 — Bypassed the rules by design
`MQL_Reason__c = 'Partner - Deal Registration'` OR owner is `Leads - Partner Deal Registration` (`00G6g000003ESwrEAG`). **High confidence** — 25 of 28 unrouted MQLs in 7 days. Evaluated first; this is what keeps bucket 3 a short high-signal list instead of ~91% false positives.

### 3 — Didn't route at all
Any of: `Lead_Routed_Date__c` null · `Lead_Routing_Rule__c` null · owner in a failure queue (`Marketing Leads Routing Fallback` `00GPb00000QVSx7MAH`, `Leads Queue - Team Member Not Found` `00GPb00000Po3dqMAB`, `Leads - 72 hour lead re-route` `00GPb00000RiAddMAF`) · an **MQL** still sitting in `Leads - Marketing Queue`. **High confidence** — direct field states, zero inference.

> `Leads - Marketing Queue` (`00G6g000003ESwpEAG`) holds **1,252,779** leads. It is the pre-routing resting state and the nurture pool — **not** a failure signal unless an MQL is stuck there.

**Severity within bucket 3** — "High confidence" above describes the *detection* (direct field states). The `Severity` column is a separate question: is anyone currently accountable for this lead?

| Condition | Flags | Severity | Why |
|---|---|---|---|
| MQL with `Lead_Routed_Date__c` **and** `Lead_Routing_Rule__c` both null | `NO_ROUTED_DATE;NO_RULE_STAMPED` | **High** | A qualified lead nobody owns — the worst outcome on the sheet |
| MQL still sitting in `Leads - Marketing Queue` | `MQL_STUCK_PRE_ROUTING` | **High** | Qualified but never left the pre-routing pool |
| Owner is `Marketing Leads Routing Fallback` | `QUEUE_ROUTING_FALLBACK` | **High** | The highest-precision routing failure that exists (only 2 leads org-wide) |
| Owner is `Leads Queue - Team Member Not Found` | `QUEUE_ATM_NOT_FOUND` | **High** | A terminal parking spot with 4 leads and no watcher. Rare *because* the engine normally falls through (§ recompute chain) — a lead landing here means fall-through didn't save it |
| Owner is `Leads - 72 hour lead re-route` | `QUEUE_72H_REROUTE` | **Medium** | This queue **is** the escalation path. The SLA breach already happened and the re-route mechanism is engaged, so it's tracked, not dropped |

Bucket 1 and bucket 4 rows are **always `Low`** — nothing is wrong with them, and the explanatory-only flags never escalate.

### 2 — Routed through the rules, wrong rep
Routed and rule-stamped, plus ≥1 sub-signal below. **Compare against the router-set owner, not current `OwnerId`** — this is the load-bearing mechanic. Current owner is contaminated: 8 of 40 sampled routed leads had since been DQ'd into `Leads - DQ/Recycle Queue` by the rep they were correctly routed to.

> **Router-set owner** = `NewValue` of the earliest `LeadHistory` row where `Field='Owner'` AND `CreatedBy.Name='B2BMA Integration'` AND `OldValue='00G6g000003ESwpEAG'`. Owner changes are **double-logged** (a display-names row and an 18-char-Id row) — dedupe by keeping the Id row, display from the names row. Fallbacks in order: current `OwnerId` if there were no post-routing Owner changes; else `OldValue` of the earliest post-routing change.

| Sub-signal | Flag | Precision | Role |
|---|---|---|---|
| Owner ≠ expected, deterministic modes | `OWNER_MISMATCH` | **High** | drives the bucket, High severity |
| Owner is an inactive user | `RULE_USER_INACTIVE` | **High** | drives the bucket, High severity |
| RR pool role inconsistent with the rule | `RR_POOL_MISMATCH` | Medium | drives the bucket, Medium severity |
| Human reassigned within **4 business hours** | `REASSIGNED_<n>H` | Medium | Medium severity |
| Rule fired ≠ recomputed rule | `RULE_DIVERGENCE` | **Low** | **column only — never the sole driver** |
| Owner's region ≠ lead's region, but routing was correct | `ACCOUNT_OWNER_REGION_MISMATCH` | n/a | **explanatory column only — stays in bucket 1** |
| ATM rule matched but no team member existed | `ATM_FALLTHROUGH` | n/a | **explanatory only — bucket 1** |
| Fired rule needs no account, lead now has one | `ACCOUNT_LINKED_AFTER_ROUTING` | n/a | **explanatory only — bucket 1** |
| No active rule matches the lead at all | `RECOMPUTE_NO_RULE` | **High** | drives the bucket, but **point the row at Config Linter check 4** — it's the known coverage gap, not a fresh diagnosis |
| Routing stamp not written by `B2BMA Integration` | `ROUTER_STAMP_NOT_B2BMA` | n/a | provenance note on the router-set owner |

**`Account Owner` mode inherits its correctness from account ownership** (28% of routed MQLs). A lead can be *correctly* routed and still land cross-region, because the account is owned cross-region. Verified: `00QPb00001ldfhFMAQ` is a Romanian (EMEA) lead correctly routed by rule 90 to the account's owner, Andrew Brayton (`AE NA Commercial`). Two reviewers will ask "why does a Romanian lead have a US AE?" every time — so put the answer on the row (`ACCOUNT_OWNER_REGION_MISMATCH` + `Expected_Basis`) rather than letting them re-derive it. It's an account-ownership question, not a routing defect, and must not be counted as a misroute.

- **Why rule-divergence is demoted:** lead `00QPb00001kS1PcMAK` fired rule 780 where the recompute says rule 52 — and **both resolve to the same person**. Rule-identity divergence with an identical owner is not a misroute.
- **Why the window is 4 business hours, not 24:** lead `00QPb00001ldfhFMAQ` was routed *correctly*, then DQ'd 19h later by that same rep. A 24h window calls that a misroute. Also exclude reassignments to DQ/Recycle carrying a `Disqualified_Reason__c`, self-reassignments, and anything by `B2BMA Integration` / `MA Qualified Integration` / `MA Saleswings Integration`.

### 1 — Routed properly
No sub-flag fired. **Keep these rows** — at n≈11 the denominators and the week-over-week trend are half the value.

## Expected-owner recompute

Null rule criteria = **wildcard** (confirmed: the "All Regions"/"All Products" rules are literally null). All five dimensions must match; **lowest `Order_by__c` wins**. `Order_by__c` is `Number(5,0) (Unique)` — platform-enforced, so ties are impossible and no tie-break heuristic is needed.

| Dimension | Predicate |
|---|---|
| Tier | `rule.Tier__c == tier(lead)` — never wildcard (non-null on all 43) |
| Region | `rule.Region__c` null OR `== lead.Region__c` |
| Country | `rule.Country__c` null OR `lead.CountryCode ∈ split(rule.Country__c, ';')` |
| Product | `rule.Product__c` null OR `== lead.Product_Cloud_Interest__c` |
| Account type | `rule.Account_Type__c` null OR (`lead.Account__c != null` AND `Account.Type ∈ split(rule.Account_Type__c, ';')`) |

**There is no `Matched__c` field.** The Matched/Unmatched grid *is* `Account_Type__c` — non-null means the lead must carry a matching account, and the Matched twins simply sit at lower `Order_by__c`, so first-match-wins produces the grid for free. The words in `Name` are descriptive only.

`tier(lead)`: `Marketing_Tier__c` `'1'→'Tier 1'`, `'2'→'Tier 2'`. Cross-check the `MQL_Reason__c` prefix and flag disagreement as `TIER_SIGNAL_CONFLICT`. Tiers 3 / 4 / `No Tier` / null match no rule.

No match → `expected_rule = NONE`, expected owner = `Marketing Leads Routing Fallback`. Then per `Route_to__c`:

| Mode | Expected owner | Confidence |
|---|---|---|
| `Direct To User` | `rule.User__c`; flag `RULE_USER_INACTIVE` if `User__r.IsActive = false` | High |
| `Account Owner` | `Account.OwnerId` of `lead.Account__c`; unresolvable if account null | High |
| `Account Team Member` | any `AccountTeamMember` on `lead.Account__c` with `TeamMemberRole = rule.Account_Team_Role__c`. Zero rows → the engine **falls through to the next matching rule** (it does *not* park the lead in Team-Member-Not-Found). Multiple rows → set-membership assertion | High for "in the set", not for "which one" |
| `Round Robin` | **pool-level only** — parse the trailing pool token from `rule.Name`, assert the router-set owner's `UserRole.Name` is in the mapped set (Reference tab) | Medium |
| `Queue` | unused — `Queue_Name__c` is null on all 43 active rules. Its appearance is a **linter error**, not a routing outcome | n/a |

**Never conflate `User.UserRole.Name` with `AccountTeamMember.TeamMemberRole`.** An account owner whose *role* is `DAM Sales Advisor` is not an ATM row with that `TeamMemberRole`. Confusing the two invents ATM matches that don't exist — see `reference.md` §10, first case.

**A rule whose `Account_Team_Role__c` matches no real `TeamMemberRole` can never route anyone** — skip it in the recompute and report it (currently orders 70 and 1722, both `'Monsido Account Executive'`, which exists nowhere in the org). `reference.md` §6.

### The recompute returns a chain, not one rule

Two engine behaviours aren't expressible in the table, and both were measured at ~8–9% of leads (`reference.md` §7b). Ignoring them put false divergence at 23.9% — enough noise to get the sheet ignored by week two.

1. **ATM fall-through** — an ATM rule doesn't terminate the walk; keep collecting matches until a deterministic mode. Fired rule anywhere in the chain → `ATM_FALLTHROUGH`, bucket 1.
2. **Account-link timing** — `Account__c` isn't history-tracked. Recompute again with the account forced null; if the fired rule is in *that* chain → `ACCOUNT_LINKED_AFTER_ROUTING`, bucket 1.

Two criteria groups identical on all five dimensions fired **different rules** in the same cohort — so a divergence means the engine disagreed with a literal reading of the table, never that the engine is provably wrong. Say so on the row.

## Query plan — 8 calls, timeout-safe

Q1 active rules (43) → Q2/Q3 the day's leads by each date predicate, union by Id (~11–15) → Q4 `Account WHERE Id IN (…)` → Q5 `AccountTeamMember` (only if an ATM rule is expected) → Q6 `LeadHistory` semi-join → Q7 `User WHERE Id IN (<unresolved owner ids>)` → Q8 linter (weekly). Exact field lists in `reference.md`.

**Hard rules — each one is a verified failure, not caution:**
- **Never** `getObjectSchema('Lead')` — 300 fields / 132KB, blows the token cap. Field lists are always explicit; use `FieldDefinition` for lookups (~1KB).
- **Never** filter `LeadHistory` by `CreatedDate` — times out even at `LAST_N_DAYS:1 LIMIT 10`. Always `LeadId`-scoped; the semi-join `LeadId IN (SELECT Id FROM Lead WHERE …)` returns instantly.
- **Never** `GROUP BY` or `WHERE` on `Region__c`, `AVP_Geography__c`, `Business_Segment__c`, `Account_Id__c` — 1300-char formula fields; confirmed error. `SELECT` them per-lead; aggregate on `CountryCode`.
- **Never** use `Routing_MQL_Age__c` / `MQL_Age__c` — 0 on most routed leads, non-zero with no pattern on others.
- Use `Account__c` (the real lookup, filterable), not its text-formula mirror `Account_Id__c`.
- **Never** `GROUP BY Lead_Routing_Rule__r.Order_by__c` — a number field is not groupable through a relationship (`"field 'Order_by__c' can not be grouped in a query call"`). Group by `Lead_Routing_Rule__r.Name` instead. Aggregates over a wide `CountryCode IN (…)` list across 90 days also time out — narrow the window or drop to a `LIMIT`ed detail query.
- **A lead can be stamped with a now-inactive rule.** Verified: `00QPb00001U8FBNMA3` carries order 1700, deactivated since. Load the rule by Id from the lead's own lookup for `Rule_Fired`, and only recompute against **currently active** rules — then label the row `RULE_NOW_INACTIVE` rather than calling it a mismatch. Only matters when auditing past days; today's leads route against today's table.

## Output — the Sheet (4 tabs)

1. **`Daily Summary`** — append, one row/day. What they read at 9am: date, leads in scope, count per bucket, count per severity, top divergence class, rows appended, `PARTIAL` marker. **Never a percentage without its denominator beside it.**
2. **`Detail`** — append-only, ~11 rows/day. Columns in `reference.md`. `Bucket` is a sortable string (`1 Routed OK` / `2 Wrong rep` / `3 Not routed` / `4 Bypassed`) — not emoji, which break sort and search. `Expected_Basis` carries the *why* (`rule 30 → Account Owner (001Pb…)`) so the standup never re-derives the logic. The skill writes **only `A:X`** — the last two columns (`Reviewer_Note`, `Resolved`) are human-owned and must survive every append.
3. **`Config Linter`** — *not* appended. A state snapshot of the rules, `values.update` to a fixed range, with `First_Seen` / `Last_Checked` so a bug open three weeks is obvious.
4. **`Reference`** — human-owned, read by this skill (see Inputs).

Also print a short digest to the caller — bucket counts, then the High-severity rows in full. A run whose only output is "rows appended" is unreviewable.

Any **file** deliverable (e.g. a config-audit fix spec handed to an SF admin, `Lead_Routing_Config_Audit_YYYY-MM-DD.md`) goes in `outputs/lead-routing/` — one subfolder per subject, never `outputs/` root. See `.claude/rules/output-files.md`.

## Writes — follow the contract
- **Sheet append** (`gws sheets values.append`) = **Class A** — reversible, nobody notified. Idempotency key = `Run_Date` + `Lead_Id`: read `Detail!A:D` first and skip existing keys; if today's date already exists on `Daily Summary`, the run is a **no-op** (contract §4).
- A deliberate re-run means delete-and-rewrite of that date's block. Because that can destroy reviewer annotations, it is **always confirmed, never automatic**.
- `valueInputOption=RAW` + `insertDataOption=INSERT_ROWS` so `00Q…` Ids and dates aren't coerced. Introspect exact request shapes with `gws schema sheets.spreadsheets.values.append` rather than guessing.
- **No Salesforce writes** — every finding is a spec plus a handoff, never a completion claim (§9). That includes the 4 broken rules: report them, don't claim them fixed.
- Log every run in `decisions/actions.md`, including `RESULT: skipped` no-ops (§7).

## Notes
- **Round robin is asserted at pool level only.** This report cannot say which rep was next in a rotation — no rotation-pointer object exists, `Queue_Name__c` is null on all 43 active rules, and the rule→pool link is not in the data. A passing RR row means "consistent with the pool," never "the correct rep." 64% of routed MQLs are RR, so this caveat covers most of the volume — state it, don't bury it.
- **The recompute reflects the declarative rule table only.** The engine is Flow/Apex this connector cannot see (no `Flow`, `ApexClass`, or `Territory2` exposed). Precedence is inferred from `Order_by__c`. A divergence means the engine disagreed with a literal reading of the table — it does not prove the engine is wrong.
- **Known systematic divergence:** the order-1001 AMER fallback (above). Until someone with Flow visibility confirms it, region divergences are reported as a **counted class with examples**, not as N individual misroutes. Re-check this whenever the rule table changes.
- **`Account__c` is not history-tracked** (0 `LeadHistory` rows for it across 30 days). When a lead hit an "Unmatched" rule while carrying an account, this cannot tell whether the link existed at routing time — those rows are Medium confidence and labelled so.
- **No speed/SLA metric is claimed.** If time-to-route is wanted it's `Lead_Routed_Date__c − Marketing_Qualified_Date__c` as a raw interval; one record has an MQL date **7 seconds before** `CreatedDate`, so negatives are real and print as-is.
- **No segment breakdown** — `Business_Segment__c` is `"Incomplete Data"` on most sampled MQLs and is an unGROUPable formula. It is in no bucket predicate.
- **Small-n discipline.** ~11 leads/day means one lead is ~9%. Percentages always carry the denominator. A failed query means the affected bucket reads `not measured — <query> failed` and the day is marked `PARTIAL` — a gap is reported as a gap, never estimated around.
- **The 14,636 inactive-owner leads stay out of daily scope** — that's a backlog cleanup project, not a daily QA signal. Flag only when it lands on one of today's leads.
- Pairs with `campaign-hygiene-audit` (same read-only-auditor shape, campaign side). Assumptions still needing confirmation from the routing owner are listed at the bottom of `reference.md` — check that list before treating a divergence as fact.
