# Forest Fire Susceptibility — Bagmati Province, Nepal

Machine-learning pipeline for forest fire susceptibility mapping in **Bagmati Province, Nepal** (2015–2026). The project downloads raw geospatial data from public APIs, engineers features on a ~100 m canonical grid, and prepares datasets for model training.

## Requirements

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) (package manager)

## Quick Start

```bash
# Clone the repo and install dependencies
uv sync

# Create a .env file with your API keys (see Environment Variables below)

# Download all raw datasets
uv run python scripts/download_data.py

# Extract GEE-based datasets (NDVI, burned area, etc.)
uv run python scripts/gee_extractor.py --authenticate
uv run python scripts/gee_extractor.py --module all
```

> **Note:** `data/raw/` and `data/processed/` are gitignored. Run the download scripts locally to populate them, or place your own copies of the datasets under `data/raw/`.

## Environment Variables

Create a `.env` file at the project root. `scripts/config.py` loads these automatically.

| Variable | Required | Purpose |
|---|---|---|
| `FIRMS_MAP_KEY` | For FIRMS API downloads | NASA FIRMS Archive API key — [free registration](https://firms.modaps.eosdis.nasa.gov/usfs/api/) |
| `GEE_PROJECT_ID` | For GEE extraction | Google Cloud / Earth Engine project ID |
| `NASA_EARTHDATA_USERNAME` | Optional | NASA Earthdata login (some raster sources) |
| `NASA_EARTHDATA_PASSWORD` | Optional | NASA Earthdata password |

Example `.env`:

```env
FIRMS_MAP_KEY=your_firms_key
GEE_PROJECT_ID=your-gcp-project-id
NASA_EARTHDATA_USERNAME=your_username
NASA_EARTHDATA_PASSWORD=your_password
```

## Data Download

### Public API sources

```bash
# All modules
uv run python scripts/download_data.py

# Single module
uv run python scripts/download_data.py --module <name>
```

| Module | Source | Output |
|---|---|---|
| `firms` | NASA FIRMS Archive API | Fire hotspot CSVs |
| `weather` | Open-Meteo | Historical daily weather |
| `dem` | CGIAR SRTM | 30 m elevation GeoTIFF |
| `lulc` | ESA WorldCover | 10 m land cover GeoTIFF |
| `osm` | Overpass API | Roads, settlements, water (JSON) |
| `worldpop` | WorldPop | 100 m population GeoTIFF |
| `hansen` | Hansen Global Forest Change | Tree cover & loss year tiles |
| `gadm` | GADM | Nepal admin boundaries (GeoJSON) |

### Google Earth Engine sources

```bash
# One-time OAuth (opens browser)
uv run python scripts/gee_extractor.py --authenticate

uv run python scripts/gee_extractor.py --module ndvi
uv run python scripts/gee_extractor.py --module firms
uv run python scripts/gee_extractor.py --module all
```

GEE extracts NDVI composites (MODIS MOD13Q1), FIRMS hotspots, and MODIS MCD64A1 burned-area rasters into `data/raw/ndvi/` and `data/raw/burned_area/`.

## Pipeline Overview

```
Public APIs / GEE
       │
       ▼
scripts/download_data.py  ──►  data/raw/
scripts/gee_extractor.py         ├── firms/        (fire hotspots)
                                 ├── dem/          (SRTM 30 m)
                                 ├── lulc/         (ESA WorldCover 10 m)
                                 ├── osm/          (roads, settlements, water)
                                 ├── worldpop/     (population density)
                                 ├── hansen/       (forest change)
                                 ├── gadm/         (admin boundaries)
                                 ├── ndvi/         (MODIS 250 m via GEE)
                                 ├── climate/      (gridded climate via GEE)
                                 ├── sentinel2/    (Sentinel-2 via GEE)
                                 └── burned_area/  (MCD64A1 monthly GeoTIFFs)
       │
       ▼
Feature engineering  ──►  data/processed/
(notebooks/feature.ipynb)    ├── forest_fire_dataset_100m.parquet
                             ├── forest_fire_training_100m.parquet
                             ├── ndvi_sequences_100m.npz
                             └── feature_stack_100m/   (43 feature GeoTIFFs)
       │
       ▼
Model training / evaluation
```

## Project Structure

```
sem_project/
├── scripts/
│   ├── config.py           # Paths, constants, API keys, canonical grid
│   ├── download_data.py    # Raw data downloader (public APIs)
│   └── gee_extractor.py    # Google Earth Engine extractor
├── notebooks/
│   └── feature.ipynb       # Feature engineering notebook
├── data/
│   ├── raw/                # Downloaded source data (gitignored)
│   └── processed/          # Engineered features & datasets (gitignored)
├── outputs/                # Model outputs and figures
├── logs/                   # Pipeline logs (gitignored)
├── pyproject.toml
└── .env                    # Credentials (gitignored)
```

## Key Design Notes

- **Study area:** Bagmati Province bounding box (27.0°–28.4° N, 84.0°–86.6° E), reconstructed from 13 GADM district polygons (`NAME_3` field).
- **Fire season:** February–May (pre-monsoon).
- **Canonical grid:** 3 arc-seconds (~92 m, tagged as 100 m) — approximately 2,880 × 1,680 cells. All rasters are reprojected to this grid before stacking.
- **Flammable land cover:** ESA WorldCover classes 10 (tree), 20 (shrub), 30 (grass) — defined in `scripts/config.py` as `FLAMMABLE_CLASSES`.
- **Configuration:** All scripts import paths and constants from `scripts/config.py`; do not duplicate values elsewhere.

## Dependencies

Managed via `uv` / `pyproject.toml`. Core libraries include `geopandas`, `rasterio`, `pandas`, `scikit-learn`, `numpy`, and `requests`.
