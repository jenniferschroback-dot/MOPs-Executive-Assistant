---
name: campaign-naming
description: Proposes a standardized campaign name from classified intake data for use in Asana tasks and Salesforce campaign records. Use after intake-classification, whenever a campaign or task needs a name before it's created in Asana or Salesforce.
---

# Campaign Naming

## Naming convention (confirmed 2026-07-08, see `decisions/log.md`)

```
type_subtype_region_description_year_quarter
```

Example: `evt_ws_all_dam workshop boston_2026_q3`
(type=`evt` event, subtype=`ws` workshop, region=`all`, description=`dam workshop boston`, year=`2026`, quarter=`q3`)

**Known codes** (only what's been confirmed so far — extend this table as new ones are confirmed, don't invent codes):

| Segment | Confirmed values |
|---|---|
| type | `evt` = event |
| subtype | `ws` = workshop |
| region | `all` = all regions |

## Process

1. Take the classified intake data (request type, product/segment, region, description, target date).
2. Map each field to its code using the table above.
3. **If a needed type/subtype/region code isn't in the table yet, ask what code to use** rather than inventing one — then add the confirmed code to the table above so it's reusable.
4. Build the name in the exact segment order: `type_subtype_region_description_year_quarter`.
5. Show the generated name before writing it into Asana or Salesforce — a quick sanity check, not a full re-negotiation of the format.
6. Use the same name consistently for the Asana task/sub-tasks and the Salesforce campaign record so they match.

## Why this matters
Naming drift is a named MOPS pain point — inconsistent names break Salesforce reporting and attribution. The format itself is now fixed; the only open work is filling in the code table as new types/subtypes/regions come up.
