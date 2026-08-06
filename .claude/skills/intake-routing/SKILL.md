---
name: intake-routing
description: Assigns a classified MOPS intake ticket to the right owner (per the 2026 regional-assignment table), sets its Priority from the SLA clock + urgency, and surfaces workload imbalance before assigning. Use right after intake-classification, when a ticket has a Region + Project Type and needs an owner and priority.
argument-hint: [Asana task URL or ticket id]
---

# Intake Routing

The triage step between "a ticket is classified" and "the right person owns it, at the right priority." Runs **after `intake-classification`** (Region + Project Type are known) and closes three documented gaps at once:

- **No routing today** — ownership lives in a static table nobody applies; Aayushi assigns by hand.
- **Priority left null** on nearly all new intake tasks.
- **Workload imbalanced** — Harish/Aayushi carry a disproportionate share.

This skill only **triages** (owner + priority + load check). It doesn't classify (`intake-classification`), decide the SF-Campaign gate (`campaign-gate-check`), name anything (`campaign-naming`), or build a spec (`sf-campaign-spec`).

## Inputs
- A classified ticket: Asana `Project Type`, Region, requester, named stakeholder(s), target date, stated urgency.
- Source of the routing table: `references/MOps Docs/MOps Team Structure 2026 - Team Structure.md`.

> ⚠️ **Region and requester are not actually on the board** (verified live 2026-08-03, 50 open tickets). There is **no Region field** on `[MOPs] Intake`; `Requesting Team` is **49/50 null** and `Requestor` is **50/50 null**. Consequences: rules **2 (LATAM)** and **5 (Region × Project Group)** below cannot fire on real tickets — do **not** synthesize a Region to make them fire. Use `created_by` (populated 38/50, and its names map directly onto the stakeholder lists) as the requester for rule 4. The Region-free subset — rules **1, 3, 4, 6** — covers **64%** of the open board. See `intake-dispatch` for the measured engine, and note that rule 4 concentrates **75% of routable tickets on Aayushi**, which makes the workload check below load-bearing rather than advisory.

## Routing table (Region × Project Group × Stakeholder → owner)

Owners are **Harish, Aayushi, Felipe** for all campaign-facing work, plus **Jennifer** for the ad-hoc/no-pattern types only (see rule 0). Forkan (intern) is not a work-owner in this table.

| Owner | Owns | Named stakeholders | Additional role |
|---|---|---|---|
| **Harish Pandey** | NA Partner Marketing · EMEA Field & Events · Global Campaigns · Non-region Ad hoc / Special projects (Creative, Data Integrity) | Glenn, Katharine, Jemma, Karen | System Integrations — Zoom Webinar, CVent, Alice |
| **Aayushi Sharma** | EMEA Partner Marketing · NA Field & Events · Global Campaigns | Zhenya, Kim Bonilla, Kathy, Shannon, Jessica, Sam, Melissa | Task / Asana Management |
| **Felipe Tencio** | All regions — **Segmentation** (owns Naming Convention / Platform (Asset) Management in Account Engagement) · **LATAM** support | — | Segmentation Lead for every row |
| **Jennifer Schroback** (manager) | **Ad-hoc / no-pattern types only** — `IT/Integration`, `Automations \| Martech`, `UAT` | — | Scopes work that has no repeatable shape |

## Routing logic — apply in this order, first match wins

0. **No-pattern / ad-hoc types → Jennifer, with no sub-tasks.** `IT/Integration`, `Automations | Martech`, `UAT`. Evaluated **first** because it's the most specific and entirely unambiguous — Project Type alone decides it, no Region or stakeholder needed. These are the three types the sub-task catalog classes as "ad-hoc, no repeatable pattern," so they get scoped by the manager rather than templated. **Never propose sub-tasks for these** (authorized by Forkan 2026-08-03).
   - This **overrides rule 6** for `UAT`, which rule 6 would otherwise send to Aayushi. `Form Request` / `Other` / `Reporting` are unaffected and still route per rule 6.

1. **Segmentation / audience / list work → Felipe.** Any `Audiences`, segmentation task, or target-list pull, in any region. Felipe owns segmentation across all rows.
2. **LATAM region → Felipe.** LATAM support is his.
3. **System integration work → Harish.** Anything Zoom Webinar / CVent / Alice(11x) setup-flavored (his additional role).
4. **Named-stakeholder match.** If the requester or a named stakeholder appears in a single owner's stakeholder list above, route there — this is the strongest signal (the table is stakeholder-specific).
5. **Region × Project Group.** Map `Project Type` → Project Group (table below), combine with Region, look up the owner. Note **Global Campaigns is split** (Harish: Karen; Aayushi: Sam, Melissa) — disambiguate by stakeholder; if neither stakeholder is named, treat as ambiguous (step 7).
6. **Asana/task-management & unclassifiable ops** (`Form Request`, `Other`, `Reporting` with no stakeholder signal) → **Aayushi** (Task / Asana Management is her role) — unless workload (below) says otherwise, then flag. **`UAT` was removed from this list** — rule 0 sends it to Jennifer.
7. **Ambiguous → don't guess.** If two owners tie, no signal resolves it, or it's a **Global event / Engage / Drupalcon** (assignments deliberately vary for cross-region consistency — see the Team Structure doc's note), do **not** assign. Post a short heads-up to the MOPS team Slack channel naming the ticket + why it's ambiguous, and ask a human to pick.

### Project Type → Project Group (bridge — treat as unconfirmed, confirm with the team before relying on it as fixed)
| Project Type | Project Group |
|---|---|
| `Webinar Request`, `Event (+ SFDC Campaign)` | Field & Events |
| `SFDC Campaign only (single)`, `SFDC Campaigns only (multiple)`, `UTM(s) (+ SFDC Campaign)`, `Email(s) only \| Nurture Sequences` | Campaigns |
| `Audiences`, `List Upload` | Segmentation (Felipe) |
| Partner-flagged request (stakeholder-driven) | Partner Marketing |
| `Reporting`, `Form Request`, `Other`, internal ops | route by step 6 |

If a `Project Type` or Region doesn't map cleanly, **ask** — don't invent an owner (same discipline as `campaign-gate-check`).

## Priority

Priority is derived, not guessed:

1. **Base** = the SLA tightness for this Project Type. The per-type SLA table is the single source in **`sla-watchdog`** — read it there, don't duplicate it here.
2. **Escalate** if: stated Rush/Urgent (SOP routes these to VP/Director approval — flag that, don't silently set high), target date is inside the SLA window already, or a hard external date (event/webinar live date) is near.
3. Map to the Asana **Priority** custom field value (field gid `1211656326879675` — note the **trailing space** in its name, `"Priority "`). **Verified live 2026-08-03** — the picklist is *not* Low/Med/High, and there are five values, each a long descriptive string that must be written in full:

| Value (write this exact string) | gid |
|---|---|
| `Urgent (customer-facing, business-critical, immediate action needed, no workaround)` | `1211656326879678` |
| `Critical (customer-facing, business-critical, action this week needed, no workaround)` | `1211656326879679` |
| `High (major usability impact with few workarounds, customer-facing)` | `1211656326879680` |
| `Medium (moderate impact with some workarounds)` | `1211656326879681` |
| `Low (nice to have, several workarounds, inconvenient)` | `1211656326879682` |

The extra `Urgent` / `Critical` tiers matter: the SOP's Rush/Urgent → VP/Director escalation maps to `Urgent`, not `High`. **94% of open intake tickets have this field null** (47/50 sampled), so there is no baseline of human-set values to calibrate a derived priority against — treat early derivations as proposals.

## Workload check (surface, don't override)

Before assigning, query the routed owner's current open `[MOPs] Intake` task count (Asana `search_tasks` by assignee + not-completed).
- Route **per the table** — it's the ownership model, not a suggestion.
- **Report the load** alongside the recommendation, and if the routed owner is already above a heavy threshold, **flag it** so a human can rebalance. Don't silently reassign someone's owned region because they're busy.

## Writes — follow the contract

Every write goes through `.claude/rules/write-actions.md`:
- **Assign / reassign owner** (`update_tasks`) = **Class B** — notifies the assignee. Confirm first, naming who gets pinged. Idempotency: if already assigned to the target, **skip silently**.
- **Set Priority / due date** (`update_tasks`) = **Class A**. State it, then set it.
- **Slack heads-up for ambiguous tickets** (`slack_send_message`) = **Class C** — confirm before sending; not an unattended send unless explicitly authorized in the contract §6.
- Log every write attempt (including skips) to `decisions/actions.md`.

## Output format (propose before writing)

```
**Ticket:** [name / link]
**Classified as:** [Project Type] · [Region] · requester [name]
**→ Recommended owner:** [Harish/Aayushi/Felipe]  (rule matched: [1–7])
**Current load (that owner):** [N open intake tasks]  [⚠ heavy — consider rebalancing]
**→ Recommended priority:** [value]  (base SLA [type] = [days]; escalation: [none/urgent/date])
**Writes to confirm:** assign owner (Class B, notifies [name]); set Priority (Class A)
```

## Notes
- Runs after `intake-classification`; pairs with `campaign-gate-check` (gate) and `sla-watchdog` (SLA clock) but owns neither.
- The Region × Project Group → Project Group bridge is inferred, not team-confirmed — every low-confidence route should be flagged for confirmation, and confirmed answers folded back into the bridge table.
- If the routing table itself drifts (people change regions/stakeholders), update the Team Structure doc first, then this table.
