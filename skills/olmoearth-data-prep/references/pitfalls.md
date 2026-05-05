# The 8 known OE data-prep pitfalls

Each one cost real debugging time across the Karst / Chesapeake / Potomac case studies. The skill prevents each procedurally; this doc explains why each one breaks so the skill's defaults are easy to override correctly.

## 1. Wrong field names (and confusing the THREE valid schemas)

**Symptom**: Studio import fails silently. Or olmoearth_run's `prepare_labeled_windows` reads no labels. Or rslearn `add_windows` runs without error but every label loads as nodata.

**Root cause**: There are **three verified schemas** that look similar but are not interchangeable, each tied to a different point in the pipeline:

- **Studio import** (verified against `OlmoEarth_sample_file.geojson`): the class label lives at `properties.sample_category`. Project metadata follows the `sample_*` prefix; framework fields are `task_name` and `observation_time`. Used when you upload raw labels TO Studio.
- **Studio export** (verified against `olmoearth_projects/olmoearth_run_data/sample/annotation_features.geojson`): the class label lives at `properties.es_label` (integer index). Companion fields `es_start_time`, `es_end_time`, `es_annotations_task_id`. **This is what `olmoearth_run` consumes** — its `LabeledWindowPreparer` classes read these fields directly.
- **rslearn / AWF tutorial** (verified against the AWF tutorial cell that reads labels): the class label lives at `properties.oe_labels.category`. Older / alternate format.

The wrong names are `tag`, `label`, `class`, top-level `category` (no prefix), or nesting things outside `properties`. All of those look plausible and ship silently.

**Fix**: `scripts/audit.py` accepts all three verified schemas and tells you which one it found. Match the schema to your destination — Studio upload uses #1, olmoearth_run consumes #2, the rslearn AWF flow uses #3. `scripts/write_config.py` passes the user's input through, so start from a verified sample structure and you won't drift.

**Reference cases**: Karst classification (~144K → 47K rebalanced) hit this on the import side. The audit also caught the same class of bug on the export side when first run against the official olmoearth_run sample (`es_label` wasn't recognized until the skill was patched).

## 2. Bbox AOIs instead of real watersheds

**Symptom**: Embeddings see ocean / urban / unrelated cover that has nothing to do with the watershed signal you're modeling. Model accuracy looks plausible but doesn't generalize.

**Root cause**: Drawing a bbox around a station is fast on a map but the bbox includes whatever happens to be inside it — not the upstream contributing area. For hydrology this is the difference between modeling "water near this point" and "what flows to this point."

**Fix**: `scripts/fetch_aoi.py` calls NLDI for upstream basin polygons (by NHDPlus COMID) and WBD for HUC-12 subbasins. NLDI gives precise upstream-of-station polygons; HUC-12 gives named subbasin polygons for event-scale work.

**Reference case**: Chesapeake Bay nutrient loads (121 stations × 10 years × 3 metrics). Bbox AOIs gave the model nothing useful; switching to NLDI basins recovered signal.

## 3. Studio range-locking when uploading multiple metrics

**Symptom**: Uploading TN, TP, and TSS as one dataset locks the visualization colorbar to a single range. Two of the three metrics become unreadable.

**Root cause**: Studio fits one min/max per upload session, and multi-metric uploads fight over it.

**Fix**: emit one Studio import file per metric. `scripts/write_config.py` (with the user splitting features by metric upstream) writes them as separate import files in the same output directory.

**Reference case**: Chesapeake (3 nutrient metrics on the same station set). Three separate uploads → three readable visualizations.

## 4. `.geojson` rejected as `application/octet-stream` on Windows

**Symptom**: Studio rejects the upload with a MIME error, even though the file is valid GeoJSON. Same file uploads fine from a Mac.

**Root cause**: Windows browsers tag unknown extensions as `application/octet-stream`. Studio whitelists `application/json` and `application/geo+json` but not the generic blob type. The OS — not the file content — determines the MIME.

**Fix**: always emit both `.geojson` and `.json` versions of the same payload. The user picks whichever Studio accepts on their browser/day. `scripts/write_config.py` does this by default.

**Reference case**: Karst & Potomac uploads from a Windows laptop.

## 5. Quantile binning gave 96 / 2.5 / 1.3 / 0.1 % imbalance

**Symptom**: 4-class regression-as-classification ends up with 96% in one bucket and the rest in single-digit percentages. The model trivially predicts the majority class for ~95% accuracy that means nothing.

**Root cause**: Quantile binning splits at percentile boundaries of the *distribution*, but environmental data is heavy-tailed — most of the distribution sits in one bucket. A 25/50/75% split on a log-normal distribution is *not* a 25/25/25/25% sample split.

**Fix**: equal-frequency binning (sort, then split into equal-size groups) is the default. `scripts/audit.py` warns when max/min class ratio > 10. Quantile binning becomes opt-in via an explicit flag if the user actually wants distribution-aware bins.

**Reference case**: Chesapeake nutrient regression buckets.

## 6. Random splits inflate reported accuracy

**Symptom**: Validation accuracy looks great (90%+); production performance is mediocre. Easy to ship a model that "works" on val and fails in the field.

**Root cause**: Random splits put samples from the same watershed / scene / event in both train and val. The model learns scene-specific shortcuts (sensor artifacts, regional cover patterns) instead of the underlying signal. Val measures memorization, not generalization.

**Fix**: spatial leave-out CV is the default. `scripts/write_config.py` sorts features by longitude and assigns every Nth feature to val (matches the AWF tutorial's spatial split pattern at cell 9).

**Reference case**: All three case studies — caught most clearly in Karst, where geographic correlation between sinkholes is strong.

## 7. 14K+ records timed out Studio (1-hour upload limit)

**Symptom**: Studio upload progress bar gets to ~95% then 504s. The dataset shows up as "failed" with no actionable error.

**Root cause**: Studio caps each upload session at 1 hour of processing time. Empirically, ~10K records is the safe upper bound; larger datasets hit the wall.

**Fix**: `scripts/write_config.py` auto-splits at 10K records, partitioning by longitude so each shard is geographically coherent rather than randomly distributed. This means each shard can be uploaded as its own Studio "region" or rejoined later for training.

**Reference case**: Karst (~47K rebalanced records, split into 5 regional shards).

## 8. Class imbalance with no negative class

**Symptom**: Fine-tuned model predicts the positive class everywhere; precision tanks. Operationally, the model is unusable because every alert is a false positive.

**Root cause**: A classifier with only positive classes can't learn what the *absence* of the signal looks like. Without examples of "definitely not a sinkhole," the model has no reason to predict "not a sinkhole" for anything.

**Fix v0**: `scripts/audit.py` fails the audit when no class named `other` / `background` / `negative` / `stable` / `none` / `no_event` is present. The user must add negative-class samples — typically by:

- **Geographic sampling**: random points outside the labeled positive class polygons, in the same broad region.
- **Value binning**: for regression-as-classification, the lowest equal-freq bin can serve as the negative class.

**Fix v0.1 (planned)**: `scripts/write_config.py --negative-class auto` will pick the strategy based on label type and emit the negatives directly.

**Reference case**: Karst — sinkhole / surface_depression / cave_entrance / surface_mine all needed a `stable` negative class to make the classifier useful in production.
