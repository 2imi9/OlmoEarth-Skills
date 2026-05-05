# OlmoEarth Studio schema literals

There are **two schemas** in play and they're not the same — confusing them is pitfall #1.

## Studio import schema (verified)

What you upload TO Studio. Verified against the official `OlmoEarth_sample_file.geojson`:

```json
{
  "type": "Feature",
  "geometry": { "type": "Point", "coordinates": [-122.3321, 47.6062] },
  "properties": {
    "task_name": "task_name_value_1",
    "observation_time": "2025-06-01T00:00:00Z",
    "sample_category": "sample_category_value_1",
    "sample_number": 86,
    "sample_true_false": true
  }
}
```

| Field | Type | Role |
|-------|------|------|
| `properties.task_name` | string | Annotation task name (one Studio task = one grid cell) |
| `properties.observation_time` | ISO 8601 datetime (Z suffix) | When the observation was made |
| `properties.sample_category` | string | The class label (the tag-group value in Studio) |
| `properties.sample_number` | integer | Number metadata (project-defined) |
| `properties.sample_true_false` | boolean | Boolean metadata (project-defined) |

**The `sample_*` prefix is the convention for project-defined metadata fields.** `task_name` and `observation_time` are framework fields without the prefix. The metadata types — Text / Number / Boolean / Enum / Multi-Value enum / Tag group — match Studio's annotation form primitives in the OE Studio Admin Guide.

## rslearn / fine-tune export schema (verified)

What Studio writes (or you write directly) FOR rslearn fine-tuning. From the AWF tutorial cell that reads labels:

```python
sample["properties"].get("oe_labels", {}).get("category")
```

Structure:

```json
{
  "type": "Feature",
  "geometry": { "type": "Point", "coordinates": [-2.345, 36.789] },
  "properties": {
    "oe_labels": { "category": "Woodland Forest" }
  }
}
```

This is what `rslearn dataset add_windows --fname train.geojson` consumes. Use this format for fine-tuning, not for Studio uploads.

## Which one do I use?

| Goal | Schema |
|------|--------|
| Upload labels to Studio for annotation review | Studio import (`sample_category`) |
| Run rslearn fine-tuning directly on labeled points | rslearn export (`oe_labels.category`) |
| Annotate in Studio, then export and fine-tune | Start with Studio import; Studio writes the rslearn export for you |

`scripts/audit.py` accepts either schema and tells you which one it found.

## Anti-patterns the audit flags

These names show up in past mistakes — none of them work in either schema:

| Wrong | Right (Studio import) | Right (rslearn export) |
|-------|----------------------|------------------------|
| `properties.tag` | `properties.sample_category` | `properties.oe_labels.category` |
| `properties.label` | `properties.sample_category` | `properties.oe_labels.category` |
| `properties.class` | `properties.sample_category` | `properties.oe_labels.category` |
| `properties.category` (top-level, no prefix) | `properties.sample_category` | nested under `oe_labels` |
| top-level `tag` (sibling of `geometry`) | move under `properties` | move under `properties` |

## Project-defined metadata fields

The `sample_number` and `sample_true_false` in the verified sample are placeholders for **project-defined** metadata. Real projects might have `sample_confidence`, `sample_severity`, `sample_observer_id`, etc. — they all share the `sample_*` prefix and map to the Annotation tags configured in Project settings.

If Studio rejects an upload with a "field not recognized" error:

1. Open Project settings → Annotation tags in Studio.
2. Verify the literal field name and prefix there.
3. Rename in your GeoJSON to match exactly.

## OE Studio Admin Guide pointer

The Nov 2025 OE Studio Admin Guide is the authoritative reference for:

- Metadata types: Text / Number / Boolean / Enum / Multi-Value enum / Tag group.
- Tag groups bundle enum tags (example: "Crop type" → grapes / tree fruit / trees / vegetables / water / wheat).
- Project setting: Label type = Point / Polygon / Line.
- Build Task Dataset area selection: Draw on map (< 20,000 km²) / Input coordinates / Upload CSV/GeoJSON.
- Default grid: 2 km — close to OE pretraining tile geometry of 2.56 km × 2.56 km @ 10 m/px.

When in doubt, mirror Studio's UI literals exactly.

## Image vs Point/Polygon labels

Studio supports image annotations in addition to Point / Polygon / Line. This skill targets the geometric label flow — image annotation export uses a different schema and isn't covered here.
