# Display-BRA: A Unified Output Arbitration Model for TreeOS Graphics Architecture

## Overview

In TreeOS, final image composition is not performed by traditional GPU pipelines or mixed rendering units. Instead, all rendering results—whether UI elements, text, video frames, or 3D game output—are dispatched to a specialized structural endpoint known as the **Display-BRA (Branch of Rendering Arbitration)**.

This branch is responsible for a single unified task:

> **Collect all visual outputs, sort by layer, and show the topmost visible pixel per coordinate.**

This approach removes the need for:
- Shader-based layer blending
- Centralized GPU mixing or framebuffer merging
- Deferred compositing stages or overlay managers

---

## Key Principles

### 1. **No Mixed GPU Clusters**

TreeOS does not rely on a central "compositor" or hybrid blending unit. All rendering outputs remain within their respective functional clusters:
- UI-BRA → renders text, widgets, controls
- Video-BRA → decodes video frames
- Game-BRA → renders 3D content
- Others (e.g. Notification-BRA, Sensor-Overlay-BRA) follow the same pattern

Each branch writes its own framebuffer region along with metadata:
- **Layer index** (integer depth)
- **Bounding box** (screen-space coordinates)
- **Timestamp/frame ID** (optional)

These outputs are not pre-merged.

---

### 2. **Display-BRA as Arbiter, Not Renderer**

Display-BRA is not a rendering unit. It performs no drawing or shader execution.  
Instead, it functions as a final **pixel arbitration layer** that:

- Receives all framebuffer outputs from relevant branches
- Sorts them by **layer index**
- For each pixel `(x, y)`:
  - Traverses layers from top to bottom
  - Selects the topmost non-transparent pixel
  - Sends it to display buffer

> Think of it as Z-buffer logic without rasterization: pure structure-driven visibility control.

---

### 3. **Advantages of Decoupled Display Arbitration**

| Feature | Traditional GPU | TreeOS Display-BRA |
|--------|------------------|--------------------|
| Composition | Shader-based blending | Pixel-layer arbitration |
| Pipeline pressure | High | Minimal |
| UI + Video + Game contention | Yes (shared resource) | None (logical separation) |
| Layer bugs (flashing, tearing) | Frequent | Eliminated |
| Real-time debugability | Difficult | Fully traceable by path output |

---

## Display-BRA Workflow

```text
[UI-BRA] → framebuffer + layer → 
[Video-BRA] → framebuffer + layer → 
[Game-BRA] → framebuffer + layer →
   ↓
     [Display-BRA] → per-pixel visibility sort →
     [Display hardware buffer] → screen

•	The Display-BRA is triggered after all rendering BRA complete their tasks
	•	Arbitration is performed in hardware (if supported) or optimized memory scan pass
	•	The output is clean, flicker-free, and fully deterministic

⸻

Compatibility & Extensions
	•	Supports alpha blending via layer flags (optional)
	•	Can merge frames from external sources (e.g. USB-C screen output)
	•	Path-encoded rendering makes every pixel’s origin traceable
	•	Ideal for temporal rendering, multi-screen mirroring, and layered HUD systems

⸻

Summary

The TreeOS rendering pipeline ends not in a compositor, but in a judge.
Display-BRA arbitrates what gets shown—not by guessing, but by rule:

“If a pixel is present and not covered by a higher layer, it shows. Else, it waits.”

This makes the visual output predictable, inspectable, and structure-aligned—a fitting end for a system built entirely on path integrity and logical traceability.
