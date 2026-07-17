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

## Interaction 4: Critiquing the Original Rainfall Visualization 

**User Prompt:** 
> our team is redesigning a published rainfall visualization. we discussed for a bit and here are some points we came up with 
> 
> we think its strengths are: 
> it shows rainfall over a long period 
> it includes the rainfall values for all 12 months 
> it allows readers to see that rainfall changes between years 
> 
> we think its weaknesses are: 
> there are too many colours and visual elements 
> it is difficult to compare the same month across different years 
> the reader has to keep looking between the graph and the legend 
> the large amount of information makes the main pattern difficult to notice 
> 
> based on these points, what other strengths or weaknesses might we have missed? do not repeat our points unless you are adding a new explanation 

**AI Response:**
Here are some additional points you may have missed. 

Other strengths 

1. The long time span may help reveal seasonal patterns.Readers can check whether certain months are usually wetter or drier. 
2. All months are shown using the same dataset and structure. This gives readers a complete yearly view instead of focusing on only selected months. 
3. Unusual rainfall periods may stand out. Very wet or dry months could be noticed when they look different from the surrounding values. 

Other weaknesses 

1. The visual may not clearly separate exact values from general patterns. Readers might see that one area looks wetter, but still struggle to understand how much wetter it is.
2. Years with extreme rainfall may affect the scale. A few unusually high values could make normal differences between other months harder to see. 
3. The order of the years may be unclear or difficult to follow. If the layout does not have an obvious chronological direction, readers may struggle to identify changes over time. 
4. Missing or incomplete data may not be clearly explained. Readers could mistake missing values for zero rainfall or assume that all months have complete records. 
5. The visualization may not guide the reader toward the most important findings. Without a clear title, annotations, or highlighted patterns, readers have to work out the main message themselves. 
6. Some colours may be too similar or inaccessible. This could make the visualization difficult to read for people with colour-vision deficiencies or when viewed from a distance.

---

## Interaction 5: Refining the Visualization Critique 

**User Prompt:** 
> some of your points are useful, but somw are too general and could apply to almost any graph. focus only on problems that affect the reader’s ability to compare monthly rainfall across many years. i want it to be specific to what we're working on 
> 
> summarize the 3 most important strengths and 3 most important weaknesses. keep each point short and explain why it matters 

**AI Response:** 
3 most important strengths 

1. Covers all months across many years This allows readers to look for long-term seasonal rainfall patterns. 

2. Keeps the monthly breakdown Readers can see which months contributed to each year’s rainfall instead of only seeing yearly totals. 

3. Shows variation between years Unusually wet or dry years can be identified from the wider historical context. 

3 most important weaknesses 

1. The same month is difficult to compare across years Monthly values are placed in different positions, so readers cannot easily track January, February, or another month over time. 

2. Too many month colours make comparison harder Readers must remember which colour represents each month while comparing more than 40 years. 

3. The stacked structure hides differences between months Most sections do not share a common baseline, making their sizes difficult to compare accurately.

---
## Interaction 6: Refining the Target Audience and Task

**User Prompt:**
> okok now we're working on the design brief 
> 
> our possible target audience are students, urban planners, residents and policy makers in Singapore. we want the redesigned visualization to help them quickly identify: 
> wetter and drier months 
> unusually wet or dry years 
> repeating seasonal rainfall patterns 
> changes in rainfall patterns across many years 
> 
> Is this audience and task clear enough? suggest how we can make it more specific. do not write the final design brief yet 

**AI Response:** 
The task is mostly clear, but the audience is too broad. Students, residents, urban planners, and policymakers may use the visualization for different reasons. It would be better to choose one main audience and treat the others as secondary. 

For example: Primary audience: Singapore students and residents interested in understanding historical rainfall patterns. 

Main task: Compare monthly rainfall across many years to quickly identify seasonal patterns, unusually wet or dry periods, and possible changes over time. Urban planners and policymakers usually need more decision-focused information, such as flood risk, drainage planning, or infrastructure needs. Your rainfall data alone may not be enough to support those decisions directly. 

You should also clarify that the visualization shows historical rainfall at Changi station, rather than rainfall across all of Singapore. This prevents the audience from assuming the results represent every location in the country. 

---

## Interaction 7: Revising the Design Brief 

**User Prompt:** 
> we got feedback the design brief should be more succinct and should focus mainly on the audience and the problem we are trying to solve 
> 
> im going to do another draft and lets see if its ok 
> 
> dont make claims about climate change because our visualization alone cannot prove climate change. focus on comparing historical monthly rainfall patterns 
>
> This redesign is intended for students and residents in Singapore who want to understand historical rainfall patterns at Changi. The original visualization makes it difficult to compare the same month across many years because it contains too many colours and stacked elements. The redesigned visualization will make monthly rainfall easier to compare, helping readers identify wetter and drier months, seasonal patterns, and unusually wet or dry years.

**AI Response:** 
Design Brief 

It clearly focuses on the audience, the original problem, and the comparison task without making unsupported climate claims. 

---

## Interaction 8: Rewriting the Planned Design Direction 

**User Prompt:** 
> we are going to rewrite our planned design direction after receiving the abpve feedback
> 
> our original idea was a stacked bar chart with one bar for every year and 12 coloured sections representing the months. however, we realised that it is difficult to compare the same month across different years because the sections do not share a common starting point 
> 
> we are considering changing it to a heatmap where: 
> the x-axis shows the 12 months 
> the y-axis shows the years 
> colour represents the monthly rainfall total 
>
> Our original plan was to use a stacked bar chart, with one bar for each year and 12 coloured sections representing the months. However, this makes it difficult to compare the same month across different years because most sections do not share a common starting point. We are considering using a heatmap instead. The x-axis will show the 12 months, the y-axis will show the years, and colour will represent the monthly rainfall total. This design should make it easier to compare rainfall across months and years and quickly identify seasonal patterns and unusually wet or dry periods. This keeps the focus on the comparison problem and clearly explains why the heatmap may be more suitable. 

**AI Response:** 
This sounds better than your previous version. It clearly does not beat around the bush and communicates the point effectively. Good job

---

## Interaction 9: Comparing the Stacked Bar Chart and Heatmap 

**User Prompt:** 
> wait first, 
> 
> compare these two options for showing monthly rainfall across more than 40 years: 
> a stacked bar chart with one bar per year and 12 sections per bar 
> a heatmap with months on the x-axis, years on the y-axis, and rainfall represented by colour 
> 
> our main task is to help students and urban planners compare the same month across different years and quickly notice wet, dry, seasonal, and unusual patterns 
> 
> Which option better supports this task, and why? Also explain one weakness of the option you recommend. 

**AI Response:** 
The heatmap better supports your task. A stacked bar chart makes it difficult to compare the same month across different years because most monthly sections do not share a common baseline. Readers also need to remember 12 different colours and visually separate many stacked sections. A heatmap places each month in the same column and each year in the same row. This makes it easier to: 
1. compare the same month across many years; 
2. scan across a year to identify wetter and drier months; 
3. notice repeating seasonal patterns; 
4. spot unusually wet or dry cells. 

One weakness is that colour is less precise than length or position. Readers can quickly identify general patterns, but they may struggle to compare two months with similar rainfall totals or read exact values. A clear colour legend and a small number of annotations can reduce this weakness. 

---

## Interaction 10: Addressing the Heatmap's Precision Limitation 

**User Prompt:** 
> ok so we agreed that the heatmap makes month-to-month and year-to-year patterns easier to scan. however, colour is less precise than position or length for reading exact values. 
> 
> should we still use the heatmap? suggest ways to reduce this weakness without placing a number inside every cell, because that would make the visualization cluttered 

**AI Response:** 

Yes, you should still use the heatmap because your main task is to reveal overall patterns, not to support precise comparison of every rainfall value. 

You can reduce the weakness in several ways: 
1. Use a clear colour legend with labelled rainfall values in millimetres. Readers can estimate the range represented by each colour. 
2. Use a small number of colour groups, such as five or six rainfall ranges, instead of many similar shades. This makes differences easier to recognise. 
3. Annotate only important cells, such as the wettest month, driest month, or unusually high rainfall periods. 
4. Use a simple ordered colour scale, where light consistently means lower rainfall and dark consistently means higher rainfall. 
5. Mention exact values in the caption or supporting text when discussing important findings. 
6. Provide the detailed data through a QR code or supplementary table for readers who need exact monthly values. 

The heatmap remains suitable because it prioritises fast comparison and pattern recognition, while the legend and selected annotations provide enough numerical detail without cluttering every cell. 

---

## Interaction 11: Reviewing the Heatmap Design 

**User Prompt:** 
> okok, so this is what we ended up on: 
> we have decided to use a heatmap for the redesigned visualization 
> 
> current plan: 
> x-axis: January to December 
> y-axis: years in chronological order 
> cell colour: monthly rainfall total in millimetres 
> sequential colour scale where darker colour means more rainfall 
> a separate neutral colour for incomplete or unavailable data 
> a clear legend explaining the rainfall values 
> only a small number of annotations for unusual values 
> 
> review this plan. what should we keep, change, or remove to make the heatmap readable and accessible on a poster? focus on layout, colour, labels, legend, missing data, and annotations. 

**AI Response:** 
Your plan is strong and fits the comparison task well. 

Keep 
1. Months on the x-axis and years on the y-axis. This makes it easy to compare the same month vertically across many years. 
2. Sequential colour scale. Using darker shades for higher rainfall gives the colours a clear order. 
3. Neutral colour for missing or incomplete data. This prevents missing values from being mistaken for zero rainfall. 
4. Limited annotations. A few annotations can guide attention without covering the heatmap. 

Change or refine 
1. Make the chronological direction obvious. Choose either earliest year at the top and latest at the bottom, or the reverse. Keep it consistent and make the order easy to understand. 
2. Use short month labels. Labels such as `Jan`, `Feb`, and `Mar` will save space and remain readable from a distance. 
3. Avoid too many similar shades. Consider using around five or six clearly separated colour ranges. Too many shades may be difficult to distinguish on a printed poster. 
4. Use a colour-blind-friendly scale. A light-to-dark blue scale would suit rainfall. Avoid rainbow and red-green scales. 
5. Label the legend clearly. Use a title such as "Monthly rainfall total (mm)". The legend should show whether the colours represent exact continuous values or rainfall ranges. 
6. Separate partial and unavailable data only when necessary** Using different patterns or colours for both may add complexity. If the distinction is important, explain both clearly. Otherwise, one neutral colour labelled "Incomplete or unavailable" may be simpler. 
7. Annotate meaningful values, not only extremes** You could highlight the wettest month, one unusual dry period, and one noticeable repeated pattern. Each annotation should explain why the value matters. 

Remove or avoid 
1. Numbers inside every cell 
2. Heavy borders around every tile 
3. Too many grid lines - Decorative icons or extra colours 
4. Annotations that only repeat values already clear from the legend 

One remaining concern is that more than 40 year labels may become crowded. You could label every fifth year while keeping all rows in chronological order. 

---

## Interaction 12: Reviewing the Final Heatmap Plan 

**User Prompt:** 
> this is our final heatmap plan: 
> months are placed on the x-axis 
> years are placed on the y-axis 
> monthly rainfall total is represented using an ordered sequential colour scale 
> darker cells represent higher rainfall 
> incomplete or unavailable data are shown separately from valid rainfall values 
> the title explains what is being compared 
> the legend includes the unit in millimetres 
> annotations are limited to a few important or unusual periods 
> the design avoids unnecessary borders, text, and decoration 
> 
> check whether this plan supports our target audience of students and urban planners who want to identify historical monthly and yearly rainfall patterns 
> 
> identify: 
> what is working well 
> any remaining weakness 
> one final improvement we should consider 

**AI Response:** 
Yes, this plan supports your target audience quite well. It matches the goal of helping students and urban planners quickly scan historical rainfall patterns without too much clutter. 

What is working well 
- Months on the x-axis and years on the y-axis. This makes it easy to compare the same month across different years and also scan across a single year for seasonal patterns. 
- Sequential colour scale. Using darker cells for higher rainfall gives a clear visual order, which helps readers quickly spot wetter and drier periods. 
- Separate treatment for incomplete or unavailable data. This improves accuracy because readers will not confuse missing data with low rainfall. 
- Clear title and labelled legend. These help readers understand what the chart shows and what the colours mean without extra explanation. 
- Limited annotations and minimal decoration. This keeps the heatmap clean and allows important patterns to stand out. 

Any remaining weakness 
Exact values may still be hard to compare** The heatmap is good for pattern recognition, but colour is less precise than position or length. Readers may find it harder to judge small differences between similar rainfall values. 

One final improvement to consider 
Reduce label crowding on the y-axis** Since you have more than 40 years, showing every year label may make the poster look crowded. You could label every 5th year or every 10th year while still keeping all rows in the chart. This keeps the layout cleaner but still readable. 