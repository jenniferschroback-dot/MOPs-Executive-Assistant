# Graph Report - /Users/forkane.lebdi/mops-executive-assistant  (2026-07-28)

## Corpus Check
- 39 files · ~101,479 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 179 nodes · 236 edges · 28 communities (15 shown, 13 thin omitted)
- Extraction: 85% EXTRACTED · 15% INFERRED · 0% AMBIGUOUS · INFERRED: 35 edges (avg confidence: 0.83)
- Token cost: 89,873 input · 0 output

## Community Hubs (Navigation)
- Naming, Gate & Tone Rules
- PPTX Slide Builders
- Command Center Dashboard & Jira
- Reporting Rules & Pardot Sync
- Priorities, Goals & Routines
- Acquia Brand System
- Email Performance & Pipeline
- Skill Authoring
- MOPS Team & Conventions
- On-Time SLA Model
- Dev/Analytics Icons
- Performance & Network Icons
- Email Metrics & Personas
- Top Email Performance Charts
- Webinar/Event Dual-Campaign Pattern
- Community 15
- Community 16
- Community 17
- Community 18
- Community 19
- Community 20
- Community 21
- Community 22
- Community 23
- Community 24
- Community 25
- Community 26
- Community 27

## God Nodes (most connected - your core abstractions)
1. `_run()` - 11 edges
2. `ParBot Email Performance Skill` - 10 edges
3. `_blank()` - 9 edges
4. `_set_bg()` - 9 edges
5. `add_logo()` - 9 edges
6. `_eyebrow_title_intro()` - 9 edges
7. `add_stat_cards_slide()` - 9 edges
8. `Intake Classification Skill` - 9 edges
9. `add_content_chart_slide()` - 8 edges
10. `add_priority_list_slide()` - 8 edges

## Surprising Connections (you probably didn't know these)
- `Email campaign performance section` --semantically_similar_to--> `Email Performance Brief Q2 2026 (output)`  [INFERRED] [semantically similar]
  .claude/skills/mops-command-center/SKILL.md → outputs/Email_Performance_Brief_Q2-2026_2026-07-26.md
- `Intake-to-Launch Timing Metric` --semantically_similar_to--> `MOps Weekly Review Skill`  [INFERRED] [semantically similar]
  .claude/skills/intake-tracking/SKILL.md → .claude/skills/MOps-weekly-report/SKILL.md
- `Intake Pipeline Automation project` --conceptually_related_to--> `Naming enforcement pain point`  [INFERRED]
  projects/intake-pipeline-automation/README.md → context/work.md
- `Email Performance Brief Q2 2026 (parbot skill copy)` --shares_data_with--> `Email Performance Brief Q2 2026 (output)`  [INFERRED]
  .claude/skills/parbot-email-performance/Email_Performance_Brief_Q2-2026_2026-07-26.md → outputs/Email_Performance_Brief_Q2-2026_2026-07-26.md
- `Current Priorities (MOPS)` --conceptually_related_to--> `Intake Pipeline Automation project`  [INFERRED]
  context/current-priorities.md → projects/intake-pipeline-automation/README.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Acquia Deck Generation Stack** — _claude_skills_acquia_brand_deck_skill_acquia_brand_deck, _claude_skills_acquia_brand_deck_skill_pptx_helpers, _claude_skills_mops_weekly_report_skill_mops_weekly_review, _claude_skills_parbot_email_performance_skill_parbot_email_performance [INFERRED 0.75]
- **Email Send Calendar Data Flow** — _claude_skills_intake_classification_skill_multi_send_extraction, _claude_skills_email_send_calendar_skill_milestone_subtask_pattern, _claude_skills_mops_weekly_report_skill_email_calendar_slides [EXTRACTED 0.95]
- **Intake-to-Launch Pipeline Skill Chain** — _claude_skills_campaign_gate_check_skill_campaign_gate_check, _claude_skills_intake_classification_skill_intake_classification, _claude_skills_campaign_naming_skill_campaign_naming, _claude_skills_sf_campaign_spec_skill_sf_campaign_spec, _claude_skills_intake_tracking_skill_intake_tracking [EXTRACTED 0.95]
- **MOps Command Center system (skill + briefing + two dashboards + routine)** — _claude_skills_mops_command_center_skill_mops_command_center, _claude_skills_mops_command_center_skill_briefing_object, _claude_skills_mops_command_center_dashboard_live_dashboard, _claude_skills_mops_command_center_dashboard_template_dashboard, routines_mops_command_center_readme_routine [EXTRACTED 0.85]
- **On-time SLA computation chain** — _claude_skills_mops_command_center_skill_ontime_sla_metric, _claude_skills_mops_command_center_skill_sla_turnaround_table, references_sops_mops_sla_timeline_sla_doc, _claude_skills_mops_command_center_dashboard_live_computeontime [EXTRACTED 0.85]
- **Email performance + pipeline reporting flow** — _claude_skills_mops_command_center_skill_email_perf_section, _claude_skills_pipeline_influenced_email_skill_pipeline_influenced_email, _claude_skills_pipeline_influenced_email_skill_amountallopportunities, outputs_email_performance_brief_q2_2026_2026_07_26_brief, _claude_skills_mops_command_center_dashboard_live_computeemailreport [INFERRED 0.85]

## Communities (28 total, 13 thin omitted)

### Community 0 - "Naming, Gate & Tone Rules"
Cohesion: 0.12
Nodes (27): Communication Style Rules, Internal vs External Tone, Campaign Naming Convention (fixed format), Campaign Gate Check Skill, SF Campaign Gate Table (Project Type to yes/no), Asana Project Type Custom Field (gid 1206591746930193), Single Shared Gate Principle, Campaign Naming Skill (+19 more)

### Community 1 - "PPTX Slide Builders"
Cohesion: 0.23
Nodes (23): add_content_chart_slide(), add_flow_emphasis_slide(), add_footnote(), add_logo(), add_overview_cards_slide(), add_priority_list_slide(), add_process_steps_slide(), add_stat_cards_slide() (+15 more)

### Community 2 - "Command Center Dashboard & Jira"
Cohesion: 0.10
Nodes (21): closeJiraIssue() (live), computeJira() (live), dashboard-live.html (self-refreshing surface), dashboard-template.html (snapshot render target), briefing object (shared KPI snapshot), Jira Close write action, Jennifer Schroback (Sr. Manager, Agentic Marketing Ops), Jira intake channel (second front-door) (+13 more)

### Community 3 - "Reporting Rules & Pardot Sync"
Cohesion: 0.14
Nodes (16): Connected MCP Servers (Asana/Salesforce/Jira/Slack), MOPS Operations Rules, Pardot Has No MCP Connection, Member Status Scaffolding Must Match Campaign Type, Weekly Review Drive Folder (0ACqafLRVUxJzUk9PVA), MOps Weekly Review Skill, No Fabricated Figures Rule, Results-Driven Reporting Principle (+8 more)

### Community 4 - "Priorities, Goals & Routines"
Cohesion: 0.18
Nodes (13): Current Priorities (MOPS), Q3 2026 Goals, Naming enforcement pain point, Campaign status scaffolding pain point, Priyanka, Intake Pipeline Automation project, MOps Intake Pipeline routine (disabled), MOps Weekly Review 2.0 routine (+5 more)

### Community 5 - "Acquia Brand System"
Cohesion: 0.21
Nodes (12): Acquia Brand Reference, Acquia Color Palette, Droplet Motif and Logo, Acquia Icon Set, Revenue-Marketing-Intern-Overview.pptx (source deck), Acquia Typography (Proxima Nova / Montserrat), Acquia Brand Deck Skill, gws Drive Upload Workflow (+4 more)

### Community 6 - "Email Performance & Pipeline"
Cohesion: 0.29
Nodes (10): computeEmailReport() / Email Performance Report generator (live), Email campaign performance section, Email Performance Brief Q2 2026 (parbot skill copy), AmountAllOpportunities (Campaign rollup), pipeline-influenced-email metric skill, MOPS Executive Assistant (project), Forkan (MOPS intern, automation builder), Driver: audience precision (+2 more)

### Community 7 - "Skill Authoring"
Cohesion: 0.25
Nodes (8): CLAUDE.md vs Skills Distinction, context: fork Subagent Execution, Dynamic Context Injection (!command syntax), Skill Frontmatter Field Reference, Skill Builder Reference, Skill Audit Checklist, Discovery Interview Process, Skill Builder Skill

### Community 8 - "MOPS Team & Conventions"
Cohesion: 0.25
Nodes (8): Aayushi, Felipe, Forkan, Harish, Jennifer, MOPS (Marketing Operations) Team, Salesforce Campaign Name character limit (~40-50 chars), Email send calendar via milestone sub-tasks (audience clash avoidance)

### Community 9 - "On-Time SLA Model"
Cohesion: 0.40
Nodes (6): computeOnTime() (live), On-time SLA % metric, Turnaround SLA table (by Project Type), Decision: real on-time SLA model (2026-07-21), Weekly send-day ownership / 3-per-week cap, MOps Requests SLA Timeline Requirements (2025)

### Community 10 - "Dev/Analytics Icons"
Cohesion: 1.00
Nodes (3): Code/development icon: monitor screen showing angle-bracket code tag (</>) with a gear cog badge, blue outline line-art with light-blue fill, Growth/analytics icon: ascending bar chart with an upward trend line and node dots, blue outline line-art with light-blue fill, Launch/send icon: paper airplane in flight with motion lines, blue outline line-art with light-blue accents

### Community 11 - "Performance & Network Icons"
Cohesion: 1.00
Nodes (3): Monitor with rising line-graph and up-arrow icon (blue outline, light-blue fill), Network hub icon: central node linked to six surrounding nodes (blue outline, light-blue fill), Partnership handshake icon: two hands clasping (blue outline, light-blue cuff fill)

### Community 12 - "Email Metrics & Personas"
Cohesion: 0.67
Nodes (3): Core email metrics: Open Rate 10.66% (up from 8.62% prior), CTR 0.35% (up from 0.28% prior), Email engagement by persona: IT Architect/IT Operations most engaged (252), Legal least/underengaged (5), Persona base by prospects vs customers: Marketing largest (3,378 leads / 8,759 contacts), Website Design/Dev smallest

### Community 13 - "Top Email Performance Charts"
Cohesion: 0.67
Nodes (3): Subject-line pattern CTR chart: content-episode subjects lead at 2.11% CTR / 30.5% open, Top 10 emails by CTR chart: Acquia TV: Engage London w/ Daniel leads at 2.89%, Top emails by clicks chart: Acquia product roadmap session leads at 1,140 clicks

### Community 14 - "Webinar/Event Dual-Campaign Pattern"
Cohesion: 0.67
Nodes (3): Dual-campaign pattern for webinar/event (event + email campaign), Parent campaign linkage (webinar/event campaign as parent of email campaign), Promotional email flag (do you require a promotional email?)

## Knowledge Gaps
- **61 isolated node(s):** `Acquia blue droplet logo mark — solid teardrop/water-drop shape in Acquia brand light blue (#1E9FDA-like cyan-blue) on transparent background; the core brand motif used as logo and decorative element across the deck system`, `Build/media icon — blue line-art icon of a framed image/screen with mountain-scene thumbnail, a text-lines card, and a play button; light-blue fills on a two-tone blue palette; represents building media, content, or creative assets in the deck`, `Calendar/report icon — blue line-art icon of a wall calendar with date cells, a green-blue checkmark, and a speedometer/gauge overlay; light-blue fills on two-tone blue palette; represents scheduling, reporting, or performance tracking in the deck`, `Systems Expand icon — light-blue circle at center with four diagonal arrows pointing outward to corners, inside a rounded blue-bordered square frame`, `Target Plan icon — concentric bullseye target rings in Acquia blue with a blue arrow striking the center` (+56 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **13 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `ParBot Email Performance Skill` connect `Reporting Rules & Pardot Sync` to `Acquia Brand System`?**
  _High betweenness centrality (0.045) - this node is a cross-community bridge._
- **Why does `MOps Weekly Review Skill` connect `Reporting Rules & Pardot Sync` to `Naming, Gate & Tone Rules`?**
  _High betweenness centrality (0.042) - this node is a cross-community bridge._
- **Why does `Acquia Brand Deck Skill` connect `Acquia Brand System` to `Reporting Rules & Pardot Sync`?**
  _High betweenness centrality (0.029) - this node is a cross-community bridge._
- **Are the 2 inferred relationships involving `ParBot Email Performance Skill` (e.g. with `MOps Weekly Review Skill` and `Acquia Brand Deck Skill`) actually correct?**
  _`ParBot Email Performance Skill` has 2 INFERRED edges - model-reasoned connections that need verification._
- **What connects `Acquia blue droplet logo mark — solid teardrop/water-drop shape in Acquia brand light blue (#1E9FDA-like cyan-blue) on transparent background; the core brand motif used as logo and decorative element across the deck system`, `Build/media icon — blue line-art icon of a framed image/screen with mountain-scene thumbnail, a text-lines card, and a play button; light-blue fills on a two-tone blue palette; represents building media, content, or creative assets in the deck`, `Calendar/report icon — blue line-art icon of a wall calendar with date cells, a green-blue checkmark, and a speedometer/gauge overlay; light-blue fills on two-tone blue palette; represents scheduling, reporting, or performance tracking in the deck` to the rest of the system?**
  _61 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Naming, Gate & Tone Rules` be split into smaller, more focused modules?**
  _Cohesion score 0.1168091168091168 - nodes in this community are weakly interconnected._
- **Should `Command Center Dashboard & Jira` be split into smaller, more focused modules?**
  _Cohesion score 0.10476190476190476 - nodes in this community are weakly interconnected._