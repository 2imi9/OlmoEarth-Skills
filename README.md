# OlmoEarth-Skills

Claude Code / agent skills for OlmoEarth workflows — bundled instructions and small utility scripts that help an LLM agent prepare data, build datasets, and ship OlmoEarth projects without re-learning the same pitfalls every time.

## Skills

| Skill | What it does |
|-------|--------------|
| [`olmoearth-data-prep`](skills/olmoearth-data-prep/) | Convert raw geospatial labels into OlmoEarth-ready datasets. Recognizes the 3 verified schemas (Studio import `sample_category`, Studio export raw `es_label`, production / rslearn standard `oe_labels.{key}`), emits both AWF-style and production-style rslearn `dataset.json`, fetches real watershed AOIs (NLDI / WBD HUC-12), and runs a 7-criteria audit. Pre-empts 8 known prep pitfalls. |

More skills will land here as workflows stabilize.

## Verified ground-truth sources

The skill is grounded in canonical sources rather than invented conventions. Every claim in the reference docs cites one of:

- **`OlmoEarth_sample_file.geojson`** — Studio's import format (`sample_category` + `task_name` + `observation_time`).
- **[`allenai/olmoearth_projects`](https://github.com/allenai/olmoearth_projects)** — `olmoearth_run_data/sample/` (`dataset.json`, `olmoearth_run.yaml`, `annotation_features.geojson`) and `scripts/oer_annotation_creation.py` line 565 (`"oe_labels": labels` — the standardization script that establishes `properties.oe_labels.{key}` as the production schema).
- **The official AWF tutorial** — verifies the direct-rslearn-fine-tune flow and the 3-bandset Sentinel-2 layout with `zoom_offset`.

## Install (Claude Code)

Symlink the skill folder into your Claude Code skills directory. User-global (available across projects):

```bash
ln -s "$(pwd)/skills/olmoearth-data-prep" ~/.claude/skills/olmoearth-data-prep
```

Windows — junction works without admin if both paths are on the same volume:

```powershell
New-Item -ItemType Junction `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-data-prep" `
    -Target "$(Get-Location)\skills\olmoearth-data-prep"
```

Or a real symlink (PowerShell as administrator):

```powershell
New-Item -ItemType SymbolicLink `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-data-prep" `
    -Target "$(Get-Location)\skills\olmoearth-data-prep"
```

The skill auto-loads when its description matches the user's request — no manual `/invoke` needed. **Restart your Claude Code session after installing** so the boot-time skills scan picks up the new entry.

## Standalone use (no agent)

The bundled scripts are plain Python with no skill-specific imports — runnable directly:

```bash
# 7-criteria audit on a labels GeoJSON (accepts all 3 verified schemas)
python skills/olmoearth-data-prep/scripts/audit.py path/to/labels.geojson

# fetch a real watershed polygon
python skills/olmoearth-data-prep/scripts/fetch_aoi.py --nldi-comid 12345 --out basin.geojson

# emit rslearn config + Studio import + optional fine-tune YAML
# AWF style (default — single sentinel2 layer with 3 zoom_offset bandsets)
python skills/olmoearth-data-prep/scripts/write_config.py labels.geojson out/ --finetune --num-classes 9

# production style (12 per-month layers with alias + time_offset, matches sample/dataset.json exactly)
python skills/olmoearth-data-prep/scripts/write_config.py labels.geojson out/ --config-style production
```

Only the standard library is required for the basic flow. `shapely` (optional) enables polygon validity checks in the audit.

## What this skill teaches the agent

The bundled `SKILL.md` + `references/` capture three things that took multiple debugging sessions to learn:

1. **The three schemas are not interchangeable.** `properties.sample_category` is what Studio accepts on upload. `properties.es_label` is what Studio's "Export Annotations" tab writes — but it's *not* what `olmoearth_run` consumes. Drop a raw `es_label` file into a project dir and `prepare_labeled_windows` runs cleanly while emitting **zero** labeled windows. The fix is to rename `es_*` → `oe_labels.{key}` via `oer_annotation_creation.py` (or a 5-line jq pass) before kicking off the pipeline.
2. **There are two valid `dataset.json` layouts.** AWF style (3 zoom_offset bandsets, single `sentinel2` layer, `query_config.PER_PERIOD_MOSAIC`) is the tutorial path. Production style (12 per-month layers, each `aliased` to one model input, with explicit `time_offset` per layer, vector label) is what `olmoearth_run`'s `sample/dataset.json` uses. They're both valid but not interchangeable — `--config-style {awf,production}` picks one.
3. **The 8 prep pitfalls** — wrong field names, bbox-vs-watershed AOIs, Studio MIME rejection on Windows, quantile vs equal-frequency binning, random vs spatial splits, 10K-record / 1-hour Studio upload limit, missing negative class, schema-rename trap. Each maps to a defensive default in `audit.py` or `write_config.py`.

## Evaluation

The skill was tested with the canonical skill-creator eval loop — 4 realistic prompts × 2 conditions (with-skill vs no-skill baseline) run as parallel subagents, scored with evidence-quoted assertions, aggregated into a benchmark.

| Iteration | Evals | With skill | No-skill baseline | Note |
|-----------|-------|-----------|-------------------|------|
| 1 | 3 | 16/16 (100%) | 5/16 (31%) | Original 3 prompts (Karst, Chesapeake, wetland) |
| 2 | 4 | 21/22 (96%) | 7/22 (32%) | Added Studio export → olmoearth_run prompt; **exposed a real bug** |
| 3 (spot check) | 1 | 6/6 ✓ | n/a | Bug fixed in commit `623c63d`; rename trap now caught |

The iteration-2 bug was a real misdocumentation in `references/schema.md` — the skill claimed `es_label` is what `olmoearth_run` consumes, but the canonical `oer_annotation_creation.py` (line 565) and `sample/olmoearth_run.yaml` (`label_property: "category"`) show `oe_labels.{key}` is the production schema. The fix is grounded in citations to those canonical sources.

Variance asymmetry is consistent across iterations: with-skill is tight (pass-rate stddev ~8%, time stddev ~2 s, token stddev ~3K); without-skill is wide (pass-rate stddev ~37%, time stddev ~80 s, token stddev ~17K). The skill removes the "did the agent happen to know OE conventions" lottery.

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
