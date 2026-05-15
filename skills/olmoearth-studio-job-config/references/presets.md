# Studio job presets

Verified-from-Studio-wizard configs for common Earth-observation tasks. Each is a starting point — adjust the model size up/down by sample count and class count per [`model_sizes.md`](model_sizes.md).

The output schema matches the wizard's six fields plus a `rationale` block.

## Crop type mapping

```json
{
  "output_type": "per_pixel_classification",
  "foundation_model": "tiny",
  "label_field": "oe_labels.category",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [3, 4] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Bump to Base if >5 crop classes or <2K samples. Move start months to Sep/Oct in Southern Hemisphere.

## Mangrove extent

```json
{
  "output_type": "per_pixel_classification",
  "foundation_model": "tiny",
  "label_field": "oe_labels.category",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Mangrove is stable year-round; any start month is valid. Add S1 in heavily tidal areas where waterline shifts confuse S2.

## Land cover

```json
{
  "output_type": "per_pixel_classification",
  "foundation_model": "base",
  "label_field": "oe_labels.category",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Use Base for 5+ classes (urban, water, forest, cropland, bare, …). 640 m patch if labels include broad transitions.

## Soil moisture

```json
{
  "output_type": "per_pixel_regression",
  "foundation_model": "tiny",
  "label_field": "oe_labels.moisture_pct",
  "time_frame": { "mode": "single_moment_with_context", "before_months": 3, "after_months": 0, "before_offset_days": 0 },
  "imagery_sources": ["sentinel2", "sentinel1"],
  "patch_size_m": 320
}
```

S1 is the moisture signal (dielectric constant); S2 carries vegetation/NDVI as a proxy. 3 months of before-context captures the drying trend.

## Tree height / canopy density

```json
{
  "output_type": "per_pixel_regression",
  "foundation_model": "base",
  "label_field": "oe_labels.height_m",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [6] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Mid-year start captures peak leaf-on phenology. Add S1 if forest is dense (radar penetrates canopy).

## Biomass (regional average)

```json
{
  "output_type": "window_regression",
  "foundation_model": "base",
  "label_field": "oe_labels.biomass_mg_per_ha",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [6] },
  "imagery_sources": ["sentinel2", "sentinel1"],
  "patch_size_m": 640
}
```

640 m patch matches typical biomass plot scale. Base because regression with high variance benefits from richer features.

## Ecosystem type (regional)

```json
{
  "output_type": "window_classification",
  "foundation_model": "tiny",
  "label_field": "oe_labels.ecosystem",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Matches Studio's own wizard example ("ecosystem type in a 320 × 320 m region").

## Vessel detection

```json
{
  "output_type": "point_detection",
  "foundation_model": "tiny",
  "label_field": "oe_labels.vessel_type",
  "time_frame": { "mode": "single_moment", "observation_window_hours": 12 },
  "imagery_sources": ["sentinel2", "sentinel1"],
  "patch_size_m": 1280
}
```

S1 catches small vessels and night detections. Move to ±24 h if dataset is sparse. Use Base if vessel-class taxonomy is fine-grained (>3 classes).

## Solar array detection

```json
{
  "output_type": "point_detection",
  "foundation_model": "base",
  "label_field": "oe_labels.array_type",
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 1280
}
```

Solar arrays are stable installations → period mode, not single-moment. Base helps with fine-grained array-type distinctions.

## Oil slick detection

```json
{
  "output_type": "point_detection",
  "foundation_model": "tiny",
  "label_field": "oe_labels.slick",
  "time_frame": { "mode": "single_moment", "observation_window_hours": 12 },
  "imagery_sources": ["sentinel1", "sentinel2"],
  "patch_size_m": 1280
}
```

S1 is primary — slicks suppress wind-driven roughness → dark patch. S2 secondary for daylight confirmation.

## Flood damage / extent

```json
{
  "output_type": "per_pixel_classification",
  "foundation_model": "base",
  "label_field": "oe_labels.flood_state",
  "time_frame": { "mode": "single_moment_with_context", "before_months": 1, "after_months": 2, "before_offset_days": 7, "after_offset_days": 0 },
  "imagery_sources": ["sentinel2", "sentinel1"],
  "patch_size_m": 640
}
```

Before-offset 7 days skips event-day cloud cover. S1 is essential — floods happen under storm clouds. After-context 2 months captures persistence vs recovery.

## Drought monitoring

```json
{
  "output_type": "per_pixel_regression",
  "foundation_model": "tiny",
  "label_field": "oe_labels.drought_index",
  "time_frame": { "mode": "single_moment_with_context", "before_months": 6, "after_months": 0, "before_offset_days": 0 },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

6 months of before-context captures the dry-down trajectory. NDVI/EVI in S2 carries the signal; S1 not essential.

## Burn scar / fire scar

```json
{
  "output_type": "per_pixel_classification",
  "foundation_model": "tiny",
  "label_field": "oe_labels.burned",
  "time_frame": { "mode": "single_moment_with_context", "before_months": 1, "after_months": 1, "before_offset_days": 0, "after_offset_days": 0 },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

S2 SWIR band is *the* burn signal. Before/after lets the model see contrast between pre-burn and post-burn surfaces.

## Embeddings (general purpose)

```json
{
  "output_type": "embeddings",
  "foundation_model": "tiny",
  "label_field": null,
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1] },
  "imagery_sources": ["sentinel2"],
  "patch_size_m": 320
}
```

Embeddings have no label field. Tiny (192-dim) is the sweet spot for downstream consumers; Nano (128-dim) is fine for clustering only.

## Embeddings (water / wetland clustering)

```json
{
  "output_type": "embeddings",
  "foundation_model": "tiny",
  "label_field": null,
  "time_frame": { "mode": "period", "period_months": 12, "start_months": [1] },
  "imagery_sources": ["sentinel2", "sentinel1"],
  "patch_size_m": 320
}
```

Add S1 when the downstream task involves water-surface or wetland classification — radar adds the texture signal that pure S2 embeddings lack.
