# MOPS Operations

- Asana, Salesforce, Jira, and Slack MCP servers are connected — use them directly to look up, create, or update records instead of asking the user to do it manually.
- Pardot is part of the daily workflow but has no MCP connection yet — flag when a task needs Pardot access.
- Campaign naming convention is fixed: `type_subtype_region_description_year_quarter` (e.g. `evt_ws_all_dam workshop boston_2026_q3`). See `.claude/skills/campaign-naming/SKILL.md` for the code table. If a type/subtype/region code isn't confirmed yet, ask rather than inventing one.
- Campaign member status scaffolding must match the campaign type; when creating or updating a Salesforce campaign, check what statuses that campaign type requires before assuming defaults.
