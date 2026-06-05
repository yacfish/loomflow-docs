# Current Pipeline (Implementation State)

**Last Updated:** June 2026  
**Purpose:** This document describes the **actual current data flow** of LoomFlow as implemented in the code. It is meant to be kept up to date as the architecture evolves.

> **Note:** This document only reflects what is currently implemented. Planned or proposed features (such as `extractPalette()`, `setPalette()`, `region_mask` parameter, `IDirectionFieldComputer`, or `StreamlineEngine`) are **not** included here.

---

## High-Level Data Flow

```text
User
│
├── loadImage(path)
│       └── Preprocessor.load()
│           └── Stores original image + default full content_mask (255 everywhere)
│
├── generateStrokes(params)          ← Only public generation method
│       │
│       ├── Extract palette internally
│       │       └── PaletteExtractor.extract(original, content_mask, params.palette_size)
│       │           └── Stored in impl_->palette
│       │
│       ├── Compute direction fields
│       │       └── DirectionFieldComputer.computePerColor(
│       │               original, content_mask, impl_->palette
│       │           )
│       │           └── Returns PerColorDirectionFields
│       │               (one DirectionField + mask per palette color)
│       │
│       ├── Initialize coverage
│       │       └── CoverageMap.initialize(content_mask)
│       │
│       ├── Main stroke generation pass
│       │       └── StreamlineGenerator.generatePerColor(
│       │               per_color_fields,
│       │               coverage,
│       │               params,
│       │               palette
│       │           )
│       │               │
│       │               ├── For each palette color:
│       │               │       ├── SeedSampler.sampleInRegion(...)
│       │               │       │       └── Importance or Random sampling
│       │               │       │
│       │               │       └── For each valid seed:
│       │               │               ├── Early coherence filtering
│       │               │               ├── traceStroke(...)
│       │               │               │       ├── Euler integration along direction
│       │               │               │       ├── Brightness-aware thickness (if enabled)
│       │               │               │       ├── Coherence + coverage stop conditions
│       │               │               │       └── CoverageMap.addStroke(...)
│       │               │               └── Collect Stroke with color
│       │               │
│       │               └── Return partial StrokeList
│       │
│       ├── Iterative refinement loop
│       │       └── While refinement passes remain and uncovered areas exist:
│       │               ├── Compute uncovered mask (threshold 0.60)
│       │               ├── StreamlineGenerator (refinement seeds)
│       │               └── Append new strokes to result
│       │
│       └── Finalize
│               ├── Store strokes in impl_->last_strokes
│               ├── Compute final hard coverage %
│               └── Return impl_->last_strokes
│
└── Result: StrokeList
```

---

## Component Summary (Current State)

| Component                    | Role                                      | Notes |
|-----------------------------|-------------------------------------------|-------|
| `Preprocessor`              | Image loading + content mask              | Simple wrapper |
| `PaletteExtractor`          | K-means palette extraction                | Called internally inside `generateStrokes()` |
| `DirectionFieldComputer`    | Per-color structure tensor computation    | Concrete class (no interface yet) |
| `SeedSampler`               | Seed point generation                     | Supports Random + Importance modes |
| `StreamlineGenerator`       | Stroke tracing logic                      | Contains `traceStroke()` and `generatePerColor()` |
| `CoverageMap`               | Soft coverage tracking + refinement       | Optimized implementation |
| `StrokeRenderer`            | Final raster rendering                    | Supports brightness-aware thickness |
| `LoomFlow`                  | Main orchestrator                         | Still contains most high-level logic |

---

## Key Characteristics of Current Implementation

- `generateStrokes()` is the **only** public entry point for generation.
- Palette extraction happens **internally** on every call.
- Direction fields are computed **per color**.
- Refinement is handled as an **iterative loop** inside `generateStrokes()`.
- No public support yet for:
  - Extracting or setting the palette manually
  - Processing specific regions via mask
  - Swapping direction field strategies

---

*This file should be updated whenever significant changes are made to the generation pipeline.*