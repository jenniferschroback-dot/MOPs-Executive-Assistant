# MOPS Task → Sub-task Catalog

_Derived 2026-08-03 from live Asana `[MOPs] Intake` (project gid `1205660951274722`, 2,664 completed tasks). Method: for each `Project Type` option, pulled the 12–15 most-recently-completed **parent** tasks (`is_subtask=false`) with `num_subtasks`, then deep-read the 12 parents carrying the richest sub-task sets to extract actual names._

**What this is:** the deliverable-level sub-task sets MOPS actually creates per request type. Companion to [`mops-sla-timeline.md`](mops-sla-timeline.md), which covers *deadlines and approval checkpoints* for the same types. This one covers *what work items exist*; that one covers *when each is due*.

**Consumed by:** `intake-classification` step 6 ("expand into sub-tasks"). That skill's own table decides the **Salesforce campaign gate** — for the gate, `campaign-gate-check` wins. For sub-task expansion, this file wins.

---

## Evidence strength — read this before automating

| Tier | Types | Meaning |
|---|---|---|
| **Template** | `Webinar Request` | A real, stable, repeated template. Safe to expand automatically. |
| **Template, inconsistently applied** | `Event (+ SFDC Campaign)` | A clear template exists but 5 of 15 recent parents had **zero** sub-tasks. Propose it; don't assume it. |
| **Formulaic** | `Email(s) only \| Nurture Sequences`, `List Upload` | No fixed list, but a mechanical rule that scales with send/list count. Safe to generate from the rule. |
| **Leaf** | `SFDC Campaign only (single)`, `SFDC Campaigns only (multiple)`, `Form Request`, `Audiences`, `Reporting`, `Issues`, `Other` | The ticket **is** the work item. Creating sub-tasks here is wrong. |
| **Ad-hoc — no pattern** | `IT/Integration`, `Automations \| Martech`, `UAT` | Project work sized case-by-case. Never auto-expand; ask. |

---

## Automation coverage — who already handles which type

An **n8n workflow** runs in production, polling Asana every 15 minutes to detect new tickets, decide the campaign gate, and generate the campaign name. Its coverage is narrow and conditional:

| Project Type | n8n | Consequence |
|---|---|---|
| `SFDC Campaign only (single)` | ✅ always | Naming is handled — don't propose one |
| `UTM(s) (+ SFDC Campaign)` | ✅ but **dead** — the option is disabled in Asana (`1206591746930196`, `enabled: false`), so no new ticket can select it | Tell whoever maintains the workflow; it's dead code, not a gap |
| `Webinar Request` | ⚠️ **only if** the submission explicitly states **no** companion promotional email | Promo email = yes, or unanswered → human path |
| `Event (+ SFDC Campaign)` | ⚠️ same condition | Same |
| everything else, incl. `SFDC Campaigns only (multiple)` | ❌ | Fully human |

**"Explicitly states no" is the literal rule** — a missing or ambiguous answer to the promo-email question means **not covered**. Never infer "no" from silence.

The division of labour: **n8n owns detect → gate → name. The MOPS skills own assign → prioritize → expand sub-tasks.** Nothing in n8n assigns owners, sets Priority, or creates sub-tasks, so this catalog stays authoritative for the sub-task half regardless of n8n's scope.

**Two unknowns that would change this:** whether n8n writes back into Asana (so coverage can be *read* rather than inferred from Project Type), and whether it holds **Salesforce write credentials** — which, if true, means `sf-campaign-spec` should hand off to n8n instead of to a human, and the standing "no SF write path" constraint (`write-actions.md` §9) has a route around it.

---

## 1. `Webinar Request` — the 20-step template ✅

Strongest finding. Four deep-read parents (`EDU Webinar` 21, `DAM Finserv` 21, `Acquia x Drupal Association Q226` 20, `Conductor Partner Master Class` 18) share a near-identical ordered set. Sub-tasks are prefixed with the webinar name in practice; `[Webinar Name]` below marks where.

| # | Sub-task | Owner tag seen |
|---|---|---|
| 1 | Complete Webinar Planning Doc | — |
| 2 | Webinar Campaign Naming | `[Webinar Owner]` |
| 3 | Create Webinar Campaign in SFDC | `[Webinar Owner]` |
| 4 | Add Webinar to Events Calendar Once Confirmed | — |
| 5 | Edit Webinar Details in Zoom | `[Webinar Owner]` |
| 6 | Stakeholder Copy Review | — |
| 7 | `[Webinar Name]` Webinar Audience Selection | MOPS (Felipe) |
| 8 | Webinar Kickoff Meeting | — |
| 9 | `[Webinar Name]` Webinar Banner Request - Email / Social | Creative |
| 10 | `[Webinar Name]` MOPs Create Webinar Landing Page with Form | `[MOPs]` |
| 11 | `[Webinar Name]` Test Form | `[MOPs]` |
| 12 | `[Webinar Name]` Push Page Live | `[MOPs]` |
| 13 | `[Webinar Name]` Confirm Page Go-Live | `[WEB]` |
| 14 | `[Webinar Name]` Webinar Email Creation | MOPS — **has its own children, see 1a** |
| 15 | `[Webinar Name]` Social Copy and Scheduling Request | Social |
| 16 | `[Webinar Name]` Webinar Slide Creative Request | Creative |
| 17 | Webinar Dry Run | — |
| 18 | Webinar Hosting | `[For Day Of Moderator]` |
| 19 | Upload Webinar Recording to DAM | — |
| 20 | Webinar Landing Page Updated to On Demand | `[MOPs]` |

**1a. `Webinar Email Creation` nests a second level.** Observed on gid `1215985796251438`:
- Email 1 Promo · Email 2 Promo · Email 3 Promo
- Post-Event Attendee Email · Post-Event No-Show Email
- Attendee and No-Show Report Upload

These are the items `email-send-calendar` turns into dated milestones. Promo email count varies (1–4).

**Situational add-ons** (present on some, not template):
- `UTM Links for <partners>` — partner/sponsor co-marketing webinars
- `LinkedIn Webinar Promotion` — paid promotion attached
- `Post webinar list upload` + `Post webinar social copy and scheduling request` — 3rd-party-hosted webinars, where the attendee list arrives externally

**3rd-party webinar variant** (e.g. ` CMI Webinar Feb 25 `, gid `1213044620018510`): skips Zoom setup, dry run, and hosting (the partner runs the event). Substitutes `<partner> (3rd party webinar) SFDC code request` and adds `Email Audience List Import` + post-webinar list upload. **Check who hosts before expanding steps 5, 17, 18.**

---

## 2. `Event (+ SFDC Campaign)` — 16-step template, distinct from webinar ⚠️

Deep-read `DrupalCon Chicago 2026` (gid `1213029686065382`, 16 sub-tasks). This is **not** the webinar list — no Zoom, no dry run, no on-demand conversion; instead pre/post-event email, staff logistics, and a summary.

| # | Sub-task |
|---|---|
| 1 | `[Event]` Create Event Planning Doc |
| 2 | `[Event]` Planning Meeting |
| 3 | Add Event To Global Events Tracker |
| 4 | `[Event]` SFDC Campaigns |
| 5 | `[Event]` Landing Page |
| 6 | `[Event]` Event LP Header Image & Events Page Card |
| 7 | `[Event]` Creative Requests |
| 8 | `[Event]` Organic Social Request |
| 9 | `[Event]` Pre Event Marketing Email |
| 10 | Schedule Know Before You Go |
| 11 | `[Event]` List(s) Upload |
| 12 | `[Event]` Post Event Marketing Email |
| 13 | `[Event]` Sales Emails |
| 14 | `[Event]` Field Update |
| 15 | `[Event]` Create Post Event Summary |
| 16 | `[Event]` Staff Feedback |

⚠️ **Adoption is inconsistent.** Of 15 recent completed Event parents: 5 had 0 sub-tasks, 4 had 2–6, 6 had 7–16. Large events (DrupalCon) get the full set; regional dinners/user groups often get nothing. Propose the list, flag it as the full-scale template, let the owner trim.

### 2a. Workshop / small-event variant — a third distinct pattern

`DAM Workshop DC October 2025` (gid `1210988062310880`) and `DAM Workshop Milwaukee October 2025` (gid `1211118925190758`) are **identical, 11 sub-tasks each**, and neither uses the DrupalCon shape:

- SF Campaign creation
- Form creation
- Email 1 / 2 / 3 creation → `default_task`
- Email 1 / 2 / 3 Approval → **`approval` subtype**
- Email 1 / 2 / 3 Send → **`milestone` subtype**

This is the "campaign + form + N-email promo sequence" pattern. Use it for workshops, dinners, user groups, and partner master classes — not the 16-step DrupalCon set.

> **Note:** the Milwaukee copy is filed under `Project Type = Form Request`, the DC copy under `Event (+ SFDC Campaign)`. Same work, two types. See §6.

---

## 3. `Email(s) only | Nurture Sequences` — formulaic, scales with send count

No fixed list. Two shapes observed:

**Shape A — single send with campaign setup** (`Acquia AI Advisor Launch`, gid `1216531795994729`):
- SFDC Campaign Creation
- Provide Email Copy _(stakeholder)_
- `[Name]` - Email Build
- `[Name]` - Email Audience
- MOPs Create UTM's

**Shape B — per-send triple** (`Partner Master Class`, gid `1211012624202827`): for each email N —
- Email N Creation → `default_task`
- Email N Approval → `approval`
- Email N send → `milestone`

Recent parents run 0–6 sub-tasks. Single-email promos frequently have **zero** — the ticket is the work.

**Rule for expansion:** count the sends, emit the Shape B triple per send, plus Shape A's setup items once if no campaign exists yet. Hand the `milestone` items to `email-send-calendar` for dating.

---

## 4. `List Upload` — exactly one sub-task

The most consistent type in the whole board. **12 of 15** recent parents had exactly 1 sub-task; name is a variant of **`Upload list into SF`**.

The two exceptions (`Sherpa Partner Recruitment Campaign`, 2 sub-tasks) were multi-batch uploads — one sub-task per batch.

**Rule:** one sub-task per distinct list/batch. Never more.

---

## 5. Leaf types — do not create sub-tasks

The ticket is the work item. Recent-completed parents with **zero** sub-tasks:

| Project Type | Zero-subtask rate | Note |
|---|---|---|
| `Issues` | 12 / 12 | Always a leaf |
| `SFDC Campaign only (single)` | 12 / 15 | The 3 exceptions had 1 sub-task literally named "Campaigns creation" — redundant with the parent |
| `SFDC Campaigns only (multiple)` | 13 / 15 | Same; `Trials Digital Program` (gid `1215539382737090`) → one "Campaigns creation" child |
| `Audiences` | 9 / 12 | Exceptions are multi-list enrichment projects. **Execution skill: `audience-pull`** — owner is always Felipe (rule 1), no campaign, no name, no sub-tasks; deliverable is a CSV + spec in `outputs/audiences/` |
| `Reporting` | 9 / 12 | Exceptions (`LinkedIn CAPI` 8, `SF List creation` 4) are integration projects mis-typed as Reporting |
| `Form Request` | 9 / 15 | Genuine standalone form requests are leaves; the 4–11 rows are misclassified events (§2a) |
| `Other` | 9 / 12 | Catch-all; mostly single execution steps ("Push Page Live", "Territory batch update") |

---

## 6. Ad-hoc types — no sub-tasks, and they go to Jennifer

**Routing decision (Forkan, 2026-08-03): these three route to Jennifer with no sub-tasks.** They have no repeatable shape, so the manager scopes them rather than an automation templating them. This is `intake-routing` **rule 0**, evaluated first, and it overrides rule 6 for `UAT` (which previously went to Aayushi).

| Project Type | Recent parents | Sub-task range | Read |
|---|---|---|---|
| `IT/Integration` | 12 | 0–4 | Real project work (Cvent, G2, Snowflake, 6Sense↔Pardot). Scoped per integration. |
| `Automations \| Martech` | 6 total, ever | 0–2 | Same. Very low volume. |
| `UAT` | **1 total, ever** | 3 | Effectively unused since 2025-08. |

Never propose a sub-task list for these — any list would be invented rather than derived.

---

## 7. Field-level corrections to `intake-classification`

Verified against the live project's `custom_field_settings` on 2026-08-03:

1. **`Project Type` has 17 options, not 15.** The skill table is missing `6Sense CE` (gid `1210870069976681`, **disabled**).
2. **`UTM(s) (+ SFDC Campaign)` is confirmed disabled** (gid `1206591746930196`). The skill's open question — *"confirm it's actually retired before assuming"* — is now answered: it is disabled in Asana. No new tickets can select it.
3. **An undocumented `MOPs - Project Subtype` field exists** (gid `1211571206556215`): `UTMs` · `SFDC ID` · `Email Send` · `Landing Page` · `Email`. Not referenced anywhere in the skills. Likely the finer-grained discriminator that would resolve several `Other` / `Form Request` ambiguities — worth asking Harish about.
4. **An `Out of SLA/Rush` Yes/No field exists** (gid `1211856242548764`) — directly relevant to `sla-watchdog`, which currently derives rush status from the SOP rather than reading this field.
5. **Sub-task `resource_subtype` is load-bearing.** Email sends are created as `milestone`, approvals as `approval`, everything else `default_task`. Any sub-task creation must set this — a send created as `default_task` won't appear on the email send calendar.

---

## 8. Open questions for Harish / Aayushi

- Is the 20-step webinar set an **enforced** template (from an Asana project template) or convention? If templated, the template is the source of truth, not this file.
- Should the §2a workshop pattern be its own `Project Type`? It's currently split across `Event` and `Form Request`, which corrupts per-type SLA and volume reporting.
- `Event (+ SFDC Campaign)`: is the 16-step set expected for all events, or DrupalCon-scale only? Determines whether 0-sub-task events are a gap or correct.
- Is `UAT` retired in practice? One completed task in ~12 months.
