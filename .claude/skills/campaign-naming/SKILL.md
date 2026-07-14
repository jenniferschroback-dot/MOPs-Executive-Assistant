---
name: campaign-naming
description: Proposes a standardized campaign name from classified intake data for use in Asana tasks and Salesforce campaign records. Use after intake-classification, whenever a campaign or task needs a name before it's created in Asana or Salesforce.
---

# Campaign Naming

## Naming convention (confirmed 2026-07-10, see `decisions/log.md` — supersedes the 2026-07-08 `type_subtype_region_description_year_quarter` format)

```
Region_Channel_Product_Description_YYYY-Qn
```

Segments: `Region` (geo scope), `Channel` (campaign channel/type, e.g. event, email, webinar), `Product` (product/segment the campaign is for), `Description` (free text), `YYYY-Qn` (year-quarter, e.g. `2026-Q3`).

**Known codes** (only what's been confirmed so far — extend this table as new ones are confirmed, don't invent codes):

| Segment | Confirmed values |
|---|---|
| Region | `All` = all regions |
| Channel | `Email` = every one-off/promotional email type (promo, post-webinar, post-event, or any other one-off send) — they all share the same `Email` channel code, don't split them further. `Nurture` (exact code still open — Harish suggested `NURT`, "or anything you find suitable" — **not locked yet, confirm the short code with Harish before using it**) = nurture-sequence email requests only (`Project Type` = `Email(s) only \| Nurture Sequences` with subtype Nurture). Everything else not yet confirmed — ask. |
| Product | not yet confirmed — ask |

## Process

1. Take the classified intake data (request type, product/segment, region, description, target date).
2. Map each field to its code using the table above.
3. **If a needed Region/Channel/Product code isn't in the table yet, ask what code to use** rather than inventing one — then add the confirmed code to the table above so it's reusable.
4. Build the name in the exact segment order: `Region_Channel_Product_Description_YYYY-Qn`.
5. **Cap the `Description` segment length.** Harish flagged (2026-07-13) that Salesforce's Campaign Name field has a hard character limit, and stakeholder-provided event/description names sometimes run long enough to blow past it — his manual workaround was silently truncating (e.g. "Leading University..." → "Leading university"), which loses information. Exact cap isn't finalized (he suggested 40–50 chars); until it's confirmed, if the full name would be unusually long, propose a shortened `Description` and confirm it with MOPS/the stakeholder rather than truncating silently.
6. **Dual-campaign naming (Webinar/Event requests with a companion email):** when `sf-campaign-spec` is creating a paired Event/Webinar Campaign + Email Campaign for the same request (see that skill's gate), give both the identical name except the `Channel` segment — one uses `Event`/`Webinar`, the other uses `Email`. That's what lets the pair be visually recognized as belonging to the same initiative.
7. Show the generated name before writing it into Asana or Salesforce — a quick sanity check, not a full re-negotiation of the format.
8. Use the same name consistently for the Asana task/sub-tasks and the Salesforce campaign record so they match.

## Why this matters
Naming drift is a named MOPS pain point — inconsistent names break Salesforce reporting and attribution. The format itself is now fixed; the only open work is filling in the code table as new types/subtypes/regions come up, locking the Nurture code, and settling the exact Description character cap.
