# MOPS Operations

- Asana, Salesforce, Jira, and Slack MCP servers are connected — use them directly to look up, create, or update records instead of asking the user to do it manually.
- Pardot is part of the daily workflow but has no MCP connection yet — flag when a task needs Pardot access.
- Campaign naming convention is fixed: `Region_Channel_Product_Description_YYYY-Qn`. See `.claude/skills/campaign-naming/SKILL.md` for the code table. If a Region/Channel/Product code isn't confirmed yet, ask rather than inventing one.
- Campaign member status scaffolding must match the campaign type; when creating or updating a Salesforce campaign, check what statuses that campaign type requires before assuming defaults.
