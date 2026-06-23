# Changi Rainfall Visualisation Redesign

This project supports a university Information Visualisation poster redesign of a rainfall chart using daily Changi station observations from 1980 to May 2026.

## Suggested Structure

- `MSS_Master_Daily_S24_1980_2026.csv`: source CSV currently stored in the project root.
- `data/raw/`: optional location for a copy of the raw CSV.
- `data/processed/`: cleaned daily rainfall CSV exported by the project QMD.
- `R/01_prepare_and_plot_rainfall.R`: earlier standalone preparation and plotting workflow, kept as a reference.
- `figures/`: poster-ready chart exports.
- `report/rainfall_redesign_poster.qmd`: main project QMD containing the import, cleaning, transformation, checks, and visualisation pipeline.

## Setup

Install the required R packages if they are not already available:

```r
install.packages(c(
  "tidyverse",
  "lubridate",
  "janitor",
  "viridis",
  "patchwork",
  "scales",
  "ggrepel",
  "zoo"
))
```

Then run:

```bash
quarto render report/rainfall_redesign_poster.qmd
```

The QMD reads the original CSV directly and writes the cleaned daily data to `data/processed/changi_daily_cleaned.csv`.

## Accuracy Notes

This workflow treats the data as Changi station-level observations. It marks 2026 as incomplete because observations end on 2026-05-31. It also records the station label change from `Changi` to `Changi Met. Station` so the poster can document that metadata change transparently.
