Exploratory Data Analysis
================

``` r
library(tidyverse)
library(ggridges)
library(patchwork)

knitr::opts_chunk$set(
  fig.width = 6,
  fig.asp = .6,
  out.width = "90%"
)

theme_set(theme_minimal() + theme(legend.position = "bottom"))

options(
  ggplot2.continuous.colour = "viridis",
  ggplot2.continuous.fill = "viridis"
)

scale_colour_discrete = scale_colour_viridis_d
scale_fill_discrete = scale_fill_viridis_d
```

## Load data

``` r
sparcs_clean =
  read_csv(
    "./data/sparcs_clean.csv",
    col_types = cols(
      condition = col_factor(levels = c("Heart failure", "Respiratory infection", "Stroke", "Sepsis", "Kidney/UTI")),
      severity  = col_factor(levels = c("Minor", "Moderate", "Major", "Extreme")),
      age_group = col_factor(levels = c("0 to 17", "18 to 29", "30 to 49", "50 to 69", "70 or Older")),
      payer     = col_factor(levels = c("Medicare", "Medicaid", "Private", "Other")),
      ownership = col_factor(levels = c("Nonprofit", "Public"))
    )
  )

# apply our rule: drop stays with costs under $500 (too low to be real)
sparcs_eda =
  sparcs_clean |>
  filter(costs >= 500)

# how many we dropped
nrow(sparcs_clean) - nrow(sparcs_eda)
```

    ## [1] 20

## Overview

``` r
# basic numbers for each condition
sparcs_eda |>
  group_by(condition) |>
  summarize(
    stays          = n(),
    hospitals      = n_distinct(hosp_id),
    median_los     = median(los),
    median_charges = median(charges),
    median_costs   = median(costs)
  ) |>
  knitr::kable(digits = 0, format.args = list(big.mark = ","))
```

| condition | stays | hospitals | median_los | median_charges | median_costs |
|:---|---:|---:|---:|---:|---:|
| Heart failure | 59,633 | 189 | 5 | 46,607 | 14,074 |
| Respiratory infection | 35,005 | 190 | 5 | 50,495 | 14,760 |
| Stroke | 26,520 | 182 | 4 | 60,426 | 16,288 |
| Sepsis | 131,038 | 187 | 6 | 64,343 | 18,104 |
| Kidney/UTI | 30,228 | 188 | 4 | 39,427 | 10,684 |
