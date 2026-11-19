# Same Diagnosis, Different Bills?

## Project Overview

This project looks at how much New York hospitals differ in what they charge for the same condition. Using about 282,000 inpatient stays from 2023 statewide discharge data (NY SPARCS), we compare hospital charges and estimated costs for five common conditions: heart failure, major respiratory infections, stroke, sepsis, and kidney and urinary tract infections.

**Course:** P8105 Data Science I | Fall 2026

## Team Members

- Yimeng Cao (yc4959)
- Zeheng Wei (zw3268)
- Yichuan Ma (ym3197)
- Jiaxuan Su (js7161)
- Yulin Zhu (yz5458)

## Project Website

**Website URL:** TBD

**GitHub Repository:** https://github.com/yc4959/p8105_final

## Key Findings

Our analysis answers three questions:

1. **How far apart are hospitals?** How much median charges and estimated costs vary across hospitals within each condition.
2. **Is it the patients or the hospital?** How much of the gap is left after accounting for severity, age group and length of stay.
3. **Expensive for everything?** Whether hospitals that are expensive for one condition are also expensive for the others.

We also built a hospital map and a charge estimator that shows what the same typical patient would be charged at different hospitals.

## Repository Structure

```
p8105_final/
├── README.md
├── Proposal.Rmd            # Project proposal
├── data_check.Rmd          # Picking the five conditions, checking the download
├── data_pipeline.Rmd       # Download and cleaning
├── analysis_rules.Rmd      # Rules every analysis uses
├── eda.Rmd                 # Exploratory analysis
├── q1_gaps.Rmd             # Q1: differences across hospitals
├── q2_adjusted.Rmd         # Q2: differences after accounting for patients
├── q3_consistency.Rmd      # Q3: consistency across conditions
├── estimator.Rmd           # Charge estimator
├── map.Rmd                 # Hospital map
├── report.Rmd              # Full report
├── _site.yml               # Website configuration
├── index.Rmd               # Homepage
├── results/                # One page per condition
└── data/
    ├── sparcs_clean.csv           # Cleaned stays (made by data_pipeline.Rmd)
    ├── hospital_medians.csv       # Q1 output
    ├── hospital_adjusted.csv      # Q2 output
    ├── hospital_consistency.csv   # Q3 output
    └── estimator_grid.csv         # Estimator output
```

## Data Sources

- **Hospital discharges:** [Hospital Inpatient Discharges (SPARCS De-Identified): 2023](https://health.data.ny.gov/d/46xm-urtu) via Health Data NY
- **Hospital information:** [Health Facility General Information](https://health.data.ny.gov/d/vn5v-hh5r) via Health Data NY (ownership and location)

## Analysis Rules

1. Drop 20 stays with estimated costs under $500.
2. For rankings, regression and comparing conditions, keep hospitals with at least 30 stays for that condition.
3. Keep stays where charges are lower than costs (mostly BronxCare Hospital Center).

Details are in `analysis_rules.md`.

## Reproducing the Analysis

1. Clone the repository and open `p8105_final.Rproj` in RStudio.
2. The cleaned data `data/sparcs_clean.csv` is already included. To download and clean it again, knit `data_pipeline.Rmd`.
3. Knit the analysis files in this order (each one saves a file the next ones use):

```
q1_gaps.Rmd -> q2_adjusted.Rmd -> estimator.Rmd -> q3_consistency.Rmd -> map.Rmd
```

4. Build the website:

```r
rmarkdown::render_site()
```

## Notes

- Charges are hospital list prices, not what patients or insurers actually paid. Costs are estimates made by the state.
- Data are for 2023 only.

## Contact

For questions, please contact any team member listed above.