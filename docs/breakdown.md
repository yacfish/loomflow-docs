# Detailed Component Breakdown

This document describes the current state of the main components in LoomFlow (as of June 2026), including recent evolutions and the direction toward a more modular architecture.

## Coverage Map

- Stored as `CV_32F`, same size as the image, initialized to 0.0
- Only pixels inside the **content mask** contribute to coverage
- Each stroke adds soft coverage using its effective thickness (influenced by `coverage_width_offset`)
- Current implementation uses an optimized single `polylines` + `findNonZero` approach for performance
- Two ways to measure coverage:
  - **Estimated coverage** (fast, based on accumulated sum)
  - **Hard coverage** (threshold-based, e.g. 0.60)
- Refinement passes continue until hard coverage target is reached or progress stalls

## Per-Color Direction Fields

- The system computes one `DirectionField` **per palette color** (not a single unified field)
- For each color:
  - A binary mask is created from hard assignment
  - Structure tensor is computed only on pixels belonging to that color
  - Direction field is smoothed and aggressively gap-filled in low-coherence areas
- This enables strokes to follow direction coherent with their own color region
- A modular `IDirectionFieldComputer` interface is being introduced to support alternative sources (e.g. depth maps)

## Importance Map & Seed Sampling

- Combines three weighted factors:
  - Direction coherence
  - Inverse coverage (favor uncovered areas)
  - Local contrast
- Two seeding modes supported:
  - **Importance sampling** (default) using CDF
  - **Random sampling** with optional minimum distance constraint
- CDF is rebuilt periodically during generation
- Minimum seeds per color enforced (currently set low to allow small regions)

## Streamline Generation & Tracing

- Uses Euler integration along the local direction field
- Controllable randomness and flow adherence
- Key stop conditions:
  - Leaves the region mask / content mask
  - Local coverage exceeds threshold
  - Coherence drops below `min_coherence_to_trace`
  - Maximum stroke length reached
- **Brightness-aware thickness** (Level 3): Stroke thickness can be mapped from color luminance (black = thick, white = thin)
- **Level 1 color variation**: In low-coherence areas, strokes can occasionally use nearby palette colors for optical texture

## Iterative Refinement

- After the main pass, uncovered areas are detected using a hard threshold (default 0.60)
- Additional refinement passes are run with higher seed density
- Stops when:
  - No significant coverage improvement
  - Maximum number of refinement passes reached
  - Target coverage achieved
- This significantly improves coverage in difficult regions

## Rendering & Ordering

- `StrokeRenderer` supports two modes:
  - Constant thickness
  - Per-point / per-segment thickness (used with brightness mapping)
- Optional stroke ordering by brightness (`darker_colors_first`)
- Supports global `brush_opacity` for blending

## Current Modular Direction (Phase 1+)

The architecture is evolving toward clearer separation:

- `IDirectionFieldComputer` — Swappable strategy for computing direction fields (image-based, depth-based, user-provided, etc.)
- `StreamlineEngine` — Handles seeding, tracing, and coverage accumulation
- `LoomFlow` — High-level orchestrator that can expose the above components

This design makes it easier to support advanced features like depth-guided strokes and regional processing without major rewrites.
