---
name: campaign-gate-check
description: Decides whether a classified intake ticket needs a Salesforce Campaign at all, before campaign-naming/sf-campaign-spec run. Use right after a ticket's Asana Project Type is known, as the single shared gate — don't re-derive this logic inside other skills.
---

# Campaign Gate Check

One decision, one place: does this ticket need a Salesforce Campaign? `intake-classification`, `sf-campaign-spec`, and any automation chaining them (e.g. the MOps Intake Pipeline routine) call this instead of keeping their own copy of the gate logic.

## Input
- The ticket's Asana `Project Type` custom field value (gid `1206591746930193`).
- For the two exception types below only: whether a Salesforce Campaign already exists for the named event/webinar (check via Salesforce `find`/`soqlQuery` before answering).

## Output
One of:
- **`yes`** — this ticket needs a Salesforce Campaign (one or more records).
- **`no`** — confirmed, no Salesforce Campaign needed.
- **`needs-human-input`** — `Project Type` is missing, unmapped, or one of the exception types where the existing-Campaign check itself is ambiguous.

## Gate table

Confirmed 2026-07-11 by cross-referencing live Asana `[MOPs] Intake` tasks against actual Salesforce Campaign records — see `decisions/log.md`. This is the classification vocabulary; use exact values, don't invent new ones.

| Project Type (Asana value) | Gate result |
|---|---|
| `Webinar Request` | **yes** |
| `Event (+ SFDC Campaign)` | **yes** |
| `SFDC Campaign only (single)` | **yes** |
| `SFDC Campaigns only (multiple)` | **yes** |
| `UTM(s) (+ SFDC Campaign)` | **yes** (option currently disabled in Asana — confirm it's actually retired before assuming) |
| `Email(s) only \| Nurture Sequences` | **no** |
| `Audiences` | **no** |
| `Reporting` | **no** |
| `Other` | **no** |
| `UAT`, `IT/Integration`, `Automations \| Martech`, `Issues`, `Team OOO` | **no** |
| `List Upload` | **Check first.** Per Harish (2026-07-13): ~70% of the time this uses the Campaign already created for that event/webinar → **no**. The ~30% exception is a partner-hosted event/webinar with no existing Acquia Campaign → **yes** (new Campaign needed just for the list import). Look up whether a Campaign already exists for the named event before answering; if that lookup itself is inconclusive, return `needs-human-input`. |
| `Form Request` | **Check first.** Per Harish (2026-07-13): ~70% of the time the Campaign already exists (created when the Event/Webinar Campaign was set up) → **no** (just wire the existing Campaign ID into the form). The ~30% exception is when the web team built the landing page ahead of any Campaign → **yes**. Same lookup-first rule as `List Upload`. |
| Missing / unmapped value | **needs-human-input** |

## Notes
- This skill only answers the yes/no/needs-human-input question — it doesn't classify the rest of the ticket (that's `intake-classification`), name anything (`campaign-naming`), or build the spec (`sf-campaign-spec`).
- If a `Project Type` value shows up that isn't in this table, don't guess — return `needs-human-input`, then once a human confirms the answer, add it to the table so it's reusable.
- Callers decide what to *do* with each result (e.g. whether "no" means "skip silently" or "skip + Slack mention" is a caller-level policy, not this skill's concern).
