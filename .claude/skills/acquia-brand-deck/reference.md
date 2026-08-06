# Acquia Brand Reference

Extracted directly from `Revenue-Marketing-Intern-Overview.pptx` (Karen Plant, Summer Intern Program, July 2026 — `~/Downloads/`) by reading the theme XML, slide XML, and embedded media, then sampling exact pixel colors. Not eyeballed from a PDF render — these are ground-truth values.

## Colors

| Name | Hex | Used for |
|---|---|---|
| Navy | `#232C61` | Slide backgrounds (title/section slides), H1 headings, eyebrow labels, card borders, card titles on navy |
| Navy Dark (reserve) | `#141938` | Defined in the theme's extended palette; no confirmed usage in the source deck — available for a darker variant if needed |
| Acquia Blue | `#036BB5` | Card/subsection titles on white slides (e.g. "Plan the Programs") |
| Sky Blue | `#26A3DD` | Logo wordmark, droplet graphic, icon linework/strokes |
| Body Gray | `#3D4F5C` | Paragraph/body text on white slides |
| Card Fill | `#F2F9FC` | Light card/panel backgrounds on white slides |
| Icon Light Fill | `#D3ECF8` | Secondary fill accent inside icons (e.g. inner circle, highlight bar) |
| White | `#FFFFFF` | Text on navy, slide backgrounds |
| Teal (secondary) | `#66C8CA` | Theme accent — not used in source deck; available for tags/categories |
| Pink (secondary) | `#E1126E` | Theme accent — not used in source deck |
| Orange (secondary) | `#F47A20` | Theme accent — not used in source deck |
| Yellow (secondary) | `#FEC231` | Theme accent — not used in source deck |

Card styling specifics: rounded rectangle, `#F2F9FC` fill, `#232C61` border at ~1.25pt weight, corner radius ≈ 9% of the shorter side (python-pptx `adj` value ~0.09).

## Typography

- **Headings:** Proxima Nova Extrabold — bold, rounded geometric sans, used for slide titles and hero text.
- **Body/subheadings:** Proxima Nova (Regular / Bold / Italic).
- Proxima Nova is a commercial (Adobe/Mark Simonson) font not available in Google Slides or most systems without a license. **Fallback: Montserrat** (ExtraBold for headings, Regular/SemiBold for body) — closest free geometric-sans match and natively available in Google Fonts, so a Google Slides upload renders it without substitution artifacts. Use Montserrat by default unless generating a file for a machine with Proxima Nova actually installed/licensed.
- Type scale observed: Title ~46pt bold, section H1 ~28-32pt bold, card/subsection titles ~14-16pt bold, body ~13-14pt regular, eyebrow label ~11-12pt bold caps.

## Logo

- `assets/logo-blue.png` — "Acquia" wordmark in Sky Blue (`#26A3DD`). Default logo — used on both white and navy backgrounds in the source deck (works on navy because of contrast).
- `assets/logo-navy.png` — same wordmark in Navy (`#232C61`). Alternate for special cases (e.g. printed collateral on a very light/textured background where blue would wash out).
- The "q" in the wordmark has a distinctive droplet-shaped counter — this droplet motif is the core brand shape, echoed at large scale as the decorative graphic (below).
- Placement: top-left, ~0.5in margin, ~1.5in wide, on every slide (title slides: near top; content slides: bottom-right corner, small, per source deck's page 2-7 pattern).

## Droplet graphic

- `assets/droplet-blue.png` — solid Sky Blue (`#26A3DD`) teardrop, transparent background, portrait aspect ~0.77:1 (w:h).
- Used as a large decorative element bleeding off the right edge on **navy title/section/closing slides only** (never on white content slides). In the source deck it appears as **two copies stacked with a slight vertical overlap**, positioned so both bleed off the right edge and the lower one also bleeds off the bottom edge.
- Reference coordinates (10×5.625in slide, scale to any canvas by the same ratio):
  - Droplet 1: x=7.75in, y=-0.55in, w=2.55in, h=3.31in
  - Droplet 2: x=7.75in, y=2.76in, w=2.55in, h=3.31in (touches/overlaps droplet 1's bottom seam)

## Background texture (optional, subtle)

- `assets/pattern-subtle.png` — near-invisible thin droplet/circle outline pattern, full-bleed, used behind navy slides for subtle texture (barely visible — don't rely on it to carry any contrast).
- `assets/pattern-tiled.png` — busier tessellated pattern mixing solid and outline droplets at low opacity; an alternate texture option, not used alongside the subtle pattern in the same deck.
- Both are optional flourishes — a plain navy or white background is always acceptable and was used on some slides (e.g. image-heavy ones) without either pattern.

## Icon set

All icons: Sky Blue (`#26A3DD`) linework with an Icon Light Fill (`#D3ECF8`) accent, square canvas, transparent background, ~512×512px source. Used inside cards on white slides, always paired with a bold Acquia-Blue card title and gray body text below.

| File | Depicts | Used for (source deck) |
|---|---|---|
| `icons/target-plan.png` | Target with arrow | "Plan the Programs" / Plan step |
| `icons/build-media.png` | Image + play button | "Build" step |
| `icons/launch-paperplane.png` | Paper airplane | "Launch" step |
| `icons/growth-chart.png` | Ascending line/bar chart | "Grow the Pipeline" / Optimize step |
| `icons/systems-expand.png` | Expand arrows around a circle | "Run the Systems" / Marketing Operations |
| `icons/partnership-handshake.png` | Handshake | "Partner Marketing" |
| `icons/network-hub.png` | Central node with 6 spokes | "Campaigns Team" |
| `icons/monitor-performance.png` | Monitor with trend line | "Check Performance" (day-in-the-life) |
| `icons/code-gear.png` | Code brackets + gear | "Run the Systems" (alt context) |
| `icons/calendar-report.png` | Calendar + checkmark + gauge | "Follow Up & Report" (day-in-the-life) |

When a new concept doesn't match an existing icon, either reuse the closest conceptual match or generate a new same-style icon (thin Sky Blue stroke, `#D3ECF8` fill accent, square canvas) — don't mix in a different icon style (no filled solid icons, no photographic icons, no other brand's icon set).

## Layout templates (from the source deck, in order of appearance)

1. **Title / section-break slide** (navy bg): logo top-left, subtle pattern behind, two overlapping droplets bleeding off bottom-right, large bold white title + smaller colored subtitle line, optional presenter/date footer text. Used for: deck title, and would be used for section dividers and the closing slide.
2. **Overview + N-card row** (white bg): small bold navy eyebrow label, bold navy H1 below it, gray intro paragraph, then a row of 2-3 equal-width rounded cards (icon, bold Acquia-Blue title, gray description). Logo bottom-right, small.
3. **Process/step row** (white bg): eyebrow + H1 + intro, then a horizontal row of numbered steps (numeral label, icon, bold title, gray description), connected by right-arrow glyphs between steps.
4. **Flow/emphasis comparison** (white bg): eyebrow + H1 + intro, then side-by-side boxes connected by arrows; one box is "highlighted" (solid navy fill, white text) to show current position/emphasis, the others are light-outline boxes (white fill, navy border, navy text).
5. **Two-column highlight box** (white bg): one solid-navy rounded box (teal or white sub-heading, white bullet list) beside one light-card rounded box (bold Acquia-Blue heading, gray bullet list) — used to contrast two related lists (e.g. "core plays" vs. "channels").
6. **Image/screenshot slide** (white bg): simple bold navy title top-left, no eyebrow, supporting screenshots/images below with minimal decoration — used for "campaigns in action" style proof slides.
7. **Closing/thank-you slide**: same template as #1 (navy + droplets + logo), with a closing headline and optional contact/profile card.
8. **Stat card row** (white bg, added when rebuilding the Q2 Pardot deck): eyebrow + H1 + headline sentence, then a row of KPI cards (big navy value, signed colored delta, gray label) — no chart, per the dataviz rule that a single headline number is a stat tile, not a chart. Delta color: Teal `#66C8CA` for a good-direction change, Orange `#F47A20` for bad, gray for "no data" — this is the deck's one sanctioned use of those two reserve accents, as status colors, not identity.
9. **Numbered priority list** (white bg, added when rebuilding the Q2 Pardot deck): eyebrow + H1 + headline, then stacked rows — each a light card with a colored left accent bar, a bold "N · Title" line, and a description line. Use the accent color to group priorities (e.g. Acquia Blue for "start now," Orange for "schedule later").

See `pptx_helpers.py` for working python-pptx functions implementing all 9 templates (`add_title_slide`/`add_closing_slide`, `add_overview_cards_slide`, `add_process_steps_slide`, `add_flow_emphasis_slide`, `add_two_column_highlight_slide`, `add_content_chart_slide`, `add_stat_cards_slide`, `add_priority_list_slide`).

## Provenance note

The source deck's image `descr` (alt-text) attributes reference paths like `/mnt/skills/plugins/marketing-essentials:acquia-pptx/assets/...`, implying it was built with a pre-existing Acquia-branded plugin/skill in a different Claude Code environment. That plugin isn't installed here — this skill reconstructs the same brand system from the actual embedded assets/colors/fonts so it's usable standalone in this project.
