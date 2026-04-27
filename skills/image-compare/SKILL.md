---
name: image-compare
description: Use when comparing two images for visual differences — including visual QA after implementing UI from a Figma design or spec, when a section has been coded and you need to verify fidelity, or when a broad visual inspection has missed a difference. Mandatory during the figma-driven-development visual gate.
---

# Image Comparison

## Overview

Systematically locate subtle visual differences between two images using a binary spatial decomposition to guide the LLM's attention. Instead of scanning the whole image and relying on intuition, you build an average-colour tree for each image, diff the trees to find the most divergent region, then examine only that region with vision.

## Core Pattern

```
Build colour tree (both images) → Diff trees → Zoom into most-divergent region → Examine with vision → Step out for context if needed
```

## Step 1: Build the Colour Tree

For each image, recursively compute average colour per region. Absolute coordinates are
tracked in `build_tree` itself — **do not** try to reconstruct them in `diff_trees`.

```python
from PIL import Image
import numpy as np
from skimage.metrics import structural_similarity as sk_ssim

def avg_colour(img_array: np.ndarray) -> tuple[int, int, int]:
    return tuple(img_array.mean(axis=(0, 1)).astype(int)[:3])

def node_ssim(region_a: np.ndarray, region_b: np.ndarray) -> float | None:
    h, w = region_a.shape[:2]
    if min(h, w) < 11:
        return None
    gray_a = region_a.mean(axis=2).astype(np.float32) if region_a.ndim == 3 else region_a.astype(np.float32)
    gray_b = region_b.mean(axis=2).astype(np.float32) if region_b.ndim == 3 else region_b.astype(np.float32)
    return sk_ssim(gray_a, gray_b, data_range=255.0)

def build_tree(img_a: np.ndarray, img_b: np.ndarray, max_depth: int = 6, depth: int = 0,
               abs_x: int = 0, abs_y: int = 0,
               node_id: int = 0, parent_id: int | None = None,
               _counter: list | None = None) -> dict:
    if _counter is None:
        _counter = [0]
    h, w = img_a.shape[:2]
    my_id = _counter[0]
    _counter[0] += 1
    node = {
        "id": my_id,
        "parent_id": parent_id,
        "depth": depth,
        "abs_region": (abs_x, abs_y, w, h),
        "colour": avg_colour(img_a),
        "ssim_score": node_ssim(img_a, img_b),
        "children": [],
    }
    if depth < max_depth and min(h, w) >= 11:
        if w >= h:
            mid = w // 2
            halves = [
                (img_a[:, :mid], img_b[:, :mid], abs_x,       abs_y),
                (img_a[:, mid:], img_b[:, mid:], abs_x + mid, abs_y),
            ]
        else:
            mid = h // 2
            halves = [
                (img_a[:mid, :], img_b[:mid, :], abs_x, abs_y),
                (img_a[mid:, :], img_b[mid:, :], abs_x, abs_y + mid),
            ]
        node["children"] = [
            build_tree(ha, hb, max_depth, depth + 1, ax, ay, _counter[0], my_id, _counter)
            for ha, hb, ax, ay in halves
        ]
    return node
```

**max_depth guidance:**
- 4 → ~16 leaf regions (fast, good for large layout differences)
- 6 → ~64 leaf regions (default, catches most UI changes)
- 8 → ~256 leaf regions (use when colour shift is extremely subtle)

## Step 2: Diff the Trees

Walk both trees together and score each node by colour distance. Record absolute coordinates for every node — these are used both to crop the image and to map findings back to the page (e.g. for `elementFromPoint` or DevTools inspection).

```python
def colour_distance(c1, c2) -> float:
    return sum((a - b) ** 2 for a, b in zip(c1, c2)) ** 0.5

def diff_trees(tree_a: dict, tree_b: dict) -> tuple[list[dict], dict[int, dict]]:
    """Return (diffs sorted by distance, index mapping node_id → diff_node).
    abs_region is (x, y, w, h) in image pixels — use these to locate the
    element on the source page via document.elementFromPoint(x, y).
    parent_id links to the same node_id in the index for walking up the tree.

    NOTE: abs_region is read directly from each node — build_tree must be
    called with abs_x/abs_y so coordinates are absolute from the start.
    Do NOT recompute offsets here; that pattern is broken (child nodes always
    store (0,0) as their top-left relative to their own slice).
    """
    results = []
    stack = [(tree_a, tree_b)]
    while stack:
        a, b = stack.pop()
        x, y, w, h = a["abs_region"]
        results.append({
            "id": a["id"],
            "parent_id": a["parent_id"],
            "depth": a["depth"],
            "abs_region": (x, y, w, h),
            "centre": (x + w // 2, y + h // 2),
            "distance": colour_distance(a["colour"], b["colour"]),
            "colour_a": a["colour"],
            "colour_b": b["colour"],
        })
        for child_a, child_b in zip(a["children"], b["children"]):
            stack.append((child_a, child_b))
    sorted_results = sorted(results, key=lambda n: n["distance"], reverse=True)
    index = {n["id"]: n for n in sorted_results}
    return sorted_results, index
```

**Prefer leaf nodes when selecting candidates** — parent nodes accumulate divergence from all children. Filter to the deepest depth before picking top-N:

```python
diffs, index = diff_trees(tree_a, tree_b)
max_d = max(n["depth"] for n in diffs)
leaf_diffs = [n for n in diffs if n["depth"] == max_d]
top_candidates = leaf_diffs[:3]
```

## Step 3: Zoom and Examine

Take the top-N most-divergent **leaf** nodes (start N=3). For each, crop both images to that region and examine with vision:

```python
def crop_region(img_path: str, abs_region: tuple, padding: int = 20) -> Image.Image:
    img = Image.open(img_path)
    x, y, w, h = abs_region
    x1 = max(0, x - padding)
    y1 = max(0, y - padding)
    x2 = min(img.width,  x + w + padding)
    y2 = min(img.height, y + h + padding)
    return img.crop((x1, y1, x2, y2))
```

Save crops as temp files, then pass both to the Read tool (which renders images inline for vision).

**Examination prompt to send yourself:**
> "These are two crops of the same UI region. Identify every visual difference: text, colour, size, spacing, visibility, icon, state."

## Step 4: Step Out for Context

If you find a candidate difference but can't tell what UI element it belongs to, step out by walking **up the index** to the parent node — no re-cropping needed, the parent's `abs_region` is already computed:

```python
def step_out(node: dict, index: dict, img_a: Image.Image, img_b: Image.Image,
             padding: int = 20) -> tuple[Image.Image, Image.Image, dict]:
    """Return crops of the parent node and the parent dict itself."""
    parent = index[node["parent_id"]]
    crop_a = crop_region_from_img(img_a, parent["abs_region"], padding)
    crop_b = crop_region_from_img(img_b, parent["abs_region"], padding)
    return crop_a, crop_b, parent
```

Repeat — each call moves one level up the tree, doubling the region — until you can answer: "What changed and where in the UI?"

**Stopping rule:** Stop when the cropped region contains enough surrounding UI to name the component (e.g. you can see the button label, the card boundary, the nav section).

## Step 5: Report

For each difference found, state:
- **Location**: approximate position (e.g. "top-right, navigation bar")
- **Pixel coordinates**: `abs_region` and `centre` from the diff node — e.g. `(412, 88, 64, 32)`, centre `(444, 104)`
- **Page element** (if working from a live browser): run `document.elementFromPoint(cx, cy)` in DevTools using the centre coords to identify the DOM element
- **What changed**: specific description (e.g. "button label changed from 'Save' to 'Update'")
- **Severity**: cosmetic / functional / breaking

If the diff trees show zero divergence above threshold (~5.0 distance), the images are likely identical or the change is sub-pixel. Raise max_depth or check that the images are the same dimensions first.

## Quick Reference

| Situation | Action |
|-----------|--------|
| Images different sizes | Resize to same dimensions before building tree |
| No regions diverge | Increase max_depth or check colour space (RGBA vs RGB) |
| Difference found but context unclear | Step out: walk up the index to parent node |
| Multiple high-distance nodes | Examine top 3–5, they may be the same logical region split across depth |
| Very subtle colour shift (e.g. opacity) | Use max_depth=8 and check RGB channels individually |
| Need to identify DOM element on live page | Use `centre` coords with `document.elementFromPoint(cx, cy)` in DevTools |

## Common Mistakes

- **Examining the full image directly** — the LLM's attention disperses; the subtle difference gets missed. Always zoom first.
- **Stopping at the first divergent node** — parent nodes accumulate child divergence; check that the children aren't pointing at a more specific sub-region.
- **Forgetting padding** — a 4×4 pixel crop gives no context. Always add 20px padding minimum.
- **Re-cropping to step out** — the parent node already has the right coordinates. Use `step_out()` with the index rather than computing a new crop size.
- **Assuming one difference** — run the full top-N scan; there may be multiple independent changes.
- **Reconstructing offsets in `diff_trees` instead of tracking them in `build_tree`** — every node slice stores its top-left as `(0, 0)` relative to itself, so `child["region"][:2]` is always `(0, 0)`. Any attempt to accumulate offsets during tree traversal produces wrong coordinates. Always pass `abs_x`/`abs_y` into `build_tree` and store `abs_region` there; `diff_trees` must read those values directly without modification.
