# Changi Rainfall Visualisation Redesign

This project supports a university Information Visualisation poster redesign of a rainfall chart using daily Changi station observations from 1980 to May 2026.

## Suggested Structure

- `data/MSS_Master_Daily_S24_1980_2026.csv`: raw source CSV used by the project.
- `data/processed/`: cleaned daily rainfall CSV exported by the project QMD.
- `figures/`: poster-ready chart exports.
- `report/team_coral_final_submission.qmd`: main project QMD containing the import, cleaning, transformation, checks, and visualisation pipeline.
- `report/team_coral_final_submission.html`: self-contained rendered project report.

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
quarto render report/team_coral_final_submission.qmd
```

The QMD reads the original CSV directly and writes the cleaned daily data to `data/processed/changi_daily_cleaned.csv`.

## Accuracy Notes

This workflow treats the data as Changi station-level observations. It marks 2026 as incomplete because observations end on 2026-05-31. It also records the station label change from `Changi` to `Changi Met. Station` so the poster can document that metadata change transparently.
