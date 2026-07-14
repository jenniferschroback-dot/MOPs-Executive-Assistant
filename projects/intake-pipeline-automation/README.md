# Intake Pipeline Automation

Automates the MOPS intake process: classifies incoming intake forms, generates standardized campaign names, and creates the Salesforce campaign spec (including status scaffolding) — replacing manual re-entry into Asana and Salesforce.

**Status:** Active

**Key dates:** Target completion this week (as of 2026-07-08).

## Sequencing decision (2026-07-13, per Harish)
Build order agreed with Harish: **campaign creation form first**, then layer the AI naming/classification automation onto webinar requests, then event requests. Reasoning: right now stakeholders submit email/campaign requests inconsistently (ticket comments, a shared doc, or nothing standardized), and the AI can't reliably automate on top of an unstandardized intake — the form has to exist and be consistently used before the automation is trustworthy.

**Campaign creation form — mandatory fields (per Harish):**
- Campaign name (system-generated, not stakeholder-entered)
- Type (stakeholder-provided — maps to `Project Type` in `intake-classification/SKILL.md`)
- PCI flag
- China flag
- Currency (defaults to USD, rarely needs override)
- Campaign region (existing SF picklist)
- Program type (Brand Awareness / Market Development / Sale Acceleration / Partner)
- For Webinar/Event requests: a yes/no "do you require a promotional email as part of this request?" question (mirrors the existing webinar landing-page form) — drives the dual-campaign gate in `sf-campaign-spec/SKILL.md`.
- For Email-channel requests: support for more than one send per submission (first send's subject/pre-header/body required; additional sends addable) — see `email-send-calendar/SKILL.md`.

**Ownership:** Priyanka is building the Asana-native form itself (as of 2026-07-13). Forkan is exploring whether an Asana-MCP-driven form (built via Claude Code, working directly off Asana sub-tasks) could run this same flow instead/alongside — undecided as of this meeting, follow up with Priyanka before committing to one path.

**Open gap:** no documented reference exists yet for the full sub-task template per request type (Webinar, Event, Email) — the "Known request types" table in `intake-classification/SKILL.md` is the closest thing today but is explicitly marked "not yet MOPS-confirmed as the fixed standard." Building this out (possibly with Samantha, who worked on the original MOPs intake form) is still open.

See `transcripts/Harish_Forkane meeting .txt` (2026-07-13) for the full discussion this is drawn from.

## Reference sources
Check in this order when you need info on the intake pipeline automation:
1. This project (`context/`, `decisions/log.md`, this README)
2. Obsidian vault (access pending)
3. GitHub repo: https://github.com/forkanelebdi-ACQ/mops-ai-automation-routines — only if the info isn't in the two sources above
