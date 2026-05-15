# Patch size — how much context the model sees

Studio offers four patch sizes (160 / 320 / 640 / 1280 m). The wizard renders this as a single dropdown, but it's two questions in disguise:

1. How large is the *thing* you're predicting? (the *object* or *region*)
2. How much surrounding context does the model need to identify it?

A larger patch helps with context but increases compute (linearly with area) and reduces output resolution at inference (the window slides; bigger windows = coarser map).

## Defaults by output type

| Output type | Default patch |
|-------------|---------------|
| Per-pixel regression | 320 m |
| Per-pixel classification (segmentation) | 320 m |
| Window-level regression | 320 m (match the window the user wants to predict) |
| Window-level classification | 320 m (Studio's own "320 × 320 m region" wording) |
| Point / bbox detection | **1280 m (Studio's explicit recommendation)** |
| Embeddings | Match downstream consumer; 320 m if unsure |

## Detection: why 1280 m

Detection models need to see object *relationships and patterns*, not just isolated objects:

- A vessel is more identifiable when the model can see the surrounding water + wake + nearby vessels
- A solar array is easier to detect with adjacent panel rows in view
- An oil slick has variable shape but reliable surrounding "smooth water" texture

Studio explicitly recommends 1280 m for object detection. Drop below 1280 m only when:

- Objects are dense (>10 per km²) and small (<5 m) — fields full of irrigation pivots
- You're memory-constrained and willing to trade accuracy

## Object size → patch size rule

Roughly, your patch should be **20–80× the object's longest dimension**:

| Object size | Recommended patch |
|-------------|-------------------|
| <5 m (cars, small vessels) | 320 m |
| 5–20 m (large vessels, small solar arrays, oil drums) | 640–1280 m |
| 20–100 m (large solar arrays, ships, oil slicks) | 1280 m |
| >100 m (oil slicks, large concentrations) | 1280 m (Studio's max) |

If the object would barely span more than the patch, the model can't see context — increase the patch.

## Per-pixel: why 320 m

For segmentation, 320 m is the right default because:

- Each pixel still gets enough surrounding context (~32 × 32 pixels at 10 m S2)
- Inference produces a 320 m resolution map; that's typically what users want
- Compute scales linearly with patch area, so 640 m doubles cost

Bump to 640 m when:

- Labels depend on *neighboring* features (crop type where field boundaries matter)
- The label is a landscape property (broad ecosystem categories)

Drop to 160 m when:

- Labels are sparse points; smaller windows reduce the empty-background problem
- Narrow linear features (rivers, roads) need fine resolution and are sensitive to wasted patch area

## Window-level: match the prediction window

If the user wants ecosystem-type predictions for *320 × 320 m tiles*, the patch must be 320 m. The patch *is* the prediction unit for window-level tasks. Don't oversize — you'll predict at a coarser resolution than the user wants.

For region-aggregate regression (e.g., "average biomass per 1 km square"), match the patch to the desired aggregation tile.

## Trade-off summary

```
Patch size       Compute   Resolution   Context   Best for
160 m            1×        Fine         Minimal   Sparse points, narrow features
320 m            4×        Medium       Moderate  Default segmentation, window classification
640 m            16×       Coarse       Good      Landscape labels, larger regions
1280 m           64×       Coarsest     Maximum   Detection (Studio default)
```

Compute scales with the square of patch size (area), so 1280 m is 64× more expensive per window than 160 m. But you process *fewer* windows to cover the same area, so total cost scales linearly with area (not patch size) — patch size mostly affects *per-window* cost and *output resolution*.

## Anti-patterns

- **160 m for vessel detection** — vessel is bigger than the patch; no context. Use 1280 m.
- **1280 m for fine-resolution land cover** — output map is at 1280 m resolution, which is much coarser than S2's native 10 m. Drop to 320 m.
- **640 m for a project where the user only has a 100 km² AOI** — wastes patches on irrelevant context; the AOI itself is the size of a few patches. Drop to 320 m or 160 m.
- **320 m for tiny dense objects (parking lots, individual panels)** — objects are barely > a pixel; model can't learn them. Either go to 1280 m for context, or aggregate to window-level classification.
