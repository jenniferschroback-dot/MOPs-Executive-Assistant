---
name: audience-pull
description: Builds a target audience from Salesforce — Contacts and/or Leads filtered by region, product interest, segment, persona/title and engagement — applies the mandatory mailability suppression, and delivers it as a CSV in outputs/audiences/ with a reproducible query spec. Use for any Audiences intake ticket, a target-list or contact-list pull, or an event/webinar invite list.
argument-hint: [Asana ticket URL/id, or a plain-English audience description]
---

# Audience Pull

Turns an `Audiences` request into an actual list. Queries Salesforce, applies suppression, writes `outputs/audiences/audience-<slug>-YYYY-MM-DD.csv` plus a companion `.md` spec so the pull is reproducible and auditable.

**This is the one intake type that can genuinely be finished here**, because the deliverable is data and Salesforce reads are unrestricted. Everything else campaign-shaped stops at spec-plus-handoff.

**Owner is always Felipe** (`intake-routing` rule 1 — segmentation, all regions). **No campaign, no name, no sub-tasks** — `Audiences` is a leaf type with no SF Campaign gate.

## Scope boundary — read this first

| | |
|---|---|
| ✅ **Can do** | Query SF, compute the audience, export rows to CSV, report size + suppression breakdown |
| ❌ **Cannot do** | Create a Salesforce Report, add Campaign Members, or create/update a Pardot list — all writes. Pardot has **no connector at all** |

So a request like *"create a competitor suppression list in Pardot"* (a real past ticket) produces the member list plus a handoff, never a completion claim (`write-actions.md` §9).

---

## Step 1 — Resolve the request into filter dimensions

Read the ticket (description **and** comments — audience criteria are usually in a comment or linked doc). Extract:

| Dimension | Ask if missing |
|---|---|
| **Object** — Contacts, Leads, or both | Default to **Contacts** and say so; Leads are a different field set (see §Asymmetry) |
| **Region** | Often implied by the requesting team |
| **Product interest** | The strongest available filter |
| **Segment / company size** | Lead-only field — see below |
| **Persona / title** | `Title` is 86% populated |
| **Engagement** | ⚠️ read §Engagement before promising this |
| **Exclusions** | Existing campaign members? Competitors? Customers vs prospects? |
| **Size expectation** | A pull returning 74k when they expected 500 means the criteria were misread |

**Never invent criteria.** If the request says "our EMEA marketing audience" with nothing else, propose an interpretation and get it confirmed before exporting — an audience is a mailing list, and a wrong one gets sent to real people.

## Step 2 — Build the query from validated fields

All fields below verified live 2026-08-04 against the org. Population rates are from a real 74,530-contact cohort.

### Core selectors — Contact

| Field | Type | Notes |
|---|---|---|
| `Region__c` | Formula (Text) | ⚠️ **Filterable with `=`, NOT groupable** — `GROUP BY Region__c` errors with `field 'Region__c' can not be grouped`. Values seen: `EMEA`, others. |
| `MailingCountry` | Text | Indexed and groupable — **prefer this for filtering**, use `Region__c` for display |
| `Product_Cloud_Interest__c` | Picklist | **The best filter.** 55% populated. Values + counts: `Acquia Source` 278,797 · `DXP` 135,717 · `Marketing Cloud` 111,840 · `Drupal Cloud` 56,760 · `Content Cloud` 50,699 · `DXO` 23,615 · `Monsido` 6,188 · null 552,872 |
| `Product_Interest__c` · `Product_Interest_Multi__c` | Multi-select | Two *additional* product fields exist. Three total — confirm which the requester means before assuming |
| `Title` | Text(128) | 86% populated — viable for persona targeting |
| `DAM_Buyer_Persona__c` · `DAM_UX_Personas__c` · `Partner_Persona__c` | Picklist | Explicit persona fields |
| `X6sense_Segments__c` | Long Text(10000) | Segment membership. Use `LIKE '%name%'` |

### Asymmetry — Contact vs Lead

Do **not** assume the same fields exist on both:

| | Contact | Lead |
|---|---|---|
| `Business_Segment__c` | ❌ absent | ✅ Formula (Text) |
| `Most_Recent_Product_Cloud_Interest__c` | ❌ | ✅ Formula |
| `Subscribed_to_*` (6 fields) | ✅ | ❌ |
| `DAM_Opted_out_of_*` (6) + `DAM_Unsubscribed_from_ALL_Email__c` | ✅ | ❌ |
| `Region__c`, `Product_Cloud_Interest__c`, `pi__*`, `HasOptedOutOfEmail` | ✅ | ✅ |

A both-objects pull needs **two queries with different WHERE clauses**, unioned in the CSV with a `Source` column (`Contact` / `Lead`). Never one query.

⚠️ **Never run `getObjectSchema('Lead')`** — 300 fields / 132KB, blows the token cap. Use `FieldDefinition` instead:
```sql
SELECT QualifiedApiName, Label, DataType FROM FieldDefinition
WHERE EntityDefinition.QualifiedApiName = 'Contact' AND QualifiedApiName LIKE '%Region%'
```

## Step 3 — Suppression is mandatory, not a filter

**Always applied. Never optional. Never removed to hit a size target.** This is the step that keeps a pull from mailing someone who opted out.

Minimum clause for **Contact**:
```sql
Email != null
AND HasOptedOutOfEmail = false
AND IsEmailBounced = false
AND pi__pardot_hard_bounced__c = false
```

For **Lead**, drop `IsEmailBounced` if absent and keep the rest.

**Measured impact:** 111,840 Marketing Cloud contacts → **74,530 mailable**. That's **33% removed**. Suppression is not a rounding error.

**Consider adding**, depending on the send:
- `Subscribed_to_ALL_email_types__c = true` — for broad marketing sends
- `DAM_Unsubscribed_from_ALL_Email__c = false` and the relevant `DAM_Opted_out_of_*` — for any DAM/Widen audience
- `Email_Validation_Status__c` — if deliverability is a concern
- `Employment_Status__c` — to drop known job-changers

**Report the funnel in the spec file**, every time:
```
matched criteria        111,840
− opted out / bounced    37,310  (33.4%)
= mailable               74,530
```

### The opt-out clause is not enough — screen the rows too

**Validated 2026-08-04 on a real 59-row pull: 12 rows (20%) had defects that every opt-out filter passes.** These are not theoretical. Screen for them and emit a `Flag` column; keep the rows so exclusions are auditable rather than silently dropped.

| Flag | Detection | Action |
|---|---|---|
| `EXCLUDE_COMPLIANCE_REMOVED` | `Email LIKE '%removedforcompliance%'`, or name contains `Removed per Request` | **Hard exclude.** These are GDPR-erased records and they pass every opt-out check. Mailing one is the worst outcome this skill can cause. |
| `EXCLUDE_LEFT_ORG` | `FirstName`/`LastName` contains `LEFT`, `(left org)`, `left org` | **Hard exclude** — person has left, mail will bounce or reach a stranger |
| `EXCLUDE_INVALID_EMAIL` | `Email LIKE '%.invalid'` | **Hard exclude** — deliberately invalidated, yet `IsEmailBounced` stays `false` |
| `ACCOUNT_MISMATCH` | Email domain inconsistent with `Account.Name` | Warn. Seen at 5/59, and 4 of those pointed at **Australian** accounts on Dutch contacts — a bad bulk association, not noise |
| `ENCODING_CORRUPT` | Non-UTF8 replacement chars in name/title | Warn — renders broken in a greeting. Worth fixing before a personalised send |
| `POSSIBLE_DUP` | Same surname + same email local-part across different domains | Warn |
| `ROLE_ACCOUNT` | Shared-inbox pattern (`info@`, `ondernemersinformatie@`) stored under a person's name | Warn — don't personalise |

Report the extra funnel step:
```
+ criteria narrowed      59
− flagged hard-exclude    6
= clean mailable         53
```

**Screening cannot be done by SOQL alone** for most of these — the name-based patterns need the rows in hand. That's a reason to prefer pulls small enough to actually export (§Step 5), and a reason never to treat a `COUNT(Id)` as the deliverable size.

## Step 4 — Engagement: the obvious field is the wrong one

⚠️ **Pardot's engagement fields are effectively dead in Salesforce.** Measured on the 74,530-contact cohort:

| Signal | Populated | Verdict |
|---|---|---|
| `X6sense_Engagement_Score__c` | 74,433 — **99.9%** | ✅ best coverage |
| **`CampaignMember` in last 365d** | 62,337 — **83.6%** | ✅ **best real signal** — actual campaign response history |
| `Behavior_Score__c` | 52,055 — **69.8%** | ✅ usable |
| `pi__score__c` > 0 | 9,988 — 13.4% | ⚠️ thin |
| `Groove_Last_Engagement__c` | 2,141 — 2.9% | ❌ |
| `pi__last_activity__c` | 638 — **0.9%** | ❌ |
| `pi__grade__c` | **0 — 0%** | ❌ **completely empty** |

This is counterintuitive — Pardot *is* the email platform, so `pi__last_activity__c` and `pi__grade__c` look like the natural engagement filters. They aren't; the sync isn't bringing activity data across. **Never filter on `pi__grade__c`** (it will return zero rows) and treat `pi__last_activity__c` as unusable.

Prefer campaign-response history:
```sql
Id IN (SELECT ContactId FROM CampaignMember WHERE CreatedDate = LAST_N_DAYS:365)
```
For *responded* rather than merely *targeted*, add `AND HasResponded = true` to the subquery.

## Step 5 — Size check before exporting

Run `COUNT(Id)` first, always. Then pick a path by size — this is a hard limit, not a preference:

| Size | Path |
|---|---|
| **≤ 500** | Export directly. Comfortable in one or two calls. |
| **500 – 2,000** | Export via keyset pagination (below). Warn that it costs several calls. |
| **> 2,000** | **Do not export rows.** Deliver the count, the exact SOQL, and the suppression funnel — then hand off to a Salesforce report export or Data Loader. |

**Why the ceiling:** query results return through the tool into context, and a 50-task Asana result already hit ~65k chars. A 74,530-row audience cannot come back this way at any page size. `OFFSET` also caps at **2,000** in SOQL, so deep paging isn't available.

**Keyset pagination** (the only unbounded method):
```sql
... AND Id > '<lastIdFromPreviousPage>' ORDER BY Id ASC LIMIT 200
```
Repeat until a page returns fewer than the limit. Never `OFFSET`.

If the count exceeds 2,000, say so plainly and ask whether to tighten criteria or hand off — don't silently truncate to the first 2,000, which produces a list biased by record Id (i.e. by creation order).

## Step 6 — Write the CSV

Save the raw query JSON to the scratchpad, then convert with Python's `csv` module — never hand-assemble CSV, because titles and account names contain commas and quotes.

**Files, both in `outputs/audiences/`** (the canonical subfolder for this skill — `.claude/rules/output-files.md`; `mkdir -p` it if it doesn't exist, and never write to `outputs/` root):
- `audience-<slug>-YYYY-MM-DD.csv` — the list
- `audience-<slug>-YYYY-MM-DD.md` — the spec: ticket link, criteria as agreed, **the exact SOQL**, the suppression funnel, row count, and any caveat

The `.md` is what makes the pull reproducible and reviewable. A CSV with no spec is unauditable — nobody can tell later what "EMEA marketing audience" meant.

**Standard columns:** `Source` (Contact/Lead) · `Id` · `FirstName` · `LastName` · `Email` · `Title` · `AccountName` · `Region__c` · `MailingCountry` · `Product_Cloud_Interest__c` · plus whichever engagement field was used.

Always include `Id` — it's what makes the list loadable back into Salesforce or Pardot by whoever does the import.

**These are local repo files, not live-system writes.** No contract verb applies, nothing is notified, nothing goes in `decisions/actions.md`.

⚠️ **The CSV contains personal data** (names, emails, employers). It lands in a git repo — don't commit it without checking, and don't attach it anywhere outside Acquia.

## Step 7 — Report

```
**Audience:** <name>   ·  ticket <link>
**Criteria:** <as agreed, explicitly>
**Object(s):** Contact | Lead | both

matched criteria   <n>
− suppressed       <n>  (<pct>%)
= mailable         <n>

**Engagement signal:** <field> (<coverage>% populated)
**Delivered:** outputs/audiences/audience-<slug>-<date>.csv  (<n> rows)  ·  spec alongside
**Owner:** Felipe (rule 1)
**Not done:** <SF report / campaign members / Pardot list — handoff, per §9>
```

---

## Routine mode — daily sweep (`$ARGUMENTS` = `--sweep`)

Runs unattended at 06:00 PT via `routines/mops-audience-pull/`. Same skill, one behavioural change that matters: **an unattended run may never invent criteria.** Attended, a missing dimension is a question; unattended, there is nobody to ask, so the ticket is reported as blocked rather than guessed at.

### Detection — do not key on `Project Type` alone

Measured on the live board 2026-08-04: **`Project Type = Audiences` is set on 1 open ticket.** Twelve other audience-shaped tickets in the same window carry `Project Type = null`. Keying on the picklist alone finds almost nothing.

Match a new ticket if **either**:
- `Project Type` = `Audiences` (field gid `1206591746930193`), **or**
- the name matches `Audience Report` · `Audience Pull` · `Target List` · `Contact List` · `Audience Request` (case-insensitive)

Report which test matched, per ticket. A name-matched ticket with a null `Project Type` is itself a finding — the classification gap, not just an audience request.

### Per-ticket outcome — four states, and only one produces a CSV

| Outcome | When | What gets written |
|---|---|---|
| `READY` | Every dimension in Step 1 is explicit in the description or comments | Full pull: CSV + `.md` spec, exactly as attended |
| `NEEDS_INPUT` | Any dimension is missing or admits more than one reading | **Spec stub only, no CSV.** List the exact questions and the SOQL that *would* run once answered |
| `BLOCKED` | `[Event Name]` placeholder, or no parent event resolvable | No CSV, no spec. Name the blocker |
| `TOO_LARGE` | `COUNT(Id)` > 2,000 (Step 5) | Count + exact SOQL + funnel + handoff. **Never a truncated CSV** |

**`NEEDS_INPUT` is the expected outcome, not the failure case.** An audience is a mailing list; a CSV built from a guessed reading of "our EMEA audience" is worse than no CSV, because it looks finished. Prefer the stub every time the reading is not forced.

Suppression (Step 3) and row screening are **not** relaxed for being unattended — they are the mandatory part.

### Digest

Write `outputs/audiences/audience-sweep-YYYY-MM-DD.md` every run, **including when nothing new arrived** — a missing file is ambiguous, an empty one isn't. Per ticket: link, matched-by, outcome, and either the delivered filenames or the exact blocker. Head the digest with the four outcome counts and the `[Event Name]` placeholder count.

### Hard limits on the unattended run

- **Zero live-system writes.** No Asana comment, no assignment, no Salesforce anything. Salesforce is read-only regardless (`write-actions.md` §9); Asana writes are simply out of scope for this routine. Nothing goes in `decisions/actions.md`.
- **New arrivals only**, via `state.json` watermark + floor date. Never lower the floor to sweep the backlog.
- **Advance the watermark to the run's start time, and only on success.** A failed run leaves it, so the next run re-covers the window. Re-reporting is harmless; skipping isn't.
- **The CSVs contain personal data** and land in a git repo. Unattended generation makes that accumulate without anyone deciding to — see the gitignore note in the routine README.

---

## Worked example

`outputs/audiences/audience-emea-marketingcloud-nl-2026-08-04.{csv,md}` is a real validated run — EMEA · Marketing Cloud · Netherlands · has Title · campaign activity in 365d → 59 rows, 53 clean after flags. Use its spec file as the template for the `.md` companion.

Verified in that run: 59 unique Ids, 59 unique emails, 0 malformed rows, comma-bearing titles and account names correctly quoted.

## Known gaps

1. **No SLA exists for `Audiences`.** The SOP's per-type table has no row for it — closest analogue is `List Upload` (3–7 days, complexity-dependent). Priority derivation is a guess until someone sets one. Flag it rather than asserting a due date.
2. **Effort varies ~10× within the type** — a webinar audience selection is a filter; `GTM Plays Audience Orchestrations - Contact Data Enrichment` was a 5-sub-task project. Read the description; don't size from the Project Type.
3. **`[Event Name]` placeholder tickets.** Two of 12 sampled were literally named `[Event Name] Pre-Event Email Audience Report - MOPs` — the template automation's substitution bug. For an audience pull this is blocking, not cosmetic: the ticket doesn't say which event. Go to the parent.
4. **Three product-interest fields** (`Product_Cloud_Interest__c`, `Product_Interest__c`, `Product_Interest_Multi__c`) with no documented distinction. Ask Felipe which is authoritative.
5. **`Region__c` is a formula** — filterable but unindexed, so large pulls filtering on it may be slow. Prefer `MailingCountry` where the mapping is known.
