# Decision Log

Append-only. When a meaningful decision is made, log it here.

Format: [YYYY-MM-DD] DECISION: ... | REASONING: ... | CONTEXT: ...

---

[2026-07-08] DECISION: Built initial skill scaffolds for intake-classification, campaign-naming, sf-campaign-spec, and intake-tracking, with unconfirmed schema/convention details explicitly flagged rather than guessed. | REASONING: No campaign naming convention or Salesforce record type/status schema was confirmed yet; guessing silently would repeat the exact "naming drift" and "wrong status scaffolding" problems these skills are meant to fix. | CONTEXT: First skills built after initial MOPS assistant setup; refine each skill's placeholders as real Asana/Salesforce schema and a naming convention get confirmed through use.
