# OlmoEarth-Skills

Claude Code / agent skills for OlmoEarth workflows — bundled instructions and small utility scripts that help an LLM agent prepare data, build datasets, and ship OlmoEarth projects without re-learning the same pitfalls every time.

## Skills

| Skill | What it does |
|-------|--------------|
| [`olmoearth-data-prep`](skills/olmoearth-data-prep/) | Convert raw geospatial labels into OlmoEarth-ready datasets — schema validation, real watershed AOI fetchers, rslearn config emission, and a 7-criteria audit. Pre-empts the 8 known prep pitfalls (wrong field names, bbox-vs-watershed, Studio MIME issues, quantile imbalance, random splits, 10K timeouts, missing negative class). |

More skills will land here as workflows stabilize.

## Install (Claude Code)

Symlink the skill folder into your Claude Code skills directory.

User-global (available across projects):

```bash
ln -s "$(pwd)/skills/olmoearth-data-prep" ~/.claude/skills/olmoearth-data-prep
```

Project-local:

```bash
ln -s "$(pwd)/skills/olmoearth-data-prep" /path/to/project/.claude/skills/olmoearth-data-prep
```

Windows (PowerShell, run as administrator):

```powershell
New-Item -ItemType SymbolicLink `
    -Path "$env:USERPROFILE\.claude\skills\olmoearth-data-prep" `
    -Target "$(Get-Location)\skills\olmoearth-data-prep"
```

The skill auto-loads when its description matches the user's request — no manual `/invoke` needed.

## Standalone use (no agent)

The bundled scripts are plain Python with no skill-specific imports — runnable directly:

```bash
# 7-criteria audit on a labels GeoJSON
python skills/olmoearth-data-prep/scripts/audit.py path/to/labels.geojson

# fetch a real watershed polygon
python skills/olmoearth-data-prep/scripts/fetch_aoi.py --nldi-comid 12345 --out basin.geojson

# emit rslearn config + Studio import + (optional) Lightning fine-tune YAML
python skills/olmoearth-data-prep/scripts/write_config.py labels.geojson out/ --finetune --num-classes 9
```

Only standard library is required for the basic flow. `shapely` (optional) lets the audit check polygon validity.

## Repo layout

```
OlmoEarth-Skills/
├── skills/
│   └── olmoearth-data-prep/
│       ├── SKILL.md            main entry — workflow + the 7 criteria + the 8 pitfalls
│       ├── references/         loaded by agent on demand
│       │   ├── schema.md
│       │   ├── rslearn_config.md
│       │   └── pitfalls.md
│       └── scripts/            standalone Python helpers
│           ├── audit.py
│           ├── fetch_aoi.py
│           └── write_config.py
├── LICENSE
└── README.md
```

## License

MIT. See [LICENSE](LICENSE).
