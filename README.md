# OlmoEarth-Skills

Agent skills for OlmoEarth workflows. Bundled instructions and small utility scripts that help any LLM prepare data, build datasets, and ship OlmoEarth projects without re-learning the same pitfalls every time.

## Skills

| Skill | What it does |
|-------|--------------|
| [`olmoearth-data-prep`](skills/olmoearth-data-prep/) | Convert raw geospatial labels into OlmoEarth-ready datasets. Recognizes the 3 verified schemas (Studio import `sample_category`, Studio export raw `es_label`, production / rslearn standard `oe_labels.{key}`), emits both AWF-style and production-style rslearn `dataset.json`, fetches real watershed AOIs (NLDI / WBD HUC-12), and runs a 7-criteria audit. Pre-empts 8 known prep pitfalls. |

More skills will land here as workflows stabilize.

## Verified ground-truth sources

Every claim in the skill is anchored to a canonical source, not invented from training data:

- **`OlmoEarth_sample_file.geojson`** — Studio's import format (`sample_category` + `task_name` + `observation_time`).
- **[`allenai/olmoearth_projects`](https://github.com/allenai/olmoearth_projects)** — `olmoearth_run_data/sample/` (`dataset.json`, `olmoearth_run.yaml`, `annotation_features.geojson`) and `scripts/oer_annotation_creation.py` line 565 (`"oe_labels": labels` — establishes `properties.oe_labels.{key}` as the production schema).
- **The official AWF tutorial** — verifies the direct-rslearn-fine-tune flow and the 3-bandset Sentinel-2 layout with `zoom_offset`.

## Use with any LLM

Two install paths. Pick whichever fits the LLM you're using.

### A. Claude Code — auto-load

Symlink (or junction on Windows) the skill folder into your Claude Code skills directory:

```bash
ln -s "$(pwd)/skills/olmoearth-data-prep" ~/.claude/skills/olmoearth-data-prep
```

```powershell
# Windows, no admin needed:
New-Item -ItemType Junction `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-data-prep" `
    -Target "$(Get-Location)\skills\olmoearth-data-prep"
```

The skill auto-loads when its description matches a request — no `/invoke` needed. **Restart your Claude Code session after installing** so the boot-time skills scan picks it up.

### B. ChatGPT / Gemini / Mistral / Llama / any other LLM — paste a prompt

Most LLMs don't have an auto-loading skill system, but you can give them the same context by pasting the prompt below at the start of a session. The LLM will then follow the same workflow when you ask about OlmoEarth data prep.

<details>
<summary><b>Click to expand the universal LLM prompt</b></summary>

```
You have access to the olmoearth-data-prep skill from
https://github.com/2imi9/OlmoEarth-Skills. When the user asks about
preparing data for OlmoEarth — OlmoEarth Studio uploads, rslearn
dataset configs, fine-tuning, watershed AOIs, label schema questions,
embedding workflows, or any task involving Sentinel-2 windows /
olmoearth_run / Planetary Computer queries in an Earth-observation
context — consult this skill before answering, and cite specific
pitfall numbers + the reference doc you used.

THREE VERIFIED SCHEMAS, NOT INTERCHANGEABLE:

1. Studio import — properties.sample_category + task_name + observation_time.
2. Studio export raw — properties.es_label + es_start_time + es_end_time
   + es_annotations_task_id. NOT directly consumable by olmoearth_run —
   must rename to schema 3 first via olmoearth_projects/scripts/oer_annotation_creation.py
   or a 5-line jq/Python pass (es_label -> oe_labels.<label_property>,
   es_start_time -> oe_start_time, es_end_time -> oe_end_time,
   es_annotations_task_id -> oe_annotations_task_id; zero-index
   integer labels if your project's class IDs start at 1).
3. Production / rslearn standard — properties.oe_labels.{key}, where
   {key} matches label_property in olmoearth_run.yaml (typically
   "category"). What olmoearth_run actually reads (verified against
   oer_annotation_creation.py line 565 and sample/olmoearth_run.yaml's
   label_property: "category").

TWO VALID dataset.json LAYOUTS:

- AWF style: 1 sentinel2 layer with 3 zoom_offset bandsets (10m / 20m
  / 60m) and query_config.PER_PERIOD_MOSAIC. Used in the AWF tutorial.
- Production style: 12 per-month layers (sentinel2_l2a_mo01..mo12),
  each alias="sentinel2_l2a" with per-layer time_offset stepping
  -180d to +150d in 30d steps. Vector label layer. Used by
  olmoearth_run.

EIGHT KNOWN PITFALLS (each with a defensive fix):

1. Wrong field names + the es_label -> oe_labels.{key} rename trap.
2. Bbox AOIs instead of real watersheds — use NLDI for upstream basins.
3. Multi-metric upload range-locking in Studio — emit one file per metric.
4. .geojson MIME-rejected on Windows as application/octet-stream —
   emit both .geojson AND .json.
5. Quantile binning gives heavy-tail imbalance — use equal-frequency bins.
6. Random splits inflate val accuracy — use spatial leave-out CV
   (sort by longitude, every Nth -> val).
7. 14K+ records timeout Studio (1-hour upload limit) — auto-shard at 10K.
8. No negative class — classifier has no "absence" signal — add a
   stable / background / other class.

WORKFLOW:

1. Validate schema (which of the three is in this file?).
2. Attach real AOIs (NLDI basin or HUC-12) — never bbox for hydrology.
3. Write outputs: dataset.json + dual-extension Studio import +
   optional Lightning fine-tune YAML.
4. Audit against the 7 OE quality criteria (volume, schema, class
   distribution, per-class volume, negative class, spatial
   distribution, polygon cleanliness).

REFERENCE DOCS (fetch if the user's question goes deeper than this prompt):

- SKILL.md:       https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-data-prep/SKILL.md
- schema.md:      https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-data-prep/references/schema.md
- rslearn config: https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-data-prep/references/rslearn_config.md
- pitfalls:       https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-data-prep/references/pitfalls.md

BUNDLED PYTHON SCRIPTS (stdlib only, runnable standalone):

- audit.py:        7-criteria audit, accepts all 3 schemas.
- fetch_aoi.py:    NLDI basin + WBD HUC-12 fetcher.
- write_config.py: emit dataset.json (--config-style awf|production),
                   Studio import (dual extension), Lightning fine-tune YAML.
```

</details>

If the LLM can't fetch URLs, copy [`SKILL.md`](skills/olmoearth-data-prep/SKILL.md) directly into the prompt — it's about 150 lines and fits comfortably in any modern context window. The reference docs and scripts can also be pasted on demand when the user's question requires them.

## Standalone use (no agent)

The bundled scripts are plain Python with no skill-specific imports — runnable directly:

```bash
# 7-criteria audit on a labels GeoJSON (accepts all 3 verified schemas)
python skills/olmoearth-data-prep/scripts/audit.py path/to/labels.geojson

# fetch a real watershed polygon
python skills/olmoearth-data-prep/scripts/fetch_aoi.py --nldi-comid 12345 --out basin.geojson

# emit AWF-style rslearn config + Studio import + Lightning fine-tune YAML
python skills/olmoearth-data-prep/scripts/write_config.py labels.geojson out/ --finetune --num-classes 9

# emit production-style config (structurally identical to olmoearth_run sample/dataset.json)
python skills/olmoearth-data-prep/scripts/write_config.py labels.geojson out/ --config-style production
```

Only the standard library is required. `shapely` (optional) enables polygon validity checks.

## Evaluation

Benchmarked with the canonical skill-creator eval loop — 4 realistic prompts × 2 conditions (with skill vs no-skill baseline), run as parallel subagents, scored against evidence-quoted assertions.

| Eval | With skill | No-skill baseline |
|------|-----------|-------------------|
| Studio import (Karst, 142K rows — schema + sharding + Windows MIME) | 6 / 6 | 0 / 6 |
| Watershed AOI (Chesapeake, 121 USGS stations) | 5 / 5 | 3 / 5 |
| rslearn fine-tune (wetlands, 800 points, 4 classes) | 5 / 5 | 0 / 5 |
| Studio export → `olmoearth_run` (6,500 features, `es_label` rename) | 6 / 6 | 4 / 6 |
| **Total** | **22 / 22 (100%)** | **7 / 22 (32%)** |

Variance is also asymmetric: with-skill is tight (pass-rate stddev ~8%, time stddev ~2 s, token stddev ~3K); the baseline is wide (pass-rate stddev ~37%, time stddev ~80 s, token stddev ~17K). The skill removes the *"did the agent happen to know OE conventions"* lottery.

## Repo layout

```
OlmoEarth-Skills/
├── skills/
│   └── olmoearth-data-prep/
│       ├── SKILL.md                 main entry — workflow, 7 criteria, 8 pitfalls
│       ├── references/              loaded by agent on demand
│       │   ├── schema.md            3 verified schemas + es_label → oe_labels rename
│       │   ├── rslearn_config.md    AWF + production config layouts
│       │   └── pitfalls.md          8 pitfalls with cause + fix + reference case
│       └── scripts/                 standalone Python helpers (stdlib only)
│           ├── audit.py             7-criteria audit, accepts all 3 schemas
│           ├── fetch_aoi.py         NLDI basin + WBD HUC-12 fetcher
│           └── write_config.py      emit dataset.json + Studio import + fine-tune YAML
├── LICENSE
└── README.md
```

## License

MIT. See [LICENSE](LICENSE).
