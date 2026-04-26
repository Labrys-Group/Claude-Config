---
name: figma-driven-development
description: Use when implementing UI from a Figma link. Required before writing any code when a Figma URL is provided. Also use when auditing an existing implementation for Figma fidelity.
---

# Figma-Driven Development

Two modes: **Build** (Phases 1–6) and **Audit** (Phase 7).

## Phase 1: Discover Figma Tools

```
ToolSearch("figma")
ToolSearch("select:mcp__claude_ai_Figma__get_metadata,mcp__claude_ai_Figma__get_design_context,mcp__claude_ai_Figma__get_screenshot")
```

If ToolSearch returns nothing: ask the user to connect Figma via claude.ai settings (`/mcp`) and restart. Do not implement from memory or screenshots without Figma access.

**Tools:**
- `get_metadata` — XML structural tree (node IDs, names, sizes). Always call via subagent — root responses exceed context limits predictably.
- `get_design_context` — Reference code + screenshot + design values. Use when exact values are needed.
- `get_screenshot` — Visual render of a node. Use first — it's cheap. Escalate to `get_design_context` only to resolve discrepancies.

## Phase 2: Parse the URL

```
https://www.figma.com/design/sHPq6WL754484sETatkL51/Name?node-id=9936-6
                              ^^^^^^^^^^^^^^^^^^^^^^                ^^^^^^
                              fileKey                               nodeId (convert - to :)
```

Use the URL's `node-id` directly — it's the designer's intended frame. Never start from `0:1` (page root) — it returns the entire file and exceeds token limits.

## Phase 3: Check for Existing Catalogue

Catalogue lives at `.figma/[fileKey].md`.

- **Exists + section present** → Phase 5 (build) or Phase 7 (audit)
- **Exists + section missing** → Check for naming mismatch first. Then `get_screenshot(fileKey, rootNodeId)` to visually search. If still not found, ask the user for the specific node URL — do not guess.
- **Not exists** → Phase 4

## Phase 4: Build the Catalogue

### 4a. Fetch root metadata via subagent

Always route `get_metadata` through a subagent — root frames for full pages predictably exceed context. Copy the exact file path from the tool result verbatim (UUID paths are easy to mistype).

Subagent prompt:
> "File at **[exact path]** is JSON `[{type, text}]` where text is XML. Using jq or python3 (NOT Read tool): (1) root node name, type, width, height; (2) markdown table of ALL direct children: name | id | type | width | height. Return markdown only."

The XML `name` attribute is the authoritative section name. If frames are generically named (e.g. `Frame 168…`), screenshot each child before writing the catalogue and name from visual content.

### 4b. Screenshot all sections in parallel

```
get_screenshot(fileKey, sectionNodeId)  — one per section, all at once
```

A full-page overview is too small to read — screenshot sections individually.

### 4c. Write the catalogue

Save to `.figma/[fileKey].md`:

```markdown
# Figma Catalogue — [File Name]
File key: [key]
Root node: `[nodeId]` ("[name]" — [width]×[height]px)
Catalogued: [date]

## Sections
### [Layer name]
- Node ID: `9936:51`
- Size: 1440×803px
- Description: [layout, key elements, purpose — from screenshot]

## Shared Styles
| Token | Observed value | Notes |
|-------|---------------|-------|
| Background | #?????? | Verify via get_design_context |
> Exact values must be confirmed by fetching design context.

## DRY Opportunities
- **[Pattern]**: Appears in [Section A] ([nodeId]), [Section B] ([nodeId]) — candidate for `<Component />` or `.class`
```

### 4d. Identify DRY opportunities

Before implementing, scan for: repeated background layers, shared button variants, common wrapper widths, repeated typography patterns. Finding DRY after implementation is expensive — find it now.

## Phase 5: Implement a Section

1. Re-read the catalogue — find the node ID.
2. Sanity-check: screenshot the node, verify it matches the catalogue description. Re-catalogue if wrong.
3. **Sub-component check:** If the node is a sub-component (e.g. `PieChart`, `Card`, `Button`), fetch the **parent node's metadata** first — labels and decorators are often siblings, not children.
4. Screenshot first. Fetch `get_design_context` only if you spot a discrepancy needing exact values.
5. From `get_design_context`, extract:
   - Layout: direction, alignment, padding, gap, max-width
   - Background: colour, gradient stops, image refs
   - **All text content** — copy every text node exactly. Screenshots can't catch copy mismatches.
   - Interactive elements: button labels, variants, states
   - Decorative elements: asset URLs, connector lines, dividers, positions
   - Responsive variants: Mobile / Tablet / Desktop frames
6. **Check the config/data layer** (e.g. `site-config.ts`) for copy — a mismatch may live entirely in config, invisible in JSX.
7. **Download assets immediately after fetching design context — before writing any code.** Asset URLs (`figma.com/api/mcp/asset/...`) expire in 7 days. Each new `get_design_context` call returns new URLs — do not mix URLs from different fetches.
   - **Do not assume file format from context** — inspect the URL or run `curl -sI "[url]" | grep content-type` to confirm. Figma assets labelled as images are frequently SVGs, not PNGs or JPEGs.
   - Images/illustrations → `curl -o public/images/[name].[ext] "[url]"` (use confirmed extension, e.g. `.svg`, `.png`, `.jpg`)
   - Icons (SVG) → `curl -o public/icons/[name].svg "[url]"` — check for existing equivalents first
   - Structural elements (connector lines, dividers, shapes) → **do not download** — reconstruct with CSS (`border`, `linear-gradient`, inline SVG). Never use expiring URLs for UI elements.
   - A static asset approximating a Figma vector is technical debt — flag it; correct impl is CSS.
8. Cross-reference every colour and spacing value against the existing design system before adding new tokens.
9. Apply DRY opportunities from the catalogue.
10. **Fidelity over DRY:** If a shared component can't express the Figma layout, inline the markup. Don't distort layout to fit a component.
11. Batch all edits, then run typecheck/build once at the end.
12. **REQUIRED — Visual gate. Do not move on until this is done:**
    - Navigate and screenshot the live browser: `mcp__chrome-devtools__navigate_page(url: "http://localhost:3000")` then `mcp__chrome-devtools__take_screenshot()`
    - Crop to the section — read its bounding box: `mcp__chrome-devtools__evaluate_script(script: "JSON.stringify(document.querySelector('[data-section=\"NAME\"]').getBoundingClientRect())")`
    - Fetch the Figma screenshot for the same node: `get_screenshot(fileKey, sectionNodeId)`
    - **Delegate the visual diff to a skeptical subagent.** Do not review it yourself — your first pass will miss things. Spawn an Agent with the following prompt, passing both screenshots and the catalogue entry for this section as context:

      > You are a meticulous visual QA reviewer. Your job is to find mistakes — assume they are there, because they almost always are. You will be shown two images: a Figma design and a browser implementation. Your disposition is skeptical and critical. Do not give the benefit of the doubt.
      >
      > **Section context from the Figma catalogue:**
      > - Section name: [name from catalogue]
      > - Node ID: [nodeId]
      > - Expected size: [width×height from catalogue]
      > - Catalogue description: [description field verbatim]
      >
      > Use this context to understand what elements should be present and to anchor your diff. If the implementation is missing something the catalogue describes, that is a confirmed missing element.
      >
      > **Scale warning:** Scale errors are extremely common and easy to miss — elements that are too small, too large, or incorrectly proportioned relative to their surroundings. Do not trust that something "looks about right." Actively compare the relative size of every element (icons, images, text blocks, buttons, cards) against the Figma design. Ask yourself: does this element occupy the same proportion of the section as it does in the design?
      >
      > Go through every category below. For each one, describe what you see in both images and call out any discrepancy, no matter how small:
      > - **Layout**: flex direction, alignment (horizontal and vertical), gap, padding, margin — compare every axis
      > - **Typography**: font size, weight, line height, letter spacing, colour, text-transform, text-decoration — read each text node character by character
      > - **Copy**: every word, punctuation mark, and line break — do not skim
      > - **Colour**: backgrounds, borders, text, icon fills — flag anything that looks even slightly off
      > - **Spacing**: internal padding, gaps between elements, outer margins
      > - **Scale & sizing**: widths, heights, aspect ratios — this is a high-risk category. Elements are frequently too small or too large. Compare proportions carefully against the Figma design, not just against your expectations.
      > - **Borders & shadows**: radius, width, colour, box-shadow offsets and blur
      > - **Visual effects**: gradients (direction, stops, colours), opacity, blur, overlay effects — these are the most commonly wrong
      > - **Charts & data visualisations**: bar heights, line paths, colours, labels, axes, legends — treat every detail as suspect
      > - **Icons & images**: correct asset, correct size, correct colour/fill, correct orientation
      > - **Missing elements**: scan the Figma image for anything absent from the browser — decorative lines, badges, indicators, overlays, subtle background patterns
      > - **Extra elements**: scan the browser image for anything not present in Figma
      > - **Responsive state**: confirm the viewport matches the intended breakpoint variant
      >
      > Output a numbered diff list. An empty list is almost certainly wrong — if you find nothing, re-examine. Justify an empty list explicitly.

    - Read the subagent's diff list. Triage using the priority order in Phase 7d. **Fix all layout, copy, and missing-element issues before proceeding to the next section.**

## Phase 6: Zooming Into Child Nodes

```
get_screenshot(fileKey, childNodeId)      — visual confirmation first (cheap)
get_design_context(fileKey, childNodeId)  — exact values only if needed
```

For repeated similar elements (connector lines, cards, icons): screenshot each child individually — `get_design_context` may flatten them, hiding style differences.

**Watch for Figma coordinate artefacts** — these only work at the original design dimensions:
| Artefact | Risk |
|----------|------|
| Percentage insets on absolute elements (`inset: "6.52% 3.42%"`) | Collapses without explicit parent size |
| Negative insets >100% (`inset: "-102.42% -72.65%"`) | Overflows at different sizes |
| `calc()` with large px offsets (`left: calc(50% + 227.91px)`) | Breaks at narrower containers |
| Pixel positions referencing parent size (`top: -220.86px`) | Only valid at exact original height |

Verify parent has an explicit fixed size matching Figma dimensions, or convert to size-safe values.

## Phase 7: Audit Mode

### 7a. Read the existing implementation first

Read all relevant source files — components, views, config/data layer — before fetching from Figma.

### 7b. Screenshot all sections in parallel

```
get_screenshot(fileKey, sectionNodeId)  — all sections at once
```

### 7c. Build a diff list before touching code

**Delegate the visual diff to a skeptical subagent — do not review it yourself.** Spawn an Agent per section (run in parallel) with the following prompt, passing both the Figma screenshot and the browser screenshot as context:

> You are a meticulous visual QA reviewer. Your job is to find mistakes — assume they are there, because they almost always are. You will be shown two images: a Figma design and a browser implementation. Your disposition is skeptical and critical. Do not give the benefit of the doubt.
>
> **Section context from the Figma catalogue:**
> - Section name: [name from catalogue]
> - Node ID: [nodeId]
> - Expected size: [width×height from catalogue]
> - Catalogue description: [description field verbatim]
>
> Use this context to understand what elements should be present and to anchor your diff. If the implementation is missing something the catalogue describes, that is a confirmed missing element.
>
> **Scale warning:** Scale errors are extremely common and easy to miss — elements that are too small, too large, or incorrectly proportioned relative to their surroundings. Do not trust that something "looks about right." Actively compare the relative size of every element (icons, images, text blocks, buttons, cards) against the Figma design. Ask yourself: does this element occupy the same proportion of the section as it does in the design?
>
> Go through every category below. For each one, describe what you see in both images and call out any discrepancy, no matter how small:
> - **Layout**: flex direction, alignment (horizontal and vertical), gap, padding, margin — compare every axis
> - **Typography**: font size, weight, line height, letter spacing, colour, text-transform, text-decoration — read each text node character by character
> - **Copy**: every word, punctuation mark, and line break — do not skim
> - **Colour**: backgrounds, borders, text, icon fills — flag anything that looks even slightly off
> - **Spacing**: internal padding, gaps between elements, outer margins
> - **Scale & sizing**: widths, heights, aspect ratios — this is a high-risk category. Elements are frequently too small or too large. Compare proportions carefully against the Figma design, not just against your expectations.
> - **Borders & shadows**: radius, width, colour, box-shadow offsets and blur
> - **Visual effects**: gradients (direction, stops, colours), opacity, blur, overlay effects — these are the most commonly wrong
> - **Charts & data visualisations**: bar heights, line paths, colours, labels, axes, legends — treat every detail as suspect
> - **Icons & images**: correct asset, correct size, correct colour/fill, correct orientation
> - **Missing elements**: scan the Figma image for anything absent from the implementation — decorative lines, badges, indicators, overlays, subtle background patterns
> - **Extra elements**: scan the implementation for anything not present in Figma
> - **Responsive state**: confirm the viewport matches the intended breakpoint variant
>
> Classify each finding: **Layout deviation** / **Copy mismatch** / **Missing element** / **Wrong element** / **Correct**. Output a numbered diff list. An empty list is almost certainly wrong — if you find nothing, re-examine. Justify an empty list explicitly.

Only fetch `get_design_context` for sections where the subagent identified layout deviations needing exact values.

### 7d. Triage priority

1. Wrong layout or structure — fix immediately
2. Wrong copy — fix immediately
3. Missing element — fix immediately
4. Minor spacing deviation (1–4px) — fix if visible; skip if within Tailwind rounding
5. Pixel-perfect padding differences with no visual impact — skip

### 7e. Batch all edits, verify once

Make all fixes, then run typecheck/build once. Update catalogue if any description was wrong.

## Catalogue File

`.figma/[fileKey].md` — relative to project root.
- Add to `.gitignore` — generated, may go stale
- `mkdir -p .figma/` on first use in a project
- Shared across agents — avoids re-fetching

## Key Rules (do / don't)

| Don't | Do |
|-------|----|
| Fetch `0:1` (page root) | Start from URL's `node-id` |
| Call `get_metadata` in main session | Route through subagent |
| Name sections from visual guesses when layer names exist | Use XML `name` attribute |
| Use layer names when frames are generic | Screenshot each child; name from visual content |
| Fetch `get_design_context` for every section upfront | Screenshot first; escalate only for discrepancies |
| Ignore copy when checking visual fidelity | Cross-check all text nodes explicitly |
| Check only component JSX for copy | Also check config/data layer |
| Leave Figma asset URLs in code | Download and commit before writing code |
| Use expiring URLs for connector lines/dividers | Reconstruct with CSS |
| Assume repeated elements are identical | Screenshot each child to confirm |
| Force a shared component to fit the layout | Inline markup when fidelity and DRY conflict |
| Hardcode a colour without checking tokens | Cross-reference against design system |
| Run typecheck after each edit | Batch edits; verify once at the end |
| Re-fetch root metadata for a missing section | Screenshot root first to visually search |
| Implement a section not found in Figma | Ask user for the specific node URL |
| Implement a sub-component node in isolation | Fetch parent metadata first |
| Re-fetch a node before downloading assets | Download immediately after each fetch |
| Move on without visual verification | Step 12 is a hard gate — screenshot, compare, fix HIGH priority issues first |
