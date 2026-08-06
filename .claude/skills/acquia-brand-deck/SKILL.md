---
name: acquia-brand-deck
description: Applies Acquia's brand system (colors, fonts, logo, droplet motif, icon set) and layout templates (title/section, overview cards, process steps, flow/emphasis, two-column highlight, chart/content, closing) to any presentation. Use when asked to brand, re-skin, or style a deck like Acquia's internal decks, or to build a new Acquia-branded .pptx from scratch.
argument-hint: [content/topic for the deck, or a deck to re-skin]
---

# Acquia Brand Deck

Builds (or re-skins) a `.pptx` using Acquia's actual brand assets and the layout patterns from `Revenue-Marketing-Intern-Overview.pptx` (Karen Plant, July 2026) — not an approximation, but colors/fonts/images pulled directly from that file's theme XML and media. Full provenance and exact specs are in `reference.md`; don't re-derive them from memory.

## When to use this vs. other skills

- **Building/re-skinning a deck to match Acquia's brand** → this skill.
- **Generating the actual charts/graphs to put on a slide** → use the `dataviz` skill first to design the chart, then place the resulting image with `add_content_chart_slide` (template 6) below.
- **The specific content/data for a MOPS deck** (e.g. Pardot performance) → that's a separate skill (`parbot-email-performance`, `MOps-weekly-report`, etc.) — this skill only handles the visual system, not the data/analysis.

## Step 1: Get the content structured first

Before touching layout, have the finished outline: slide-by-slide title, the 1-sentence intro/headline for each, and the body content (bullets, card items, step items, chart image paths). If re-skinning an existing deck, extract its content losslessly — don't summarize or drop detail while re-templating.

## Step 2: Pick a layout template per slide

Match each slide's content shape to one of the 7 templates in `pptx_helpers.py` (full visual spec for each in `reference.md`):

1. `add_title_slide` — navy bg, droplets, logo, big title + subtitle + footer. Use for the deck title and any section-break slide.
2. `add_overview_cards_slide` — eyebrow + H1 + intro, then 2-4 icon cards in a row. Use for "what we do" / "three teams" style overviews.
3. `add_process_steps_slide` — eyebrow + H1 + intro, then a numbered step row with arrows between. Use for a sequential process (plan/build/launch/optimize, etc.).
4. `add_flow_emphasis_slide` — eyebrow + H1 + intro, then boxes connected by arrows with one box highlighted solid navy. Use to show where something sits in a flow/funnel.
5. `add_two_column_highlight_slide` — one solid-navy box + one light-card box side by side, each with a heading and bullet list. Use to contrast two related lists.
6. `add_content_chart_slide` — eyebrow + H1 + intro, then a chart/screenshot image (optionally with a bullet sidebar), plus an optional footnote. Use for data findings, charts, or proof/screenshot slides.
7. `add_closing_slide` — same as template 1 (alias `add_title_slide`). Use for the final "Thank you" slide.
8. `add_stat_cards_slide` — eyebrow + H1 + headline sentence, then a row of KPI cards (big value, signed colored delta, label). Use for a period's headline metrics — a single number is a stat tile, not a chart (per `dataviz`'s form heuristic). Delta color: Teal `#66C8CA` = good direction, Orange `#F47A20` = bad, gray = no data.
9. `add_priority_list_slide` — eyebrow + H1 + headline, then stacked numbered cards with a colored left accent bar. Use for a prioritized action plan / roadmap closing slide.

Don't invent a new visual style for a slide that doesn't fit — pick the closest template and adapt the content to it, or ask if none fit.

## Step 3: Generate charts before building slides that need them (if applicable)

If a slide needs a chart (not just cards/bullets), invoke the `dataviz` skill to design and render it as a PNG first, using the palette in `reference.md` (Navy `#232C61`, Acquia Blue `#036BB5`, Sky Blue `#26A3DD`, Body Gray `#3D4F5C` — swap dataviz's placeholder palette for these). Save the chart PNG, then pass its path to `add_content_chart_slide`.

## Step 4: Build the deck

Copy `pptx_helpers.py` next to your build script (or import it directly — it resolves `assets/` relative to its own file location, so it works from anywhere as long as the file itself stays in this skill folder). Example:

```python
import sys
sys.path.insert(0, "/Users/forkane.lebdi/mops-executive-assistant/.claude/skills/acquia-brand-deck")
from pptx_helpers import *

prs = new_presentation()
add_title_slide(prs, "Deck Title", "Subtitle", "Date | Context")
add_overview_cards_slide(prs, "Eyebrow", "H1 Title", "Intro sentence.",
    cards=[(ICONS["growth-chart"], "Card Title", "Card description.")])
# ... one call per slide, using the template that matches each slide's content ...
prs.save("outputs/decks/My_Deck.pptx")   # or the calling skill's own outputs/ subfolder
```

`ICONS` is a dict keyed by filename (no extension) — see `reference.md`'s icon table for what each depicts. Reuse the closest existing icon before generating a new one; if you must generate one, match the style exactly (Sky Blue `#26A3DD` thin stroke, `#D3ECF8` light fill accent, square canvas, transparent background) — never mix in a different icon style.

**Keep the file small** (see `.claude/skills/parbot-email-performance`'s history for why this matters): don't add unused slide layouts/masters beyond python-pptx's default single blank layout, and don't embed the full-resolution background patterns unless the slide actually needs one — a plain navy or white background is always acceptable.

## Step 5: Verify before considering it done

Don't just trust that python-pptx didn't crash — actually look at the rendered slides:
1. Upload the `.pptx` to Drive with `gws` (see Notes below), converting to Google Slides.
2. Get each slide's thumbnail: `gws slides presentations get --params '{"presentationId":"<id>","fields":"slides.objectId"}'` to list page IDs, then `gws slides presentations pages getThumbnail --params '{"presentationId":"<id>","pageObjectId":"<pN>","thumbnailProperties.thumbnailSize":"LARGE"}'` per page to get a `contentUrl`.
3. `curl` each `contentUrl` to a local PNG and view it with the Read tool before telling the user it's done.
4. If it's a throwaway verification copy (not the deliverable), delete it after: `gws drive files delete --params '{"fileId":"<id>"}'`.

## Notes

- **Upload via `gws`, not the Google Drive MCP tool.** `gws drive files create --json '{...metadata...}' --upload <local-path> --upload-content-type <mime>` uploads directly from disk — no base64 encoding needed. The Drive MCP tool's `create_file` requires the entire file inline as a base64 string, which is slow/expensive for anything beyond a trivial file size; `gws` was found specifically to solve this after that approach repeatedly stalled on a Pardot deck upload. `gws --upload` paths must resolve inside the current working directory — `cd` into the project first if the file lives elsewhere.
- **Fonts:** default to Montserrat (see `reference.md` for why — Proxima Nova is the true brand font but isn't available in Google Slides or on most machines without a license).
- **Card/box corner radius, border weight, and colors** are all encoded in `pptx_helpers.py` already — don't hand-roll new shape styling per deck, adapt the existing helpers instead so every Acquia-branded deck this project produces actually looks consistent.
- If brand assets ever need updating (new icon, new color), edit `reference.md` and the relevant file in `assets/` together — don't let them drift out of sync.
