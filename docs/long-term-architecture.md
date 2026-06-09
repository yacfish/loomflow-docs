# Granular Pipeline Architecture in LoomFlow Core

**Status:** Recommendation for long-term architecture  
**Applies to:** LoomFlow Core + future Argus integration + iOS Metal backend

---

## Overview

Instead of building LoomFlow as one large monolithic "image → artistic strokes" component, the recommended approach is a **granular, stage-based architecture**.

The artistic stroke pipeline is deliberately broken down into small, well-defined, independent stages. Each stage has its own focused interface:

- `IStructureTensorComputer` — gradient analysis, local orientation + anisotropy
- `IOrientationFieldProcessor` — smoothing, coherence enhancement, diffusion
- `IStrokeSeedSelector` / `IDensityMapGenerator` — importance sampling
- `IStrokeGenerator` — streamline integration or parallel random walks following the field
- `IStrokeRenderer` — variable-width stroke rasterization (for live preview)
- `IVectorExporter` — SVG / G-code / plotter-ready output

A lightweight coordinator (`LoomFlowEngine`) wires the active stages together according to the current parameters and desired output type.

This design keeps the **mathematical core** completely platform-agnostic while allowing hardware-specific optimizations only where they deliver real value.

---

## Why This Granular Approach Matters for the Project

### 1. Maximum Reuse Across Very Different Hardware Targets
LoomFlow needs to run in multiple environments:
- iOS / iPadOS (Metal + Core ML)
- Future desktop / Argus nodes (CPU reference, Vulkan, or Metal via metal-cpp)
- Pen plotter pipeline (headless, CPU-only export)

With a granular design, the **same high-level logic and orchestration code** works everywhere. Only the performance-critical inner loops of specific stages are re-implemented for Metal. Everything else (parameter handling, pipeline wiring, export, UI glue) stays 100% common C++.

### 2. Incremental and Low-Risk Metal Porting
You do **not** need to rewrite the entire algorithm in Metal from day one.

Realistic split (based on current LoomFlow flow):
- **High GPU value** (port early): Structure tensor computation, orientation field smoothing/diffusion
- **Medium GPU value** (port second): Parallel stroke generation (reformulated as many independent walks), raster preview splatting
- **Low / no GPU value** (keep on CPU): High-level scheduling, seed selection strategy, color sampling logic, all export code, parameter validation

This means only **~25–40%** of the core algorithm typically needs a Metal implementation. The rest stays in clean, testable, portable C++.

You can validate the first Metal stages on real hardware (iPhone 11 Pro Max) while the rest of the pipeline still runs on the reference CPU implementation — dramatically reducing risk and debugging time.

### 3. Excellent Testability and Correctness
Each stage can (and should) be tested in complete isolation.

Recommended practice:
1. Finish and freeze a high-quality **CPU reference implementation** first.
2. Write numerical + image-comparison (golden image) tests against it.
3. Only then implement the Metal version of a stage.
4. Add a test that runs both implementations on identical input and verifies they produce equivalent results within tolerance.

This pattern is the professional standard in production computer vision and graphics engines. It turns the classic "GPU results look different" problem into a manageable, incremental process.

### 4. Natural and Powerful Integration with Argus
Because each stage is a small, self-contained unit with clear inputs and outputs, individual stages (or small groups of stages) can be exposed directly as **Argus nodes**.

Benefits:
- Users can mix LoomFlow stages with other Argus nodes (cameras, filters, recorders, display nodes, etc.) in a visual graph.
- LoomFlow becomes a first-class citizen inside the Argus ecosystem instead of a black-box external library.
- Future plugin system can let users create or share new stages (e.g., a custom `IStrokeGenerator` variant) without touching the rest of the codebase.

### 5. Future-Proofing and Extensibility
The project has several long-term directions:
- Standalone iOS app with beautiful SwiftUI interface
- Deep Argus integration (node-based creative pipelines)
- Pen plotter / physical output workflows
- Potential community plugin / node sharing system

A granular architecture makes all of these easier:
- New stages can be added (advanced tensor voting, ML-based seed selection, style transfer along strokes, etc.) without modifying existing code.
- Different backends (Metal, future Vulkan, CPU SIMD) can coexist cleanly.
- The same core can power both a simple "one-click artistic filter" mode and an expert node-graph mode.

### 6. Reduced Long-Term Maintenance Burden
Monolithic GPU ports tend to become unmaintainable over time. When you later want to improve the algorithm, add a new feature, or support a new platform, you often have to touch GPU code everywhere.

With clear stage boundaries, algorithmic improvements stay in the common code, and only the affected Metal kernels need updating. This keeps the project sustainable even as a solo developer moving between machines (old Intel Mac → M1/M2 → future hardware).

---

## Recommended Implementation Pattern

Use **abstract interfaces** for each stage + a lightweight coordinator that owns the active pipeline.

Key principles:
- Interfaces live in `include/loomflow/stages/`
- CPU reference implementations live in `src/common/stages/`
- Metal implementations live in `src/metal/stages/` (only compiled for iOS targets)
- The coordinator (`LoomFlowEngine`) is responsible for stage selection, data flow between stages, and parameter propagation.
- Use a factory or simple dependency injection so the calling code (iOS app, Argus node, CLI exporter) never needs to know which concrete implementations are active.

This pattern scales cleanly from the current prototype phase all the way to a polished 1.0 product and beyond.

---

## Summary

The granular stage-based approach is not just "nice to have" — it is the key architectural decision that makes LoomFlow viable as both:

- A high-performance on-device creative tool on iOS, and
- A modular, reusable component inside the larger Argus real-time streaming framework.

It minimizes the amount of code that must be rewritten for Metal, maximizes testability and correctness, enables deep Argus integration, and keeps the project maintainable and extensible for years to come — all while allowing pragmatic, incremental development even when working on limited hardware.

---

*Add this note to `docs/ARCHITECTURE.md` or create `docs/GRANULAR_PIPELINE_DESIGN.md`.*