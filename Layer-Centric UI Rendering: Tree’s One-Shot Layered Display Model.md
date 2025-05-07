# Layer-Centric UI Rendering: Tree’s One-Shot Layered Display Model

## Overview

TreeOS introduces a radical rethinking of UI rendering: rather than relying on diffing, event-driven redraw triggers, or layered repaint queues, Tree executes a full-path, parallelized rendering pass across all active UI nodes and merges their outputs directly into a shared framebuffer with depth and coordinate tracking. We call this model **Layer-Centric Rendering**.

The guiding philosophy:

> "Render everything in one go. Write to the framebuffer. Track layer and position. If a pixel is not covered by a higher layer, it gets displayed."

This principle eliminates the need for:
- Reactive diff-based UI updates
- Complex invalidation trees
- Frame-jittering caused by thread congestion
- Redundant z-index sorting or repaint pipelines

## Key Concepts

### 1. **Leaf-as-Renderer**
Each visible UI component corresponds to a **Leaf**, a modular, stateless rendering node. Every Leaf, once triggered by a path traversal, independently produces:
- Pixel output (texture buffer or raw draw command)
- Bounding box (screen-space coordinates)
- Assigned layer (integer z-order)

Leaf rendering is **deterministic**, **side-effect free**, and can be performed in parallel across multiple compute clusters.

### 2. **One-Shot Rendering Pass**
All triggered Leaf nodes perform rendering in parallel. The results are written into a **layer-aware framebuffer**. No layering logic is applied during this phase—only raw pixel output is stored, associated with their respective coordinates and layer tags.

This eliminates intermediate state mutation, progressive recomposition, or multiple frames of update propagation.

### 3. **Layer Visibility Resolution**
Once all outputs are committed to the shared framebuffer, a final scan pass resolves visibility per pixel:
- For each screen coordinate `(x, y)`, select the **highest-layer** pixel present at that position.
- Discard pixels from lower layers if occluded.
- Directly blit the resolved pixel data to display hardware.

This finalization is a deterministic O(n) process over the framebuffer, optimized via hardware if available.

### 4. **No Diff, No Reconciliation**
Because rendering is always triggered **explicitly by path state**, there is:
- No need to track previous component states
- No `shouldComponentUpdate` logic
- No dependency on virtual DOMs or reactive state engines

Tree’s UI is a **projection of path state**, not a stateful view tree.

---

## Example Flow

1. 100 UI components are active (scrollable list, pop-up, background).
2. Path engine triggers corresponding 100 Leaf nodes.
3. Each Leaf emits:
    - Its visual output (texture or vector)
    - A screen-space bounding box
    - A layer index (e.g., 0 = background, 10 = modal)
4. All outputs are written into a framebuffer.
5. Final pixel scan determines what’s visible per coordinate.
6. Frame is pushed to the screen—**no delay, no reflow, no debounce**.

---

## Benefits

- **Ultra-fast UI rendering**, typically within 10–30μs for complex screens
- **No flicker or frame tearing** caused by partial repaint
- **Ideal for multi-core and GPU/NPU clustered systems**
- **Scales naturally with hundreds of UI elements**
- **Structure-first design**: display is a result of structure, not reaction

---

## Future Extensions

- Layer-aware occlusion culling at the Leaf level
- Hierarchical layer grouping (for drag & drop or modals)
- Sub-pixel collision optimizations
- Integration with TreeOS memory pathways and SapClarify scheduling

---

## Quote Origin

This model was born from a simple, profound idea during the development of TreeOS:

> _“I render everything at once. Push to framebuffer. If a pixel’s not covered, it shows.”_

That’s not optimization. That’s liberation.
