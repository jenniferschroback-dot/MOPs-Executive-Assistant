# Available Tools

_Last updated: 2026-08-03_

What's actually reachable in a Claude Code session right now, by MCP connection. This is a capability reference — see `.claude/skills/` for how these get used in an actual workflow.

## Connected & usable now

### Asana
Full task/project management: create, read, update, delete tasks; search tasks/objects; manage projects and portfolios; add comments and read attachments; look up users, teams, and workspace agents; post project status updates.

### Salesforce
**Read/query only right now** — find records, run SOQL queries, get object schema, get related records, get user info, list recent records for an object.
**No create/update/delete tool is exposed yet.** This means the `sf-campaign-spec` skill's "create the campaign record via Salesforce MCP" step currently has no write path — campaign records still need to be created manually (or this needs a different/expanded Salesforce connection) until that changes.

### Atlassian (Jira + Confluence)
- **Jira:** create/edit/transition issues, add comments and worklogs, link issues, look up issue types and project metadata, run JQL searches.
- **Confluence:** create/update pages, read pages and their descendants, comment on pages, search spaces via CQL.
- Also includes Compass (component catalog) and Teamwork Graph (cross-entity relationship context) tools.

### Slack
Read channels, threads, files, and user profiles; search channels/users/messages; add reactions; send or schedule messages and drafts; create/read/update canvases.

### Google Drive
Search and list files, read/download file content, get file metadata and permissions, create and copy files.
**Cannot append to or update an existing file** — `create_file` only creates. It also requires the whole file inline as base64, which stalls on anything non-trivial. For uploads and for any Sheets/Slides/Docs edit, use the `gws` CLI below instead.

### `gws` CLI — not MCP, but the widest Google surface available
Installed and authed locally (keyring backend) at `~/.nvm/versions/node/*/bin/gws`. Services: `drive`, `sheets` (read **and write**), `docs`, `slides`, `gmail`, `calendar`, `admin-reports`. Introspect any method with `gws schema <service.resource.method>`.

This is the **only** path to Google Sheets writes — there is no Sheets MCP connector, and none exists in the registry. Used by `acquia-brand-deck` (Drive upload + Slides thumbnails) and `lead-routing-audit` (daily Sheet append).

**Two constraints that bite:**
- `--upload` paths must resolve inside the current working directory — `cd` into the project first.
- **Credentials are local to Forkan's machine.** A cloud routine has neither the binary nor the keyring, so any routine depending on `gws` must run locally or be given its own credentials. Don't assume a scheduled cloud agent can reach Sheets.

## Awaiting authorization (visible, not yet usable)
These connectors show up as available but need the user to complete authorization first — via claude.ai connector settings, or `/mcp` in an interactive Claude Code session:
- Acquia DAM (ADAM)
- Asana (a second, separate Asana connector beyond the one already authorized)
- Box
- Canva
- Conductor
- Figma
- G2
- Gamma
- Gmail
- Google Calendar
- HubSpot
- Intercom
- Linear
- Miro
- Notion
- Porter
- Supermetrics Marketing Analytics
- Vimeo
- Windsor Custom
- Zapier
- Zoom for Claude / zoom-mcp
- monday.com

## Used daily, no MCP connection
- **Pardot** — part of the daily workflow but has no MCP server connected. Flag any task that needs Pardot access rather than assuming it's reachable.

## Maintenance
This list reflects tool access, which can change independently of anything in `context/` or `.claude/skills/`. Re-check it (or ask to re-check it) if a skill starts failing on a step that assumes a tool it doesn't actually have — like Salesforce writes above.
