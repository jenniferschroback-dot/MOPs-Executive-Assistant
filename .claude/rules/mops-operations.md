# MOPS Operations

- Asana, Salesforce, Jira, and Slack MCP servers are connected — use them directly to look up, create, or update records instead of asking the user to do it manually.
- Pardot is part of the daily workflow but has no MCP connection yet — flag when a task needs Pardot access.
- No fixed campaign naming convention exists yet. Until one is defined and logged in @decisions/log.md, do not invent one silently — ask before creating Asana tasks or Salesforce campaign records with a generated name.
- Campaign member status scaffolding must match the campaign type; when creating or updating a Salesforce campaign, check what statuses that campaign type requires before assuming defaults.
