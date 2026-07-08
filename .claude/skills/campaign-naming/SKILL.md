---
name: campaign-naming
description: Proposes a standardized campaign name from classified intake data for use in Asana tasks and Salesforce campaign records. Use after intake-classification, whenever a campaign or task needs a name before it's created in Asana or Salesforce.
---

# Campaign Naming

There is **no fixed naming convention yet** (per @context/work.md and @.claude/rules/mops-operations.md). This skill's job right now is to propose a consistent, sensible name and get it confirmed — not to silently enforce a convention that doesn't exist.

## Process

1. Take the classified intake data (request type, product/segment, region, target date).
2. Propose a name using a simple, readable pattern, e.g.:
   `[RequestType]_[Product/Segment]_[Region]_[YYYYMM]`
   (This exact pattern is a starting guess — treat it as a draft, not policy.)
3. **Always show the proposed name and ask for confirmation** before writing it into any Asana task or Salesforce campaign record.
4. Once a name is approved:
   - Use it consistently for both the Asana task/sub-tasks and the Salesforce campaign record so they match.
   - If the user corrects the pattern (not just the specific name), log the corrected pattern to `decisions/log.md` as the emerging convention, e.g.:
     `[YYYY-MM-DD] DECISION: Campaign naming convention is X | REASONING: ... | CONTEXT: ...`
5. Once a convention has been logged in `decisions/log.md`, use it going forward instead of re-proposing from scratch each time — check the log first.

## Why this matters
Naming drift is a named MOPS pain point — inconsistent names break Salesforce reporting and attribution. This skill exists to converge on one convention over a few real uses, not to guess forever.
