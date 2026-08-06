# Write Actions — Standing Contract

Every write the assistant performs goes through this, regardless of entry point: chat, a skill, a dashboard button, or an unattended routine. One definition — nothing re-derives its own guardrails.

**Applies to:** any tool call that creates, updates, deletes, transitions, sends, or schedules. Reads are unconstrained.

---

## 1. Verb registry

Only these writes are in scope. A request needing anything else → describe what it would take and ask; don't improvise a new write.

### Asana — full write
| Verb | Tool | Class |
|---|---|---|
| Update task fields (due date, Priority, MOPS-Status, custom fields) | `update_tasks` | A |
| Mark complete / incomplete | `update_tasks` | A |
| Assign or reassign an owner | `update_tasks` | B — notifies the assignee |
| Create task / sub-task / milestone | `create_tasks` | B — notifies followers |
| Comment on a task | `add_comment` | B |
| Post a project status update | `create_project_status_update` | B |
| Create a project | `create_project` | B |
| Delete a task | `delete_task` | **C** |

### Jira (project `MOPS`) — full write
| Verb | Tool | Class |
|---|---|---|
| Edit issue fields | `editJiraIssue` | A |
| Add a worklog | `addWorklogToJiraIssue` | A |
| Link issues | `createIssueLink` | A |
| Comment on an issue | `addCommentToJiraIssue` | B |
| Create an issue | `createJiraIssue` | B |
| Transition an issue (any hop) | `transitionJiraIssue` | **C** |
| Transition to Done | `transitionJiraIssue` | **C** — terminal |

**Why every MOPS transition is Class C:** this project's workflow is forward-only and linear (To Do → In Progress → In Review → Done) and **Done is terminal — no reopen transition exists** (verified against closed Ticket and Sub-task issues, both returning zero transitions). There is no backward hop, so no transition can be undone through the connector. Done also requires a `resolution` (Done / Won't Do / Outdated = ids 10000 / 11010 / 6).

### Confluence — full write
| Verb | Tool | Class |
|---|---|---|
| Update a page | `updateConfluencePage` | A — versioned, restorable |
| Create a page | `createConfluencePage` | B |
| Comment on a page | `createConfluenceFooterComment` / `createConfluenceInlineComment` | B |

### Slack — full write
| Verb | Tool | Class |
|---|---|---|
| Add a reaction | `slack_add_reaction` | A |
| Update a canvas | `slack_update_canvas` | A |
| Create a canvas | `slack_create_canvas` | B |
| Send a message to a channel | `slack_send_message` | **C** |
| DM a person | `slack_send_message` | **C** |
| Schedule a message | `slack_schedule_message` | **C** |

**Why Slack sends are Class C:** no delete-message or cancel-scheduled-message tool is exposed. Once sent, the assistant cannot unsend it. `slack_send_message_draft` is likely gentler but its exact delivery behavior has not been verified — treat as B and verify before relying on it.

### Google Drive — write
| Verb | Tool | Class |
|---|---|---|
| Create or copy a file in a private location | `create_file` / `copy_file` | A |
| Create a file in a shared team folder | `create_file` / `copy_file` | B — visible to the team |

### Google Sheets — write, via the `gws` CLI only
| Verb | Tool | Class |
|---|---|---|
| Append rows to an existing sheet | `gws sheets spreadsheets values append` | A |
| Update a bounded, machine-owned range | `gws sheets spreadsheets values update` | A |
| Create a new spreadsheet | `gws sheets spreadsheets create` | A — private on creation |
| Overwrite or clear a range a human edits | `values update` / `values clear` | **C** — destroys work no connector can restore |

**No Sheets MCP connector exists** (none in the registry). Writes go through the local `gws` CLI, which is authed against Forkan's own keyring — so **Sheets writes are unavailable in a cloud routine environment** unless `gws` is installed there with its own credentials. A routine that needs one must run locally or ship credentials; don't assume the cloud env can do it.

Rules for any sheet a human also edits:
- Write only machine-owned columns/ranges. Reviewer-annotation columns are off-limits — appending must never shift or overwrite them.
- Prefer `append` over `update`. An `update` that lands on a human-edited range is Class C, not A.
- `valueInputOption=RAW` so Salesforce Ids and dates aren't coerced by Sheets' parser.
- Introspect the request shape with `gws schema sheets.spreadsheets.values.append` rather than guessing.

### No write path exists
- **Salesforce** — query-only. No create/update/delete tool is exposed.
- **Pardot** — no MCP connector at all.
- **Gmail** — draft-only, and not currently authorized.
- **Google Calendar** — not currently authorized.

See §9 for how to handle a request that needs one of these.

---

## 2. Reversibility classes → what each requires

**Class A — Reversible.** Prior state can be restored with the same connector; no human is notified.
→ State the change, make it, log it. Confirm first for anything non-obvious. The bulk rule (§3) still applies.

**Class B — Notifying.** Undoable in the system, but a human has already been pinged and the notification itself can't be recalled.
→ Confirm before the call, **naming the audience**: "this comments on the ticket (Harish gets notified)" / "this posts to #mops-team".

**Class C — Terminal.** Cannot be undone with the tools available.
→ Confirm naming the **exact consequence** and that it is **one-way**. Never batch-approve. Never fire from an unattended routine unless explicitly listed in §6.

If the class is unclear, treat it as the higher class.

---

## 3. Bulk policy

- **1–3 records** — normal confirm per §2.
- **4+ records** — show a preview table (every record and the exact change), take **one** approval for the batch, then execute sequentially and report **per-record** results. Partial failures are reported explicitly; never summarize a partial batch as "done".
- **Class C actions are never batch-approved.** Each item is confirmed individually, or the human does it in-system.
- **Hard stop at 25.** More than 25 records in one gesture → stop and ask for a written go-ahead that names the count.

---

## 4. Idempotency — check before you write

Re-runs are the most likely real failure mode: a routine re-fires, or someone re-runs the same ticket. Check first, every time.

| Write | Check before |
|---|---|
| Create sub-tasks | Read the parent's existing sub-tasks; match by name; skip matches |
| Assign an owner | Read the current assignee; already the target → skip **silently** (no comment, no note) |
| Append rows to a Sheet | Read the existing key columns; skip rows whose natural key is already present. A day already logged → the whole run is a no-op, not a second block of rows |
| Create a Jira issue | JQL the project for the same summary in the last 30 days |
| Send a digest / alert | Dedupe on a run key (date + alert identity) against the last run |
| Create a Drive file | Search the target folder for the same filename; version it, don't duplicate |

A skipped write is still logged (§7) with `RESULT: skipped`.

---

## 5. Failure policy — never auto-retry a write

A failed write may have applied. Retrying blind is how you get duplicates and double-sends.

- **"May have applied"** — `server_unavailable`, `upstream_error`, `cancelled`, timeout → report **AMBIGUOUS**, read the record back to establish the truth, then decide with the human. Never fire again blind.
- **"Definitely didn't apply"** — `needs_reauth`, `server_not_connected`, validation / invalid-params → safe to fix and re-issue.
- **Success means the tool's own success signal**, not the absence of an error. Example: Asana `update_tasks` succeeded only if `failed[]` is empty.

---

## 6. Authorization — unattended writes

Only these targets may be written **without a human in the loop**:

| Target | Verb | Status |
|---|---|---|
| `#mops-team` | Weekly review post (link + headline summary) | **Standing** — in production via the `mops-weekly-review-2.0` routine. Re-confirm with Jennifer if the scope widens beyond the weekly review. |
| Sheet `1gFu34xeJJ2Jrk1p9jtvgkzp_WtspHNyCGZ1zorbNNOA` — "Lead Routing QA — Daily (MOPs + RevOps)" | `values.append` to `Detail!A:X` and `'Daily Summary'!A:N`; `values.update` to `'Config Linter'` | **Standing** — authorized by Forkan 2026-08-03 for the `lead-routing-audit` launchd job (05:30 PT daily). **This sheet only, these ranges only.** `Detail!Y:Z` (`Reviewer_Note`, `Resolved`) and the `Reference` tab are human-owned and must never be written unattended. |
| Everything else | — | **Per-instance confirmation.** |

No channel, DM, project, or record outside this table gets an unattended write. Adding a row is a decision → log it in `decisions/log.md` first.

---

## 7. Audit

**Every write attempt appends one line to `decisions/actions.md`** — including skipped and ambiguous ones. Format is defined in that file.

A write that isn't logged didn't happen as far as anyone can audit. This is what makes delegated write access trustworthy; treat it as part of the write, not paperwork after it.

`decisions/log.md` is for design decisions. `decisions/actions.md` is for things done to live records. Don't mix them.

---

## 8. Attribution

Writes run as the human whose credentials are in play — dashboard actions run as the viewer, chat actions as Forkan, routine actions as the routine's identity. Never phrase a ticket-facing comment as the assistant acting on its own authority.

---

## 9. When there's no write path

For Salesforce, Pardot, Gmail send, and Calendar (§1), a request produces a **spec plus a handoff** — never a completion claim.

- Say plainly that the step is manual, and why (query-only connector / no connector).
- Produce the exact values a human needs to enter (this is what `sf-campaign-spec` is for).
- Do not report the task as done, and do not log it in `decisions/actions.md` as a write.

---

## Routine environments

Cloud routines clone specific skill folders and **do not inherit `.claude/rules/`**. Any routine repo that performs writes must ship a copy of this file, and its README must name it as a dependency. Re-push after any change here, or the routine runs unguarded.
