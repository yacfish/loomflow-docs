# Hybrid Architecture + Depth Integration for LoomFlow – Phase 0 & Phase 1

**Status:** Draft v1.3  
**Date:** June 2026  
**Author:** Yacine Sebti + Grok

## Relationship to Long-term Architecture

This document describes the **pragmatic near-term development plan** (Phase 0: depth integration + Phase 1: palette editing and regional processing) using targeted, lightweight modularity.

The broader **long-term architectural vision** (fine-grained stage-based design for maximum cross-platform reuse and deep Argus integration) is documented in `long-term-architecture.md`.

---

## 1. Overview & Goals

This document defines a pragmatic **hybrid architecture** for LoomFlow with a new **Phase 0** focused on **depth map integration**, followed by **Phase 1** (palette editing + region masks).

**Phase 0 Goal**: Introduce depth-aware stroke generation via `IDirectionFieldComputer` for significantly better artistic results (3D flow, depth-aware seeding, smart thickness, occlusion ordering).

**Phase 1 Goal**: Deliver manual palette editing and basic regional processing with low effort.

The approach favors **targeted modularity** — particularly around direction field computation — rather than a broad refactoring of the entire stroke generation pipeline.

---

## 2. Current State of the Project

As of June 2026, LoomFlow has reached a solid but still relatively monolithic stage in its core architecture.

### Existing Structure

- **`LoomFlow`** is the main high-level class (using the Pimpl idiom). It acts as the central orchestrator and currently exposes a simple one-call API via `generateStrokes()`.
- The heavy lifting is delegated to well-separated modules in `src/core/`:
  - `PaletteExtractor` — Performs k-means palette extraction (already a separate class).
  - `DirectionFieldComputer` — Computes per-color structure tensors from image luminance using Sobel gradients + smoothing/filling.
  - `SeedSampler` — Handles random and importance-based seeding.
  - `StreamlineGenerator` — Performs the actual stroke tracing (Euler integration, coherence stopping, coverage updates).
  - `CoverageMap` — Manages soft coverage accumulation and refinement logic.
  - `StrokeRenderer` — Renders the final `StrokeList` to raster images (CPU + OpenCV).
- The `Stroke` struct already contains a `layer_id` field, clearly added in anticipation of future layering features.
- `RenderParams` has grown significantly and already contains many artistic controls (seeding mode, brightness mapping, color variation, refinement settings, etc.).

### Current Strengths

- Good separation of concerns at the module level (`src/core/`).
- Strong focus on artistic quality (per-color direction fields, Level 1 color variation, brightness-aware thickness, iterative refinement).
- Clean intermediate representation (`StrokeList` + `PointWithWidth`).
- The project is already thinking ahead about extensibility (`layer_id`, `RenderParams` richness).

### Current Limitations & Coupling Points

Despite the good module separation, several important capabilities remain difficult or impossible without modifying core logic:

- **Palette editing** is not supported. The palette is extracted internally inside `generateStrokes()` with no public way to inspect or modify it beforehand.
- **Regional processing** is not exposed. There is no way to restrict stroke generation to a specific part of the image using a mask.
- **Direction field flexibility** is limited. `DirectionFieldComputer` is a concrete class. There is no clean way to inject alternative sources such as depth maps (e.g. Depth Anything) or user-provided direction fields.
- **Multi-region / multi-layer workflows** require users to manually instantiate multiple `LoomFlow` objects and combine results, with no helper utilities.
- `LoomFlow` still tightly couples palette extraction, direction field computation, and stroke generation inside a single method.

In short, while the low-level modules are reasonably well designed, the **high-level orchestration** in `LoomFlow` remains quite monolithic, making advanced use cases (palette editing, custom direction fields, regional control) harder than they should be.

---

## 3. Proposed Architecture (Hybrid + Modular)

We adopt a **targeted modular design** that introduces two new key abstractions while keeping most of the existing logic cohesive.

### Core Components

```cpp
// Existing
class PaletteExtractor;

// New modular components
class IDirectionFieldComputer {
    virtual PerColorDirectionFields compute(
        const cv::Mat& image,
        const cv::Mat& content_mask,
        const std::vector<PaletteColor>& palette,
        const RenderParams& params,
        const cv::Mat& depth = cv::Mat()   // optional depth map input
    ) = 0;
};

class StreamlineEngine {
    StrokeList generate(...);
};

// Rendering
class StrokeRenderer;

// High-level class
class LoomFlow {
public:
    PaletteExtractor&        getPaletteExtractor();
    IDirectionFieldComputer& getDirectionFieldComputer();
    StreamlineEngine&        getStreamlineEngine();
    StrokeRenderer&          getRenderer();

    // Phase 1 high-level API
    std::vector<PaletteColor> extractPalette(const RenderParams& params);
    void setPalette(const std::vector<PaletteColor>& palette);
    StrokeList generateStrokes(const RenderParams& params,
                               const cv::Mat& region_mask = cv::Mat());

    void setDirectionFieldComputer(std::unique_ptr<IDirectionFieldComputer> computer);
};
```

### Depth Map Integration (Key Phase 0 Enhancement)

`IDirectionFieldComputer` can accept an optional depth map. This enables:

- Strokes that follow **3D surface geometry** (better flow on fabric, hair, faces, etc.)
- **Depth-aware seeding** (prioritize foreground elements)
- **Smart thickness variation** (thinner strokes for distant areas)
- Automatic **layer ordering** based on depth for correct occlusion

Depth maps are generated **externally** (e.g. via Depth Anything V2 on Core ML / ONNX) and passed into the platform-agnostic C++ core. The core itself stays lightweight and does not embed ML models.

**Design rationale:**
- `IDirectionFieldComputer` is the primary extension point (supports image gradients, depth maps, or custom fields).
- `StreamlineEngine` groups the more stable tracing and coverage logic.
- `LoomFlow` remains a convenient high-level facade while exposing components when needed.
- Depth maps are generated externally and passed in, keeping the C++ core platform-agnostic.

---

## 4. Phase 0: Depth Map Integration (Next Focus)

**Goal**: Enable depth-aware strokes as the immediate priority.

### Key Deliverables – Phase 0

- Extend `IDirectionFieldComputer` to accept optional depth maps
- Support depth-aware seeding, stroke flow, and thickness variation
- Enable automatic layer ordering based on depth (occlusion)
- Keep depth generation external (Core ML / ONNX) — C++ core stays platform-agnostic

Depth integration is the highest-value near-term improvement for stroke quality and will drive the creation of the `IDirectionFieldComputer` interface.

### External Depth Generation (Implemented)

A working external depth map generator has been created:

- **Tool**: `scripts/generate_depth.py` using **Depth Anything V2 Small** via ONNX Runtime
- Can be run standalone or automatically via `tests/run_test.sh`
- Successfully produces high-quality depth maps on real images
- Fully external to the C++ core (as designed)

This gives us a practical way to generate and test with real depth maps during development. On-device Core ML integration can be added later for the final app.

---

## 5. Phase 1 Implementation Plan (Following Phase 0)

### 5.1 Key Deliverables in Phase 1

| Feature                              | Priority | Notes |
|--------------------------------------|----------|-------|
| `extractPalette()` + `setPalette()`  | High     | Enables manual palette editing |
| `generateStrokes(..., region_mask)`  | High     | Regional processing support |
| `layer_id` in `RenderParams`         | Medium   | Basic layer tagging |
| `mergeStrokeLists()` utility         | Medium   | Help combine multi-region results |
| Public `setDirectionFieldComputer()` | High     | Now expected after Phase 0 |

### 5.2 Direction Field Strategy

- The current structure tensor implementation becomes the default `IDirectionFieldComputer`.
- We refactor internally to use the interface during Phase 1.
- Public swapping of direction field computers is deferred to early Phase 2.

---

## 6. Updated Data Flow (Phase 0 + Phase 1)

```
User
  │
  ├── loadImage()
  │
  ├── extractPalette()                    ← New
  │
  ├── setPalette(edited)                  ← New (optional)
  │
  ├── generateStrokes(params, region_mask) ← Enhanced
  │     ├── PaletteExtractor or provided palette
  │     ├── IDirectionFieldComputer
  │     ├── StreamlineEngine (respects region_mask)
  │     └── Return StrokeList
  │
  └── (Optional) mergeStrokeLists()
```

---

## 7. Component Responsibilities

| Component                    | Responsibility                              | Extensibility Focus             |
|-----------------------------|---------------------------------------------|---------------------------------|
| `PaletteExtractor`          | Palette extraction                          | Low                             |
| `IDirectionFieldComputer`   | Compute per-color direction fields          | **High** (Depth, Custom fields) |
| `StreamlineEngine`          | Seeding, tracing, coverage                  | Medium                          |
| `StrokeRenderer`            | Final rendering                             | Medium                          |
| `LoomFlow`                  | Orchestration + convenient high-level API   | Coordinator                     |

---

## 8. Benefits of This Design

- Delivers immediate value (palette editing + regional processing) with low effort.
- Introduces the most important extension point (`IDirectionFieldComputer`) early.
- Maintains backward compatibility.
- Creates a clean evolutionary path toward full Smart Layers.
- Avoids over-fragmentation of the tracing logic in Phase 1.

---

## 9. Future Phases (Summary)

**Phase 2**
- Richer multi-region merging
- Full exposure of `StreamlineEngine`

**Phase 3 – Smart Layers**
- Higher-level `SmartLayerManager` built on top of the modular components
- Integration of segmentation models (YOLO, SAM, etc.)

---

## 10. Open Questions

- Should `IDirectionFieldComputer` receive the `region_mask` directly?
- When should `setDirectionFieldComputer()` become public?
- Is `StreamlineEngine` expected to become swappable in the future?

---

*This document replaces earlier informal notes about the hybrid approach.*
