# Intake Dispatch — 2026-08-05

**Window:** 2026-08-04T12:00:31Z → 2026-08-05T13:05:13Z
**2 new submissions · 1 triaged already (skipped) · 0 proposed · 2 blocked**

⚠️ **100% of this window is blocked on classification.** Both new tickets have a null `Project Type`, so the routing engine cannot fire on either. Nothing is proposable — this digest has zero approvable actions.
⚠️ **Both tickets are LATAM (Mexico / Chile) and the board has no Region field.** Rule 2 (LATAM → Felipe) is the rule that should decide these, and it is exactly the rule that cannot fire. See _Blocked_ for the region evidence sitting in the form body.
⚠️ **Possible n8n divergence** — both are `Salesforce Campaigns` requests by their own form body, the type n8n always covers, and neither has a `Project Type` or a generated campaign name a day after arrival. See _n8n check_.
🧹 **0 `[Event Name]` placeholder sub-tasks created this window.** (The standing backlog population is unchanged and still large — out of scope here.)
**Load (open intake tasks):** Aayushi 23 · Felipe 22 · Harish 21 — balanced this run; no imbalance headline.

## Proposals — new top-level submissions

None. Both new submissions are blocked (below).

## Secondary — new standalone sub-task requests

None proposable.

- `Atlanta Tech Week 2026 Pre-Event Email Build` (parent: `Atlanta Tech Week 2026 Pre Event Marketing Email`) — **triaged already**: assigned to Aayushi Sharma and already completed, `MOPS- Status = Completed`. Owner proposal omitted per contract §4 (already-assigned is a silent skip). Counted, not argued with.

## Excluded

- 0 template-generated sub-tasks
- 0 test rows
- 0 blank orphans
- 0 template placeholders

## Blocked

Both tickets were filed through the MOps Intake Form by **Alexandra Vargas C.** (`alexandra.vargas@acquia.com`) on 2026-08-04, ~16 minutes apart. `created_by` is **null on both** — so even the `created_by` proxy that normally rescues rule 4 is unavailable here; the requester name exists only in the free-text body.

### 1. `SFDC Campaign Creation + Fintech MX` — gid `1217170258402614`
- **Blocked:** needs classification (null `Project Type`)
- **Body says:** `What are you Requesting? = Salesforce Campaigns` · Campaign Type `Event` · Subtype `Tradeshow` · Campaign Date `08/12/2026` · GTM/PCI `DXP` · Cost `4000` · Partner `Julius`
- **Region evidence (body only):** "Sponsorship of the Fintexh Summit, an industry event in **MX**"
- **Proposed due date:** Aug 5, 2026 — **today**
- `Out of SLA/Rush = No` · unassigned · 0 sub-tasks

### 2. `SFDC Campaign Creation + Havas CL Tradeshow` — gid `1217170017808944`
- **Blocked:** needs classification (null `Project Type`)
- **Body says:** `What are you Requesting? = Salesforce Campaigns` · Campaign Type `Event` · Subtype `Tradeshow` · Campaign Date `07/29/2026` · GTM/PCI `DXP` · Cost `0` · Partner `Havas`
- **Region evidence (body only):** Havas **CL** tradeshow — Chile
- **Proposed due date:** Aug 5, 2026 — **today**
- ⚠️ **Campaign Date `07/29/2026` is 7 days in the past** — this is retroactive campaign creation, which changes what "on time" means for it.
- ⚠️ **Name mismatch:** the ticket title says *Havas*, but the `SFDC Campaign Name` field says **`Vass CL Tradeshow`**. One of the two is wrong and a human needs to say which before anything is created in Salesforce.
- `Out of SLA/Rush = No` · unassigned · 0 sub-tasks

### What unblocks them

Setting `Project Type` is the only blocker for routing. The body is unambiguous about *what* is being requested — `Salesforce Campaigns`, singular campaign per ticket — so the classification is a 10-second human confirmation, not an investigation. This skill does not set it, and deliberately does not guess it.

**Non-binding note for whoever triages:** if these are classified and a Region is established as LATAM, rule 2 sends both to **Felipe** (LATAM support is his). Treat that as a candidate, not a proposal — the region comes from prose in the body, not a field, and the skill's standing rule is not to synthesize a Region to make rule 2 fire. There is also a live counter-signal: **Harish currently owns several LATAM items** on this board (`Email Invitation for a Webinar in LATAM`, `LATAM Webinar AIContent BR Email send 3`, `Email Invitation for a Webinar in Brasil`), so actual practice may not match rule 2. Worth one question to Aayushi or Harish rather than an inference.

## n8n check

n8n polls at 6am and 12pm daily, so it has had **two passes** over both tickets (arrived 2026-08-04 ~19:42 and ~19:58 UTC). Observed state as of this run: `Project Type` null, no campaign name written anywhere on either ticket.

Two readings, and they need different fixes:
- **If n8n gates on `Project Type`** — it never fired, because the field is null. Then the null-`Project Type` problem is not just a routing blocker, it silently disables the naming automation too, which makes it more expensive than it looks.
- **If n8n gates on the form body** (`What are you Requesting? = Salesforce Campaigns`) — it should have fired on both and did not. That's a live failure worth reporting to whoever maintains the workflow.

Distinguishing them requires the still-open question of **whether n8n writes back into Asana**. Until that's answered, coverage here is inferred, and this run cannot tell a silent n8n failure apart from a correctly-skipped ticket. Flagging rather than concluding.

## Writes

**None.** This skill performs zero writes, and this run had nothing proposable anyway. Nothing logged to `decisions/actions.md`.
