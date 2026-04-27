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

Catalogue index lives at `.figma/[fileKey].md`. Section files live at `.figma/[fileKey]/[section-slug].md`. Shared styles live at `.figma/[fileKey]/shared-styles.md`.

- **Index exists + section present** → Phase 5 (build) or Phase 7 (audit)
- **Index exists + section missing** → Check for naming mismatch first. Then `get_screenshot(fileKey, rootNodeId)` to visually search. If still not found, ask the user for the specific node URL — do not guess.
- **Not exists** → Phase 4

## Phase 4: Build the Catalogue

The catalogue is split into three file types so each subagent loads only what it needs:

| File | Purpose | Who reads it |
|------|---------|--------------|
| `.figma/[fileKey].md` | Index: section list, node IDs, slugs, one-line descriptions | Every agent at the start |
| `.figma/[fileKey]/shared-styles.md` | CSS tokens, typography scale, shared spacing | Every implementation agent |
| `.figma/[fileKey]/[section-slug].md` | Full detail for one section: layout, colours, copy, assets | Only the agent implementing/auditing that section |

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

### 4c. Write the catalogue index

Save to `.figma/[fileKey].md`. This file must stay small — it is loaded into every agent's context. No style values, no copy, no code blocks here.

```markdown
# Figma Catalogue — [File Name]
File key: [key]
Root node: `[nodeId]` ("[name]" — [width]×[height]px)
Catalogued: [date]

Shared styles: `.figma/[fileKey]/shared-styles.md`

## Sections

| Section | Node ID | Size | Slug | Description |
|---------|---------|------|------|-------------|
| Hero | `9936:51` | 1440×803px | `hero` | Full-width hero with gradient background, heading, subheading, two CTAs, and hero illustration |
| Features | `9936:102` | 1440×960px | `features` | Three-column feature grid with icon cards |
| Pricing | `9936:188` | 1440×720px | `pricing` | Two-tier pricing cards with CTA buttons |

## DRY Opportunities
- **[Pattern]**: Appears in [Section A] (`hero`), [Section B] (`features`) — candidate for `<Component />` or `.class`
```

### 4d. Write the shared styles file

Save to `.figma/[fileKey]/shared-styles.md`. Include every value that appears in two or more sections. Values observed only from screenshots are flagged `[confirm]` — verify via `get_design_context` before use.

```markdown
# Shared Styles — [File Name]
File key: [key]

> Values marked `[confirm]` were observed from screenshots — verify exact values via get_design_context before use.

## Colours
```css
--color-bg-primary: #0F0F1A;             /* Page background */
--color-bg-card: rgba(255,255,255,0.04); /* Card surface */
--color-accent-start: #6C63FF;           /* Gradient start [confirm] */
--color-accent-end: #48CAE4;             /* Gradient end [confirm] */
--color-text-primary: #FFFFFF;
--color-text-secondary: #8B8BA7;
--color-border: rgba(255,255,255,0.08);
```

## Typography
```css
--font-family: 'Inter', sans-serif;
--text-xs:   12px / 16px;
--text-sm:   14px / 20px;
--text-base: 16px / 24px;
--text-lg:   20px / 28px;
--text-2xl:  24px / 32px;
--text-4xl:  36px / 44px;
--text-5xl:  48px / 56px;
```

## Spacing
```css
--section-padding-y: 80px;
--section-padding-x: 120px;
--card-padding:      32px;
--card-gap:          24px;
--card-radius:       16px;
```
```

### 4e. Write each section file

**Write ALL section files before fetching `get_design_context` for any section.** Screenshots from 4b are enough to write the stub — design context is fetched later, per-section, during Phase 5. If you fetch design context before the section files are on disk, you will rationalise skipping the write ("I already have the values in context") and the catalogue will be incomplete.

Save to `.figma/[fileKey]/[section-slug].md`. One file per section. Include all concrete detail — layout code, typography values, colours, copy, assets, component mappings. An implementer should be able to start without re-fetching Figma.

```markdown
# [Section Name] — [File Name]
File key: [key]
Node ID: `9936:51`
Size: 1440×803px
Slug: `hero`

See shared styles: `shared-styles.md`

## Complexity
**Score: [1–3]** — [Simple / Moderate / Complex]

| Factor | Assessment |
|--------|------------|
| Layout complexity | [e.g. Single-column text — low] |
| Visual effects | [e.g. Two gradients, one with opacity overlay — medium] |
| Assets | [e.g. One SVG illustration — low] |
| Interactive states | [e.g. Three button variants with transitions — medium] |
| Responsive variants | [e.g. Three breakpoints with layout changes — medium] |
| Data / charts | [e.g. None — low] |
| Overlapping / absolute elements | [e.g. Decorative blobs positioned with negative insets — high] |

**Error likelihood: [1–3]** — [Low / Medium / High]

Key risks on first pass:
- [e.g. Gradient direction is subtle — easy to get wrong from screenshot alone]
- [e.g. Illustration size uses absolute px that collapses at tablet — confirm parent has explicit width]
- [e.g. CTA hover state not visible in static design — requires prototype inspection]

> Complexity score drives the number of QA passes at the visual gate (1 pass / 2 passes / 3 passes). Error likelihood is a signal to the implementer about where to spend extra care.

## Layout
```css
/* Outer wrapper */
display: flex;
flex-direction: column;
align-items: center;
padding: 80px 120px;
gap: 48px;
max-width: 1440px;
```

## Typography
| Role | Font | Size | Weight | Line-height | Colour |
|------|------|------|--------|-------------|--------|
| Heading | Inter | 48px | 700 | 56px | #FFFFFF |
| Subheading | Inter | 20px | 400 | 28px | #8B8BA7 |
| CTA label | Inter | 16px | 600 | 24px | #FFFFFF |

## Colours
| Token | Hex / gradient | Usage |
|-------|---------------|-------|
| Background | `linear-gradient(180deg, #1A0F3C 0%, #0F0F1A 100%)` | Section background [confirm] |
| CTA button | `linear-gradient(135deg, #6C63FF 0%, #48CAE4 100%)` | Primary button fill |
| Border | `rgba(255,255,255,0.08)` | Card border |

## Spacing & Sizing
| Element | Value |
|---------|-------|
| Section padding (vertical) | `80px` |
| Section padding (horizontal) | `120px` |
| CTA gap | `16px` |
| Illustration width | `560px` |

## Component Mapping
| Figma layer | Codebase component | Notes |
|-------------|--------------------|-------|
| `Button/Primary` | `<Button variant="primary">` | Use existing; verify label copy |
| `Button/Secondary` | `<Button variant="outline">` | Use existing |
| `Illustration/Hero` | — | Download from Figma; no codebase equivalent |

## Assets
| Asset | Node ID | Download path | Format |
|-------|---------|---------------|--------|
| Hero illustration | `9936:88` | `public/images/hero.svg` | SVG (confirm via curl -sI) |
| Background texture | `9936:92` | `public/images/bg-texture.png` | PNG |

## Interactive States
| Element | State | Style delta |
|---------|-------|-------------|
| `Button/Primary` | hover | `opacity: 0.9; transform: scale(1.02)` |
| `Button/Primary` | active | `transform: scale(0.98)` |
| `Button/Primary` | disabled | `opacity: 0.4; cursor: not-allowed` |
| `Button/Secondary` | hover | `border-color: #FFFFFF; color: #FFFFFF` |
> States observed from Figma prototype or component variants. Mark `[confirm]` if inferred from visual only.

## Transitions
| Element | Property | Duration | Easing | Trigger |
|---------|----------|----------|--------|---------|
| `Button/Primary` | `opacity, transform` | `150ms` | `ease-out` | hover/active |
| Section | `opacity, transform` | `600ms` | `ease-out` | scroll-into-view |
> Mark `[confirm]` if not explicitly defined in Figma prototype — do not invent values.

## Responsive Variants
| Breakpoint | Node ID | Key differences |
|------------|---------|-----------------|
| Desktop (1440px) | `9936:51` | Two-column layout; illustration visible |
| Tablet (768px) | `9936:204` | Single column; illustration hidden |
| Mobile (375px) | `9936:218` | Single column; reduced padding; stacked CTAs |
> Fetch each variant node separately — do not assume styles are identical to desktop.

## Copy
```
Heading:       "Build faster with confidence"
Subheading:    "The platform that helps teams ship without breaking things."
CTA primary:   "Get started free"
CTA secondary: "See how it works"
```
```

### 4f. Identify DRY opportunities

Before implementing, scan across section files for: repeated background layers, shared button variants, common wrapper widths, repeated typography patterns. Record findings in the index under **DRY Opportunities**. Finding DRY after implementation is expensive — find it now.

### 4g. Catalogue completion gate — mandatory blocking check

**This gate is non-negotiable. You may not write a single line of implementation code until it passes.**

Run this exact command and paste the output into your response:

```bash
ls .figma/[fileKey]/
```

Then count the files listed and confirm the count equals the number of rows in your index section table (excluding `shared-styles.md`). Write the result explicitly:

```
Section file count: 8 (matches index table ✓)
```

If the count is wrong, write the missing files now. Do not proceed until the count matches.

**Why:** Once design context is in context, agents rationalise skipping the write ("I already have the values"). They don't come back. The gate prevents this.

## Phase 5: Implement a Section

**Reading order:** Load the index → load `shared-styles.md` → load the section file. Do not load other section files.

1. Re-read the section file — confirm node ID, size, component mappings.
2. Sanity-check: screenshot the node, verify it matches the section file description. Re-catalogue if wrong.
3. **Sub-component check:** If the node is a sub-component (e.g. `PieChart`, `Card`, `Button`), fetch the **parent node's metadata** first — labels and decorators are often siblings, not children.
4. Screenshot first. Fetch `get_design_context` only if you spot a discrepancy needing exact values not already in the section file.
5. From `get_design_context`, extract and update the section file with:
   - Layout: direction, alignment, padding, gap, max-width
   - Background: colour, gradient stops, image refs
   - **All text content** — copy every text node exactly. Screenshots can't catch copy mismatches.
   - Interactive states: hover/active/disabled/focus style deltas from component variants
   - Transitions: duration, easing, trigger — from Figma prototype connections or component properties. If absent, mark `[confirm]` and use project defaults; do not invent values.
   - Responsive variants: node IDs for each breakpoint variant and what differs (layout, hidden elements, font sizes)
   - Decorative elements: asset URLs, connector lines, dividers, positions
6. **Check the config/data layer** (e.g. `site-config.ts`) for copy — a mismatch may live entirely in config, invisible in JSX.
7. **Download assets immediately after fetching design context — before writing any code.** Asset URLs (`figma.com/api/mcp/asset/...`) expire in 7 days. Each new `get_design_context` call returns new URLs — do not mix URLs from different fetches.
   - **Do not assume file format from context** — inspect the URL or run `curl -sI "[url]" | grep content-type` to confirm. Figma assets labelled as images are frequently SVGs, not PNGs or JPEGs.
   - Images/illustrations → `curl -o public/images/[name].[ext] "[url]"` (use confirmed extension, e.g. `.svg`, `.png`, `.jpg`)
   - Icons (SVG) → `curl -o public/icons/[name].svg "[url]"` — check for existing equivalents first
   - Structural elements (connector lines, dividers, shapes) → **do not download** — reconstruct with CSS (`border`, `linear-gradient`, inline SVG). Never use expiring URLs for UI elements.
   - A static asset approximating a Figma vector is technical debt — flag it; correct impl is CSS.
8. Cross-reference every colour and spacing value against `shared-styles.md` and the existing design system before adding new tokens. Update `shared-styles.md` if you confirm a value that was previously `[confirm]`.
9. Apply DRY opportunities from the index.
10. **Fidelity over DRY:** If a shared component can't express the Figma layout, inline the markup. Don't distort layout to fit a component.
11. Batch all edits, then run typecheck/build once at the end.
12. **REQUIRED — Visual gate. Do not move on until this is done:**
    - Navigate to the live browser, scroll to the section, and take a **section-level screenshot** — never a full-page thumbnail. Thumbnails hide sizing and clipping bugs.
    - Fetch the Figma screenshot: `get_screenshot(fileKey, sectionNodeId)`
    - **Read the complexity score from the section file** — do not re-derive it. Use it as the pass count (1 / 2 / 3). State the score and key risks before proceeding.

    - **Spawn the QA subagent N times sequentially** (where N = complexity score). Pass the section file content and shared-styles content inline in the prompt — do not ask the subagent to read files itself. Each pass receives the same two screenshots and catalogue context, plus a growing cumulative diff list from prior passes. Instruct each pass to: (a) use the `/image-compare` skill to systematically locate differences via binary spatial decomposition before doing free-form visual analysis; (b) re-examine everything independently, not just confirm prior findings; (c) specifically hunt for issues the prior pass may have missed; (d) add new findings to the cumulative list without removing prior ones.

      Subagent prompt (adapt for pass number — pass 1 has no prior findings; passes 2+ include them):

      > You are a meticulous visual QA reviewer. Your job is to find mistakes — assume they are there, because they almost always are. You will be shown two images: a Figma design and a browser implementation. Your disposition is skeptical and critical. Do not give the benefit of the doubt.
      >
      > **Section context from the Figma catalogue:**
      > - Section name: [name from catalogue]
      > - Node ID: [nodeId]
      > - Expected size: [width×height from catalogue]
      > - Catalogue description: [description field verbatim]
      > - Key elements: [component mapping table verbatim]
      > - Expected copy: [copy block verbatim]
      > - Complexity score: [1–3] — [Simple / Moderate / Complex]
      > - Known error risks: [key risks list from catalogue verbatim — pay extra attention to these]
      >
      > Use this context to understand what elements should be present and to anchor your diff. If the implementation is missing something the catalogue describes, that is a confirmed missing element.
      >
      > **[Pass 2/3 only] Prior findings to build on:**
      > [paste cumulative diff list from previous passes]
      > Do NOT simply confirm these — re-examine the images independently first. Then add any new findings. Prior findings may also be wrong; correct them if needed.
      >
      > **Depth of analysis — follow the instructions for the complexity score above:**
      >
      > *Score 1 — Simple:* Standard checklist pass. Cover all categories below. Flag anything that looks off.
      >
      > *Score 2 — Moderate:* Standard checklist pass, plus for every visual effect (gradient, shadow, blur, opacity), zoom into that region using the `/image-compare` binary decomposition method before moving on. Do not rely on a gestalt impression of colour or lighting — isolate each effect and compare it explicitly. For every icon and image asset, confirm correct asset identity, colour fill, and that it is not clipped.
      >
      > *Score 3 — Complex:* Everything in Score 2, plus: (a) treat each data visualisation element (bar, line, arc, label, axis tick, legend item) as a separate finding — do not summarise as "chart looks correct"; (b) for every overlapping or absolutely positioned element, verify it is not clipping its parent or siblings by inspecting each edge; (c) for every gradient, describe the exact direction and each stop colour from both images before deciding if they match; (d) use `/image-compare` to decompose the entire section into quadrants first, then recurse into any quadrant that shows any difference, no matter how minor, until you reach a 50×50px region or find the root cause; (e) if the section has small repeated elements (icon rows, stat blocks, tag lists), verify each instance individually — do not assume they are all identical.
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
      > Output a numbered diff list (cumulative across all passes). An empty list is almost certainly wrong — if you find nothing, re-examine. Justify an empty list explicitly.

    - After all passes, merge findings into a final cumulative diff list. Triage using the priority order in Phase 7d. **Fix all layout, copy, and missing-element issues before proceeding to the next section.**

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

### 7a+7b. Read implementation and screenshot Figma sections in parallel

Start both at the same time — they have no dependency on each other.

- **7a** — Read all relevant source files: components, views, config/data layer. Read the catalogue index, `shared-styles.md`, and each section file being audited.
- **7b** — Fetch screenshots of all Figma sections:

```
get_screenshot(fileKey, sectionNodeId)  — one per section, all at once
```

### 7c. Build a diff list before touching code

**Delegate the visual diff to a skeptical subagent — do not review it yourself.**

**Read the complexity score from each section file** — do not re-derive it. Use it directly as the pass count.

Spawn sections in parallel (one pipeline per section), but within each section run passes **sequentially** — each pass feeds its findings into the next. Pass the section file content inline in the prompt — do not ask the subagent to read files itself. Each subagent should use the `/image-compare` skill to systematically locate differences via binary spatial decomposition before doing free-form visual analysis. Subagent prompt (adapt for pass number):

> You are a meticulous visual QA reviewer. Your job is to find mistakes — assume they are there, because they almost always are. You will be shown two images: a Figma design and a browser implementation. Your disposition is skeptical and critical. Do not give the benefit of the doubt.
>
> **Section context from the Figma catalogue:**
> - Section name: [name from catalogue]
> - Node ID: [nodeId]
> - Expected size: [width×height from catalogue]
> - Catalogue description: [description field verbatim]
> - Key elements: [component mapping table verbatim]
> - Expected copy: [copy block verbatim]
> - Complexity score: [1–3] — [Simple / Moderate / Complex]
> - Known error risks: [key risks list from catalogue verbatim — pay extra attention to these]
>
> Use this context to understand what elements should be present and to anchor your diff. If the implementation is missing something the catalogue describes, that is a confirmed missing element.
>
> **[Pass 2/3 only] Prior findings to build on:**
> [paste cumulative diff list from previous passes]
> Do NOT simply confirm these — re-examine the images independently first. Then add any new findings. Prior findings may also be wrong; correct them if needed.
>
> **Depth of analysis — follow the instructions for the complexity score above:**
>
> *Score 1 — Simple:* Standard checklist pass. Cover all categories below. Flag anything that looks off.
>
> *Score 2 — Moderate:* Standard checklist pass, plus for every visual effect (gradient, shadow, blur, opacity), zoom into that region using the `/image-compare` binary decomposition method before moving on. Do not rely on a gestalt impression of colour or lighting — isolate each effect and compare it explicitly. For every icon and image asset, confirm correct asset identity, colour fill, and that it is not clipped.
>
> *Score 3 — Complex:* Everything in Score 2, plus: (a) treat each data visualisation element (bar, line, arc, label, axis tick, legend item) as a separate finding — do not summarise as "chart looks correct"; (b) for every overlapping or absolutely positioned element, verify it is not clipping its parent or siblings by inspecting each edge; (c) for every gradient, describe the exact direction and each stop colour from both images before deciding if they match; (d) use `/image-compare` to decompose the entire section into quadrants first, then recurse into any quadrant that shows any difference, no matter how minor, until you reach a 50×50px region or find the root cause; (e) if the section has small repeated elements (icon rows, stat blocks, tag lists), verify each instance individually — do not assume they are all identical.
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
> Classify each finding: **Layout deviation** / **Copy mismatch** / **Missing element** / **Wrong element** / **Correct**. Output a cumulative numbered diff list. An empty list is almost certainly wrong — if you find nothing, re-examine. Justify an empty list explicitly.

Only fetch `get_design_context` for sections where the subagent identified layout deviations needing exact values.

### 7d. Triage priority

1. Wrong layout or structure — fix immediately
2. Wrong copy — fix immediately
3. Missing element — fix immediately
4. Minor spacing deviation (1–4px) — fix if visible; skip if within Tailwind rounding
5. Pixel-perfect padding differences with no visual impact — skip

### 7e. Batch all edits, verify once

Make all fixes, then run typecheck/build once. Update the relevant section file if any description or value was wrong.

## Catalogue Files

```
.figma/
  [fileKey].md                   # Index — always loaded
  [fileKey]/
    shared-styles.md             # Shared tokens — loaded by every implementation agent
    hero.md                      # Section detail — loaded only when implementing/auditing Hero
    features.md
    pricing.md
```

- Add `.figma/` to `.gitignore` — generated, may go stale
- `mkdir -p .figma/[fileKey]` on first use in a project
- Shared across agents — avoids re-fetching
- When passing catalogue context to a subagent, paste the content inline — do not ask the subagent to read files itself

## Hard Rules

These are the non-obvious ones most commonly violated:

- **Never fetch `0:1`** — always start from the URL's `node-id`
- **Never call `get_metadata` in the main session** — always route through a subagent
- **Screenshot first; `get_design_context` only to resolve discrepancies** — not upfront for every section
- **Download assets immediately after `get_design_context`** — URLs expire in 7 days; never mix URLs from different fetches
- **Connector lines, dividers, shapes → CSS only** — never download or use expiring URLs for structural elements
- **Sub-component nodes: fetch the parent's metadata first** — labels and decorators are often siblings, not children
- **Fidelity over DRY** — inline markup when a shared component can't express the Figma layout
- **Check config/data layer for copy** — mismatches often live there, invisible in JSX
- **Pass catalogue content inline to subagents** — do not ask subagents to read files themselves
- **Every section gets a QA pass** — complexity score sets pass count, not whether QA runs
