# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Bayesian reanalysis of the **Andilaye WASH intervention trial** (Ethiopia) for CSSS 564 (Spring 2026). The goal is to move beyond prior frequentist work (GEE, mixed-effects models) and evaluate whether the WASH intervention had meaningful effects on household mental health outcomes using a Bayesian hierarchical model.

Authors: Yuwei Wang & William Garcia.

## Rendering the Document

```r
rmarkdown::render("final project code.Rmd")
```

Or use the **Knit** button in RStudio. Output is a PDF (`pdf_document`).

## Data

Four Stata (`.dta`) files are the raw inputs — they live in the repo root but the code reads them from a `final project/` subdirectory path:

| File | Contents |
|---|---|
| `all_hh_members.dta` | Baseline household-member-level data (HSCL items + identifiers) |
| `all_hh_members_end.dta` | Endline household-member-level data |
| `all_hh_lat.dta` | Baseline household-level data (woreda, kebele cluster, treatment assignment) |
| `all_hh_lat_end.dta` | Endline household-level data |

**Important:** The `.Rmd` reads data with paths like `"final project/all_hh_members.dta"`. The `.dta` files must be placed in a `final project/` subdirectory relative to the working directory, or the paths must be updated.

## Architecture

The analysis is structured as a single `.Rmd` with 8 sections:

1. **Research Question & Data** — data loading, scoring helpers, dataset construction
2. **Model Specification** — JAGS model strings (hierarchical logistic regression)
3. **Prior Predictive Check**
4. **Model Fitting** — MCMC via `rjags`
5. **MCMC Diagnostics** — convergence checks via `coda`
6. **Posterior Inference** — APD (Average Predictive Difference), between-cluster variance posteriors
7. **Posterior Predictive Check**
8. **Broader Context**

### Key Data Pipeline (Section 1)

Two helper functions build the analysis datasets:

- **`score_outcomes(mem_df, z1_val, a113_filter)`** — filters to `Z_1 == z1_val` (value 5 = women), recodes 888/999 to `NA`, computes row-means for anxiety (F_1_1–F_1_10) and depression (F_2_1–F_2_13) subscales, aggregates to household means, then applies caseness threshold `≥ 0.75` on the 0–3 scale. (The scale-corrected equivalent of the standard HSCL-25 cutoff of ≥ 1.75 on the 1–4 scale.)

- **`build_lat(lat_df, z1_val, a113_filter)`** — constructs household-level covariates: `treatment` (integer from `rand_var`), `woreda_id` (1=Farta/reference, 2=Fogera, 3=Bahir Dar Zuria), `cluster_id` (re-indexed 1:50 from raw `A_1_6` which has gaps up to 58).

Three final datasets are used in modeling:
- `endline_clean` — primary analysis (endline outcomes + design vars)
- `panel_clean` — sensitivity ANCOVA model (endline + baseline outcomes joined on `hh_id`; 4 households missing baseline are expected NA)

### Model Design

- Hierarchical logistic regression fit in JAGS (`rjags`)
- Three binary outcomes: `anxiety`, `depression`, `emotion` (distress)
- Fixed effects: `treatment`, woreda dummies (Farta = reference, `gamma[1] <- 0`)
- Random effects: kebele cluster (`cluster_id`, 50 levels)
- Primary estimand: **APD** — average difference in predicted caseness probability between treatment and control, with posterior probability that APD ∈ [−0.05, +0.05]
- Secondary estimand: full posterior of between-cluster variance

## R Packages

```r
tidyverse, ggplot2, knitr, kableExtra, rjags, coda, haven, loo
```

JAGS must be installed separately on the system before `rjags` will work: https://mcmc-jags.sourceforge.io/
