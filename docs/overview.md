# LoomFlow Architecture

## Overview
Organic image to variable-width brush stroke converter.  
Transforms any photo into flowing, artistic brush strokes that follow natural color shapes and direction.  
**Universal app**: iPhone, iPad and Mac (buy once, use everywhere).

## 1. Input & Preprocessing
- Load single image
- Optional region mask support for processing specific areas (Phase 1)
- Content mask (exclude background)

## 2. Palette Extraction & Editing
- Automatic extraction (k-means) respecting content mask
- Manual palette editing supported via `extractPalette()` + `setPalette()` (Phase 1)
- Future: Shared palettes across layers
  - Reduce palette size (including 1-color black-and-white mode)
  - Manually replace or edit any color

## 3. Direction Field (Structure Tensor)
- For each color in the palette:
  - Create binary mask
  - Compute Sobel gradients (Ix, Iy)
  - Build structure tensor components (Axx = Ix², Axy = Ix·Iy, Ayy = Iy²)
- Compute one `DirectionField` **per palette color** (not summed into one unified field)
- Each direction field is smoothed and gap-filled for better behavior in flat areas
- Compute orientation angle per pixel: `0.5 * atan2(2*Axy, Ayy - Axx)` + 90° rotation (to follow edges)

## 4. Real-time Preview Engine
- Progressive streamline generation with visible animation
- User-adjustable sliders: palette size, stroke spacing/density, randomness, flow adherence strength, coverage target
- Two seeding modes: Random + Importance (based on tensor coherence, contrast, and inverse coverage)
- Two stroke length modes: Fixed + Adaptive
- Coverage map (`CV_32F`) that only counts non-background pixels
- Stops automatically when target coverage is reached

## 5. Smart Layers (Future / Pro)
- Higher-level system built on top of the core (see `next-steps.md`)
- Supports per-region masks and independent `RenderParams` per layer
- Foundation for on-device segmentation (YOLO, SAM, etc.) in later phases

## 6. Batch Processing (Pro only)
- Load multiple images at once
- Extract one shared palette from the entire batch
- Apply identical settings (palette + all sliders + Smart Layers) to every image
- Save / load full parameter presets

## 7. iCloud Sync (Pro only)
- Sync presets, parameter settings, Smart Layers, batch projects, and output files across all devices via iCloud

## 8. High Quality Render & Export
- Same parameters as preview but at full resolution
- Export options depend on tier:

**Standard** ($4.99):
- Still image (screen rendering, max 1080p)
- Short video/GIF of the drawing animation (8 seconds, 720p)

**Pro** ($9.99):
- High-resolution printable still image
- Long high-quality video/GIF (up to 25 seconds, 1080p)
- SVG (with variable width / height data)
- G-code (for pen plotters)

## Tech Stack
- **Core Processing**: C++ + OpenCV
- **UI & Rendering**: Swift + Metal (Universal iOS + macOS app)
- **Cloud Sync**: iCloud (Pro only)
