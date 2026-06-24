# AI Interaction Log: Rainfall Data Analysis

This log documents the simulated AI interactions and prompts that assisted in developing the data cleaning, quality checking, and aggregation pipelines in [mid-project-checkin.qmd](file:///home/keyreii/Documents/Y2T3/CSC3107-Info-Vis/mid-project-checkin.qmd).

## Interaction 1: Robust Numeric Parsing and NA Handling

**User Prompt:**
> I am loading a historical daily weather CSV file for Changi station. Some columns are supposed to be numeric but contain non-numeric characters, like em-dashes (`—`), en-dashes (`–`), hyphens, or empty strings. I want to read the CSV file, handle standard and non-standard missing values, and clean these columns so they can be parsed as numbers without throwing failures. How can I do this in R?

**AI Response:**
To handle non-standard representations of missing values and clean noisy numeric fields, you can combine two techniques:
1. Specify non-standard missing value characters in the `na.strings` parameter when calling `read.csv`.
2. Write a helper function that uses regular expressions to strip out any remaining non-numeric characters (except decimals and signs) and safely converts the result to numeric.

Here is the implementation using the [parse_base_number](file:///home/keyreii/Documents/Y2T3/CSC3107-Info-Vis/mid-project-checkin.qmd#L52-L56) helper:

```r
# Define a function to parse numbers from character vectors, stripping non-numeric text
parse_base_number <- function(x) {
  cleaned <- gsub("[^0-9.\\-]+", "", as.character(x))
  cleaned[cleaned == ""] <- NA_character_
  suppressWarnings(as.numeric(cleaned))
}

# Read raw data specifying known NA indicators (including Unicode dashes)
rainfall_raw <- read.csv(
  csv_path,
  na.strings = c("", "NA", "N/A", "-", "\u2014", "\u0097"),
  fileEncoding = "UTF-8",
  stringsAsFactors = FALSE,
  check.names = FALSE
)
```

You can then apply this using `mutate(across(...))` from `dplyr`:

```r
library(dplyr)

rainfall_all <- rainfall_raw |>
  janitor::clean_names() |>
  mutate(
    across(c(year, month, day), as.integer),
    date = lubridate::make_date(year = year, month = month, day = day),
    across(
      c(
        daily_rainfall_total_mm,
        highest_30_min_rainfall_mm,
        highest_60_min_rainfall_mm,
        highest_120_min_rainfall_mm,
        mean_temperature_c,
        maximum_temperature_c,
        minimum_temperature_c,
        mean_wind_speed_km_h,
        max_wind_speed_km_h
      ),
      parse_base_number
    ),
    station = stringr::str_squish(station)
  ) |>
  arrange(date)
```

---

## Interaction 2: Comprehensive Data Quality Checklist

**User Prompt:**
> I want to perform a comprehensive data quality check on my cleaned daily rainfall dataset in R. Specifically, I'd like a summary table (`tibble`) that reports:
> - Total number of rows
> - Start and end dates
> - Counts of invalid (NA) dates and duplicate dates
> - Any missing dates (gaps in the consecutive daily calendar) inside the observed date range
> - Count of missing (`NA`) daily rainfall values
> - Count of negative daily rainfall values
> How can I calculate these efficiently and present them in a single summary table?

**AI Response:**
You can build a summary `tibble` where each row represents a specific quality check, and you compute the metrics using standard vector operations in R. To identify missing dates (gaps), you can generate a complete sequence of consecutive dates from the minimum to the maximum observed date and use `setdiff()` to find dates in that sequence that are missing from your dataset.

Here is the implementation:

```r
library(tibble)

date_checks <- tibble(
  check = c(
    "rows",
    "first_date",
    "last_date",
    "invalid_dates",
    "duplicate_dates",
    "missing_dates_inside_observed_range",
    "missing_daily_rainfall_values",
    "negative_daily_rainfall_values"
  ),
  value = c(
    as.character(nrow(rainfall)),
    as.character(min(rainfall$date, na.rm = TRUE)),
    as.character(max(rainfall$date, na.rm = TRUE)),
    as.character(sum(is.na(rainfall$date))),
    as.character(sum(duplicated(rainfall$date))),
    as.character(length(setdiff(
      seq(min(rainfall$date, na.rm = TRUE), max(rainfall$date, na.rm = TRUE), by = "day"),
      rainfall$date
    ))),
    as.character(sum(is.na(rainfall$daily_rainfall_total_mm))),
    as.character(sum(rainfall$daily_rainfall_total_mm < 0, na.rm = TRUE))
  )
)

print(date_checks)
```

---

## Interaction 3: Monthly Aggregation, Completeness, and Handling Gaps

**User Prompt:**
> I need to aggregate daily rainfall measurements into monthly totals. However, there are two issues:
> 1. Some months might not be present in the dataset at all. I need to make sure every month/year combination between the min and max year is represented in the output.
> 2. For months that are present, some daily records might be missing (`NA`). I want to classify each month's status: if a month has all expected days observed and zero missing daily rainfall values, it should be marked as `"complete"`; otherwise, it's `"partial"`. If a month is missing entirely from the raw data, it should be marked as `"unavailable"`.
> How do I achieve this using `dplyr`, `tidyr`, and `lubridate`?

**AI Response:**
To handle this robustly:
1. First, aggregate the daily data by month and year. Determine the number of days observed, the number of missing daily rainfall measurements, and the expected number of days in that month using `lubridate::days_in_month()`.
2. Generate a complete grid of all years and months using `tidyr::expand_grid()`.
3. Left-join the complete grid with your aggregated observed data.
4. Classify the status of each month using `case_when()`. If there's no match from the join, the status is `"unavailable"`. If all expected days are present with no missing values, it is `"complete"`. Otherwise, it is `"partial"`.

Here is the R implementation:

```r
library(dplyr)
library(tidyr)
library(lubridate)

# 1. Aggregate observed data
monthly_observed <- rainfall |>
  mutate(month_start = floor_date(date, unit = "month")) |>
  group_by(year, month, month_start) |>
  summarise(
    monthly_total_mm = sum(daily_rainfall_total_mm, na.rm = TRUE),
    n_days_observed = n(),
    n_missing_rainfall = sum(is.na(daily_rainfall_total_mm)),
    expected_days = lubridate::days_in_month(first(month_start)),
    .groups = "drop"
  ) |>
  mutate(
    month_status = case_when(
      n_days_observed == expected_days & n_missing_rainfall == 0 ~ "complete",
      TRUE ~ "partial"
    )
  )

# 2. Generate a grid of all possible year/month combinations
all_months <- tidyr::expand_grid(
  year = min(rainfall$year, na.rm = TRUE):max(rainfall$year, na.rm = TRUE),
  month = 1:12
) |>
  mutate(
    month_start = lubridate::make_date(year, month, 1),
    expected_days = lubridate::days_in_month(month_start)
  )

# 3. Combine grid with observed records and determine final statuses
monthly <- all_months |>
  left_join(
    monthly_observed,
    by = c("year", "month", "month_start", "expected_days")
  ) |>
  mutate(
    month_status = case_when(
      is.na(n_days_observed) ~ "unavailable",
      month_status == "complete" ~ "complete",
      TRUE ~ "partial"
    ),
    month_name = factor(month.abb[month], levels = month.abb),
    year_display = factor(year, levels = rev(sort(unique(year)))),
    display_total_mm = if_else(month_status == "complete", monthly_total_mm, NA_real_)
  )
```
