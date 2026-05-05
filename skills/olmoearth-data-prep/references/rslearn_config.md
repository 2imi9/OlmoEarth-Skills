# rslearn dataset config — canonical S2 template

This is the dataset `config.json` that the AWF tutorial uses verbatim. Copy it, then edit time range, output directory, and the `label` band name to match your project.

`scripts/write_config.py` emits this exact structure with the right `sort_by` for the chosen data source.

## Three bandsets at native resolutions

Sentinel-2 has bands at 10 m / 20 m / 60 m. rslearn handles this via three `band_sets` with `zoom_offset`:

```json
{
  "layers": {
    "label": {
      "type": "raster",
      "band_sets": [{ "bands": ["category"], "dtype": "int32" }]
    },
    "sentinel2": {
      "type": "raster",
      "band_sets": [
        { "bands": ["B02", "B03", "B04", "B08"], "dtype": "uint16" },
        { "bands": ["B05", "B06", "B07", "B8A", "B11", "B12"], "dtype": "uint16", "zoom_offset": -1 },
        { "bands": ["B01", "B09"], "dtype": "uint16", "zoom_offset": -2 }
      ],
      "data_source": {
        "class_path": "rslearn.data_sources.planetary_computer.Sentinel2",
        "ingest": false,
        "init_args": {
          "cache_dir": "cache/planetary_computer",
          "harmonize": true,
          "sort_by": "eo:cloud_cover"
        },
        "query_config": {
          "space_mode": "PER_PERIOD_MOSAIC",
          "max_matches": 12,
          "min_matches": 12,
          "period_duration": "30d"
        }
      }
    }
  }
}
```

## Knobs to know

| Knob | Effect |
|------|--------|
| `zoom_offset: -1` | bandset is at 2× the base resolution (20 m if base is 10 m) |
| `zoom_offset: -2` | bandset is at 4× the base resolution (60 m at base 10 m) |
| `harmonize: true` | apply the S2 BOA offset (matters for 2022+ scenes; safe to leave on) |
| `space_mode: PER_PERIOD_MOSAIC` | one mosaic per period rather than one scene per match — controls temporal density |
| `period_duration: 30d` + `max_matches: 12` | 12 monthly mosaics |
| `min_matches: 12` | reject windows that can't fill all 12 months — prevents partial training samples |

## Data source: Planetary Computer vs Element-84 Earth Search

Both serve Sentinel-2 L2A from AWS, but the cloud-cover sort key differs:

| Source | `class_path` | `sort_by` value |
|--------|--------------|-----------------|
| Planetary Computer (PC) | `rslearn.data_sources.planetary_computer.Sentinel2` | `eo:cloud_cover` |
| Element-84 Earth Search (E84) | `rslearn.data_sources.earth_search.Sentinel2` | `properties.eo:cloud_cover` |

Using the wrong key silently sorts by file order instead of cloud cover, returning cloudy mosaics that hurt model accuracy. `scripts/write_config.py --source pc|e84` selects the right one.

## Window geometry

Two standard sizes:

| Use case | Size | Resolution | Coverage |
|----------|------|------------|----------|
| Point-labeled fine-tuning | 63 × 63 | 10 m | 630 × 630 m — matches AWF tutorial; gives encoder 4-px-patch context with overlap |
| Pretraining-tile inference | 256 × 256 | 10 m | 2.56 × 2.56 km — matches OlmoEarth encoder training geometry exactly |

Studio's default 2 km grid is close to the 256 × 256 tile, so Studio tiles feed the encoder cleanly without reprojection-scale surprises.

## End-to-end add-windows flow

After writing `config.json`:

```bash
# Add windows from your labels GeoJSON
rslearn dataset add_windows \
    --root path/to/dataset \
    --group train \
    --fname train.geojson \
    --window_size 63 \
    --resolution 10 \
    --utm \
    --start 2023-01-01T00:00:00+00:00 \
    --end 2023-12-31T23:59:59+00:00

# Same for val
rslearn dataset add_windows \
    --root path/to/dataset \
    --group val \
    --fname val.geojson \
    --window_size 63 \
    --resolution 10 \
    --utm \
    --start 2023-01-01T00:00:00+00:00 \
    --end 2023-12-31T23:59:59+00:00

# Query the data source for matching scenes
rslearn dataset prepare --root path/to/dataset --workers 4

# Download and crop imagery (1–2 h for ~1500 windows)
rslearn dataset materialize --root path/to/dataset --workers 4 --no-use-initial-job
```

`prepare` only queries the catalog; `materialize` actually downloads.

## Bandset ordering note

After loading and stacking, the band order in the resulting tensor is:

```
[B02, B03, B04, B08, B05, B06, B07, B8A, B11, B12, B01, B09]
```

This is **10 m bands first, then 20 m, then 60 m** — not numeric order. The OlmoEarth normalization config (`olmoearth_pretrain.data.normalize.load_computed_config`) is keyed by band name, so as long as you keep the names with the data this works automatically. If you re-order, re-key the normalization too.
