# Roadmap

## Phase 1: Core Algorithm (C++ + OpenCV)

- [x] 1.1 Image loading and preprocessing (single image + batch mode support)
- [ ] 1.2 Background detection (Auto + Manual tap) — **Removed for now** (was too aggressive)
- [x] 1.3 Palette extraction — Now respects content mask
- [ ] 1.4 Manual palette editing (Pro only)
- [x] 1.5 Structure tensor and direction field computation — Implemented as **per-color direction fields** with smoothing and gap filling
- [x] 1.6 Streamline generation — Per-color + limited boundary crossing + early seed filtering
- [x] 1.7 Coverage map — Running sum, configurable accumulation factor and width offset
- [x] 1.8 Importance sampling and seed generation — Fully implemented with tunable weights
- [x] 1.9 Parameter system — Significantly expanded (seeding, coherence, coverage, opacity, etc.)
- [x] 1.10 Second refinement pass — Automatically seeds uncovered areas after first pass
- [ ] 1.11 High-quality final render (full resolution, same parameters as preview)
- [ ] 1.12 Rendered image export (screen 1080p for Standard, high-resolution printable for Pro)
- [ ] 1.13 Video/GIF export of drawing animation
- [ ] 1.14 SVG export (with variable width/height data)
- [ ] 1.15 G-code export (for pen plotters)
- [ ] 1.16 Multi-threaded stroke generation — Planned (per-color parallelization)
- [ ] 1.17 Prepare core library for Metal bridging (clean public APIs, parameter structs, efficient data layouts)
- [ ] 1.18 Optional CUDA acceleration path during development (direction field + prototype parallel tracing kernel)

## Phase 2: iOS + macOS Application (Universal App)

- [ ] 2.1 Universal Xcode project setup (iOS + macOS)
- [ ] 2.2 Bridge C++ core to Swift
- [ ] 2.3 Main UI with image picker (single + batch import)
- [ ] 2.4 Real-time animated preview powered by Metal Compute pipeline
- [ ] 2.5 Parameter sliders UI
- [ ] 2.6 Smart Layers UI (Pro only)
- [ ] 2.7 Batch Processing UI (Pro only)
- [ ] 2.8 Preset save / load system
- [ ] 2.9 iCloud Sync implementation (Pro only)
- [ ] 2.10 Export system with Standard / Pro tier logic
- [ ] 2.11 Metal Compute pipeline implementation (per-color direction field, parallel streamline generation, atomic coverage)
- [ ] 2.12 Unified memory (`MTLBuffer`) sharing between Swift and GPU for zero-copy data flow (coverage map, images)
- [ ] 2.13 GPU-accelerated high-quality rendering and variable-width stroke rasterization on Metal

## Phase 3: Polish & Release

- [ ] 3.1 UI/UX refinement and performance optimization
- [ ] 3.2 App icon, screenshots, and marketing assets
- [ ] 3.3 Privacy policy and legal requirements
- [ ] 3.4 App Store submission and review (iOS + macOS)

## Recent Improvements (as of June 2026)

- Per-color direction fields with coherence-guided smoothing and gap filling
- Real-time ASCII progress bar during stroke generation
- Configurable seeding density for first and refinement passes
- Second refinement pass on uncovered areas
- Coverage map now much better aligned with rendered output
- Brush thickness and brush opacity properly supported in renderer
- Extensive debug timing and logging

## Future / Nice-to-Have Features

- Revisit background detection with a more robust method
- True variable-width strokes (per-point width)
- Full export pipeline (SVG, video, G-code)
- Manual palette editing
- Multi-threaded stroke generation