# outputs/

Everything the assistant produces for a human to read, keep, or hand off. **One subfolder per subject — nothing lives in this folder's root.**

| Folder | Holds |
|---|---|
| `audiences/` | Audience pull CSVs + query specs, daily sweep digests — **gitignored, contains personal data** |
| `intake/` | Intake triage digests, classification / routing / tracking reports |
| `campaigns/` | Campaign names, SF campaign specs, gate-check results, email send calendars |
| `campaign-hygiene/` | Campaign drift and remediation reports |
| `email-performance/` | Pardot email performance decks and written briefs |
| `weekly-review/` | The weekly review deck before it goes to Drive |
| `command-center/` | Manager briefing snapshots, dashboard renders |
| `sla/` | SLA at-risk / breach sweeps |
| `lead-routing/` | Lead-routing config audits and daily QA snapshots |
| `decks/` | Ad-hoc branded decks with no other home |

Folders appear when their first file does — an absent folder just means nothing's been generated for it yet.

`<folder>/assets/` holds build intermediates (chart PNGs embedded into a deck, etc.), not deliverables.

The routing rule — which producer writes where, how to add a subfolder, filename and re-run conventions — is `.claude/rules/output-files.md`. Add a row there before a new skill starts writing files.
