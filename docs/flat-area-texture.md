# Flat Area Texture & Palette-Constrained Optical Mixing

**Status:** Draft — for discussion and iteration  
**Date:** 2026-06-03  
**Author:** Yacine Sebti + Grok

## 1. Motivation & Problem Statement

Large flat or low-texture regions (skies, walls, water, skin, solid color blocks, etc.) are currently one of the weakest areas in LoomFlow output:

- After the main + refinement passes, these regions often remain **patchy** or contain visible **stubborn blank zones**.
- Even when coverage is high, the result can feel **too uniform or "digital"** — lacking the organic, lively surface quality found in traditional painting.
- The current direction field + tracing logic tends to either stop too early or produce repetitive strokes in low-coherence areas.

Théo van Rysselberghe’s Neo-Impressionist work (and Pointillism in general) demonstrates a powerful solution: **broken color / optical mixing**. From a distance the surface reads as coherent color masses. Up close it is a vibrant, noisy field of individual colored strokes. This creates richness, vibration, and perceived depth without requiring new colors.

For LoomFlow this is especially valuable because one of the target outputs is **pen plotter** work (SVG / G-code), where we are physically limited to the colors present in the palette.

## 2. Goals & Constraints

### Primary Goals
- Give large flat / low-coherence regions a **rich, organic texture** while preserving overall color masses and direction.
- Achieve a van Rysselberghe-like effect: uniform/coherent from distance, lively/noisy up close.
- Improve visual quality on real-world photographs that contain large smooth areas.

### Hard Constraints
- **Strict palette fidelity**: Every stroke must use **exactly one color from the input palette** (no invented colors).
- Pen plotter friendly: output must remain clean vector strokes suitable for physical plotting (one pen per stroke).
- Minimal manual work: the system should not require the user to manually define good color combinations for every image.
- Backward compatible: existing good behavior on detailed/textured areas must not regress.

## 3. Proposed Approaches (Levels)

We propose four complementary levels of increasing sophistication.

### Level 1: Palette-Aware Color Variation in Low-Coherence Regions (Implemented)

**Idea**  
In regions where local coherence is low (flat areas), occasionally allow strokes to use **other colors from the same palette** instead of always using the dominant assigned color. Variation is restricted to the N closest colors to avoid ugly random jumps.

**Parameters**
- `low_coherence_color_variation` (0.0–1.0): Probability of applying color variation when coherence is low.
- `color_variation_neighbors` (int): Number of closest palette colors considered when varying (default 5).

**How it works**
- After palette extraction, for each color we pre-compute its N closest neighbors in RGB space.
- During stroke generation (main + refinement), when a seed has low coherence, with probability `low_coherence_color_variation` we randomly pick one of its N closest neighbors instead of the strict region color.
- This creates optical texture and vibration while staying 100% inside the given palette.

**Benefits**
- Controlled, harmonious color variation in flat areas.
- Directly supports pen plotter use (no new colors).
- Easy to tune and debug (closest colors are also exposed in `palette.json` and the enhanced `palette_swatch.png`).

**Risks / Limitations**
- Can look random if the distance metric is too loose.
- Does not intelligently choose *which* colors combine well together.

### Level 2: Programmatic Discovery of Good Optical Mixing Sets

**Idea**  
After palette extraction, automatically analyze the palette to discover small groups of colors that work well together for optical mixing / broken color effects.

**How it could work**
- For each base color, score other palette colors based on:
  - Color contrast / harmony (analogous, complementary, split-complementary, etc.)
  - Perceptual difference (ΔE in Lab)
  - Desaturation potential (useful for creating depth in shadows/highlights)
- Store "recommended mixing sets" per base color.
- During rendering in flat areas, deliberately interleave strokes from these discovered sets.

**Benefits**
- Much smarter than pure distance-based jitter.
- Can produce aesthetically pleasing combinations the user would not have thought of (e.g. gray + blue + yellow over dark green).
- Still 100% palette faithful.

**Risks / Limitations**
- Requires decent color theory / perceptual metrics.
- Discovery pass adds some computation time.
- Risk of over-engineering if not tuned carefully.

### Level 3: Brightness-Aware Thickness & Stroke Ordering (Implemented)

**Idea**  
Give artists and plotter users more control over how strokes behave based on color brightness, plus basic ordering control.

**Implemented Features**
- `brightness_affects_thickness`: Map color brightness to stroke thickness (`black_thickness` → thick, `white_thickness` → thin). This creates a natural ink/watercolor-like variation.
- `darker_colors_first` + `invert_layer_order`: Control rendering order by luminance (useful for physical layering on plotters).
- `low_coherence_max_stroke_length`: Use shorter strokes when Level 1 color variation is triggered (prevents long strokes from jumping between distant colors).
- Coverage map now correctly respects brightness-based thickness.

**Benefits**
- Strong visual and physical-plotter benefits with relatively simple implementation.
- Good balance between expressiveness and predictability.
- Works well in combination with Level 1 color variation.

**Risks / Limitations**
- Does not yet generate true multi-scale or layered texture strokes (see Level 4).

### Level 4: Advanced Multi-Scale & Layered Texture (Future Investigation)

**Idea**  
In flat / low-coherence regions, deliberately generate **multiple layers** of strokes with different characteristics to create richer, more traditional painterly surfaces.

**Possible Directions**
- Generate a mix of long "anchor" strokes and many short "texture" strokes in flat areas.
- Add slight random angle deviation or scribbly behavior for texture strokes.
- Support multiple passes with different densities, lengths, and angle adherence.
- Allow explicit layering order beyond simple brightness (e.g. base color first, then accents, then texture).

**Benefits (if implemented)**
- Closest approximation to traditional broken-color / Neo-Impressionist techniques.
- Very rich near/far reading.
- Highly expressive for both digital output and physical plotters.

**Risks / Limitations**
- Significantly more complex stroke generation logic.
- Higher computational cost and longer plot times.
- Requires careful tuning and possibly new user controls to avoid looking chaotic.
- Risk of over-engineering if not driven by clear artistic needs.

## 4. How Each Level Contributes to Richness

| Aspect                        | Level 1 (Variation)      | Level 2 (Smart Mixing)       | Level 3 (Brightness + Ordering) | Level 4 (Multi-Scale Layers) |
|------------------------------|--------------------------|------------------------------|---------------------------------|------------------------------|
| Visual texture in flat areas | Good                     | Very Good                    | Very Good                       | Excellent                    |
| Optical mixing / vibration   | Moderate                 | Strong                       | Good                            | Very Strong                  |
| Palette fidelity             | Perfect                  | Perfect                      | Perfect                         | Perfect                      |
| Implementation effort        | Low                      | Medium                       | Medium                          | High                         |
| Pen plotter friendliness     | Excellent                | Excellent                    | Excellent                       | Very Good                    |
| Risk of regression           | Very Low                 | Low                          | Low                             | Medium                       |

## 5. Recommended Implementation Order

1. **Level 1 first** (quick value, low risk) — Completed
2. **Level 3** (brightness thickness + ordering) — Completed
3. Evaluate Level 2 (smart mixing sets) if more intelligent color combinations are desired
4. Level 4 (true multi-scale layered texture) can be explored later as a more advanced "painterly" direction

## 6. Open Questions

- Should color variation be applied during the main pass, refinement passes, or both?
- What distance metric works best for "nearby palette color" (RGB Euclidean, HSL, perceptual ΔE)?
- How should we expose control to the user? (single slider vs per-region strength)
- Should we also allow limited cross-region bleeding for even richer optical mixing?
- For pen plotter output, should we expose stroke ordering / layering controls?

---

**Next step:** Implement **Level 1** (Palette-Aware Color Variation in low-coherence regions).

Ready when you are.
