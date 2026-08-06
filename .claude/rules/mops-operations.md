# MOPS Operations

- Asana, Salesforce, Jira, and Slack MCP servers are connected — use them directly to look up records instead of asking the user to do it manually.
- **Any write** (create, update, delete, transition, send, schedule) follows `.claude/rules/write-actions.md` — the standing contract covering the allowed verb list, confirmation by reversibility class, bulk previews, idempotency checks, no-auto-retry, unattended-send authorization, and the audit log at `decisions/actions.md`. It applies identically from chat, a skill, a dashboard button, or a routine.
- Salesforce is **query-only** — no write path exists. Pardot has no MCP connection at all. Requests needing either produce a spec plus a handoff, never a completion claim (see the contract, §9).
- Campaign naming convention is fixed: `Region_Channel_Product_Description_YYYY-Qn`. See `.claude/skills/campaign-naming/SKILL.md` for the code table. If a Region/Channel/Product code isn't confirmed yet, ask rather than inventing one.
- Campaign member status scaffolding must match the campaign type; when creating or updating a Salesforce campaign, check what statuses that campaign type requires before assuming defaults.
