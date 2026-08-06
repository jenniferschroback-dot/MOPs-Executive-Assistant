# Intake Dispatch — 2026-08-06

**Window:** 2026-08-05T12:05:13Z → 2026-08-06T13:00:29Z
**1 new submission · 1 triaged already (owner skipped) · 1 proposed · 0 blocked**

Secondary: 1 new standalone sub-task (already assigned) · 2 test rows excluded.

⚠️ **No load imbalance to report** — zero owner proposals this run (the one new ticket arrived pre-assigned). Current open intake load: **Felipe 42 · Aayushi 40 · Harish 25**.
✅ **0 need classification** — the one new top-level ticket has a `Project Type` set. First clean run on this front since the routine went live.
🧹 **0 `[Event Name]` placeholder sub-tasks created this window** — the template substitution bug did not fire in the last 24h. Not fixed, just not triggered (no event-family parent was created).

---

## Proposals — new top-level submissions

### email: DAM User Group Attendee Follow up & G2 review ask  ·  `Email(s) only | Nurture Sequences`
`1217204326890036` · created 2026-08-05 19:56 UTC by **Karen Plant** · due **2026-08-10 (Mon)** · `MOPS- Status: Assigned` · 0 existing sub-tasks

- **Owner →** _no proposal_ — already assigned. (Routing engine would have reached the same owner via rule 4: `created_by = Karen Plant` → Harish's stakeholder list. Recorded as a routing-accuracy data point, not a change request.)
- **Priority →** `High (major usability impact with few workarounds, customer-facing)`
  - SLA base: **Single Promotional Email = submit 5 days out → scheduled 1 day out** (`sla-watchdog`).
  - Escalation: **target date is already inside the SLA window.** Submitted Wed 2026-08-05 for a Mon 2026-08-10 send = **3 business days of runway against a 5-day requirement**. Under-submitted by 2 business days on arrival.
  - Not `Urgent`/`Critical`: `Out of SLA/Rush` is null and no VP/Director escalation was requested. If Karen does want it treated as a rush, the SOP path is VP/Director approval — flag, don't self-approve.
  - Field is currently null. ⚠️ Derived, uncalibrated (94% of open tickets have null Priority — no human baseline to check against).
- **Sub-tasks →** 3 proposed + 1 gate-conditional (tier: **formulaic** — one triple per send)
  1. `DAM User Group Follow-up — Email 1 Creation`   [`default_task`]
  2. `DAM User Group Follow-up — Email 1 Approval`   [`approval`]
  3. `DAM User Group Follow-up — Email 1 Send`   [**`milestone`** — required, or it won't appear on the shared email send calendar] · due 2026-08-10
  4. _(conditional)_ `SFDC Campaign(s) & Form (If Required)`   [`default_task`] — **only if `campaign-gate-check` says yes.** `Email(s) only | Nurture Sequences` is **not** an n8n-covered type, so neither the gate nor the campaign name is automated here — both are the human path (`campaign-gate-check` → `campaign-naming`). This skill proposes no name.
  - **Send count = 1**, inferred from the request: one recipient list, one subject-line/content doc, one ask ("Can you please send email to this list"). The title reads as a single email covering both the follow-up and the G2 review ask. If it's actually two sends, the triple repeats.
- **Open questions for the requester** (from Karen's comment, worth resolving before build starts):
  - Recipient list is a **Google Sheet**, not a Salesforce/Pardot list → needs a `List Upload` before it's sendable. That may be a second ticket.
  - "Can we use an Acquia letterhead template? or similar?" — template choice is unanswered.
- **Writes this would need:** set Priority (**Class A**) · create 3–4 sub-tasks (**Class B**, notifies followers). No owner write.

## Secondary — new standalone sub-task requests

- **August 2026 August Partner Newsletter Send** (`1217212720672727`, `milestone`) — parent *August 2026 August Partner Newsletter*. Already assigned to Harish → **owner proposal skipped**. Correct `resource_subtype` (`milestone`), so it will land on the send calendar. No sub-task expansion (secondary items get owner only).
  - Note: its **parent** is one of the standing genuinely-unclassified tickets (null `Project Type`). Out of this window, so not re-reported here — but the send now exists under an unclassified parent.

## Excluded

- **0** template-generated sub-tasks (no event-family parent created in this window)
- **2** test rows — `[TEST RUN - MOps Automation] SFDC Campaigns (multiple) - DrupalCon Rotterdam email build` / `... email audience`. Ours, not counted as hygiene.

## Blocked

- None. No ticket in this window hit `needs Region` or `needs classification`.

---

### Run notes

- **n8n boundary:** the one new ticket is `Email(s) only | Nurture Sequences` — **outside n8n's coverage**, so no naming divergence to check this run. Coverage remains *inferred from Project Type*; whether n8n writes back to Asana is still unanswered.
- **Volume:** 1 new top-level + 1 real new sub-task in 25 hours. Consistent with the measured ~1/day baseline; the usual 6:1 sub-task ratio did not hold this window (no template expansion fired).
