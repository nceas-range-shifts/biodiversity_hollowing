## Project

`chi_shifts` examines how cumulative human impacts change as stressors and species ranges shift over time. The current analysis uses AquaX species distribution models (SDMs) and species/functional-entity (FE) traits to map richness and turnover under climate scenarios, compares them with ISIMIP biomass change, and contrasts coastal vs. oceanic regions. All rasters share a common global grid. See `README.md` for the pipeline.

## Directory structure

- `data_mgmt/` — Quarto scripts named `data_access_*.qmd` that download/build raw inputs into `_data/`: `vuln_traits`, `funct_entities`, `isimip`, `bathymetry`, `eez_meow`. Run these first if a needed dataset is missing.
- `scripts/` — numbered analysis scripts run in order (`1_map_community_change`, `2_map_functional_change`, `3_map_biomass_change`, `4_test_turnover_coastal_oceanic`), then manuscript figure scripts `ms_fig1_...` to `ms_fig4_...`. `ms_fig*` write to `ms_figs/`.
- `scripts/common_fxns.R` — shared helper functions (see below); `source(here('scripts/common_fxns.R'))` at the top of scripts.
- `_data/` — single consolidated data directory, one subfolder per source (each has a README):
  - `gebco_2026/` — GEBCO bathymetry aggregated to 0.05°
  - `isimip_2026/` — ISIMIP3b marine fishery/biomass NetCDFs
  - `mregions_2026/` — MEOW ecoregion raster and lookups, world continents gpkg
  - `ohara_2024/` — species vulnerability framework traits/scores, FE traits
- `_output/` — generated outputs, with a README per subfolder naming the producing script: `spp_turnover_maps/` (script 1), `fe_turnover_maps/` (script 2), `biomass_change_maps/` (script 3), `figure_data/` (ms_fig4).
- `ms_figs/` — manuscript and supplemental figure PNGs.
- AquaX SDM source data lives outside the repo on the shared server at `/home/shares/data-aquax/` (accessed via `here_aquax()`), not under `_data/`.

## Grid conventions

Rasters/tables use a global lon-lat grid, default resolution `res = 0.05`, `ncols = 360/res`, with a 1-indexed row-major `cell_id` (row 1 = cells 1..ncols, scanning left to right). Outputs for mapping and analysis are projected to a Behrmann equal-area grid (`+proj=cea +lat_ts=30 ...`) at 5 km. Convert between cell_id and coordinates with the helpers in `scripts/common_fxns.R`:

- `cell_id_to_xy(df, cell_id_col = 'cell_id', res = 0.05, drop = TRUE)` — cell_id → x/y cell-center coordinates.
- `xy_to_cell_id(df, x_col = 'x', y_col = 'y', res = 0.05, drop = TRUE)` — inverse of the above.
- `calc_cellrange(i, yslice_w, res = 0.05)` — computes a `cell_id` range for a latitude slice, for chunked/batched processing.

## Key helper functions (`scripts/common_fxns.R`)

- `here_aquax(f, ...)` — builds a path under the shared AquaX data directory.
- `get_aquax_meta(meta, sdm = TRUE)` — loads AquaX species metadata.
- `get_spp_vuln()` — loads the species vulnerability framework tables from `_data/ohara_2024/...`.
- `get_sdm(aphia_id, scenario, cellrange, apply_thresh, batch_size, threads, memory_limit)` — reads per-species SDM parquet files via DuckDB, with thresholding and batching for scale.
- `sample_decomp(df)` — pools mean/sd/n across grouped samples (e.g. across species within a cell) using `collapse`.
- `plot_map()`, `map_theme()`, `plot_marginal()` — plotting helpers for the manuscript figures (Behrmann projection maps and marginal plots).

## Dependencies to know about

- Script 3 uses `_output/spp_turnover_maps/all_spp_richness.tif` (from script 1) as its ocean mask and template grid, so run script 1 first.
- `ms_fig*` and script 4 read outputs from scripts 1–3 and `_data/`.
- Several `ms_fig*` chunks skip regeneration when the output exists; delete the file to force an update.

## Coding conventions

- Use the base R pipe `|>` (not magrittr `%>%`), except where existing code already uses `%>%` for consistency within a file.
- Prefer `collapse` (`fmutate`, `fsubset`, `fsum`, `GRP`, ...) over base/dplyr equivalents in performance-sensitive code operating on large per-cell/per-species tables.
- Large per-file datasets (e.g. SDM parquet files) are queried via DuckDB (`duckdb`/`DBI`) rather than loaded fully into R.
- Use `here::here()` / `here_aquax()` for paths rather than hardcoded absolute paths (except for the shared AquaX root, which is inherently absolute).
- In `.qmd` files, pass figure paths to `knitr::include_graphics()` as relative paths (`fs::path_rel(fig_file)`) and pass `plot =` explicitly to `ggsave()`.
- Use `janitor::clean_names()` when reading in external/raw tabular data.
- When commenting for explanation, use triple hash `###` to start comment line

## Behavioral rules for Posit Assistant

- This runs on a **shared server** — when writing or modifying DuckDB queries or other parallelized code, always cap `threads` and `memory_limit` (see `get_sdm()` for existing defaults) rather than letting them run unbounded.
- Treat `/home/shares/data-aquax/` (via `here_aquax()`) as **read-only**; never write, delete, or modify files there.
- Before overwriting or deleting existing files in `_data/`, `_output/`, or elsewhere, confirm with the user first — these are expensive to regenerate.
- Do not assume `_data_raw/` still exists; all raw and processed data live under a single `_data/` directory.
- When adding new grid-indexed data, follow the existing `cell_id`/`x`/`y` conventions above rather than introducing a new indexing scheme.
- Do not read in data files (.csv, .parquet) as input tokens unless explicitly asked to do so.
