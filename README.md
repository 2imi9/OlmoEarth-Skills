# OlmoEarth-Skills

Agent skills for OlmoEarth workflows. Bundled instructions and small utility scripts that help any LLM prepare data, build datasets, and ship OlmoEarth projects without re-learning the same pitfalls every time.

## Skills

| Skill | What it does |
|-------|--------------|
| [`olmoearth-data-prep`](skills/olmoearth-data-prep/) | Convert raw geospatial labels into OlmoEarth-ready datasets. Recognizes the 3 verified schemas (Studio import `sample_category`, Studio export raw `es_label`, production / rslearn standard `oe_labels.{key}`), emits both AWF-style and production-style rslearn `dataset.json`, fetches real watershed AOIs (NLDI / WBD HUC-12), and runs a 7-criteria audit. Pre-empts 8 known prep pitfalls. |
| [`olmoearth-studio-job-config`](skills/olmoearth-studio-job-config/) | Recommend Studio "new job" wizard answers from a plain-English task description. Picks output type (per-pixel vs window vs detection vs embeddings), foundation model size (Nano / Tiny / Base), time-frame mode (period vs single-moment-with-context vs single-moment), imagery sources (S2 alone vs +S1), and patch size (160 / 320 / 640 / 1280 m). Bundles ~14 verified presets (crop / mangrove / land cover / soil moisture / biomass / vessel / solar / oil slick / flood / drought / burn scar / embeddings) and a validator that catches detection-with-tiny-patch and other cross-field traps. |
| [`olmoearth-embeddings`](skills/olmoearth-embeddings/) | Decide embeddings vs fine-tuning for an OlmoEarth task (grounded in the AWF Kenya tutorial's measured accuracy/time/VRAM table), then emit a runnable Jupyter notebook that extracts OlmoEarth Nano / Tiny / Base / Large embeddings for the user's rslearn dataset and trains kNN + linear-probe heads. Handles the small-dataset (<100 samples), limited-compute (T4 / Colab), similarity-search, and "no labels yet" cases. |

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
ln -s "$(pwd)/skills/olmoearth-data-prep"          ~/.claude/skills/olmoearth-data-prep
ln -s "$(pwd)/skills/olmoearth-studio-job-config"  ~/.claude/skills/olmoearth-studio-job-config
ln -s "$(pwd)/skills/olmoearth-embeddings"         ~/.claude/skills/olmoearth-embeddings
```

```powershell
# Windows, no admin needed:
New-Item -ItemType Junction `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-data-prep" `
    -Target "$(Get-Location)\skills\olmoearth-data-prep"
New-Item -ItemType Junction `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-studio-job-config" `
    -Target "$(Get-Location)\skills\olmoearth-studio-job-config"
New-Item -ItemType Junction `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-embeddings" `
    -Target "$(Get-Location)\skills\olmoearth-embeddings"
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

---

You also have access to the olmoearth-studio-job-config skill from the
same repo. When the user is creating a Studio job and asks "what output
type / what model size / per-pixel vs window / Nano vs Tiny vs Base /
how much before-after context / Sentinel-1 or just S2 / what patch
size / 3 vs 6 vs 12 month period / how big a window for detection" —
or pastes the Studio wizard fields — consult this skill before
answering, and cite the specific reference doc used.

SIX WIZARD FIELDS, decided in order:

1. Output type: per_pixel_regression | per_pixel_classification |
   window_regression | window_classification | point_detection |
   embeddings. Drives everything else.
2. Foundation model: nano (~1.4M) | tiny (~6.2M, default) | base (~90M).
   Use Base for >5 classes or <2K samples. Use Nano only when compute is
   the binding constraint.
3. Label field: project-specific. Don't invent it — read from data
   (oe_labels.{key} / es_label / sample_category).
4. Time frame: "period" (3 / 6 / 12 / custom months + start months) for
   span-like labels (crop type, mangrove). "single_moment_with_context"
   (before + after in months, optional offset_days) for date-anchored
   predictions needing context (soil moisture, flood). "single_moment"
   (observation window in hours, default ±12 h) for transient/moving
   targets (vessels, oil slicks).
5. Imagery sources: sentinel2 default. Add sentinel1 only for
   cloudy regions, texture tasks (oil/wake/moisture), or after an
   S2-only baseline plateaus. landsat not yet available in Studio.
6. Patch size: 320 m default for per-pixel and window-level.
   1280 m default for point detection (Studio's own recommendation).
   160 m only for sparse-point-derived masks or narrow features.

ANTI-PATTERNS the skill catches:
- detection + patch_size_m=320 (too small — Studio recommends 1280 m)
- detection + multi-month period for moving targets (use single_moment)
- landsat selected (not yet available in Studio)
- single_moment_with_context with before=0 and after=0
- embeddings + label_field set (embeddings have no supervised label)

REFERENCE DOCS for olmoearth-studio-job-config:

- SKILL.md:       https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/SKILL.md
- output_types:   https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/output_types.md
- model_sizes:    https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/model_sizes.md
- time_frames:    https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/time_frames.md
- imagery:        https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/imagery_sources.md
- patch_sizes:    https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/patch_sizes.md
- presets:        https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-studio-job-config/references/presets.md

BUNDLED SCRIPT:

- recommend.py: task description -> filled config JSON; --validate
                catches detection-with-tiny-patch and other cross-
                field traps; --list-presets lists all preset keys.

---

You also have access to the olmoearth-embeddings skill from the same
repo. When the user asks "should I use embeddings or fine-tune", "I
only have 50/200/1000 labels — what now", "kNN vs linear probe", "I
want to do similarity search / clustering on satellite imagery", or
"generate a notebook to run OlmoEarth on my rslearn dataset", consult
this skill before answering.

DECISION RULE (grounded in the AWF Kenya tutorial's measured numbers):

- <100 labels                    -> embeddings + kNN (cosine, k<=20)
- 100-2000 labels                -> embeddings + linear probe first
- >2000 + strong compute + prod  -> embeddings to validate, then full
                                    30-epoch fine-tune (~82-87% vs
                                    ~70-75% for embeddings)
- similarity search / clustering -> embeddings, no classifier
- no labels yet                  -> embeddings + k-means/HDBSCAN
- weak / T4 / Colab compute      -> embeddings (fine-tune impractical)

MODEL SIZE FOR EMBEDDINGS:

- Nano (128-dim):  fastest, binary tasks, massive-scale extraction
- Tiny (192-dim):  default for downstream classification
- Base (768-dim):  fine-grained (>5 classes) or imbalanced data
- Large (1024):    rarely worth it; tutorial shows it underperforms Base

CLASSIFIER ON TOP:

- kNN, cosine, L2-normalized embeddings, k=min(20, len(train))
- Linear probe on StandardScaler-normalized embeddings (DEFAULT)
- Small MLP head only when LP plateaus AND samples > 2K

REFERENCE DOCS:

- SKILL.md:            https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-embeddings/SKILL.md
- when_to_use:         https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-embeddings/references/when_to_use.md
- classifier_choice:   https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-embeddings/references/classifier_choice.md
- model_sizes:         https://raw.githubusercontent.com/2imi9/OlmoEarth-Skills/main/skills/olmoearth-embeddings/references/model_sizes.md

BUNDLED SCRIPTS:

- recommend.py:    task description + (samples, compute) -> JSON with
                   decision (embeddings vs fine-tune), classifier,
                   model size, rationale, next-step command.
- make_notebook.py: emit a runnable .ipynb that mirrors the AWF Kenya
                    tutorial, parameterized for the user's dataset
                    path, class count, class names, and model size.
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

# recommend a Studio job config from a one-line task description
python skills/olmoearth-studio-job-config/scripts/recommend.py "predict mangrove extent in Indonesia"

# tune by class count and sample count (Tiny → Base auto-bumps under 2K samples)
python skills/olmoearth-studio-job-config/scripts/recommend.py --task "vessel detection" --num-classes 4 --num-samples 1500

# validate an existing Studio config (catches detection + 320 m patch, landsat-not-yet-available, etc.)
python skills/olmoearth-studio-job-config/scripts/recommend.py --validate my_config.json

# decide embeddings vs fine-tuning for a specific task
python skills/olmoearth-embeddings/scripts/recommend.py --task "land cover, 9 classes, 200 samples, T4 GPU"

# generate a runnable notebook that extracts OlmoEarth embeddings on the user's rslearn dataset
python skills/olmoearth-embeddings/scripts/make_notebook.py \
  --dataset-path /path/to/rslearn_dataset \
  --num-classes 9 \
  --class-names "tree_cover,shrubs,cropland,builtup,bare,grassland,water,wetland,other" \
  --model nano \
  --out my_embeddings_workflow.ipynb
```

Only the standard library is required on the generation side. `shapely` (optional) enables polygon validity checks in the data-prep audit. The notebook emitted by `make_notebook.py` needs `olmoearth_pretrain`, `rslearn`, and `scikit-learn` at *runtime* — install on Colab or your training box, not on the host running the skill scripts.

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
│   ├── olmoearth-data-prep/
│   │   ├── SKILL.md                 main entry — workflow, 7 criteria, 8 pitfalls
│   │   ├── references/              loaded by agent on demand
│   │   │   ├── schema.md            3 verified schemas + es_label → oe_labels rename
│   │   │   ├── rslearn_config.md    AWF + production config layouts
│   │   │   └── pitfalls.md          8 pitfalls with cause + fix + reference case
│   │   └── scripts/                 standalone Python helpers (stdlib only)
│   │       ├── audit.py             7-criteria audit, accepts all 3 schemas
│   │       ├── fetch_aoi.py         NLDI basin + WBD HUC-12 fetcher
│   │       └── write_config.py      emit dataset.json + Studio import + fine-tune YAML
│   ├── olmoearth-studio-job-config/
│   │   ├── SKILL.md                 Studio "new job" wizard recommender
│   │   ├── references/
│   │   │   ├── output_types.md      per-pixel vs window vs detection vs embeddings
│   │   │   ├── model_sizes.md       Nano / Tiny / Base tradeoffs
│   │   │   ├── time_frames.md       period vs single-moment-with-context vs single-moment
│   │   │   ├── imagery_sources.md   when to add Sentinel-1 (and when not to)
│   │   │   ├── patch_sizes.md       160 / 320 / 640 / 1280 m rules
│   │   │   └── presets.md           ~14 verified task presets
│   │   └── scripts/
│   │       └── recommend.py         task description → filled config JSON; --validate mode
│   └── olmoearth-embeddings/
│       ├── SKILL.md                 embeddings-vs-fine-tune decision + notebook generator
│       ├── references/
│       │   ├── when_to_use.md       full decision tree grounded in AWF tutorial benchmarks
│       │   ├── classifier_choice.md kNN vs linear probe vs MLP head, with tuning notes
│       │   └── model_sizes.md       embedding-dim tradeoffs per OE model size
│       └── scripts/
│           ├── recommend.py         (samples, compute, goal) → embeddings vs fine-tune JSON
│           └── make_notebook.py     emit a parameterized .ipynb for the user's rslearn dataset
├── LICENSE
└── README.md
```

## License

MIT. See [LICENSE](LICENSE).
