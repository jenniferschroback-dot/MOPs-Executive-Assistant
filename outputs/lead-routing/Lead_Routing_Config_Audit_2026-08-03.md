# Lead Routing — Config Audit

**Run:** 2026-08-03 · **Scope:** all 76 `Lead_Routing_Rule__c` records (43 active, 33 inactive) · **Method:** read-only SOQL, no Salesforce writes

Every finding below is a **config defect in the rule table**, not a per-lead incident. Fixing these is a handful of one-field edits by a Salesforce admin; each would otherwise keep generating misroutes daily. Salesforce is query-only through this connector — this is a spec plus a handoff, never a completion claim.

---

## 🔴 The root cause — a rule-migration bug (checks 3, 4, 5 are one defect)

The table holds **two generations** of rules: the 33 inactive ones are name-prefixed `1-` … `32-`; the 43 active ones are unprefixed. The old generation had **proper region catch-alls with `Country__c` = null (wildcard)**:

| Inactive rule | Criteria | Routed to |
|---|---|---|
| order 1500 `22- T1 Unmatched - EMEA - All Other Products` | T1 · EMEA · **country wildcard** · any product · no account required | Giuseppe Quevedo |
| order 1700 `24- T1 Unmatched - All Other Products - Rest APJ - Yoshi` | T1 · APJ · **country wildcard** · any product · no account required | Yoshi Han |

The active replacements lost that wildcard:

- **`1401` and `2200`** (EMEA all-other-products) are restricted to `Country__c = DK;GB;IE;NO;SE`.
- **`525`, `601`, `1825`** kept `Account_Type__c` populated with all five values, so they only fire for leads that **have** an account — the exact opposite of what an "Unmatched" rule exists to do. A copy-paste from the Matched twin without clearing the field.

**Consequence:** a truly unmatched (no-account) lead in **continental EMEA, MEA, non-Japan APJ, or LATAM matches no active rule at all**, and falls through to order **1001 — `T1 Unmatched - AMER - All Other Products - RR AMER BDRs`**.

### Confirmed in live data
No-account leads stamped with the AMER fallback rule:

| Lead | Country | `Region__c` | Product | Rule fired | Ended up with |
|---|---|---|---|---|---|
| `00QPb00001kFeRxMAK` | KR | APJ | Monsido | **1001 (AMER)** | Jacques Hefferan |
| `00QPb00001jQYTTMA4` | ID | APJ | DXP | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001jPlPYMA0` | ID | APJ | DXP | **1001 (AMER)** | Marco Chavarria (`BDR NA`) |
| `00QPb00001WwRRVMA3` | NZ | APJ | Content Cloud | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001WOch9MAD` | PH | APJ | DXP | **1001 (AMER)** | DQ/Recycle |
| `00QPb00001gcCDxMAM` | AU | APJ | Content Cloud | **1001 (AMER)** | Maria Cespedes |
| `00QPb00001d0tojMAA` | BR | Americas | DXP | **1001 (AMER)** | DQ/Recycle |

**LATAM is a subtler variant.** LATAM leads carry `Region__c = 'Americas'`, so they *do* match the AMER rules. But because `601` and `1825` can never fire, no-account Brazilian and Mexican prospects reach **NA** BDRs instead of Livia Russo (`BDR LatAm`).

> **Caveat:** the routing engine itself is Flow/Apex, which this connector cannot read. That order 1001 acts as the de-facto global fallback is inferred from the stamps on these leads, not read from the engine. Worth confirming with whoever owns the Flow — but the coverage gap in the table is directly observable either way.

---

## 🔴 5 active rules are unreachable

Each is **criteria-identical** on all five dimensions to a lower-`Order_by__c` twin, so first-match-wins means it never fires.

| Unreachable | Shadowed by | Criteria | Same outcome? |
|---|---|---|---|
| **525** | 46 | T1 · APJ · rest-of-APJ codes · any product · all 5 account types | Yes — both → Yoshi Han |
| **601** | 47 | T1 · LATAM codes · any product · all 5 account types | Yes — both → Livia Russo |
| **783** | 780 | T1 · Americas · Content Cloud | Yes — both → RR |
| **1825** | 1585 | T2 · LATAM codes · any product · all 5 account types | Yes — both → Livia Russo |
| **1850** | 1812 | T2 · Americas · Acquia Source | Yes — both → RR |

`1850` is harmless redundancy. `525`, `601`, `783` and `1825` are not — they were each *meant* to cover a case that now has no coverage.

---

## 🔴 4 active rules route to deactivated users

| Order | Rule | Target | Status |
|---|---|---|---|
| 46 | `T1 Matched - All - All Other Products - Rest of APJ - Yoshi` | Yoshi Han | **inactive** |
| 525 | `T1 Unmatched - All - All Other Products - Rest of APJ - Yoshi` | Yoshi Han | **inactive** (also unreachable) |
| 1590 | `T2 Matched - All - All Other Products - Rest of APJ - Yoshi` | Yoshi Han | **inactive** |
| 1800 | `T2 Unmatched - APJ - All Other Products - Japan - Nori` | Noriyuki Ishii | **inactive** |

**14,636 open leads** org-wide carry `Assigned_Inactive_User__c = true`.

Live example: `00QPb00001WNA0XMAX` (Australia) fired rule 46 and is owned by Yoshi Han. Notably, five other rule-46 leads ended up with **Jacques Hefferan** — so something downstream is already catching some of these, which means the current owner alone won't tell you the rule is broken.

---

## 🟠 Region contradiction

**Order 783** — `T1 Unmatched - EMEA - CC - RR DAM Team` has `Region__c = "Americas"`. EMEA Content Cloud leads never reach an EMEA rule. This is also why it duplicates 780 exactly.

**Fix:** set `Region__c = 'EMEA'`.

---

## 🟡 Owner/region mismatches — confirm intentional

| Order | Rule region | Target | Target's role |
|---|---|---|---|
| 950 | **EMEA** | Corey Black | `AE Americas Monsido` — confirmed firing on a live Oman lead |
| 1890 | All Regions | Corey Black | `AE Americas Monsido` — a global rule pointing at an Americas-only rep |

May well be deliberate (a single Monsido specialist covering all regions) — flagging for confirmation, not asserting a bug.

---

## 🔴 2 Monsido rules point at a team-member role that does not exist

_Added 2026-08-03 by the Phase 0 calibration backtest._

Rules **70** (`T1 Matched - All Regions - Monsido - Monsido ATM`) and **1722** (its T2 twin) both set `Account_Team_Role__c = 'Monsido Account Executive'`. That value matches **zero** rows in `AccountTeamMember` org-wide, so **neither rule can ever route anyone.**

The 16 roles that actually exist: `DAM Sales Advisor` 122,590 · `Web Governance Sales Advisor` 122,569 · `BDR` 90,854 · `DAM BDR` 86,823 · `Account Executive` 59,317 · `AE Source` 15,139 · `Account Manager` 8,111 · `Partner Manager` 5,562 · `Renewal Manager` 2,408 · `Presales Engineer` 2,010 · `Customer Success Manager` 1,567 · `TAM` 841 · `Expansion Account Executive` 354 · `Customer Value Manager` 28 · `Secondary Presales Engineer` 5 · `Professional Services` 2.

**Effect:** a Monsido lead that *does* carry an account skips its product-specific rule and falls through to order **90** / **1765** — the generic account owner. It never reaches a Monsido specialist. 6 leads in the 30-day cohort.

Almost certainly a rebrand casualty — Monsido became Acquia Web Governance, and `Web Governance Sales Advisor` exists at essentially 1:1 with `DAM Sales Advisor`. Worth confirming the intended role before editing, but it's a one-field change either way.

---

## ⚪ Cosmetic / hygiene

- **Order 1819** — `T2 Unmatched - All - All Other Products - Rest of APJ - Yoshi` actually routes to **Keith Pettinger** (`SVP APJ/AMC`). The name is stale, and an SVP receiving inbound leads is worth a sanity check. Region itself is correct.
- **Order 48** — `Route_to__c = 'Account Team Member'` but `User__c` is set to Scott Delea. The value is inert; it reads as the routing target and will mislead the next person who audits this.
- **Order 950** — `Name` misspells "Monside".
- **16 of 16 Round Robin rules have `Queue_Name__c` = null.** The rule→pool link lives outside this table, so round-robin targets are not auditable from Salesforce data. Systemic, and the reason the daily audit can only assert RR *pool* consistency, never the specific rep.

---

## Proposed fix spec

Each is a single field edit. **A Salesforce admin has to apply these — the assistant has no write path.**

| Order | Field | Current | Set to |
|---|---|---|---|
| 525 | `Account_Type__c` | all 5 values | *(clear)* |
| 601 | `Account_Type__c` | all 5 values | *(clear)* |
| 1825 | `Account_Type__c` | all 5 values | *(clear)* |
| 1401 | `Country__c` | `DK;GB;IE;NO;SE` | *(clear)* — or add a continental-EMEA/MEA catch-all |
| 2200 | `Country__c` | `DK;GB;IE;NO;SE` | *(clear)* — or add a continental-EMEA/MEA catch-all |
| 783 | `Region__c` | `Americas` | `EMEA` |
| 46 · 525 · 1590 | `User__c` | Yoshi Han (inactive) | active APJ owner, or deactivate the rule |
| 1800 | `User__c` | Noriyuki Ishii (inactive) | active Japan owner, or deactivate the rule |
| 1850 | `Active__c` | true | false — pure duplicate of 1812 |
| 70 · 1722 | `Account_Team_Role__c` | `Monsido Account Executive` (no such role) | `Web Governance Sales Advisor` — confirm intended role first |
| 1819 | `Name` | `… - Yoshi` | reflect Keith Pettinger, or repoint |

**Suggested order of operations:** clear `Account_Type__c` on 525/601/1825 and fix the two inactive-user targets first — those close the APJ and LATAM gaps and stop leads landing on deactivated users. The EMEA `Country__c` change is the larger blast-radius edit and deserves a conversation about who *should* own continental EMEA and MEA leads before it's made.

---

## What this audit cannot tell you

- **Whether the right rep was picked inside a Round Robin pool.** No rotation-pointer object exists and `Queue_Name__c` is null on all 16 RR rules.
- **The engine's actual precedence logic.** Inferred from `Order_by__c`; the Flow/Apex is not readable through this connector.
- **Whether an account was linked at routing time.** `Account__c` is not history-tracked, so "Unmatched rule fired on a lead that has an account" can't be fully adjudicated retroactively.
