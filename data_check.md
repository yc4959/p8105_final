data_check
================
Yimeng Cao
2026-11-13

``` r
library(tidyverse)
library(httr)

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

# API link for the 2023 SPARCS data, we use this everywhere below
base_url = "https://health.data.ny.gov/resource/46xm-urtu.csv"
```

## Explore available diagnosis groups

Data come from the [Hospital Inpatient Discharges (SPARCS
De-Identified): 2023](https://health.data.ny.gov/d/46xm-urtu) dataset on
Health Data NY.

``` r
# grab a few rows just to see what the columns are called
GET(base_url, query = list("$limit" = "5")) |>
  content("parsed", col_types = cols(.default = "c")) |>
  names()
```

    ##  [1] "hospital_service_area"          "hospital_county"               
    ##  [3] "operating_certificate_number"   "permanent_facility_id"         
    ##  [5] "facility_name"                  "age_group"                     
    ##  [7] "zip_code_3_digits"              "gender"                        
    ##  [9] "race"                           "ethnicity"                     
    ## [11] "length_of_stay"                 "type_of_admission"             
    ## [13] "patient_disposition"            "discharge_year"                
    ## [15] "ccsr_diagnosis_code"            "ccsr_diagnosis_description"    
    ## [17] "ccsr_procedure_code"            "ccsr_procedure_description"    
    ## [19] "apr_drg_code"                   "apr_drg_description"           
    ## [21] "apr_mdc_code"                   "apr_mdc_description"           
    ## [23] "apr_severity_of_illness_code"   "apr_severity_of_illness"       
    ## [25] "apr_risk_of_mortality"          "apr_medical_surgical"          
    ## [27] "payment_typology_1"             "payment_typology_2"            
    ## [29] "payment_typology_3"             "birth_weight"                  
    ## [31] "emergency_department_indicator" "total_charges"                 
    ## [33] "total_costs"

``` r
# count how many stays each DRG has
# (let the API do the counting so we don't download everything)
drg_list =
  GET(base_url, query = list(
    "$select" = "apr_drg_description, count(*) as n",
    "$group"  = "apr_drg_description",
    "$order"  = "n DESC",
    "$limit"  = "400"
  )) |>
  content("parsed")
```

    ## Rows: 335 Columns: 2
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (1): apr_drg_description
    ## dbl (1): n
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
# top 10 most common
drg_list |> print(n = 10)
```

    ## # A tibble: 335 × 2
    ##    apr_drg_description                                                         n
    ##    <chr>                                                                   <dbl>
    ##  1 NEONATE BIRTH WEIGHT > 2499 GRAMS, NORMAL NEWBORN OR NEONATE WITH OTH… 165929
    ##  2 SEPTICEMIA AND DISSEMINATED INFECTIONS                                 131139
    ##  3 VAGINAL DELIVERY                                                       122271
    ##  4 HEART FAILURE                                                           59668
    ##  5 CESAREAN SECTION WITHOUT STERILIZATION                                  59104
    ##  6 MAJOR RESPIRATORY INFECTIONS AND INFLAMMATIONS                          35044
    ##  7 KIDNEY AND URINARY TRACT INFECTIONS                                     30248
    ##  8 OTHER PNEUMONIA                                                         29802
    ##  9 SCHIZOPHRENIA                                                           28240
    ## 10 CVA AND PRECEREBRAL OCCLUSION WITH INFARCTION                           26529
    ## # ℹ 325 more rows

## Select diagnosis groups

We selected five of the ten most common APR-DRGs in 2023, one acute
medical condition per organ system: heart failure, major respiratory
infections, ischemic stroke, sepsis, and kidney and urinary tract
infections. Childbirth, newborn, and psychiatric groups were excluded
because of their distinct care settings.

``` r
# our 5 DRGs (names have to match the data exactly, all caps)
drgs = c(
  "HEART FAILURE",                                   # heart
  "MAJOR RESPIRATORY INFECTIONS AND INFLAMMATIONS",  # lungs
  "CVA AND PRECEREBRAL OCCLUSION WITH INFARCTION",   # brain (stroke)
  "SEPTICEMIA AND DISSEMINATED INFECTIONS",          # whole body (sepsis)
  "KIDNEY AND URINARY TRACT INFECTIONS"              # urinary
)

# turn the list into a filter the API understands, e.g. in('A', 'B', ...)
where_clause = str_c(
  "apr_drg_description in(",
  str_c("'", drgs, "'", collapse = ", "),
  ")"
)

# check how many stays and how many hospitals each DRG has
GET(base_url, query = list(
  "$select" = "apr_drg_description, count(*) as n, count(distinct permanent_facility_id) as n_hosp",
  "$where"  = where_clause,
  "$group"  = "apr_drg_description"
)) |>
  content("parsed")
```

    ## Rows: 5 Columns: 3
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (1): apr_drg_description
    ## dbl (2): n, n_hosp
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

    ## # A tibble: 5 × 3
    ##   apr_drg_description                                 n n_hosp
    ##   <chr>                                           <dbl>  <dbl>
    ## 1 CVA AND PRECEREBRAL OCCLUSION WITH INFARCTION   26529    182
    ## 2 HEART FAILURE                                   59668    189
    ## 3 KIDNEY AND URINARY TRACT INFECTIONS             30248    188
    ## 4 MAJOR RESPIRATORY INFECTIONS AND INFLAMMATIONS  35044    190
    ## 5 SEPTICEMIA AND DISSEMINATED INFECTIONS         131139    187

## Download data

``` r
# only download our 5 DRGs (~280k rows instead of ~2 million)
# read everything as character for now since length_of_stay has "120 +"
# we'll fix the types when we clean the data
sparcs_raw =
  GET(base_url, query = list(
    "$where" = where_clause,
    "$limit" = "300000"
  )) |>
  content("parsed", col_types = cols(.default = "c")) |>
  janitor::clean_names()

# should be 282,628
nrow(sparcs_raw)
```

    ## [1] 282628

## Quick checks

``` r
# how many stays per DRG (should match the counts above)
sparcs_raw |> count(apr_drg_description)
```

    ## # A tibble: 5 × 2
    ##   apr_drg_description                                 n
    ##   <chr>                                           <int>
    ## 1 CVA AND PRECEREBRAL OCCLUSION WITH INFARCTION   26529
    ## 2 HEART FAILURE                                   59668
    ## 3 KIDNEY AND URINARY TRACT INFECTIONS             30248
    ## 4 MAJOR RESPIRATORY INFECTIONS AND INFLAMMATIONS  35044
    ## 5 SEPTICEMIA AND DISSEMINATED INFECTIONS         131139

``` r
# length of stay: any "120 +" values?
sparcs_raw |> filter(str_detect(length_of_stay, "\\+")) |> count(length_of_stay)
```

    ## # A tibble: 1 × 2
    ##   length_of_stay     n
    ##   <chr>          <int>
    ## 1 120 +            129

``` r
# costs and charges: values have commas like "12,345.67", so use parse_number()
sparcs_raw |>
  summarise(
    bad_cost   = sum(is.na(parse_number(total_costs))),
    bad_charge = sum(is.na(parse_number(total_charges)))
  )
```

    ## # A tibble: 1 × 2
    ##   bad_cost bad_charge
    ##      <int>      <int>
    ## 1        0          0

``` r
# peek at a few raw values
sparcs_raw |> select(total_costs, total_charges) |> head()
```

    ## # A tibble: 6 × 2
    ##   total_costs total_charges
    ##   <chr>       <chr>        
    ## 1 13,064.47   84,655.83    
    ## 2 49,762.09   252,898.29   
    ## 3 30,199.12   139,725.51   
    ## 4 14,289.91   73,038.76    
    ## 5 14,635.68   74,938.79    
    ## 6 37,075.43   154,945.35

``` r
# severity levels
sparcs_raw |> count(apr_severity_of_illness_code, apr_severity_of_illness)
```

    ## # A tibble: 4 × 3
    ##   apr_severity_of_illness_code apr_severity_of_illness      n
    ##   <chr>                        <chr>                    <int>
    ## 1 1                            Minor                    17264
    ## 2 2                            Moderate                 76747
    ## 3 3                            Major                   108661
    ## 4 4                            Extreme                  79956

``` r
# payers (we'll need to group these later)
sparcs_raw |> count(payment_typology_1, sort = TRUE)
```

    ## # A tibble: 9 × 2
    ##   payment_typology_1             n
    ##   <chr>                      <int>
    ## 1 Medicare                  193281
    ## 2 Medicaid                   48023
    ## 3 Private Health Insurance   19601
    ## 4 Blue Cross/Blue Shield     13230
    ## 5 Federal/State/Local/VA      3023
    ## 6 Self-Pay                    2418
    ## 7 Managed Care, Unspecified   2004
    ## 8 Miscellaneous/Other          882
    ## 9 Department of Corrections    166

``` r
# records with no hospital id (abortion records are blanked out)
sparcs_raw |> summarise(no_hosp_id = sum(is.na(permanent_facility_id)))
```

    ## # A tibble: 1 × 1
    ##   no_hosp_id
    ##        <int>
    ## 1        184

``` r
# what's inside the respiratory group? (checking for COVID)
sparcs_raw |>
  filter(apr_drg_description == "MAJOR RESPIRATORY INFECTIONS AND INFLAMMATIONS") |>
  count(ccsr_diagnosis_description, sort = TRUE) |>
  head(10)
```

    ## # A tibble: 10 × 2
    ##    ccsr_diagnosis_description                                          n
    ##    <chr>                                                           <int>
    ##  1 COVID-19                                                        22870
    ##  2 Aspiration pneumonitis                                           7700
    ##  3 Pneumonia (except that caused by tuberculosis)                   3445
    ##  4 Tuberculosis                                                      383
    ##  5 Pleurisy, pleural effusion and pulmonary collapse                 351
    ##  6 Gangrene                                                          140
    ##  7 Other specified and unspecified lower respiratory disease          91
    ##  8 Postprocedural or postoperative respiratory system complication    29
    ##  9 Bacterial infections                                               17
    ## 10 Mediastinal disorders                                              10

## Hospital info join

``` r
# hospital info table (ownership, health system, lat/long)
fac_info =
  GET("https://health.data.ny.gov/resource/vn5v-hh5r.csv",
      query = list("$limit" = "50000")) |>
  content("parsed", col_types = cols(.default = "c")) |>
  mutate(fac_id = as.numeric(fac_id))

# hospitals in our data
# SPARCS ids are zero-padded ("000541"), the hospital table isn't ("541"),
# so turn both into numbers before joining
sparcs_hosp =
  sparcs_raw |>
  filter(!is.na(permanent_facility_id)) |>
  count(permanent_facility_id, facility_name) |>
  mutate(permanent_facility_id = as.numeric(permanent_facility_id))

# hospitals that don't match (sorted by number of stays)
sparcs_hosp |>
  anti_join(fac_info, by = c("permanent_facility_id" = "fac_id")) |>
  arrange(desc(n))
```

    ## # A tibble: 6 × 3
    ##   permanent_facility_id facility_name                                          n
    ##                   <dbl> <chr>                                              <int>
    ## 1                  1439 Mount Sinai Beth Israel                             1607
    ## 2                  3376 Cohen Children's Medical Center                      462
    ## 3                  1153 Wyoming County Community Hospital                    282
    ## 4                   325 The University of Vermont Health Network - Alice …   239
    ## 5                   798 Claxton-Hepburn Medical Center                        62
    ## 6                   565 Eastern Niagara Hospital - Lockport Division          41

``` r
# ownership types for the hospitals that do match
fac_info |>
  semi_join(sparcs_hosp, by = c("fac_id" = "permanent_facility_id")) |>
  count(ownership_type, sort = TRUE)
```

    ## # A tibble: 5 × 2
    ##   ownership_type                 n
    ##   <chr>                      <int>
    ## 1 Not for Profit Corporation   180
    ## 2 Municipality                  12
    ## 3 Public Benefit Corporation     9
    ## 4 State                          7
    ## 5 County                         1

``` r
# some hospitals show up more than once in fac_info -- check which ones
fac_info |>
  semi_join(sparcs_hosp, by = c("fac_id" = "permanent_facility_id")) |>
  count(fac_id) |>
  filter(n > 1) |>
  left_join(fac_info, by = "fac_id") |>
  select(fac_id, facility_name, description, ownership_type)
```

    ## # A tibble: 46 × 4
    ##    fac_id facility_name                  description              ownership_type
    ##     <dbl> <chr>                          <chr>                    <chr>         
    ##  1      5 St. Peter's Hospital           Hospital                 Not for Profi…
    ##  2      5 St. Peter's Hospital           Hospital                 Not for Profi…
    ##  3     66 Olean General Hospital         Hospital                 Not for Profi…
    ##  4     66 Olean General Hospital         Hospital                 Not for Profi…
    ##  5    116 Arnot Ogden Medical Center     Hospital                 Not for Profi…
    ##  6    116 Arnot Ogden Medical Center     Hospital                 Not for Profi…
    ##  7    170 Margaretville Hospital         Primary Care Hospital -… Not for Profi…
    ##  8    170 Margaretville Hospital         Primary Care Hospital -… Not for Profi…
    ##  9    181 Vassar Brothers Medical Center Hospital                 Not for Profi…
    ## 10    181 Vassar Brothers Medical Center Hospital                 Not for Profi…
    ## # ℹ 36 more rows
