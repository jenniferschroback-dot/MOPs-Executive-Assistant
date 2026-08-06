# Output Files — Where Deliverables Go

Every file this assistant produces as a **deliverable** lands in a subfolder of `outputs/`, never in `outputs/` root and never loose in the repo. One routing table — skills and routines don't each invent a path.

**Applies to:** anything written for a human to read, keep, or hand off — CSVs, `.md` reports/digests/specs, `.pptx` decks, chart images. Scratch/working files go in the session scratchpad instead, not `outputs/`.

---

## 1. Routing table — producer → subfolder

| Subfolder | What lands there | Produced by |
|---|---|---|
| `outputs/audiences/` | Audience CSVs + their `.md` query specs; the daily sweep digest | `audience-pull`, `mops-audience-pull` routine |
| `outputs/intake/` | Intake triage digests, classification/tracking/routing reports | `intake-dispatch`, `intake-classification`, `intake-routing`, `intake-tracking`, `mops-intake-dispatch` routine |
| `outputs/campaigns/` | Campaign name proposals, SF campaign specs, gate-check results, email send calendars | `campaign-naming`, `sf-campaign-spec`, `campaign-gate-check`, `email-send-calendar` |
| `outputs/campaign-hygiene/` | Campaign drift / remediation reports | `campaign-hygiene-audit` |
| `outputs/email-performance/` | Pardot email performance decks and written briefs | `parbot-email-performance`, `pipeline-influenced-email` |
| `outputs/weekly-review/` | The weekly review `.pptx` before it goes to Drive | `MOps-weekly-report` |
| `outputs/command-center/` | Manager briefing snapshots, dashboard render output | `mops-command-center` |
| `outputs/sla/` | SLA at-risk / breach sweeps | `sla-watchdog` |
| `outputs/lead-routing/` | Lead-routing config audits and daily QA snapshots | `lead-routing-audit` |
| `outputs/decks/` | Ad-hoc branded decks with no other home | `acquia-brand-deck` |

Subfolders are created on demand — if the target doesn't exist yet, `mkdir -p` it and write. Only these ten names are canonical.

## 2. New producers

A new skill or routine that writes files **must** name its subfolder here before it ships. Pick by **subject matter**, not by which skill ran:

- Does one of the ten already cover this subject? Use it. Don't add a near-synonym (`audience-lists/` next to `audiences/`).
- Genuinely new subject? Add a row, and log the addition in `decisions/log.md`. Keep the name short, plural-or-domain, kebab-case.

Never write to `outputs/` root. A file whose home is unclear is a signal the table needs a row, not a reason to drop it at the top level.

## 3. Filenames

The subfolder carries the category, so the filename carries the **instance** — keep the date, drop nothing else that disambiguates:

- Dated series: `<thing>-YYYY-MM-DD.md` (e.g. `intake/intake-dispatch-2026-08-04.md`)
- Per-request deliverables: `<thing>-<slug>-YYYY-MM-DD.<ext>` (e.g. `audiences/audience-emea-marketingcloud-nl-2026-08-04.csv`)
- Decks keep their existing report-style names (`Pardot_Email_Performance_Q2_2026.pptx`)

Existing filename conventions in each skill stay as they are — this rule changes the **directory**, not the name. A CSV and its companion spec share a basename and sit side by side.

## 4. Build intermediates

Files produced only to build a deliverable (chart PNGs embedded into a deck, intermediate JSON) go in `<subfolder>/assets/` — e.g. `outputs/email-performance/assets/chart_top_emails.png`. Keeps the deliverable listing readable and makes intermediates safe to clear.

## 5. Idempotency

Same rule as any other write (`write-actions.md` §4): a re-run that would produce the same filename **checks first**. Same-day digest already present → overwrite it (it's a regeneration, not a second run). A per-request deliverable that already exists → version it (`-v2`) rather than silently clobbering someone's handoff file.

## 6. Personal data

## 7. Routine environments

Local launchd routines inherit this rule with the rest of `.claude/rules/`. **Cloud routines don't** — they clone skill folders only. A cloud routine that writes files must either ship a copy of this file or have the subfolder path hardcoded in its own skill copy; otherwise it writes wherever it guesses. The paths are already inlined in each producing skill for exactly that reason — keep them inlined when editing.

## 8. Personal data

`outputs/audiences/` accumulates real names, work emails, employers and job titles. It is gitignored (`outputs/**/*.csv`, `outputs/audiences/*.md`) and must stay that way — git history isn't practically erasable, which would defeat a GDPR deletion request. Never commit these, never share them outside Acquia. Any new subfolder that will hold personal data gets a gitignore line **in the same change** that creates it.
