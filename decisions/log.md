# Decision Log

Append-only. When a meaningful decision is made, log it here.

Format: [YYYY-MM-DD] DECISION: ... | REASONING: ... | CONTEXT: ...

---

[2026-07-08] DECISION: Built initial skill scaffolds for intake-classification, campaign-naming, sf-campaign-spec, and intake-tracking, with unconfirmed schema/convention details explicitly flagged rather than guessed. | REASONING: No campaign naming convention or Salesforce record type/status schema was confirmed yet; guessing silently would repeat the exact "naming drift" and "wrong status scaffolding" problems these skills are meant to fix. | CONTEXT: First skills built after initial MOPS assistant setup; refine each skill's placeholders as real Asana/Salesforce schema and a naming convention get confirmed through use.

[2026-07-08] DECISION: Campaign naming convention is `type_subtype_region_description_year_quarter` (e.g. `evt_ws_all_dam workshop boston_2026_q3`). | REASONING: Forkan confirmed this is the actual format to use, replacing the draft/no-convention state. | CONTEXT: Only `evt` (type), `ws` (subtype), and `all` (region) are confirmed codes so far — see `.claude/skills/campaign-naming/SKILL.md` for the growing code table.
