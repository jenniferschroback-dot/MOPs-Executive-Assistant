# Lead Routing Audit — reference

Everything here was verified against the live Acquia org (read-only) on 2026-08-03. Where a claim is inferred rather than observed, it says so. Re-verify the rule table on any run that reports an unexpected class of divergence — it changes without notice.

> ## ⚠️ The rule table changed on or before 2026-08-06 — read §0 first
>
> **47 active rules, not 43.** Three of §6's proposed fixes were applied and 14 rules are new. Several §6 findings and the §8 calibration are stale. §0 records exactly what moved; the rest of this file is unrevised as of 2026-08-03 except where §0 overrides it.

---

## 0. Table revision, discovered 2026-08-06

Found by the `/lead-routing-audit yesterday` run for report day 2026-08-05. `Order_by__c` values active: 30, 45, 46, 47, 48, **51**, **53**, **57**, **59**, **61**, 70, 90, 130, 450, 520, 525, 601, **715**, **719**, 780, 783, **901**, 950, 1001, 1401, 1550, 1580, 1585, 1590, **1711**, **1715**, **1719**, 1722, 1765, 1766, 1780, 1790, 1800, 1812, 1819, 1825, 1850, **1860**, **1880**, 1890, **2000**, 2200. Order **52 no longer exists**.

### Fixes applied in Salesforce

| Order(s) | Change | Consequence for this skill |
|---|---|---|
| **1401**, **2200** | `Country__c` cleared (was `DK;GB;IE;NO;SE`) | **The continental-EMEA/MEA coverage gap in §6 check 4 is CLOSED.** Verified 2026-08-05: NL, DE, NO, JE leads all reached EMEA rules. `RECOMPUTE_NO_RULE` was 0/20 — its first zero. The §8 baseline of 7.5% is obsolete on the high side. |
| **525** | `Account_Type__c` cleared | §6 checks 3 and 5 resolved for this rule. It is now **broader** than order 46 and reachable — do not treat it as shadowed. |
| **70** | `Account_Team_Role__c` → `Web Governance Sales Advisor` | §6 check 10 resolved for 70; confirmed live that ATM rows with this role exist. **Twin 1722 was NOT fixed** — half-applied. |

### Still open from §6
Orders 46 / 525 / 1590 / 1800 (deactivated users) · 783 (`Region__c = 'Americas'`) · 601 / 1825 (LATAM `Account_Type__c`) · 1850 (duplicate of 1812) · 1722 (Monsido ATM role) · 950 / 1890 (Corey Black region) · 1819 (name) · 16 RR rules with null `Queue_Name__c`. Check 12 grew to **3** hits — the new ATM rules 51 and 53 copied order 48's inert `User__c`.

### Remaining coverage gaps (§6 check 4, narrowed)
1. **Non-listed APJ countries** — `Region__c = 'APJ'` with a country outside `JP` (520/1800) and the 13-code rest-of-APJ list (525/1819): CN, KR, KG, UZ, TW, HK, IN … → no match, falls to 1001 / 2000.
2. **LATAM no-account** — carries `Region__c = 'Americas'`, so it matches the AMER rules; 601/1825 can never fire, so it reaches NA BDRs instead of Livia Russo.

### New engine mechanic — an inactive-user rule leaves the lead UNOWNED

Verified on `00QPb00001mehOfMAI` (2026-08-05, rule 46 → Yoshi Han, deactivated). `LeadHistory` at 07:07:29Z holds **three Owner writes in the same second**, all by `B2BMA Integration`: `Leads - Marketing Queue` → `Yoshi Han` → `Leads - Marketing Queue`.

- Router-set owner extraction (§3) still works — take the **first** qualifying row (`Yoshi Han`), not the last.
- The lead lands back in the pre-routing queue as an unowned MQL → **bucket 3**, flags `MQL_STUCK_PRE_ROUTING;RULE_USER_INACTIVE`. Bucket 3 is evaluated before bucket 2, so it wins even though `RULE_USER_INACTIVE` normally drives bucket 2.
- **`Assigned_Inactive_User__c` stays `false`** because the final owner is a queue. The §8 figure of 14,636 leads therefore *undercounts* inactive-user damage, and this failure mode is invisible to that tripwire. A real count needs a `LeadHistory` scan for same-second owner→queue reversions.
- The account was also owned by the same dead user, so rule 90 would have failed identically. There was no path to a live human.

### Also confirmed on 2026-08-05
- Rules **57** (per-region CC ATM) and **719** (T1 EMEA Source RR) both routed correctly — 719 caught a Jersey/EMEA/Source no-account lead that under the old table would have fallen to the AMER pool.
- All RR pool assertions passed: Marco Chavarria `BDR NA` ×2, Grace Safadi `BDR EMEA` ×2, **Francesca Bravo `BDR EMEA`** (verified live, previously only assumed on the Reference tab).
- **`LeadHistory` explicit-Id batching returned 50 rows instantly** for 11 lead Ids. Prefer it over the semi-join, which §2 records as "verified instant" but which timed out on 2026-08-03.
- The **internal-Acquia unrouted-MQL class** recurred (`00Q6g00000Sr5MREAZ`, Jennifer Griffin Smith). `Account.Type = 'Internal Customer'` appears in **no** rule's `Account_Type__c`, so every Matched rule is excluded. Fix belongs upstream in scoring, not in routing.

---

## 1. The routing engine, as actually configured

Standard Salesforce lead assignment is **not in use**: `SELECT Id, Name, Active FROM AssignmentRule WHERE SobjectType='Lead'` → **0 rows**. Routing is a custom Flow/Apex engine driven by the `Lead_Routing_Rule__c` table.

**`Lead_Routing_Rule__c` — 24 fields total, 76 rows, 43 active.** The fields that matter:

| Field | Type | Role |
|---|---|---|
| `Name` | text | descriptive (`T1 Matched - AMER - Source - RR AMER BDR`); the RR pool token is parsed from the tail |
| `Active__c` | boolean | 43 true |
| `Order_by__c` | `Number(5,0) (Unique)` | precedence, 30 → 2200; **platform-unique, so ties are impossible** |
| `Tier__c` | picklist | `Tier 1` / `Tier 2`; **non-null on all 43** — never a wildcard |
| `Region__c` | picklist | `Americas` / `APJ` / `EMEA` + `* Non Targeted Countries`; null = wildcard |
| `Country__c` | multipicklist | `;`-joined ISO codes; null = wildcard |
| `Product__c` | picklist | null = wildcard (covers both "All Products" and "All Other Products") |
| `Account_Type__c` | multipicklist | `Customer;Former Customer;Partner;Partner Customer;Prospect`; null = wildcard |
| `Route_to__c` | picklist, required | `Direct To User` / `Queue` / `Account Owner` / `Account Team Member` / `Round Robin` |
| `User__c` | lookup → User | target for `Direct To User` |
| `Queue_Name__c` | text | **null on all 43 active rules** — no active rule uses `Route_to__c = 'Queue'` |
| `Account_Team_Role__c` | picklist | matches `AccountTeamMember.TeamMemberRole` exactly |

**`Route_to__c` distribution.** Across the 43 active rules: Round Robin 16, Direct To User 15, Account Team Member 7, Account Owner 5. Across 225 rule-stamped MQLs (30d): **Round Robin 143 (64%)**, Account Owner 64 (28%), Direct To User 17 (8%), Account Team Member 1.

That 64% is why the RR limitation dominates the audit's honest coverage.

### There is no "Matched" field
The full `FieldDefinition` dump is 24 fields — no `Matched__c`, no equivalent. The Matched/Unmatched grid is produced by `Account_Type__c` + ordering:

- `Account_Type__c` non-null ⇒ the lead must carry an account whose `Account.Type` is in the set ("Matched").
- `Account_Type__c` null ⇒ no account requirement ("Unmatched" catch-all).
- Matched rules sit at lower `Order_by__c` than their Unmatched twins, so first-match-wins yields the grid for free.
- `Account_Type__c` listing **all five** values (orders 45, 46, 1580, 1590) means "has any account" — it is *not* a wildcard.

**Assumption, evidence-backed but undocumented** — confirm with the routing owner.

### Country/region semantics
`Country__c` serves two roles, needing no special-casing:
- **narrowing within a region** — `JP` inside APJ, `DK;GB;IE;NO;SE` inside EMEA
- **region substitute** — the LATAM rules carry `Region__c = null` plus a 24-code set

**Do not build a country→region map, and do not derive one from the rule table.** The table's `Country__c` sets cover only ~45 codes total. Sampled leads carry `OM`, `DZ`, `KR`, `UZ`, `IT`, `ID` — most appear in no rule set, yet all have a populated lead-side `Region__c`. Read `Lead.Region__c` per-lead in the `SELECT` and use `CountryCode` only for aggregation.

---

## 2. Field lists for the query plan

Copy these verbatim. **Never** call `getObjectSchema('Lead')` (300 fields / 132KB — blew the token cap). Use `FieldDefinition` for lookups:
`SELECT QualifiedApiName, DataType FROM FieldDefinition WHERE EntityDefinition.QualifiedApiName='Lead' AND QualifiedApiName LIKE '%Rout%'` (~1KB).

**Q1 — active rules (43 rows)**
```
SELECT Id, Name, Order_by__c, Tier__c, Region__c, Country__c, Product__c, Account_Type__c,
       Route_to__c, User__c, User__r.Name, User__r.IsActive, User__r.UserRole.Name,
       Account_Team_Role__c, Queue_Name__c
FROM Lead_Routing_Rule__c WHERE Active__c = true ORDER BY Order_by__c
```

**Q2 / Q3 — the day's leads (~11–15 after union by Id)**
Two calls, same field list, differing only in the date predicate: `Lead_Routed_Date__c >= <dayStart> AND < <dayEnd>` then `Marketing_Qualified_Date__c >= <dayStart> AND < <dayEnd>`. Use explicit datetime literals, never `TODAY`.
```
SELECT Id, Name, Company, Status, CountryCode, Country, StateCode, Region__c,
       Marketing_Tier__c, MQL_Reason__c, MQL_Ready__c, Marketing_Qualified_Date__c,
       Product_Cloud_Interest__c, Product_Interest__c, LeadSource, Latest_Lead_Source_Type__c,
       Account__c, Sales_Territory__c, CreatedDate,
       Lead_Routed_Date__c, Lead_Routing_Rule__c, Lead_Routing_Rule__r.Name,
       Lead_Routing_Rule__r.Order_by__c, Lead_Routing_Rule__r.Route_to__c,
       Lead_Routing_Rule__r.Account_Team_Role__c,
       OwnerId, Owner.Name, Owner.Type, Assigned_Inactive_User__c, Lead_Owner_Manager__c,
       Disqualified_Reason__c
FROM Lead WHERE <date predicate>
```

**Q4 — accounts (≤10)**
`SELECT Id, Type, OwnerId, Owner.Name, Owner.IsActive, Owner.UserRole.Name FROM Account WHERE Id IN (<lead.Account__c values>)`

**Q5 — account team members (≤30, only if an ATM rule is expected)**
`SELECT AccountId, UserId, User.Name, User.IsActive, User.UserRole.Name, TeamMemberRole FROM AccountTeamMember WHERE AccountId IN (…) AND TeamMemberRole IN ('DAM Sales Advisor','Monsido Account Executive','Partner Manager','AE Source')`

**Q6 — lead history (~40–60). Semi-join; verified instant.**
```
SELECT LeadId, Field, OldValue, NewValue, CreatedDate, CreatedBy.Name
FROM LeadHistory
WHERE Field IN ('Owner','Status') AND LeadId IN (SELECT Id FROM Lead WHERE <date predicate>)
ORDER BY LeadId, CreatedDate
```
One call per scope predicate. **Documented fallback** if the semi-join ever regresses: manual batching, `LeadId IN (<≤50 literal ids>)`. Never filter by `CreatedDate` — reconfirmed timeout at `LAST_N_DAYS:1 LIMIT 10`.

**Q7 — unresolved owners (≤15)**
`SELECT Id, Name, IsActive, UserRole.Name FROM User WHERE Id IN (<router-set owner ids not already resolved>)`

**Q8 — linter (weekly, ~91 rows)** — all 76 rules (drop `Active__c = true`), plus `SELECT Id, Name, IsActive, UserRole.Name FROM User WHERE Id IN (<all rule User__c ids>)`.

### Never do these (each is a verified failure)
| Don't | Why |
|---|---|
| `getObjectSchema('Lead')` | 300 fields / 132KB, exceeds the tool's token cap |
| Filter `LeadHistory` by `CreatedDate` | Times out even at `LAST_N_DAYS:1 LIMIT 10` |
| `GROUP BY` / `WHERE` on `Region__c`, `AVP_Geography__c`, `Business_Segment__c`, `Account_Id__c` | 1300-char formula fields — `"field 'Region__c' can not be grouped in a query call"` |
| Use `Routing_MQL_Age__c` / `MQL_Age__c` | 0 on most routed leads, 4.26 / 14.57 on others with no pattern |
| Use `Account_Id__c` as the account key | Text-formula mirror of `Account__c`; not filterable. Use the lookup |
| `listRecentSobjectRecords` on Lead | Unbounded |

---

## 3. Router-set owner extraction

The single most breakable piece of the audit. Current `OwnerId` is **not** the routing outcome — 8 of 40 sampled routed leads had since moved to `Leads - DQ/Recycle Queue`.

**Match on timing first, actor second.** The primary signal is the `Field='Owner'` row whose `CreatedDate` is nearest `Lead_Routed_Date__c` (within ~60s). The routing Flow runs in **whatever user context triggered the MQL**, so `CreatedBy` is not reliably the integration user:

| Signal | Use |
|---|---|
| Earliest `Field='Owner'` row with `\|CreatedDate − Lead_Routed_Date__c\| ≤ 60s` | **Primary** — this is the routing stamp whoever wrote it |
| That row also has `CreatedBy.Name = 'B2BMA Integration'` AND `OldValue = '00G6g000003ESwpEAG'` (`Leads - Marketing Queue`) | **Confirmatory** — high confidence when present |
| No `Field='Owner'` rows at all | Fall back to current `OwnerId` — the owner was set at record creation, which logs no history |

> **Verified counter-example — why the actor test alone fails.** Lead `00QPb00001j5qI6MAI` (2026-07-31) was routed at 19:24:39 by rule 90 to the account owner. Its routing stamp was written by **Jill Krueger** (`Marketing Manager`), not `B2BMA Integration`, and its `OldValue` was `Jake Athey`, not the Marketing Queue — the lead had been sitting with the bulk-ingest catch-all owner and she triggered the MQL. The old fingerprint matches **zero** rows here, and the "use current `OwnerId`" fallback returns Jake Athey, who a human moved it to 14 seconds *after* routing. That reads as `OWNER_MISMATCH` / High and blames the router for a human's decision. Timing-first returns José Alberto Delgado Chaves — the correct routing outcome — and the reassignment is then correctly caught as a separate Medium signal.

Take the **earliest** qualifying row's `NewValue`. Owner changes are **double-logged** — one row with display names, one with 18-char Ids:
```
Field=Owner  OldValue="Andrew Brayton"      NewValue="Leads - DQ/Recycle Queue"
Field=Owner  OldValue="005Pb00000PwgWlIAJ"  NewValue="00GPb00000SvkE5MAJ"
```
Dedupe by keeping the row whose values match `^(005|00G)[A-Za-z0-9]{12,15}$`; use the names row for display. The `005` / `00G` prefix distinguishes user from queue (`Owner.Type` also works on the Lead itself).

Automation vs. human is clean from `CreatedBy.Name` **for post-routing reassignments**: `B2BMA Integration`, `MA Qualified Integration`, `MA Saleswings Integration` are automation; anything else is a person. Do **not** reuse that test to identify the routing stamp itself (see the counter-example above) — a person can trigger the routing Flow.

---

## 4. Reassignment window — 4 business hours

Exclude from the `REASSIGNED` signal:
1. Reassignment to `Leads - DQ/Recycle Queue` where `Status = 'Disqualified'` and `Disqualified_Reason__c` is populated — **verified exoneration pattern**.
2. Reassignment performed by the routed-to rep themselves.
3. Anything by the three automation users.

**Verified case for the 4h choice:** lead `00QPb00001ldfhFMAQ` was routed correctly to the account owner (Andrew Brayton), who DQ'd it and queued it **19h later** with reason `False Interest Business Outreach`. A 24h or 48h window classifies that correct routing as a misroute.

---

## 5. `Detail` tab columns

Worst-first left-to-right, so the important fields are visible without scrolling.

```
Run_Date | Bucket | Severity | Lead_Id | Lead_URL | Lead_Name | Company | Country | Region |
Tier | MQL_Reason | Product | Account_Id | Account_Type | Rule_Fired | Route_Mode |
Router_Set_Owner | Owner_Role | Expected_Owner | Expected_Basis | Current_Owner | Status |
Flags | Action ‖ Reviewer_Note | Resolved
```

- **The skill writes only `A:X`.** `Reviewer_Note` and `Resolved` are human-owned and must survive every append.
- `Bucket` — sortable string: `1 Routed OK` / `2 Wrong rep` / `3 Not routed` / `4 Bypassed`. Conditional formatting on the column plus one saved filter view per bucket. Not emoji: they break sort and search.
- `Expected_Basis` — the *why*: `rule 30 → Account Owner (001Pb00001CJ2hqIAD)`.
- `Flags` — semicolon-joined codes: `OWNER_MISMATCH;RULE_USER_INACTIVE;REASSIGNED_2.1H`. Keeps the column count sane and stays searchable.
- `Action` — one imperative, populated only at High severity; blank on bucket 1.
- **Minimum set to act on a misroute:** `Lead_URL`, `Rule_Fired`, `Router_Set_Owner`, `Expected_Owner`, `Expected_Basis`, `Flags`. The rest is pattern fuel.

---

## 6. Config linter — 12 checks

Fix-once config bugs, not daily lead rows. Each output row: `Order_by`, rule `Name`, check id, severity, the exact contradictory values, `First_Seen`, proposed fix. **No SF writes** — spec plus handoff (§9).

Findings below are from a **full run against all 43 active + 33 inactive rules on 2026-08-03**, not estimates.

| # | Check | Confirmed hits (2026-08-03) | Severity |
|---|---|---|---|
| 1 | Rule → deactivated `User__c` | **4** — orders 46, 525, 1590 (Yoshi Han), 1800 (Noriyuki Ishii) | High |
| 2 | Region token in `Name` contradicts `Region__c` | order **783** (`…EMEA - CC…` with `Region__c='Americas'`). Exclude the LATAM rules (47/601/1585/1825) — `Region__c` null + a country set is the deliberate region-substitute pattern, not a contradiction | High |
| 3 | Fully shadowed / unreachable — a lower-`Order_by__c` rule whose criteria are a **superset on all 5 dimensions** | **5** — orders **525** (≡46), **601** (≡47), **783** (≡780), **1825** (≡1585), **1850** (≡1812). All five are criteria-*identical* to their twin, not merely subsumed | High |
| 4 | Coverage gap — evaluate a synthetic lead per (Tier × Region × no account × null product); report combos matching no active rule | **T1 APJ non-Japan**, **T1 continental EMEA/MEA**, **T2 continental EMEA/MEA** match **no active rule** | High — the root cause (see below) |
| 5 | "Unmatched" rule that requires an account (`Name` contains `Unmatched` AND `Account_Type__c` non-null → can only fire for a lead that *has* an account) | **3** — orders **525, 601, 1825**. Same three as check 3; one root cause | High |
| 6 | RR rule with no resolvable pool (`Route_to__c='Round Robin'` AND `Queue_Name__c` null) | all **16** — report as **one** systemic finding, never 16 rows | Info + tripwire |
| 7 | `Route_to__c='Queue'` with null/unresolvable `Queue_Name__c` | 0 | Tripwire |
| 8 | Direct-To-User whose `User__r.UserRole.Name` contradicts the rule's region | **2** — order **950** (`Region__c='EMEA'` → Corey Black, `AE Americas Monsido`; confirmed firing) and order **1890** (`All Regions` → the same Americas-only rep). **Not** 1819 — Keith Pettinger's `SVP APJ/AMC` role *does* match its APJ region | Medium — "confirm intentional" |
| 9 | Rule `Name` names a person who isn't `User__c`'s user | **1** — order **1819** (`…Rest of APJ - Yoshi` → Keith Pettinger). Order 1800 (`Nori` → Noriyuki Ishii) is a correct short form, not a hit | Medium |
| 10 | `Account_Team_Role__c` with no matching `AccountTeamMember.TeamMemberRole` (groupable — safe to aggregate) | **2 of 7** — orders **70** and **1722** both use `'Monsido Account Executive'`, which matches **zero** rows org-wide. Found 2026-08-03; an earlier pass wrongly recorded 0 | **High** |
| 11 | `Tier__c` null on an active rule (unreachable by tier matching) | 0 | Tripwire |
| 12 | Non-`Direct To User` rule with `User__c` populated — the value is inert but reads as the routing target | **1** — order **48** (`Route_to__c='Account Team Member'`, `User__c` = Scott Delea). Also cosmetic: order 950 `Name` misspells "Monside" | Low |

**Dropped:** duplicate `Order_by__c`. Platform-impossible — the field is `Number(5,0) (Unique)`. Check #3 replaces it with duplicate *criteria* detection, which found five real hits.

### The dead Monsido ATM rules (check 10)

The 16 `TeamMemberRole` values that actually exist, with row counts, are: `DAM Sales Advisor` 122,590 · `Web Governance Sales Advisor` 122,569 · `BDR` 90,854 · `DAM BDR` 86,823 · `Account Executive` 59,317 · `AE Source` 15,139 · `Account Manager` 8,111 · `Partner Manager` 5,562 · `Renewal Manager` 2,408 · `Presales Engineer` 2,010 · `Customer Success Manager` 1,567 · `TAM` 841 · `Expansion Account Executive` 354 · `Customer Value Manager` 28 · `Secondary Presales Engineer` 5 · `Professional Services` 2.

`Monsido Account Executive` is **not among them**, so rules **70** (`T1 Matched - All Regions - Monsido - Monsido ATM`) and **1722** (the T2 twin) can never route anyone. Monsido leads that carry an account therefore fall through to order **90** / **1765** and land on the generic account owner instead of a Monsido AE — 6 leads in the 30-day cohort.

Almost certainly a rebrand casualty: Monsido became Acquia Web Governance, and `Web Governance Sales Advisor` exists at essentially 1:1 with `DAM Sales Advisor`. **Proposed fix:** set `Account_Team_Role__c = 'Web Governance Sales Advisor'` on both. Confirm the intended role with the routing owner first — it is a one-field change either way.

### Root cause — checks 3, 4 and 5 are one bug

The rule table has **two generations**. The inactive 33 are name-prefixed `1-` … `32-`; the active 43 are unprefixed. The old generation contained **proper region catch-alls with `Country__c` = null (wildcard)**:

| Inactive rule | Criteria | Routed to |
|---|---|---|
| order 1500 `22- T1 Unmatched - EMEA - All Other Products` | T1 · EMEA · **country wildcard** · any product · no account needed | Giuseppe Quevedo |
| order 1700 `24- T1 Unmatched - All Other Products - Rest APJ - Yoshi` | T1 · APJ · **country wildcard** · any product · no account needed | Yoshi Han |

Their replacements in the active generation **lost the wildcard**:
- `1401` / `2200` (EMEA all-other-products) are restricted to `Country__c = DK;GB;IE;NO;SE`.
- `525` (APJ rest-of-APJ) kept `Account_Type__c` = all five values, so it only fires for leads that **have** an account — which is exactly the opposite of what an "Unmatched" rule is for. Same copy-paste error on `601` and `1825` (LATAM).

So a truly unmatched (no-account) lead in continental EMEA, MEA, non-Japan APJ, or LATAM **matches no active rule at all** and falls through to order **1001 `T1 Unmatched - AMER - All Other Products - RR AMER BDRs`**.

**Confirmed in live data** — no-account leads stamped with the AMER fallback rule 1001:

| Lead | Country | `Region__c` | Product | Rule fired | Ended up with |
|---|---|---|---|---|---|
| `00QPb00001kFeRxMAK` | KR | APJ | Monsido | **1001 (AMER)** | Jacques Hefferan |
| `00QPb00001jQYTTMA4` | ID | APJ | DXP | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001jPlPYMA0` | ID | APJ | DXP | **1001 (AMER)** | Marco Chavarria (`BDR NA`) |
| `00QPb00001WwRRVMA3` | NZ | APJ | Content Cloud | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001WOch9MAD` | PH | APJ | DXP | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001gcCDxMAM` | AU | APJ | Content Cloud | **1001 (AMER)** | Maria Cespedes |
| `00QPb00001d0tojMAA` | BR | Americas | DXP | **1001 (AMER)** | DQ/Recycle |

LATAM is a subtler case: LATAM leads carry `Region__c = 'Americas'`, so they *do* match the AMER rules — meaning no-account Brazilian and Mexican prospects reach **NA** BDRs instead of Livia Russo (`BDR LatAm`), because 601 and 1825 can never fire.

**Proposed fix (spec only — no SF write):** clear `Account_Type__c` on 525, 601, 1825; clear `Country__c` on 1401 and 2200 (or add a continental-EMEA/MEA catch-all); set 783's `Region__c` to `EMEA`; repoint or deactivate the four inactive-user rules; delete 1850. Each is a one-field change a Salesforce admin applies directly.

**Do not report this as N individual daily misroutes.** It is one config defect with a large blast radius — the skill reports it once on the `Config Linter` tab and counts the affected leads as a class.

---

## 7. Queue reference

| Queue | Id | Open leads | Meaning |
|---|---|---|---|
| `Leads - Marketing Queue` | `00G6g000003ESwpEAG` | **1,252,779** | Pre-routing resting state + nurture pool. **Not** a failure signal unless an MQL is stuck here |
| `Leads - DQ/Recycle Queue` | `00GPb00000SvkE5MAJ` | 12,827 | Downstream DQ. Contaminates current `OwnerId` — 94 of 312 MQLs (30d) end here |
| `Leads - Not Targeted Countries` | `00G6g000003ESwqEAG` | 9,235 | By-design geographic exclusion |
| `Leads - Partner Deal Registration` | `00G6g000003ESwrEAG` | 23 | **Bucket 4** — bypasses the rule engine by design |
| `Leads - Rejected Partner Deals` | `00G6g000003ESxIEAW` | 114 | Partner path |
| `Leads - 72 hour lead re-route` | `00GPb00000RiAddMAF` | 10 | **Bucket 3** — SLA breach re-route |
| `Leads Queue - Team Member Not Found` | `00GPb00000Po3dqMAB` | 4 | **Bucket 3** — ATM rule found no matching member |
| `Marketing Leads Routing Fallback` | `00GPb00000QVSx7MAH` | 2 | **Bucket 3** — highest-precision routing failure |
| `Junk Leads` | `00G6g000003ESwoEAG` | 7 | — |
| ~30 `Leads - RR *` queues | — | — | Presumed round-robin pools; **the rule→pool link is not in the data** |

88 queues total. `Owner.Type` returns `User` / `Queue` directly — no need to parse Id prefixes on the Lead itself.

---

## 7b. Two engine behaviours the rule table does not express

Both were found by the Phase 0 backtest (2026-08-03, 226 rule-stamped MQLs over 30 days, collapsed into 99 distinct criteria groups). Together they accounted for **17% of leads** and, left unmodelled, produced a 23.9% false divergence rate — above the "systematic" threshold, i.e. enough noise to sink the sheet in week one.

**The recompute therefore returns a *chain* of acceptable rules, not a single expected rule.**

### 1. An Account Team Member rule fires only if a matching team member exists

When `Route_to__c = 'Account Team Member'` and the account has no `AccountTeamMember` with the rule's `Account_Team_Role__c`, the engine does **not** park the lead in `Leads Queue - Team Member Not Found` — it continues to the next matching rule. That queue holds only 4 leads org-wide, which is consistent with fall-through being the normal path.

So an ATM rule does not terminate the recompute: keep walking and collect every subsequent match until a deterministic (non-ATM) mode is reached. If the rule that actually fired is anywhere in that chain, it is **not** a misroute → flag `ATM_FALLTHROUGH`, bucket 1. **8.4%** of leads.

### 2. `Account__c` may have been linked after routing

`Account__c` is not history-tracked (0 `LeadHistory` rows across 30 days), so a link added after the fact is invisible. An "Unmatched" rule firing on a lead that *now* carries an account is the signature.

Test: recompute a second time with the account forced to null. If the fired rule appears in *that* chain, the lead almost certainly had no account at routing time → flag `ACCOUNT_LINKED_AFTER_ROUTING`, bucket 1. **8.8%** of leads.

### The proof that the five declarative dimensions are not sufficient

Two criteria groups in the cohort are **identical** on all five dimensions (`Tier 1` · `US`/Americas · `Content Cloud` · account `Prospect`) yet fired **different rules** — 780 for 6 leads, 90 for 4. No reading of the table can produce both. Something outside it (ATM state and account-link timing at the moment of routing, per the two behaviours above) decided the outcome.

**This is why a divergence is never reported as proof the engine is wrong** — only that the engine disagreed with a literal reading of the table. Say that plainly on any row that carries `RULE_DIVERGENCE`.

---

## 8. Baselines and volume

Verified 2026-08-03:

| Measure | Value |
|---|---|
| Leads created | 125 / 1d · 2,375 / 7d · 7,167 / 30d |
| **In audit scope** (routed OR MQL'd) | **~11 / day** — 46 routed + 28 unrouted MQLs per 7d |
| MQLs (`Marketing_Qualified_Date__c`) | 73 / 7d · 312 / 30d |
| MQLs with no rule and no routed date | 87 of 312 (30d) — **79 are Partner Deal Registration** (bucket 4); only **8** are genuine failures |
| MQLs now in DQ/Recycle | 94 of 312 (30d) — 30%. Arguably a bigger problem than routing |
| Open leads with `Assigned_Inactive_User__c` | **14,636** — backlog project, out of daily scope |

`Status` is a **snapshot, not a cohort** — only ~10 leads read "Marketing Qualified" at any moment because they move past it. Always cohort on `Marketing_Qualified_Date__c`.

One catch-all owner (Alex Campbell, role `Exec/Global/Admin`) owns 4,853 of the last 30 days' leads and **zero** MQLs — bulk ingest, not a rep. Bulk-ingest leads have neither an MQL date nor a routed date, so they fall out of scope naturally. They can still enter scope later: lead `00QPb00001j5qI6MAI` sat with Alex Campbell until a marketer MQL'd it, which is exactly the case that broke the old router-set-owner fingerprint (§3).

### Phase 0 calibration — measured baselines

30 days of rule-stamped MQLs, 226 leads / 99 distinct criteria groups. These rates are on the Sheet's `Reference` tab; the reviewers use them to know what normal looks like. **Re-run the calibration whenever the rule table changes.**

| Outcome | Rate | Count |
|---|---|---|
| Recompute agrees exactly | 70% | 158 / 226 |
| Agrees via `ATM_FALLTHROUGH` | 8% | 19 / 226 |
| Agrees once account-link timing is allowed for | 9% | 20 / 226 |
| **Accepted — no misroute implied** | **87%** | **197 / 226** |
| True rule divergence | 4% | 9 / 226 |
| `RECOMPUTE_NO_RULE` (the coverage gap) | 7.5% | 17 / 226 |
| Tier outside routing scope | 1.3% | 3 / 226 |

Sub-signal firing rates: `ACCOUNT_LINK_TIMING_UNKNOWN` 10.6% · `FIRED_RULE_REGION_MISMATCH` 9.7% · `ACCOUNT_LINKED_AFTER_ROUTING` 8.8% · `ATM_FALLTHROUGH` 8.4% · `RECOMPUTE_NO_RULE` 7.5% · `RULE_DIVERGENCE` 4.0% · `RULE_USER_INACTIVE` 1.8% · `TIER_OUT_OF_SCOPE` 1.3%.

**Every signal is now below the 20% systematic threshold**, so all of them are legitimate per-lead rows. Before the two §7b corrections, `RULE_DIVERGENCE` alone sat at 23.9% and would have had to be demoted to a Config Linter class.

`RECOMPUTE_NO_RULE` and `FIRED_RULE_REGION_MISMATCH` share one root cause — the coverage gap in linter check 4. They stay as per-lead rows (each is a real lead that reached the wrong region's BDR pool) but the row must point at the check rather than inviting a fresh diagnosis each morning.

---

## 9. Assumptions to confirm with the routing owner

Nothing here is documented anywhere; all of it is inferred from data. Confirm before treating a divergence as fact.

1. **Is order 1001 (`T1 Unmatched - AMER - All Other Products - RR AMER BDRs`) intentionally the global fallback**, or is the EMEA/APJ coverage gap a bug? *Biggest open question — it likely explains the original complaint.*
2. `Account_Type__c` non-null ⇒ requires a matched account — is that the real Matched/Unmatched mechanism?
3. Is `Lead.Region__c` (the formula field) what the engine compares to `rule.Region__c`? 5 of 8 sampled agree; 3 fell to the AMER catch-all.
4. Is `Product_Cloud_Interest__c` — not `Product_Interest__c` — the engine's product input?
5. `Marketing_Tier__c` `1`/`2` → `Tier 1`/`Tier 2`; are tiers 3 / 4 / `No Tier` / null outside routing entirely?
6. The RR pool → `UserRole` map (on the Sheet's Reference tab), seeded from observed pairings: RR AMER BDR → Maria Cespedes, Marco Chavarria (`BDR NA`); RR EMEA BDR → Francesca Bravo (`BDR EMEA`), Grace Safadi; RR DAM Team → Sam Schnepf (`DAM Sales Advisor`), Stephen Deshong (`DAM Sales Manager`).
7. Is the 4-business-hour reassignment window right for this team's rhythm?
8. Is `Leads - Partner Deal Registration` + `MQL_Reason__c = 'Partner - Deal Registration'` the complete bucket-4 definition?
9. Confirm the 14,636 inactive-owner leads stay out of daily scope.

---

## 10. Verified case leads (regression set)

**All four were hand-traced end-to-end on 2026-08-03 and the design produces the correct bucket for each.** Re-run these after any change to the recompute.

### `00QPb00001kS1PcMAK` — rule divergence with an identical owner → **Bucket 1**
Brian Trombley / Ariza Content Solutions · US · Americas · T1 · Content Cloud · account `001Pb00004JOGI2IAP` (`Type = Prospect`, owned by Sam Schnepf, `DAM Sales Advisor`).

- **Fired:** rule 780 `T1 Unmatched - AMER - CC - RR DAM Team` (Round Robin)
- **Router-set owner** = Sam Schnepf, confirmed from `LeadHistory` (`OldValue = 00G6g000003ESwpEAG`, `NewValue = 0056g000006R8f4AAC`, by `B2BMA Integration`). He moved it to `Sales Qualified` 18 minutes later.
- **Recompute chain** (T1 · Americas · Content Cloud · `Prospect` account) = **[52, 90]**. 780 is not in it, but the no-account chain **is** `[780]` → `ACCOUNT_LINKED_AFTER_ROUTING`.
- → **Bucket 1**, accepted. **No `RULE_DIVERGENCE`.**

> **Correction, 2026-08-03.** An earlier version of this file said rule 52 matched and resolved to Sam Schnepf "via ATM `DAM Sales Advisor`", making this a same-owner rule divergence. That was wrong on the mechanism: Sam Schnepf is the **account owner** whose `UserRole.Name` happens to be `DAM Sales Advisor` — account `001Pb00004JOGI2IAP` has **no `AccountTeamMember` row at all**. `UserRole.Name` and `AccountTeamMember.TeamMemberRole` are different fields and must never be conflated in the recompute. The bucket was right for the wrong reason; the flag was wrong. Keep this case in the regression set precisely because it punishes that confusion.

### `00QPb00001ldfhFMAQ` — correct routing, later DQ → **Bucket 1**
Alin Istrate / Spectrum Science · Romania · EMEA · T1 · DXP · account `001Pb00001CJ2hqIAD` (`Type = Prospect`, owned by Andrew Brayton, `AE NA Commercial`).

- **Fired:** rule 90 `T1 Matched - All Regions - All Other Products - Account Owner`. **Recompute agrees** — 30 needs `Customer`; 45/46/47 are country-restricted and `RO` is in none; 48/52/70 are product-specific. 90 is correct.
- **Expected owner** = `Account.OwnerId` = Andrew Brayton. **Router-set owner** = Andrew Brayton. **Match.**
- **19h 14m** later Andrew Brayton reassigned it to `Leads - DQ/Recycle Queue` with `Disqualified_Reason__c = 'False Interest Business Outreach'` (routed `2026-08-02T18:08:44Z`, DQ'd `2026-08-03T13:23:26Z`). All three reassignment exclusions fire (past 4h, self-reassignment, DQ carve-out) → **no** `REASSIGNED` flag.
- → **Bucket 1**, plus `ACCOUNT_OWNER_REGION_MISMATCH` as explanation (EMEA lead, NA AE — inherited from account ownership, not a routing defect).
- **Also the proof that current `OwnerId` lies:** current owner is a queue. Using it would have misfiled this as a routing failure.

### `00QPb00001l3aGLMAY` — genuine cross-region misroute → **Bucket 2**
Ismail Walid · Algeria (`DZ`) · **EMEA** · T1 · Drupal Cloud · **no account**.

- **Fired:** rule **1001** `T1 Unmatched - AMER - All Other Products - RR AMER BDRs` — an Americas rule on an EMEA lead. **Router-set owner** = Marco Chavarria (`BDR NA`).
- **Recompute:** no-account means every `Account_Type__c`-bearing rule is excluded; 1001 requires `Region__c='Americas'` ≠ EMEA; 1401 is EMEA but restricted to `DK;GB;IE;NO;SE` and `DZ` isn't in it. → **`expected_rule = NONE`**, expected owner = `Marketing Leads Routing Fallback`.
- Flags: `RECOMPUTE_NO_RULE` + `RR_POOL_MISMATCH` + `RULE_DIVERGENCE`. `RR_POOL_MISMATCH` is the driver.
- → **Bucket 2**, Medium — or folded into the calibrated region-divergence class per Phase 0. Never silently passed. This is the §6 coverage-gap defect showing up as a single lead.

### `00QPb00001lyXn7MAE` — never routed, yet owned and worked → **Bucket 3**
Augusta Oparaji / CG Life · US · Americas · T1 · Marketing Cloud · no account.

- `Lead_Routed_Date__c` **null**, `Lead_Routing_Rule__c` **null**, `MQL_Reason__c = 'Tier 1 - Contact Sales and Demo Forms'` (so not bucket 4).
- `Marketing_Qualified_Date__c` is **7 seconds before** `CreatedDate` — must not crash or clamp; the negative interval prints as-is.
- **No `Field='Owner'` history rows at all** — the owner was set at record creation, which logs no history. The router-set-owner **fallback path** (use current `OwnerId`) is exercised here and yields Marco Chavarria (`BDR NA`).
- → **Bucket 3**, flags `NO_ROUTED_DATE;NO_RULE_STAMPED`. One of the ~8 genuine monthly routing failures — and note it still got worked, so "didn't route" does not imply "was ignored."

### `00QPb00001j5qI6MAI` — correct routing, human override 14s later → **Bucket 2**, Medium
Blair Haden / Collector Systems · US · Americas · T1 · product **null** · account `001Pb000025qM3pIAE` (`Prospect`, owned by José Alberto Delgado Chaves, `BDR NA`). From the verification run of 2026-07-31.

- Owned by **Alex Campbell** (bulk ingest) until Jill Krueger (`Marketing Manager`) MQL'd it.
- **Fired:** rule 90 `Account Owner`. **Recompute chain** = `[90]` — exact agreement (product is null, so the product-specific rules 48/52 don't match).
- **Router-set owner** = José Alberto Delgado Chaves = `Account.OwnerId`. **Routing was correct.**
- 14 seconds later Jill Krueger reassigned it to Jake Athey (`VP DAM/Optimize`), who worked it to `Sales Qualified`. Not self-reassignment, not DQ, not automation → `REASSIGNED_0.0H` fires and drives the bucket at **Medium**.
- **This is the case that broke the old router-set-owner rule** (§3): the routing stamp was written by Jill Krueger, not `B2BMA Integration`, and `OldValue` was a user rather than the Marketing Queue.
- Correct handling is a *question*, not an accusation: "the router put this with the account owner and a human moved it immediately — deliberate override, or is rule 90 wrong for this account?" A Medium row exists to prompt that, not to assert a defect.

### Bonus: the reviewers appear in the data
`Lucio Silvestri` is a live SF user — `lucio.silvestri@acquia.com`, active, `UserRole = Exec/Global/Admin` — and is actively transitioning these very leads to `Sales Accepted` (seen on cases 3 and 4). Consistent with a Revenue Operations role; confirm the exact team name with him.
