# Africa Hexagonal Grid Workflow

Builds a standardized 250 m flat-top hexagonal grid over a city or country in
Sub-Saharan Africa, and attaches per-hexagon urban morphology statistics
extracted from four public raster/vector data sources: building height and
volume, land cover composition, population, and a derived land cover Shannon
diversity index.

The primary output — the artifact intended for downstream analysis modules —
is a per-hexagon attribute table (Shapefile + CSV + GeoJSON) plus a dissolved
**grid boundary envelope** (GeoJSON) giving the exact spatial extent covered
by the grid, suitable for clipping other datasets to the same area.

## Contents

```
africa_hex_grid_workflow/
    __init__.py
    config.py                # data source definitions, land cover class tables, constants
    hexgrid.py                # flat-top hexagon geometry (axial <-> cartesian, tessellation)
    aoi.py                     # build_aoi(): filter -> dissolve -> buffer -> simplify -> save
    raster_extract.py         # zonal statistics: land cover percentages, batched single-column extraction
    diversity.py               # Shannon land cover diversity index
    pipeline.py                 # GridPipeline orchestrator + CLI entry point
    clip_country_rasters.py    # preprocessing: clip raw global rasters to per-country extent
data/
    example/                   # a small synthetic (fabricated) example output, for schema reference only
requirements.txt / environment.yml
pyproject.toml
```

### Why a flat module layout instead of a larger package framework

This is a single, focused pipeline (one AOI in, one enriched grid out), not a
multi-tool platform. A handful of well-separated modules under one top-level
package is easier to read end-to-end than a plugin/processor framework with
base classes, config-object hierarchies, and a CLI dispatcher — so that is
what this repository uses. `pyproject.toml` makes it pip-installable for
convenience, but you can also just run the modules directly from a checkout.

## Installation

```bash
conda env create -f environment.yml
conda activate africa_hex_grid_workflow
# or: pip install -r requirements.txt
pip install -e .   # optional, for the `africa-hex-grid` console entry point
```

## Data sources

No raw or intermediate data is bundled in this repository (the source
rasters are well over 100 GB). You must obtain your own copies of the
following four datasets and one or two boundary files.

| # | Dataset | Used for | Distributor | How to obtain |
|---|---------|----------|--------------|----------------|
| 1 | **WSF3D — World Settlement Footprint 3D, v2** | Building height and volume (`H_*`, `V_*` columns) | German Aerospace Center (DLR) | Download portal: `https://download.geoservice.dlr.de/WSF3D/`. Cite DLR's WSF3D product when using this data. |
| 2 | **Copernicus Global Land Cover 100 m, v3 (2019 epoch)** — discrete classification map + 10 cover-fraction layers | Land cover class percentages (`LC_*`), cover-fraction layers (`LCF_*`) | Copernicus Land Monitoring Service / VITO | Download viewer: `https://lcviewer.vito.be/download`. This pipeline expects the `PROBAV_LC100_global_v3.0.1_2019-nrt_*_EPSG-4326.tif` filenames as distributed by VITO (discrete classification map, plus the Bare/BuiltUp/Crops/Grass/MossLichen/PermanentWater/SeasonalWater/Shrub/Snow/Tree cover-fraction layers). |
| 3 | **WorldPop population** — Sub-Saharan Africa, 100 m, constrained, 2022 | Population per hexagon (`Pop2022`) | WorldPop (University of Southampton) | `https://www.worldpop.org`. Use the constrained, 100 m resolution, Sub-Saharan Africa 2022 product; the pipeline expects a filename of the form `sub_saharan_africa_ppp_100m_2022.tif`. |
| 4 | **AFRICAPOLIS 2020** — city/agglomeration boundaries | City-mode AOI definition (`agglomerations` boundary file) | AFRICAPOLIS (OECD / e-Geopolis) | `https://africapolis.org`. Any exported polygon layer (GPKG/GeoJSON/Shapefile) with an agglomeration name column, an ISO3 country-code column, and an ID column works — see "Boundary file schema" below. |
| 5 | **Natural Earth / country administrative boundaries** | Country-mode AOI definition, and country lookup for city mode | Natural Earth (or an equivalent African country boundary set) | `https://www.naturalearthdata.com` (or your own africa_boundaries file). Needs a country-name column and, for city mode, an ISO alpha-3 column. |

### Boundary file schema expected by this code

- **Country boundaries** (`--country-boundaries`): polygon layer with a
  country-name column (configurable via `--country-name-column`, default
  `NAME_0`) and, for city mode's country lookup, an `ISO` column with ISO3
  codes.
- **City/agglomeration boundaries** (`--city-boundaries`): polygon layer with
  a city-name column (configurable via `--city-name-column`, default
  `Agglomeration_Name`), an `ISO3` column, and an ID column named either
  `Agglomeration_ID` or `agglosID` (used to disambiguate cities that share a
  name — see "Homonym handling" below).

Neither file is included in this repository; point `--country-boundaries`
and `--city-boundaries` at your own copies.

### Preprocessing: pre-clipped per-country rasters

Per-city processing reads from a cache of rasters already clipped to each
country's extent (`{country_tif_dir}/{Country_With_Underscores}/*.tif`),
rather than masking the full continental/global raster on every run. Build
this cache once with:

```bash
python -m africa_hex_grid_workflow.clip_country_rasters \
    --country-boundaries /path/to/country_boundaries.gpkg \
    --raw-data-dir /path/to/raw_rasters_root \
    --output-dir /path/to/00_country_TIF
```

`--raw-data-dir` should contain one subdirectory per source (matching the
`source_dir` values in `clip_country_rasters.SOURCES` / `config.DATA_SOURCES`,
e.g. `WSF3D/`, `WorldPop/`, and the Copernicus land cover directory) holding
the raw files as distributed by each source's portal. The output cache is
reprojected to a single fixed continental projection (Africa Albers Equal
Area Conic, ESRI:102022) — this is just a shared intermediate representation
for the cache; the actual analysis grid for each city is generated in its
own locally-accurate UTM zone (see "Coordinate reference systems" below).

## Usage

Once you have the boundary files and the country-TIF cache in place:

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

Outputs land under `{data-dir}/03_country_grids/{Country}/{City}/` (city
mode) or `{data-dir}/03_country_grids/{Country}/` (country mode):

- `{name}_grid_250m_hex_enriched.{shp,csv,geojson}` — the per-hexagon grid
  with every attached data column.
- `{name}_grid_boundary.geojson` — the dissolved outline of all hexagons;
  this is the artifact intended for handing off to other analysis modules.

Each step (AOI, grid, raster attachment) is skipped if its output already
exists, so the same command can be re-run to resume an interrupted job. Pass
`--force-regenerate-aoi` / `--force-regenerate-grid` to force those two
specific steps to rerun; the `GridPipeline` constructor's `force_regenerate`
dict additionally supports per-data-type overrides (`land_cover`,
`land_cover_fraction`, `population`, `attachment`, `shannon`) when calling
the Python API directly rather than the CLI.

### Reproducing an example city

`data/example/` contains a small **synthetic** (fabricated, clearly labeled)
grid purely to illustrate the output schema — see `data/example/README.md`.
To generate a real example, obtain the data sources above, run
`clip_country_rasters.py` for one country, and run the command above with a
small city name (e.g. Kumasi, Ghana is a good modest-sized test case).

## Output schema

| Column(s) | Meaning | Source |
|---|---|---|
| `GridID` | `{ISO3}_{city_index:04d}_{cell_number:06d}` (city mode) or an axial-coordinate ID (standalone use) | generated |
| `City` / `Country` | AOI name | generated |
| `Area_m2`, `Center_X`, `Center_Y` | Hexagon area and centroid, in the grid's CRS | generated |
| `H_max`, `H_mean`, `H_std`, `H_sum` | Building height statistics within the hexagon, meters | WSF3D |
| `V_max`, `V_mean`, `V_std`, `V_sum` | Building volume statistics within the hexagon, m³ | WSF3D |
| `Pop2022` | Summed population count within the hexagon | WorldPop 2022 |
| `LC_Shrub`, `LC_Herb`, `LC_Crop`, `LC_Built`, `LC_Bare`, `LC_Snow`, `LC_Water`, `LC_Wetlnd`, `LC_Moss`, `LC_FC*`, `LC_FO*`, `LC_Ocean` | Percentage of hexagon area in each of 22 Copernicus land cover classes | Copernicus Global Land Cover 100m v3 |
| `LC_Shannon` | Shannon diversity index of land cover composition within the hexagon | derived |
| `LCF_Bare`, `LCF_Built`, `LCF_Crop`, `LCF_Grass`, `LCF_Moss`, `LCF_Water`, `LCF_SeaWtr`, `LCF_Shrub`, `LCF_Snow`, `LCF_Tree` | Mean cover-fraction (0-100%) for each cover type | Copernicus Global Land Cover 100m v3 cover-fraction layers |

## Design notes

### Coordinate reference systems: per-city UTM, not one continental CRS

Every AOI is gridded in its own locally-accurate UTM zone, auto-detected
from the AOI centroid (`pipeline.detect_utm_crs`), rather than one shared
continental projection. A single continent-spanning equal-area projection
distorts hexagon shape and area non-uniformly across its extent — negligible
near the projection's center, but significant near its edges — which would
make hexagon-based density/morphology metrics inconsistent between cities.
Using each city's own UTM zone keeps that distortion negligible everywhere
it is actually used.

### Global grid origin: why independently-processed grids still align

All hexagon axial coordinates are computed relative to one shared origin
(`config.GLOBAL_GRID_ORIGIN_X/Y`, expressed in the Africa Albers Equal Area
Conic projection). Even though each city is ultimately projected into its
own UTM zone rather than this shared projection, anchoring the axial
coordinate system to one fixed reference point (translated into whichever
CRS is used for a given run) is what makes grids from independent,
separately-run processing jobs tile consistently if ever reprojected into a
common CRS later, rather than each city's hexagons falling on an arbitrary,
mutually-incompatible offset.

### Homonym-safe city lookup

Agglomeration name fields are not unique: some names repeat across
countries, and a few repeat within the same country (place names separated
by a few kilometers). Resolving a city by name alone can silently return the
wrong physical location. `pipeline.find_city_with_iso3` requires (or at
least strongly prefers) an ISO3 filter to disambiguate, and falls back to a
stripped-name match only when no exact match exists (some source records
carry incidental trailing whitespace in the name field).

### Centroid-containment filtering, not polygon intersection

A hexagon is kept if its **center point** falls inside the (optionally
buffered) AOI, rather than keeping any hexagon that merely overlaps the AOI
boundary. This avoids ever emitting a hexagon that is only sliver-clipped by
the AOI edge — every hexagon in the output grid is a complete, undistorted
hexagon — at the cost of the grid's outer edge tracking the AOI boundary
only approximately (within about one hexagon radius).

### WSF3D scale factor

Building height and volume rasters are distributed as scaled integers (an
implicit x10 encoding to preserve one decimal place of precision in an
integer format); `config.DataSource.scale_factor = 0.1` for these sources
undoes that encoding to recover height in meters and volume in cubic meters.
This is not a workaround — it reflects how DLR distributes the WSF3D
product.

### `all_touched` and NoData handling

Pixel inclusion (`all_touched`) adapts to the relationship between raster
resolution and hexagon size: if a raster's pixels are large relative to a
250 m hexagon, `all_touched=True` is forced regardless of a source's default
configuration, so a hexagon can never end up with zero sampled pixels simply
because no pixel center happened to fall inside it. Every extraction
function applies a layered NoData filter (NaN, inf, explicit nodata values,
optional negative-value filtering for height/volume, and a valid-range
check) before aggregating, since a hexagon can legitimately straddle a
mix of valid and invalid pixels.

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
