---
name: campaign-hygiene-audit
description: Read-only auditor that scans live Salesforce campaigns for taxonomy/naming/region/member-status drift and produces a remediation list with fix specs. Use for a campaign hygiene/compliance pass, to find naming-convention violations or blank Campaign Type/Region, or as the weekly hygiene routine.
argument-hint: [scope — e.g. "created this quarter" | "all open" | a campaign name filter]
---

# Campaign Hygiene Audit

Salesforce is **query-only** (no write path) — which makes a read-only auditor the safest, highest-ROI campaign agent. It finds drift and hands back exact fix values; a human (or `sf-campaign-spec` for member statuses) applies them. It never claims to have fixed anything in SF (see `.claude/rules/write-actions.md` §9).

## Why this exists (documented drift)
- **1,938 campaigns with blank `Type`** org-wide — undercounts attribution-by-type.
- **Campaign Region missing** on new campaigns (e.g. 15 of 23 in one week).
- **Naming drift** — a third convention (`Event_NA_Drupal_GovCon_Booth_BM_Q32026`) found live despite the fixed convention.
- **Member-status scaffolding** wrong/missing → breaks the Pardot ↔ SF sync.

## Inputs
- Scope from `$ARGUMENTS` (a period, "all open", or a name filter). Default: campaigns created in the current quarter.
- The naming convention + code table: `.claude/skills/campaign-naming/SKILL.md` (`Region_Channel_Product_Description_YYYY-Qn`).
- Campaign taxonomy definitions: MOPs vault `wiki/concepts/campaign-taxonomy.md` (Program Type / Campaign Type / Subtype / Channel / Medium).
- Required member statuses per campaign type: `.claude/skills/sf-campaign-spec/SKILL.md`.

## Checks (run via Salesforce `soqlQuery` / `find` — read only)

1. **Blank `Type`** — Campaign records where `Type` is null within scope.
2. **Missing Region** — the Campaign Region field null/blank.
3. **Naming-convention violation** — `Name` doesn't match `Region_Channel_Product_Description_YYYY-Qn`. Flag the deviation and, where the pieces are recoverable, propose the compliant name (defer to `campaign-naming` for codes; don't invent a Region/Channel/Product code).
4. **Member-status scaffolding mismatch** — campaign's member statuses don't match what its Campaign Type requires (per `sf-campaign-spec`). This is the sync-breaker; call it out first.
5. **Orphans / duplicates** — no members, or near-duplicate names for the same initiative (a single intake task can legitimately spawn multiple campaigns — don't flag that as duplication; flag only true dupes).

## Output — a remediation report (grouped by issue, most-impactful first)

Report in chat by default. If the run is a scheduled/weekly pass or the caller wants a file, write it to `outputs/campaign-hygiene/campaign-hygiene-YYYY-MM-DD.md` — never `outputs/` root (`.claude/rules/output-files.md`).

```
## Campaign Hygiene Audit — [scope] — [N campaigns scanned]

### 🔴 Member-status mismatch (sync risk) — [n]
- [Campaign name] (Id …): has [statuses]; type [X] requires [statuses] → add [...]

### 🟠 Naming violations — [n]
- [current name] (Id …) → proposed: [compliant name or "needs codes — ask"]

### 🟡 Blank Type — [n]     ### 🟡 Missing Region — [n]
- [name] (Id …) → suggested [Type/Region] from [signal]

### ⚪ Orphans / dupes — [n]
```

Each row carries the **Campaign Id** and the **exact fix value** so a human can apply it directly.

## Handoff & writes
- **No SF writes** — produce the spec + handoff, never a completion claim (contract §9).
- Posting the summary to Slack is a **Class C** send — confirm first. As a **routine**, an unattended post to `#mops-team` is **not** authorized (only the weekly review is, contract §6) — default to producing the report for a human to post, unless authorization is explicitly added and logged.
- If run as the weekly hygiene routine, dedupe on a run key (date + audit identity) so the same findings aren't re-alerted every run (contract §4).

## Notes
- Cross-references `campaign-naming` (codes), `sf-campaign-spec` (member statuses), and the vault taxonomy — single sources; don't re-derive any of them here.
- Recurring hygiene findings with no owner are the pattern the weekly review keeps surfacing — pair this with `sla-watchdog`'s "threshold → tracked issue" idea if the team wants breaches tracked, not just reported.
- Keep the scan read-only and bounded to the requested scope; a full-org 1,938-record dump is a report, not an action item — summarize counts, list the actionable subset.
