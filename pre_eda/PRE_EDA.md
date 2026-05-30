# Pre-EDA — `data_imdc_2026`

Auto-generated overview of IMDC 2026 datasets. Generated at **2026-05-30 00:16 UTC**.

Plots were built with `uv run python scripts/generate_pre_eda.py`.

## Overview

| File | Rows | Period | Municipalities | Size (MB) |
| --- | ---: | --- | ---: | ---: |
| `dengue.csv.gz` | 4,706,650 | 2010-01-03 → 2026-03-08 | 5,570 | 39.54 |
| `chikungunya.csv.gz` | 3,548,090 | 2013-12-29 → 2026-03-08 | 5,570 | 28.79 |
| `climate.csv` | 7,621,223 | 1999-12-26 → 2026-03-15 | 5,567 | 1675.65 |
| `forecasting_climate.csv` | 6,516,900 | 2010-01-01 → 2026-03-01 | 5,570 | 415.55 |
| `datasus_population_2001_2025.csv` | 139,251 | 2001 → 2025 | 5,571 | 2.6 |
| `environ_vars.csv` | 5,570 | — | 5,570 | 0.15 |
| `ocean_climate_oscillations.csv` | 1,721 | 1993-01-04 → 2026-03-10 | — | 0.07 |
| `map_regional_health.csv` | 5,570 | — | 5,570 | 0.5 |
| `shape_muni.gpkg` | 5,570 | — | 5,570 | 24.09 |
| `shape_regional_health.gpkg` | 439 | — | — | 29.14 |
| `shape_macroregional_health.gpkg` | 118 | — | — | 44.02 |

## Datasets

### `dengue.csv.gz`

Weekly probable dengue cases by municipality (SINAN / Infodengue).

- **Columns:** `geocode`, `date`, `casos`, `epiweek`, `uf`, `macroregional_geocode`, `regional_geocode`, `uf_code`, … (+10 more)
- **Notes:** Includes train/target flags for four validation seasons and target_city marker.

![dengue.csv.gz](plots/dengue.png)

### `chikungunya.csv.gz`

Weekly probable chikungunya cases by municipality.

- **Columns:** `geocode`, `date`, `casos`, `epiweek`, `uf`, `macroregional_geocode`, `regional_geocode`, `uf_code`, … (+10 more)
- **Notes:** Same schema as dengue; period starts in 2014.

![chikungunya.csv.gz](plots/chikungunya.png)

### `climate.csv`

ERA5 climate reanalysis aggregated to epidemiological weeks per municipality.

- **Columns:** `date`, `epiweek`, `geocode`, `temp_min`, `temp_med`, `temp_max`, `precip_min`, `precip_med`, … (+9 more)
- **Notes:** Temperature, precipitation, humidity, pressure, thermal range, rainy days.

![climate.csv](plots/climate.png)

### `forecasting_climate.csv`

Copernicus seasonal climate forecasts up to six months ahead.

- **Columns:** `geocode`, `reference_month`, `forecast_months_ahead`, `temp_med`, `umid_med`, `precip_tot`
- **Notes:** One row per municipality, reference month, and forecast horizon.

![forecasting_climate.csv](plots/forecasting_climate.png)

### `datasus_population_2001_2025.csv`

Municipality population estimates from DATASUS (2001–2025).

- **Columns:** `geocode`, `year`, `population`
- **Notes:** Long format: geocode × year.

![datasus_population_2001_2025.csv](plots/datasus_population_2001_2025.png)

### `environ_vars.csv`

Static environmental descriptors per municipality (Köppen climate, biome).

- **Columns:** `geocode`, `uf_code`, `koppen`, `biome`
- **Notes:** One row per municipality; no time dimension.

![environ_vars.csv](plots/environ_vars.png)

### `ocean_climate_oscillations.csv`

Weekly ocean-atmosphere oscillation indices (ENSO, IOD, PDO).

- **Columns:** `date`, `enso`, `iod`, `pdo`
- **Notes:** National/global indices; join to case data by week.

![ocean_climate_oscillations.csv](plots/ocean_climate_oscillations.png)

### `map_regional_health.csv`

Lookup table linking municipalities to health regions and macroregions.

- **Columns:** `macroregion_code`, `macroregion_name`, `uf_code`, `uf`, `uf_name`, `macroregional_geocode`, `macroregional_name`, `regional_geocode`, … (+3 more)
- **Notes:** Use with shapefiles for spatial aggregation.

![map_regional_health.csv](plots/map_regional_health.png)

### `shape_muni.gpkg`

GeoPackage with municipality polygons and names.

- **Columns:** `geocode`, `geocode_name`, `uf`, `uf_code`, `geometry`
- **Notes:** Geometry layer for choropleth maps.

![shape_muni.gpkg](plots/shape_muni.png)

### `shape_regional_health.gpkg`

GeoPackage with regional health division polygons.

- **Columns:** `regional_geocode`, `regional_name`, `uf_code`, `geometry`
- **Notes:** Matches regional_geocode in case tables.

![shape_regional_health.gpkg](plots/shape_regional_health.png)

### `shape_macroregional_health.gpkg`

GeoPackage with macroregional health division polygons.

- **Columns:** `macroregional_geocode`, `macroregional_name`, `uf_code`, `geometry`
- **Notes:** Matches macroregional_geocode in case tables.

![shape_macroregional_health.gpkg](plots/shape_macroregional_health.png)

## Compressed copies

These `.csv.gz` files are compressed versions of datasets plotted above (no separate plot):

- `climate.csv.gz`
- `datasus_population_2001_2025.csv.gz`
- `environ_vars.csv.gz`
- `forecasting_climate.csv.gz`
- `ocean_climate_oscillations.csv.gz`

## Next steps

- Join case data with `climate.csv` on `(geocode, epiweek)` or `(geocode, date)`.
- Aggregate to UF or health region using `map_regional_health.csv` and shapefiles.
- Use `forecasting_climate.csv` for horizon-aware climate covariates.
- See [official data documentation](https://sprint.mosqlimate.org/data/) for column definitions.
