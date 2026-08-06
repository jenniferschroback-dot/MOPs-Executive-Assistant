# Lead Routing Config Audit — 2026-08-06

**Source:** live `Lead_Routing_Rule__c` (read-only), 47 active rules of 76.
**Supersedes:** `Lead_Routing_Config_Audit_2026-08-03.md` — the rule table has changed materially since then.
**Salesforce is query-only from this assistant.** Everything below is a spec for a Salesforce admin to apply; nothing here has been written.

---

## 1. What changed since 2026-08-03

The table grew from **43 → 47 active rules** and three of the audit's proposed fixes were applied.

### Fixes applied (verified live)

| Order(s) | Change | Effect |
|---|---|---|
| **1401**, **2200** | `Country__c` cleared (was `DK;GB;IE;NO;SE`) | **The continental-EMEA/MEA coverage gap is closed.** This was the root cause of the original "leads routing to the wrong reps" complaint. Confirmed on 2026-08-05: Netherlands, Germany, Norway and Jersey leads all reached EMEA rules instead of the AMER BDR fallback. `RECOMPUTE_NO_RULE` was **0/20** that day — its first zero. |
| **525** | `Account_Type__c` cleared | The rest-of-APJ Unmatched rule is reachable for the first time. Resolves linter checks 3 and 5 for this rule. **See §3 — this created a new problem.** |
| **70** | `Account_Team_Role__c`: `Monsido Account Executive` → `Web Governance Sales Advisor` | The rule can now actually route. Confirmed live that ATM rows with this `TeamMemberRole` exist on real accounts. **Its T2 twin 1722 was not fixed.** |

### 14 new rules

`51`, `53` (Americas/EMEA Source AE) · `57`, `59`, `61` (per-region Content Cloud ATM, replacing the old order `52`) · `715`, `719` (T1 Source round robin) · `901` (AMER Monsido, its own rule) · `1711`, `1715` (T2 Matched Source RR) · `1719` (T2 CC ATM) · `1860` (T2 EMEA Source RR) · `1880` (T2 CC RR) · `2000` (T2 AMER catch-all).

Two were observed routing correctly the same day: **57** (→ ATM `DAM Sales Advisor`) and **719** (→ RR EMEA BDR on a Jersey lead that under the old table would have fallen to the AMER pool).

---

## 2. Fix spec — what still needs doing

Ordered by how much lead loss each causes. Every item is a one-field change.

| # | Order | Field | Set to | Why |
|---|---|---|---|---|
| 1 | **46** | `User__c` | current Rest-of-APJ owner | Points at **Yoshi Han (deactivated)**. Actively losing leads — see §3. |
| 2 | **525** | `User__c` | current Rest-of-APJ owner | Same dead user, and the rule just became reachable. |
| 3 | **1590** | `User__c` | current Rest-of-APJ owner | Same dead user, T2 Matched. |
| 4 | **1800** | `User__c` | Takashi Fujiwara | Points at **Noriyuki Ishii (deactivated)**. Takashi is the active Japan owner — confirmed routing correctly via rules 46/520 on 2026-08-05. |
| 5 | **1722** | `Account_Team_Role__c` | `Web Governance Sales Advisor` | Half-applied fix — identical to the change already made on order 70. Until fixed, T2 Monsido leads with an account fall through to 1765/1780 and never reach a Web Governance rep. |
| 6 | **601** | `Account_Type__c` | *(clear)* | LATAM no-account T1 leads reach NA BDRs instead of Livia Russo. Same bug class as the 525 fix that *was* applied. |
| 7 | **1825** | `Account_Type__c` | *(clear)* | T2 half of the same LATAM bug. |
| 8 | **783** | `Region__c` | `EMEA` | Tagged `Americas` despite being the EMEA Content Cloud rule. Also makes it a duplicate of 780, so fixing this resolves two linter checks. |
| 9 | — | new rule or widen `525`/`1819` | `Country__c` wildcard for APJ | **Remaining coverage gap.** `Region__c = 'APJ'` leads whose country is outside `JP` and the 13-country rest-of-APJ list (CN, KR, KG, UZ, TW, HK, IN, …) still match no rule and fall to the AMER pool. |
| 10 | **1850** | — | delete or deactivate | Pure duplicate of 1812. |
| 11 | **950** | `User__c` / `Name` | EMEA Monsido rep; fix `Monside` typo | EMEA rule pointing at `AE Americas Monsido`. Harder to defend now that order 901 gives AMER its own rule with the same rep. |
| 12 | **1890** | — | confirm intentional | All-Regions rule → the same Americas-only rep. |
| 13 | **1819** | `Name` | match the actual owner | Named "Yoshi", routes to Keith Pettinger. Routing is fine; only the name misleads. |
| 14 | **48**, **51**, **53** | `User__c` | *(clear)* | Inert on `Account Team Member` rules but reads as the routing target. The pattern was copied into the two new rules — worth fixing in whatever template produces them. |
| 15 | all **16** RR rules | `Queue_Name__c` | the pool queue | Null on all 16, which is why round-robin rep selection is unauditable. Populating it would make RR verifiable at rep level instead of pool level. |

---

## 3. New finding — an inactive-user rule produces an *unowned* lead, not a wrongly-owned one

Previously treated as a latent config bug. It is an active daily lead-loss mechanism.

Lead `00QPb00001mehOfMAI` (Purchasing Lsc / Livingstone Shire Council, AU) was MQL'd and routed on 2026-08-05 at 07:07:35Z. `LeadHistory` shows **three Owner writes in the same second**, all by `B2BMA Integration`:

```
Leads - Marketing Queue  ->  Yoshi Han          (rule 46, Direct To User)
Yoshi Han                ->  Leads - Marketing Queue
```

Consequences:

- The recompute **agrees** with rule 46 — rule *selection* was correct. The target user is dead.
- The lead ended up back in the pre-routing queue with **no owner**, as a qualified MQL.
- `Assigned_Inactive_User__c` is **false**, because the final owner is a queue rather than the inactive user. **The existing inactive-owner tripwire cannot see this failure mode**, and neither can the 14,636-lead backlog count.
- The account is *also* owned by Yoshi Han, so rule 90 (`Account Owner`) would have failed identically. There was no path to a live human.

This raises fix items 1–4 above from housekeeping to urgent, and it means the true count of leads lost to inactive-user rules is **not** measurable from `Assigned_Inactive_User__c`. A proper count needs a `LeadHistory` scan for same-second owner→queue reversions.

---

## 4. Linter state

25 rows on the Sheet's `Config Linter` tab (was 23). 3 marked `RESOLVED`, 22 open. Full detail with per-rule proposed fixes:
`1gFu34xeJJ2Jrk1p9jtvgkzp_WtspHNyCGZ1zorbNNOA` → `Config Linter`.

## 5. Honest limits

- The recompute reads the **declarative rule table only**. The engine is Flow/Apex this connector cannot see; precedence is inferred from `Order_by__c`. A divergence means the engine disagreed with a literal reading of the table, not that the engine is provably wrong.
- **Round robin is asserted at pool level only** — no rotation pointer exists. 5 of the 10 routed leads on 2026-08-05 were RR; all passed pool consistency, none can be confirmed as "the correct rep".
- `Account__c` is not history-tracked, so "did this lead have an account at routing time" is never provable per lead.
