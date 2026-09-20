# Africa Hexagonal Grid Workflow

Builds a standardized 250 m hexagonal grid over African cities and countries,
and attaches per-hexagon urban morphology statistics from public raster/
vector data sources: building height and volume, land cover composition,
population, and a derived land cover diversity index.

The primary output is a per-hexagon attribute table (Shapefile + CSV +
GeoJSON) plus a dissolved grid boundary, the artifact used by downstream
analysis modules.

## Contents

```
africa_hex_grid_workflow/
    config.py                  # data source definitions, land cover class tables, constants
    hexgrid.py                 # hexagon geometry
    aoi.py                     # area-of-interest construction
    raster_extract.py          # zonal statistics
    diversity.py                # land cover diversity index
    pipeline.py                  # orchestrator + CLI entry point
    clip_country_rasters.py     # preprocessing: clip raw global rasters to per-country extent
data/
    example/                    # a small synthetic example output, for schema reference only
requirements.txt / environment.yml
pyproject.toml
```

## Installation

```bash
conda env create -f environment.yml
conda activate africa_hex_grid_workflow
# or: pip install -r requirements.txt
pip install -e .   # optional, for the `africa-hex-grid` console entry point
```

## Data sources

No raw or intermediate data is bundled in this repository. You must obtain
your own copies of the following datasets and boundary files.

| # | Dataset | Used for | Distributor |
|---|---------|----------|--------------|
| 1 | WSF3D — World Settlement Footprint 3D, v2 | Building height and volume | German Aerospace Center (DLR) |
| 2 | Copernicus Global Land Cover 100 m, v3 | Land cover composition | Copernicus Land Monitoring Service / VITO |
| 3 | WorldPop population, 100 m, 2022 | Population per hexagon | WorldPop (University of Southampton) |
| 4 | AFRICAPOLIS 2020 | City/agglomeration boundaries | AFRICAPOLIS (OECD / e-Geopolis) |
| 5 | Natural Earth (or equivalent) | Country boundaries | Natural Earth |

See the manuscript's Methods and Supplementary Information for exact source
versions, file naming, and boundary-file schema requirements.

### Preprocessing

Per-city processing reads from a cache of rasters pre-clipped to each
country's extent, rather than masking the full continental raster on every
run:

```bash
python -m africa_hex_grid_workflow.clip_country_rasters \
    --country-boundaries /path/to/country_boundaries.gpkg \
    --raw-data-dir /path/to/raw_rasters_root \
    --output-dir /path/to/00_country_TIF
```

## Usage

```bash
python -m africa_hex_grid_workflow.pipeline \
    --aoi-mode city --aoi-name "Kumasi" \
    --data-dir /path/to/output/data \
    --country-tif-dir /path/to/00_country_TIF \
    --country-boundaries /path/to/country_boundaries.gpkg \
    --city-boundaries /path/to/agglomerations.gpkg \
    --country-iso3 GHA
```

or for a whole country:

```bash
python -m africa_hex_grid_workflow.pipeline \
    --aoi-mode country --aoi-name "Ghana" \
    --data-dir /path/to/output/data \
    --country-tif-dir /path/to/00_country_TIF \
    --country-boundaries /path/to/country_boundaries.gpkg \
    --city-boundaries /path/to/agglomerations.gpkg
```

Outputs land under `{data-dir}/03_country_grids/{Country}/{City}/`:

- `{name}_grid_250m_hex_enriched.{shp,csv,geojson}` — the per-hexagon grid.
- `{name}_grid_boundary.geojson` — the dissolved outline of all hexagons.

### Reproducing an example city

`data/example/` contains a small synthetic (fabricated, clearly labeled)
grid illustrating the output schema — see `data/example/README.md`.

## Output schema

| Column(s) | Meaning | Source |
|---|---|---|
| `GridID` | Unique cell identifier | generated |
| `City` / `Country` | AOI name | generated |
| `Area_m2`, `Center_X`, `Center_Y` | Hexagon area and centroid | generated |
| `H_max`, `H_mean`, `H_std`, `H_sum` | Building height statistics, meters | WSF3D |
| `V_max`, `V_mean`, `V_std`, `V_sum` | Building volume statistics, m³ | WSF3D |
| `Pop2022` | Population count within the hexagon | WorldPop 2022 |
| `LC_*` | Percentage of hexagon area per Copernicus land cover class | Copernicus Global Land Cover |
| `LC_Shannon` | Land cover diversity index | derived |
| `LCF_*` | Mean cover-fraction per cover type | Copernicus Global Land Cover |

## Design notes

Each city is gridded in its own locally-accurate UTM zone rather than one
shared continental projection, to avoid distortion at continental scale;
grids from independently-processed cities still tile against a shared
reference origin. Hexagons are kept by centroid containment within the area
of interest, so every emitted hexagon is complete and undistorted. City
name lookups are disambiguated by ISO3 country code to handle duplicate
place names. See the manuscript's Methods and Supplementary Information for
these and other construction and validation details.

## Citation

If you use this workflow, please also cite the underlying data sources:

- German Aerospace Center (DLR), World Settlement Footprint 3D (WSF3D).
- Copernicus Land Monitoring Service, Global Land Cover 100 m v3 (Buchhorn
  et al.).
- WorldPop (www.worldpop.org — School of Geography and Environmental
  Science, University of Southampton).
- AFRICAPOLIS (OECD / e-Geopolis), 2020 city/agglomeration boundaries.

## License

No license file was found in the source project this workflow was derived
from. Until a license is chosen, all rights are reserved by default; add a
`LICENSE` file before distributing this code publicly.
