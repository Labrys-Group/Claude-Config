---
name: figma-driven-development
description: Use when implementing UI from a Figma link. Required before writing any code when a Figma URL is provided. Also use when auditing an existing implementation for Figma fidelity.
---

# Figma-Driven Development

## Overview

Use Figma as a rigid implementation scaffold. Every visual decision must trace back to a specific fetched Figma node — no guessing, no implementing from memory. A shared catalogue file on disk lets multiple agents share discovery without re-fetching.

Two modes:
- **Build mode** — implementing a section from scratch (Phases 1–6)
- **Audit mode** — checking an existing implementation against Figma (Phase 7)

## Phase 1: Discover Figma Tools

Before anything else, load the Figma tool schemas:

```
ToolSearch("figma")
ToolSearch("select:mcp__claude_ai_Figma__get_metadata,mcp__claude_ai_Figma__get_design_context,mcp__claude_ai_Figma__get_screenshot")
```

**If ToolSearch returns nothing:** The Figma integration needs to be connected. Ask the user to connect Figma via claude.ai settings (`/mcp`), then restart the session. Do not proceed without Figma access — do not implement from memory or screenshots.

**Key tools:**
- `get_metadata` — XML structural tree (node IDs, names, sizes). Use for cataloguing. **Always call via subagent** — root-level responses routinely exceed context limits and this is predictable, not a surprise.
- `get_design_context` — Reference code + screenshot + design values. Use for implementation when exact values are needed.
- `get_screenshot` — Visual render of any node. Use first — it's cheap. Only escalate to `get_design_context` when a discrepancy needs exact values to fix.

## Phase 2: Parse the URL

Extract `fileKey` and `nodeId` from the Figma URL:

```
https://www.figma.com/design/sHPq6WL754484sETatkL51/Name?node-id=9936-6
                              ^^^^^^^^^^^^^^^^^^^^^^                ^^^^^^
                              fileKey                               nodeId (convert - to :)
```

**The `node-id` in the URL is the designers' intended starting frame — use it directly.** Do NOT start from `0:1` (the page root) — it returns the entire file tree and exceeds token limits.

## Phase 3: Check for Existing Catalogue

The catalogue lives at `.figma/[fileKey].md`.

**If it exists:** Read it, then:
- Does the target section appear in the catalogue? → Go to Phase 5 (build) or Phase 7 (audit).
- Section missing from catalogue? → See "Section Not in Catalogue" below.

**If not:** Continue to Phase 4.

### Section Not in Catalogue

Do NOT re-fetch root metadata. Instead, work through these steps in order:

1. **Check for a naming mismatch.** Catalogue section names come from Figma layer names — the task may use a different label. Scan the catalogue descriptions carefully before concluding it's absent.

2. **Screenshot the root node** to visually search for the section:
   ```
   get_screenshot(fileKey, rootNodeId)
   ```
   If you can see the section, it's in the root frame but wasn't catalogued. Fetch its metadata (via subagent), add it to the catalogue, then proceed.

3. **If still not found:** The section may live in a different Figma frame not covered by the current catalogue. Ask the user to provide the specific Figma URL with the node-id focused on that section (e.g. by selecting it in Figma dev mode and copying the link). Do not guess — do not implement without a node ID.

## Phase 4: Build the Catalogue

### 4a. Fetch structural metadata for the root node

**Always route `get_metadata` through a subagent.** A root frame covering a full marketing page (height > 2000px, 5+ sections) will exceed context limits — this is predictable, not a contingency. Do not call `get_metadata` in the main session and react to the overflow; plan for the subagent upfront.

Call `get_metadata` in your main session, then immediately hand the result file path to a subagent. **Copy the exact file path from the tool result verbatim — do not retype it**, as UUID-based paths are easy to mistype.

Subagent briefing template:
> "The file at **[paste exact path here]** is a JSON array [{type, text}] where text is XML. Using jq or python3 (do NOT use the Read tool — file is too large): (1) extract the root node name, type, width, height; (2) produce a markdown table of ALL direct children with columns: name | id | type | width | height. Return markdown only."

**The XML `name` attribute on each child frame is the authoritative section name** — use it directly in the catalogue. Do not rename sections based on visual guesses; Figma layer names are what designers use to communicate intent.

**If children have generic names** (e.g. `Frame 168655…`): the designer left frames unnamed. In this case, section names in the catalogue must come from screenshots, not layer names — screenshot each child before writing the catalogue and name entries from their visual content.

### 4b. Screenshot all sections in parallel for verification

For all sections in the metadata children list, dispatch screenshots in parallel:

```
get_screenshot(fileKey, sectionNodeId)  — one per section, all at once
```

A full-page overview screenshot is too small to read section headings — screenshot sections individually. Use the screenshots to write meaningful descriptions and catch mismatches between a layer name and its actual visual content.

### 4c. Write the catalogue

Save to `.figma/[fileKey].md`:

```markdown
# Figma Catalogue — [File Name]
File key: [key]
Root node: `[nodeId]` ("[name]" — [width]×[height]px)
Catalogued: [date]

## Sections

### [Figma layer name]
- Node ID: `9936:51`
- Size: 1440×803px
- Description: [layout, key elements, purpose — from screenshot]

[... one entry per section ...]

## Shared Styles

| Token | Observed value | Notes |
|-------|---------------|-------|
| Background | #?????? | Verify via get_design_context |
| Primary accent | #?????? | Buttons, badges, highlights |
| Heading font | [family weight/size] | Verify per section |

> Exact values must be confirmed by fetching each section's design context.

## DRY Opportunities

Repeated patterns across sections — candidates for shared components or classes:

- **[Pattern name]**: Appears in [Section A] ([nodeId]), [Section B] ([nodeId]) — candidate for `<ComponentName />` or `.class-name`
```

### 4d. Identify DRY opportunities globally

Before implementing anything, scan the section list for:
- Repeated background layers (same size/type appearing in multiple sections)
- Shared button variants
- Common content wrapper widths (e.g. 1180px centred column)
- Typography/heading patterns that repeat

Note these in the catalogue. Implementing DRY later is expensive — find it now.

## Phase 5: Implement a Section

For the section you are implementing:

1. **Re-read the catalogue** — find its node ID.
2. **Sanity-check the catalogue entry.** Take a screenshot and verify the visual content matches the catalogue description. If it doesn't match, re-catalogue that entry before proceeding.
3. **Check whether the target node is a sub-component.** If the node name suggests it is a sub-component rather than a full section (e.g. `PieChart`, `Icon`, `Card`, `Button`) — fetch the **parent node's metadata** before implementing. Labels, callouts, and decorators that visually belong to the sub-component are often siblings in the parent, not children of the sub-component node. Implement the smallest container that includes all visually related siblings.
4. **Screenshot first, design context only if needed.** Take a screenshot of the section. If the section doesn't exist yet in the codebase, proceed to fetch design context. If it does exist, compare the screenshot to the current implementation visually — only fetch `get_design_context` if you spot a discrepancy that needs exact values to resolve.
4. **Fetch design context when needed:**
   ```
   get_design_context(fileKey, sectionNodeId)
   ```
   This returns reference code (React + Tailwind), a screenshot, and design values. It is a **reference, not final code** — adapt it to the project's stack, conventions, and existing components.
5. **Extract from the response:**
   - Layout: direction, alignment, padding, gap, max-width
   - Background: colour, gradient stops, image references
   - **All text content** — copy every text node's exact string. Screenshots can't catch copy mismatches. Cross-check text content in the code (and in the project's config/data layer — see note below) against these strings.
   - Interactive elements: button labels, variants, states
   - Decorative/structural elements: asset URLs, connector lines, dividers, positions
   - Responsive variants: look for Mobile / Tablet / Desktop frames
6. **Check the config/data layer.** If the project separates copy or data into a config file (e.g. `site-config.ts`) or CMS, check that layer against Figma's text nodes — not just the component output. A copy mismatch may live entirely in config, invisible in the JSX.
7. **Download assets immediately after fetching design context — before writing any code.** `get_design_context` returns `figma.com/api/mcp/asset/...` URLs that expire in 7 days. Each new call to `get_design_context` (including for a parent or sibling node) returns *new* URLs for the same assets — do not mix URLs from different fetches. Download immediately from the URLs in the response you are currently working with, before touching the codebase. Classify each URL by purpose:
   - **Images** (photos, illustrations, coin graphics) → `curl -o public/images/[name].[ext] "[url]"` — use descriptive filenames, not UUIDs
   - **Icons** (SVG) → `curl -o public/icons/[name].svg "[url]"` — check if an equivalent already exists in the project before downloading
   - **Structural elements** (connector lines, dividers, borders, decorative shapes) → **do not download** — reconstruct with CSS (`border`, `background: linear-gradient(...)`, inline SVG). Never use an expiring asset URL for a UI element.
   - **Existing committed static asset** (e.g. `pie-chart.png`) approximating a Figma vector → this is a sign the original implementation bypassed this skill. The correct implementation is CSS reconstruction. Verify visual equivalence for now, but flag it as technical debt to be replaced with a proper CSS implementation.

   After downloading, reference committed paths in code — never the Figma asset URL.
8. **Cross-reference** every colour and spacing value against the existing design system (Tailwind config, CSS variables, existing components) before adding new tokens.
9. **Check DRY opportunities** from the catalogue — use or extend shared components/classes.
10. **Shared component fidelity:** If a shared component (e.g. `<SectionHeading />`) can't express the Figma layout for this specific section (e.g. inline vs. stacked text), inline the markup for that section. **Fidelity to the Figma takes precedence over reusing a shared component** when they conflict. Do not distort the layout to fit the component.
11. **Batch edits, then verify once.** Make all your edits for the section, then run typecheck/build once at the end. Do not run typecheck after each individual edit unless an edit is likely to introduce a type error you need to diagnose immediately.

## Phase 6: Zooming Into Child Nodes

If a section is complex or a value is unclear:

```
get_screenshot(fileKey, childNodeId)       — visual confirmation (cheap, do this first)
get_design_context(fileKey, childNodeId)   — exact values and reference code (only if needed)
```

**If a section contains repeated similar elements** (e.g. multiple connector lines, multiple cards, multiple icons that may differ): screenshot each child node individually to confirm their styles before implementing. The reference code from `get_design_context` may flatten similar elements into a single container, hiding style differences between them.

**Watch for Figma coordinate artefacts.** The reference code from `get_design_context` is a direct translation of Figma's internal layout maths. Certain patterns only work correctly at the original design dimensions and will break at other sizes:

| Artefact | Example | Risk |
|----------|---------|------|
| Percentage insets on absolute elements | `inset: "6.52% 3.42%"` | Collapses if parent has no explicit size |
| Negative insets exceeding 100% | `inset: "-102.42% -72.65%"` | Overflows or clips at different sizes |
| `calc()` with large px offsets | `left: calc(50% + 227.91px)` | Breaks at narrower containers |
| Pixel positions referencing parent size | `top: -220.86px` | Only valid at the exact original height |

When you encounter these patterns: verify that the parent container has an explicit fixed size matching Figma's dimensions, or convert to values that will work at the target render size.

Child node IDs are visible in `get_metadata` output and in Figma's URL when you click an element in dev mode. Do not guess values — re-fetch the specific node.

## Phase 7: Audit Mode

Use this when checking an existing implementation for Figma fidelity, rather than building from scratch.

### 7a. Read the existing implementation first

Before fetching anything from Figma, read all relevant source files — components, views, and the config/data layer. Build a mental model of what the code currently produces.

### 7b. Screenshot all sections in parallel

Using the node IDs from the catalogue, fetch screenshots of all sections at once:

```
get_screenshot(fileKey, sectionNodeId)  — all sections in parallel
```

Do not fetch `get_design_context` yet — screenshots are cheap and may be sufficient to confirm sections are correct.

### 7c. Build a diff list before touching any code

Compare each section screenshot against the current implementation. For each section, classify:
- **Correct** — no action needed
- **Layout deviation** — wrong spacing, alignment, sizing
- **Copy mismatch** — text content differs (screenshots won't show this — cross-check text nodes separately)
- **Missing element** — Figma has something the implementation lacks
- **Wrong element** — implementation has something Figma doesn't

Only fetch `get_design_context` for sections with layout deviations that need exact values. Sections confirmed correct by screenshot need no further Figma fetching.

### 7d. Triage before editing

Not all deviations are worth fixing. Use this priority order:
1. **Wrong layout or structure** — fix immediately
2. **Wrong copy** — fix immediately
3. **Missing element** — fix immediately
4. **Minor spacing deviation** (1–4px off) — fix if it's visible; skip if it's within Tailwind's scale rounding
5. **Pixel-perfect padding differences** that don't affect the visual feel — skip

### 7e. Batch all edits, then verify once

Make all fixes across all sections, then run typecheck/build once. Update the catalogue if any section's description was wrong.

## What NOT to Do

| Bad | Good |
|-----|------|
| Fetch `0:1` (page root) — returns entire file tree | Start from the URL's `node-id` — it's the right frame |
| Read large metadata results into main context | Use a subagent to parse oversized responses |
| Name sections from visual guesses or screenshots when layer names exist | Use the Figma XML `name` attribute — it's authoritative |
| Use layer names when frames are generically named (e.g. `Frame 168…`) | Screenshot each child first — visual content is your only reliable naming signal |
| Screenshot only the full page to identify sections | Screenshot each section individually — thumbnail is too small |
| Trust catalogue entry without sanity-checking | Verify node ID + visual content before implementing |
| Fetch `get_design_context` for every section upfront | Screenshot first — only fetch design context when a discrepancy needs exact values |
| Ignore copy when checking visual fidelity | Cross-check all text nodes explicitly — screenshots can't catch copy mismatches |
| Check only component JSX for copy accuracy | Also check the config/data layer (e.g. site-config.ts) — copy may live there |
| Leave Figma asset URLs in code | Download all images/icons and commit before writing any code |
| Use expiring Figma asset URLs for connector lines or dividers | Reconstruct structural elements with CSS |
| Treat a static asset approximating a Figma vector as acceptable long-term | Flag it as technical debt — correct implementation is CSS reconstruction |
| Assume repeated elements (lines, cards) are identical | Screenshot each child to confirm — they may differ |
| Force a shared component to fit a section's layout | Inline the markup when fidelity and DRY conflict |
| Hardcode a colour without checking existing tokens | Cross-reference every value against the design system |
| Run typecheck after every individual edit | Batch all edits, then verify compilation once at the end |
| Re-fetch root metadata when a section is missing from the catalogue | Screenshot the root node first — visually check if it exists |
| Implement a section not found in Figma | Ask the user for the specific node URL rather than guessing |
| Implement a sub-component node in isolation (e.g. `PieChart`) | Fetch parent metadata first — labels and callouts are often siblings, not children |
| Download assets from one `get_design_context` call then re-fetch the node (or its parent) before writing code | Download immediately after each fetch — new calls return new URLs; do not mix |
| Use percentage insets, negative insets >100%, or `calc()` with large px offsets without checking parent size | Verify parent has an explicit fixed size matching Figma dimensions, or convert to size-safe values |

## Catalogue File Location

`.figma/[fileKey].md` — relative to the project root.

- Lives inside the project so CK Search indexes it; agents can query it semantically rather than reading the whole file
- Listed in `.gitignore` — not committed, since it's generated and may go stale as the Figma evolves
- Survives reboots (unlike `/tmp/`), so the cataloguing cost is paid once per machine per project
- Written in Phase 4, read in Phases 5 and 7; multiple agents share it without re-fetching

**On first use in a project:** create the directory with `mkdir -p .figma/` before writing the catalogue.
