---
name: intake-dispatch
description: Daily read-only sweep of NEW arrivals on the [MOPs] Intake board — buckets them by Project Type and proposes, per ticket, an owner and the sub-task set it needs, as a reviewable digest written to outputs/intake/. Never writes to Asana and never touches the backlog. Use for the daily intake triage digest, or to see what just came in and needs routing.
argument-hint: [optional: "since YYYY-MM-DD" to override the watermark for one run]
---

# Intake Dispatch — propose only

The daily triage sweep. Reads **what arrived since the last run** on `[MOPs] Intake`, buckets it by `Project Type`, and for each ticket proposes **an owner** and **the sub-task set it needs**. Then stops.

**Scope is new arrivals only.** The ~158 open tickets already sitting on the board are deliberately out of scope — the digest is about what just came in, not the backlog. That's enforced structurally by a watermark plus a floor date, not by judgment. See _Scope_.

**This skill performs zero writes.** Not "writes with confirmation" — zero. It emits a proposal a human acts on. That is the whole design: it needs no `write-actions.md` §6 authorization, so it can run unattended from day one while the routing quality is still being calibrated.

Composes existing skills rather than redefining them:
- **Owner + priority logic** → `intake-routing` (the 2026 regional table). This skill applies it; it doesn't own it.
- **Sub-task sets** → `references/sops/mops-task-subtask-catalog.md`. This skill reads it; it doesn't own it.
- **SLA clock** → `sla-watchdog`. **SF campaign gate** → `campaign-gate-check`. Neither is re-derived here.

---

## The n8n boundary — check this before proposing anything

**An n8n workflow already handles campaign naming in production.** It polls Asana **at 6am and 12pm Daily**, decides whether a new ticket is a campaign, and generates the campaign name from the submission details. This skill runs once a day, so **n8n has already seen and acted on every ticket in the window** before the digest is written.

### What n8n covers — the complete list

| Project Type | n8n handles it? |
|---|---|
| `SFDC Campaign only (single)` | ✅ always |
| `UTM(s) (+ SFDC Campaign)` | ✅ — but see the dead-branch note below |
| `Webinar Request` | ⚠️ **only if** the submission explicitly states **no** companion promotional email |
| `Event (+ SFDC Campaign)` | ⚠️ **only if** the submission explicitly states **no** companion promotional email |
| **everything else** | ❌ not covered — including `SFDC Campaigns only (multiple)` |

**The promo-email flag is the whole gate for Webinar/Event.** That's the same yes/no already described at `intake-classification` step 4 — "do you require a promotional email as part of this request?". Read it as:

- **explicitly "no" →** n8n owns it. Don't propose a name; don't re-derive the campaign gate.
- **"yes", or absent/ambiguous →** n8n does **not** own it. The ticket needs the human path, and the digest should say so plainly. Do **not** infer "no" from silence — the rule is *explicitly states no*, so a missing answer means not covered.

> **Dead branch:** `UTM(s) (+ SFDC Campaign)` is **disabled** in Asana (option gid `1206591746930196`, `enabled: false`, verified 2026-08-03). No new ticket can select it, so n8n's coverage of that type can't fire on new submissions. Worth telling whoever maintains the workflow — it's dead code, not a gap.

### What this skill must therefore NOT do

- **Never propose a campaign name.** n8n owns naming for the types it covers, and for the types it doesn't, naming is a human decision through `campaign-naming`. Two systems generating names is how a convention drifts.
- **Never re-derive the campaign gate** for an n8n-covered type. n8n already decided.
- **Do report a divergence.** If a ticket n8n should have named has no name after a full day, that's a signal worth surfacing — n8n silently failing is otherwise invisible.

### What's left for this skill — the clean split

n8n owns **detect → gate → name**. This skill owns **assign → prioritize → expand sub-tasks**. Those don't overlap: nothing in the n8n workflow assigns an owner, sets Priority, or creates sub-tasks.

So the digest's real job is the half nobody else covers, plus a daily QA read on the half n8n does.

**Open and load-bearing:** whether n8n **writes back to Asana** (custom field / comment / task rename) and whether it has **Salesforce write access**. If it writes the name into Asana, this skill should read it and skip rather than infer coverage from Project Type. If it can write to Salesforce, then `sf-campaign-spec`'s "spec plus handoff" (`write-actions.md` §9) should hand off *to n8n* rather than to a human — which would be the single biggest unlock available. Until both are answered, treat n8n coverage as inferred from the table above, and say so in the digest.

---

## Inputs

| | |
|---|---|
| Project | `[MOPs] Intake` — gid `1205660951274722` |
| Scope | **new arrivals only** — never the backlog. See _Scope_ below. |
| Default | everything created since the last successful run |

## Scope — new submissions only, and never the backlog

The board carries ~158 open tickets, most of them long-standing. **They are deliberately out of scope.** The digest reports what arrived since the last run and nothing else.

### Watermark

State lives in `routines/mops-intake-dispatch/state.json`:

```json
{ "floor_date": "<the day the routine went live>",
  "last_run_utc": "<ISO8601 of the previous run's start>" }
```

Fetch with `created_at_after = max(floor_date, last_run_utc − 1h)`.

- **`floor_date` is the backlog guard.** On the first run there is no `last_run_utc`, and without a floor the query would sweep the entire history. The floor makes "ignore what's already sitting there" structural rather than a judgment call. **Never lower it** to pick up old tickets — that's a separate, explicit request.
- **The 1-hour overlap** absorbs Asana index lag, so a ticket created seconds before a run boundary can't fall through the crack between two runs.
- **Advance the watermark to the run's *start* time, not its end.** Anything created mid-run gets caught next time rather than skipped.
- **Update it only on success.** A failed or partial run must leave the watermark alone, so the next run re-covers the same window. Re-reporting is harmless; skipping isn't.
- A closed laptop just widens the next window. Missing days self-heal.

### Empty runs are the normal case

Measured volume: **6 new top-level tickets in 7 days** (~1/day). Many days will have **zero**.

Write the digest anyway, with an explicit "no new submissions" line. A missing file is ambiguous — did nothing arrive, or did the job not run? An empty digest is unambiguous, and the file series doubles as the run log.

### ⚠️ Top-level tickets are only 1/7th of new arrivals

In the same 7 days: **6 new top-level tickets, 36 new sub-tasks.** The real inflow is sub-tasks, roughly 6:1. But they're two different things and must not be pooled:

| | What it is | Digest treatment |
|---|---|---|
| **New top-level ticket** | a genuine submission — someone filed a request | **Primary.** Full owner + sub-task + priority proposal. |
| **New sub-task, template-generated** | derived work, auto-expanded from a parent | **Excluded from proposals.** Count only. |
| **New sub-task, standalone request** | a real request that happens to be filed as a sub-task — e.g. `SFDC Campaign Request - Competitor Search Ads`, `SFDC Campaign Request (Federal ABM, Linkedin)`, `Drupal GovCon 2026 Promo Email Send #1` | **Secondary section.** Owner proposal only, no sub-task expansion. |

**Detecting template-generated sub-tasks** — either signal is sufficient:
- The name contains an unsubstituted placeholder: `[Event Name]`, `[Webinar Name]`, `[NAME]`.
- **≥3 siblings under one parent created within 60 seconds of each other.** Verified live: the DAM Workshop DC set (`SFDC Campaign(s) & Form`, `Pre Event Marketing Emails`, `Pre-Event Email Audience Report`, `Post Event Marketing Emails`, `Post-Event Email Audience Report`) was created as a 5-item batch inside 12 seconds, and the same batch appeared for DAM Workshop NYC, Drupal 12 US Holiday Party, and Atlanta Tech Week 2026.

> **This partially answers the catalog's biggest open question.** Something already auto-expands a 5-sub-task event/workshop set. So for that pattern, proposing sub-tasks would be **redundant** — the expansion is already automated. Confirm the mechanism (Asana project template? a rule? a Zap?) before proposing sub-tasks for any Event-family ticket, because the answer may delete that whole branch of this skill.

> **And it has a live bug.** Four of the ten sub-tasks that automation created on 2026-08-03 carry the literal string `[Event Name]` in their names — the substitution didn't fire. These aren't stale leftovers; they're being generated wrong right now. Report them as a hygiene line every run.

### Field gids (verified live 2026-08-03)

| Field | gid | Use |
|---|---|---|
| `Project Type` | `1206591746930193` | the bucket |
| `MOPS- Status` | `1210850678809573` | idempotency marker |
| `Priority ` | `1211656326879675` | ⚠️ **trailing space in the name** |
| `Out of SLA/Rush` | `1211856242548764` | escalation signal |
| `MOPs - Project Subtype` | `1211571206556215` | tie-breaker for ambiguous types |

**`Priority` picklist — now confirmed** (this settles the open question at `intake-routing/SKILL.md` line 58; the values are **not** plain Low/Med/High):

| Value | gid |
|---|---|
| `Urgent (customer-facing, business-critical, immediate action needed, no workaround)` | `1211656326879678` |
| `Critical (customer-facing, business-critical, action this week needed, no workaround)` | `1211656326879679` |
| `High (major usability impact with few workarounds, customer-facing)` | `1211656326879680` |
| `Medium (moderate impact with some workarounds)` | `1211656326879681` |
| `Low (nice to have, several workarounds, inconvenient)` | `1211656326879682` |

Propose the **full string** — a partial value will not match on write.

---

## Step 0 — Stamp the run

**Before any fetch**, capture the run's start time as an ISO-8601 UTC string (`date -u +%Y-%m-%dT%H:%M:%SZ`) and hold it. This is the value Step 8 writes back as the new watermark — capturing it *after* the fetch would silently drop anything created while the run was in flight.

Then read `routines/mops-intake-dispatch/state.json` and compute the window:

```
window_start = max(floor_date, last_run_utc − 1h)      # last_run_utc null → floor_date
window_end   = run_start                                # captured above
```

## Step 1 — Fetch

Two `search_tasks` calls against project `1205660951274722`, both with `created_at_after` set to `window_start`:

1. `is_subtask=false` → the primary set.
2. `is_subtask=true` → the secondary set (then split template vs standalone per _Scope_).

Do **not** filter on `completed=false`. A ticket created and closed inside the window is still a submission that happened, and the digest should say so rather than silently omitting it.

`opt_fields`: `name,created_at,completed,assignee.name,num_subtasks,parent.gid,custom_fields.name,custom_fields.display_value` — plus a second narrow pass for `created_by.name`.

**Token ceiling.** `opt_fields` including `custom_fields` **exceeds the tool output limit at 50 tasks** (~65k chars, verified). At new-arrivals volume (~1/day) this will never bite. It does bite if the watermark is ever reset or the floor lowered — in that case let the result spill to a file and `jq` it rather than reading it inline.

## Step 2 — Drop the noise before counting anything

**30% of open top-level tickets have a null `Project Type`** (15 of 50 sampled). They are not one problem — they're three, and only one needs a human:

| Group | Signature | Action |
|---|---|---|
| **Blank orphans** | empty `name`, 0 sub-tasks, unassigned | Exclude. 5 of 15. Junk rows. |
| **Template placeholders** | name contains `[Event Name]` / `[Webinar Name]` / `[NAME]` | Exclude from triage; **count them**. 5 of 15. Un-instantiated template rows — a hygiene finding, not work. |
| **Test rows** | name contains `TESTING SKILL` or `[TEST RUN` | Exclude entirely, don't even count as hygiene. These are ours. |
| **Genuinely unclassified** | real name, no `Project Type` | **Report as needs-classification.** 5 of 15. e.g. "August Partner Newsletter", "SFDC Campaign Request - Bing PMAX - Trials". |

Report the counts separately. Collapsing them into "15 unclassified" overstates the problem by 3×.

> **Unclassified is a *new-submission* problem, not backlog rot.** Of the 5 genuinely unclassified tickets, **4 were created within the previous 4 days** (2 on 2026-08-03, 2 on 2026-07-31). So a new-arrivals-only digest will hit this immediately and often — it is likely to be the single most common finding, not an edge case.

## Step 3 — Bucket by Project Type

Group the survivors by `Project Type`. At new-arrivals volume this is usually 1–3 tickets, so the "bucket" is often a single row — that's fine, the grouping exists so the per-type sub-task rule in Step 5 can be applied, not to produce statistics.

For sanity-checking a run against the standing shape of the board (all 50 most recent *open* tickets, i.e. including backlog — **not** what a daily run should look like):

| Project Type | Count |
|---|---|
| `Email(s) only \| Nurture Sequences` | 14 |
| `IT/Integration` · `Audiences` | 4 each |
| `SFDC Campaign only (single)` | 3 |
| `Other` · `List Upload` · `Form Request` | 2 each |
| `Webinar Request` · `Event (+ SFDC Campaign)` · `SFDC Campaigns only (multiple)` · `Issues` | 1 each |

Email dominates — roughly 2× everything else combined. Expect it to dominate daily arrivals too.

## Step 4 — Propose an owner (Region-free)

### The constraint that shapes this whole step

`intake-routing` routes on **Region × Project Group × Stakeholder**. On the live board:

- **There is no Region field.** Not null — absent from the project's field set entirely.
- **`Requesting Team`** (the obvious proxy) is **49/50 null**.
- **`Requestor`** (the obvious requester field) is **50/50 null**.

So rules 2 (LATAM) and 5 (Region × Project Group) **cannot fire**. Do not synthesize a Region to make them fire.

### What does work: `created_by` is the requester

`created_by` is populated on **38 of 50** tickets, and the names land directly on `intake-routing`'s stakeholder lists:

| Creator seen live | Maps to |
|---|---|
| Zhenya Thornhill · Kim Bonilla · Shannon Bystock · Jessica Salvati · Samantha Wilding ("Sam") · Melissa Hopkinson | **Aayushi** |
| Katharine Shaw · Karen Plant | **Harish** |

That makes rule 4 — the table's strongest signal — usable. Use `created_by` as the requester wherever `Requestor` is null, and **say so in the output**, because it is an inference: the person who filed the ticket is usually but not always the stakeholder.

### The engine — first match wins

| # | Rule | Needs |
|---|---|---|
| **0** | `IT/Integration` / `Automations \| Martech` / `UAT` → **Jennifer**, and **no sub-tasks** | Project Type only ✅ |
| 1 | `Audiences` / `List Upload` / any segmentation or list pull → **Felipe** | Project Type only ✅ |
| 3 | Zoom Webinar / CVent / Alice integration work → **Harish** | content signal ✅ |
| 4 | Creator or named stakeholder matches an owner's list → **that owner** | `created_by` ✅ |
| 6 | `Form Request` / `Other` / `Reporting` / `Issues`, no stakeholder signal → **Aayushi** | Project Type only ✅ |
| — | anything else | ❌ **report `BLOCKED — needs Region`**, propose nothing |
| — | null Project Type | ❌ **report `BLOCKED — needs classification`** |

**Rule 0 is evaluated first** and overrides rule 6 for `UAT` (authorized by Forkan 2026-08-03). Rules 2 and 5 are deliberately omitted — they need Region. Rule 7 (ambiguous → don't guess) still governs: a tie proposes nothing.

⚠️ Rule 0 pulls `IT/Integration` out of the BLOCKED bucket, where it previously sat with no owner — 4 of the 50 sampled tickets. It doesn't relieve the Aayushi concentration meaningfully, though: `UAT` volume is ~0 (one completed task ever).

**Measured coverage** on the 50-ticket sample: **32 routable (64%)** · 12 need Region (24%) · 6 need classification (12%).

### ⚠️ Rule 4 makes the imbalance worse — this is the important part

Of the 32 routable tickets, the engine sends **24 to Aayushi, 6 to Felipe, 2 to Harish.** That is **75% to one person** — and `intake-routing` exists specifically to fix Harish/Aayushi overload.

This is not a bug in the engine; it reflects that Aayushi's six named stakeholders file most of the board's tickets. But it means:

- **The workload check is load-bearing, not advisory.** Run it every time (`search_tasks` by assignee, not-completed) and print each proposed owner's current open count next to the proposal.
- **When one owner takes >50% of a run's proposals, say so at the top of the digest as a headline.** Don't bury it per-ticket.
- **Still propose per the table.** Per `intake-routing`, the ownership model is not a suggestion — surface the imbalance, never silently rebalance. The digest's job is to make a human's rebalance decision easy, not to make it for them.

## Step 5 — Propose the sub-task set

Read the tier from `references/sops/mops-task-subtask-catalog.md`. **The tier decides the action, and for two types the correct action is to create nothing.**

| Project Type | Propose |
|---|---|
| `Webinar Request` | The full **20-step** template. Check host first — a 3rd-party-hosted webinar drops Zoom setup, dry run, and hosting. |
| `Event (+ SFDC Campaign)` | **Choose, then propose as a question.** 16-step DrupalCon-scale set, or the workshop variant (campaign + form + N×email creation/approval/send)? 5 of 15 real events legitimately had zero sub-tasks — so "none" is a valid answer. Never auto-expand this type. |
| `Email(s) only \| Nurture Sequences` | Count the sends, emit one triple per send: `Email N Creation` (`default_task`) · `Email N Approval` (`approval`) · `Email N Send` (**`milestone`**). Add the campaign-setup items once if no campaign exists. |
| `SFDC Campaigns only (multiple)` | **Nothing.** 13 of 15 had zero sub-tasks — the ticket *is* the work item. Propose owner only. |
| `Audiences` | **Nothing.** 9 of 12 had zero. Owner only. |
| `List Upload` | Exactly **one** — `Upload list into SF`. One per distinct batch, never more. |
| `SFDC Campaign only (single)` · `Form Request` · `Reporting` · `Other` · `Issues` | **Nothing** — leaf types. |
| `IT/Integration` · `Automations \| Martech` · `UAT` | **Nothing** — and the owner is **Jennifer** per rule 0. These have no repeatable pattern, so the manager scopes them; a proposed sub-task list would be invented, not derived. |

**`resource_subtype` is load-bearing.** Email sends must be `milestone` and approvals `approval`. A send proposed as `default_task` will never appear on the shared email send calendar. Include the subtype in every proposed sub-task line.

## Step 6 — Idempotency

The watermark handles cross-run duplication: a ticket reported yesterday is outside today's window. What the watermark can't catch is a ticket that arrived new but was *already triaged by a human* before the digest ran — which happens often, since ~40% of new tickets get an assignee on creation. So still check:

- **Owner** — ticket already has an assignee → **omit the owner proposal silently** (contract §4: already-assigned is a silent skip, not a note). Count it as "triaged already", don't argue with it.
- **Sub-tasks** — read the parent's existing sub-tasks, match by name, propose only the missing ones. A ticket that already has its full set produces no sub-task proposal. This matters more than it sounds: an automation is already expanding the event/workshop 5-set (see _Scope_), often within seconds of the parent being created.
- **`MOPS- Status`** — `Assigned` correlates 1:1 with having an assignee, so it's a reliable "already triaged" signal. Skip `Deprioritized` and `Blocked` entirely; they're parked on purpose.
- **Never advance the watermark on a failed run.** Re-reporting is harmless; skipping isn't.

## Step 7 — Output

Write **one file per run** to `outputs/intake/intake-dispatch-YYYY-MM-DD.md` — the canonical subfolder for intake deliverables (`.claude/rules/output-files.md`); `mkdir -p` it if missing, never write to `outputs/` root. Print the same content to stdout so it lands in the launchd log. Write the file even when the run is empty.

This is a local repo file, not a live-system write — no contract verb applies, nothing is notified, nothing goes in `decisions/actions.md`.

```
# Intake Dispatch — <date>

**Window:** <watermark ISO> → <run start ISO>
**<N> new submissions · <N> triaged already (skipped) · <N> proposed · <N> blocked**

⚠️ **Load imbalance:** <N> of <N> proposals → <owner> (<pct>%). Current open: Aayushi <n> · Harish <n> · Felipe <n>.
⚠️ **<N> need classification** — null Project Type, cannot be routed.
🧹 **<N> `[Event Name]` placeholder sub-tasks created this window** — the template automation's name substitution is failing.

## Proposals — new top-level submissions

### <ticket name>  ·  <Project Type>
- **Owner →** <name>   (rule <n>: <why>; requester from `created_by`)  ·  their load: <n> open
- **Priority →** <full picklist string>   (SLA base <type> = <n>d; escalation: <none|rush|date>)
- **Sub-tasks →** <n> proposed  (tier: <template|propose|formulaic|none>)
  1. <name>   [<resource_subtype>]
- **Writes this would need:** assign owner (Class B, notifies <name>) · set Priority (Class A) · create <n> sub-tasks (Class B, notifies followers)

## Secondary — new standalone sub-task requests
Owner proposal only; no sub-task expansion.
- <name> (parent: <parent name>) → **<owner>** (rule <n>)

## Excluded
- <N> template-generated sub-tasks (auto-expanded, not submissions)
- <N> test rows

## Blocked
- <ticket> — needs Region (rules 1/3/4/6 don't match)
- <ticket> — needs classification (null Project Type)
```

If the window is empty, the whole body is a single line: `No new submissions in this window.` Keep the header and window range — that's what makes the file series a run log.

Every proposal names its **reversibility class** so approving it is one decision, not a research task.

## Step 8 — Advance the watermark (do this last, and only on success)

Write `last_run_utc = run_start` (the Step 0 value) back to `routines/mops-intake-dispatch/state.json`, preserving `floor_date` and `_comment` untouched.

**This is the step that makes the routine stateful — skip it and every run re-reports the same window forever.**

Ordering is deliberate: the digest is written *first*, the watermark *second*. If the run dies between them, the next run re-covers the window and re-writes that day's digest — a duplicate report, which is harmless. Advancing the watermark first would risk a silently skipped day, which is not.

An **empty window still counts as success** — advance the watermark. Nothing arriving is a valid result, not a failure.

Do **not** advance it if: a `search_tasks` call errored, the fetch was truncated, or the digest wasn't written. Leave `state.json` alone and say so in the log, so the next run picks the window back up.

---

## Writes — there are none

Nothing in this skill writes to Asana, Salesforce, Jira, Slack, or Sheets. Consequences:

- **No `write-actions.md` §6 authorization is required**, which is why this can run unattended immediately.
- **Nothing is logged to `decisions/actions.md`** — that file is for executed writes. A proposal is not a write.
- When a human approves proposals, the writes run through `intake-routing` (owner/priority) and `intake-classification` (sub-tasks) in an attended session, and *those* get logged normally.

If this ever gains write authority, the assign and sub-task-create verbs are **Class B** (both notify people) — per-instance confirmation, or a new §6 row logged in `decisions/log.md` first.

---

## Known limits — state these in the digest, don't paper over them

1. **No Region signal exists on the board.** 24% of tickets can't be routed at all until Region is captured at intake, or until the Project-Type→owner bridge is confirmed well enough to route without it. This is the single highest-value fix and it's an intake-form change, not a code change.
2. **`created_by` ≠ stakeholder.** It's the best available proxy and it's an inference. A MOPS member filing on someone's behalf (Aayushi created one of the 50) breaks it.
3. **The Project Type → Project Group bridge is unconfirmed** (`intake-routing` says so itself). Rule 6's list is inference.
4. **Priority derivation is untested against real judgment.** The picklist is confirmed but nobody has checked that a derived `Medium` matches what Aayushi would have picked. 94% of open tickets have null Priority, so there's no baseline to compare against — the first weeks of digests *are* the calibration.
5. **`Out of SLA/Rush` is 38/50 null** — so rush escalation usually has to come from the SLA clock and target date, not the field.
6. **`MOPs - Project Subtype` is unexplored.** It may resolve several `Other` / `Form Request` ambiguities. Worth asking Harish.
