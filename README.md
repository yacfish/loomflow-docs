> **Note:** This is a read-only documentation mirror for **Loomflow**.
> The full source code and active development happen in a private repository.

---

# 🎨 LoomFlow

**Turn any photo into beautiful, organic brush strokes.**

Flowing. Artistic. Direction-aware.  
Variable-width strokes that follow color, light, and form — perfect for digital art or physical pen plotters.

**Available on iPhone, iPad & Mac** (Universal — buy once, use everywhere)

---

## ✨ Core Features

- 🎨 Automatic palette extraction
- 🧭 Structure tensor direction field
- ▶️ Real-time animated preview
- 🎲 Random + Importance seeding
- 📊 Coverage-based rendering
- 🌈 **Palette-aware color variation** in flat areas (optical texture, no new colors)
- 🖌️ Manual palette editing *(Pro)*
- 🧱 Smart Layers *(Pro)*
- 📦 Batch processing *(Pro)*

## 📤 Export Options

- 🖼️ Still image (screen or high-res printable)
- 🎞️ Animated video / GIF of the drawing process
- 📐 SVG
- 🖨️ G-code *(Pro)*

## 🖼️ Examples

### Skies

| Dramatic Sky | Soft Pastel Sky |
|--------------|-----------------|
| ![Dramatic Sky](docs/gallery/loomflow_ciel-v78_strokes_preview.jpg) | ![Soft Sky](docs/gallery/loomflow_ciel2-v106_strokes_preview.jpg) |

### Flowers

| Vibrant Pink Flowers | White & Yellow Flowers |
|----------------------|------------------------|
| ![Pink Flowers](docs/gallery/loomflow_fleurs_1-v112_strokes_preview.jpg) | ![White & Yellow](docs/gallery/loomflow_fleurs_4-v111_strokes_preview.jpg) |

### Nature & Portrait

| Forest Canopy | Portrait |
|---------------|----------|
| ![Forest](docs/gallery/loomflow_foret_1-v113_strokes_preview.jpg) | ![Portrait](docs/gallery/loomflow_paris_zoom-v100_strokes_preview.jpg) |

## 🛠 Tech Stack

- **Core**: C++ + OpenCV
- **iOS + macOS**: Swift + Metal

---

## 📚 Documentation

| Document                        | Purpose                                                                 | When to Read |
|--------------------------------|--------------------------------------------------------------------------|--------------|
| [Overview](docs/overview.md)   | High-level product vision, features, and tech stack                      | Start here for context |
| [Roadmap](docs/roadmap.md)     | High-level development phases and progress                               | See overall timeline |
| [Current Pipeline](docs/current-pipeline.md) | Exact state of the implementation today                         | Understand what exists now |
| [Component Breakdown](docs/component-breakdown.md) | Detailed breakdown of current components                   | Deep dive into implementation |
| [Development Plan](docs/development-plan.md) | Active plan for Phase 0 (depth) + Phase 1 (palette + regions)     | **Current development direction** |
| [Long-term Architecture](docs/long-term-architecture.md) | Aspirational granular / stage-based vision (Argus, Metal, extensibility) | Long-term technical goals |
| [Flat Area Texture](docs/flat-area-texture.md) | Techniques for improving flat/low-coherence regions                | Specific artistic quality topic |
| [Monetization](docs/monetization.md) | Business model and tiering                                          | Product strategy |

**Note:**  
`development-plan.md` describes the **pragmatic near-term path** we're following now.  
`long-term-architecture.md` describes the **ideal long-term modular vision** (especially valuable for Argus integration).

---

## 🚀 Building the Core (Phase 1)

This repo contains the C++ core algorithm.

### Prerequisites
- CMake ≥ 3.20
- OpenCV 4.x (recommended: `brew install opencv`)

### Build

```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --parallel
```

### Run the CLI test harness

```bash
./loomflow_cli /path/to/image.jpg output_mask.png
```

### Building with a local/prebuilt OpenCV

If you compiled OpenCV yourself or are using a prebuilt version, point CMake at it:

```bash
cmake .. -DOpenCV_DIR=/path/to/your/opencv/build
cmake --build . --parallel
```

### Alternative: Older Precompiled OpenCV

We support OpenCV ≥ 4.5. If you're using an older prebuilt, point CMake at it with `-DOpenCV_DIR=...`.