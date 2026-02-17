# EPLS Methodology (Pre-Tax + Post-Tax)  
## Final pipeline notebooks (`nbs/__final`)  

Last updated: 2026-02-16

This document summarizes the production methodology implemented in `nbs/__final`, with explicit parameters, sample attrition, and year-by-year outputs for EPLS 2018, 2021, and 2024.

---

## 1. Scope and Notebook Map

### Parameter and calibration layer
- `nbs/__final/ICR_calibration/hmrc_calibration_combined.ipynb`
- `nbs/__final/model_parameters.ipynb`

### Pre-tax production notebooks
- `nbs/__final/pre_tax/00_pretax_EPLS_2018_parameters.ipynb`
- `nbs/__final/pre_tax/00_pretax_EPLS_2021_parameters.ipynb`
- `nbs/__final/pre_tax/00_pretax_EPLS_2024_parameters_some_some.ipynb`

### Post-tax production notebooks
- `nbs/__final/post_tax/00_postax_EPLS_2018_parameters.ipynb`
- `nbs/__final/post_tax/00_postax_EPLS_2021_parameters.ipynb`
- `nbs/__final/post_tax/00_postax_EPLS_2024_parameters.ipynb`

---

## 2. Parameter Governance (single source of truth)

The central parameter source is:
- `const.data_path / model_parameters.json`

### 2.1 Pre-tax core parameters by year

| Year | HMRC ICR calibration scalar (`hmrc_foi_calibration`) | Synthetic fix/var share (`mlar_fix_variable`) | Operating cost | LTV rates (`btl_rates`) low / med / high |
|---|---:|---|---:|---|
| 2018 | `1.308648` | `fix=0.45635`, `variable=0.54365` | `0.25` | `5.908% / 4.341% / 2.771%` |
| 2021 | `1.018498` | `fix=0.66178`, `variable=0.33822` | `0.25` | `4.366% / 2.906% / 2.322%` |
| 2024 | `0.884737` | `fix=0.79473`, `variable=0.20527` | `0.25` | `3.041% / 2.360% / 3.851%` |

Notes:
- 2018 and 2021 pre-tax analysis still includes synthetic fixed/variable comparability scenarios.
- 2024 also has observed `IntTyp` handling in the 2024 pre-tax/post-tax flow.
- Calibration logic follow economic common, sense, using Median for 2024 and Mean for 2021 e 2018. (if use Mean for 2024 ICR would drastically fall and back of envelope infered interest rates would explode which would not make any sense vis-a-vis BTL rates for 2024).

### 2.2 Post-tax core parameters by year

| Year | Tax year | Section 24 deductible share | Section 24 credit | Corporation tax logic | Dividend tax rate (post-tax config) |
|---|---|---:|---:|---|---:|
| 2018 | 2018/19 | `0.50` | `0.20` | flat `19%` | `0.1782620218579235` |
| 2021 | 2021/22 | `0.00` | `0.20` | flat `19%` | `0.17306939890710382` |
| 2024 | 2024/25 | `0.00` | `0.20` | marginal regime (`19%` to `25%`, 50k/250k limits, fraction 3/200) | `0.19596830601092896` |

### 2.3 `model_parameters.ipynb`: pre-tax benchmark construction

`model_parameters.ipynb` builds pre-tax benchmarks in two blocks:
- `opportunity_cost` from FTSE + gilt history
- `industry_benchmark` from year-end gilt yields plus a fixed risk premium

Inputs:
- `ftse_20y.csv` (`Close`)
- `gilt_20y.csv` (`IRLTLT01GBM156N`)

Computation logic:
1. Convert FTSE to year-end levels and annual returns:
   - `ftse_year_end = ftse_series.resample("Y").last()`
   - `ftse_returns = ftse_year_end.pct_change() * 100`
2. Convert gilts to year-end yields:
   - `gilt_dec = gilt_series.resample("Y").last()`
3. Build 10-year rolling means:
   - `FTSE_10y = rolling_mean(FTSE annual return, 10y)`
   - `Gilt_10y = rolling_mean(Gilt yield, 10y)`
4. Define opportunity-cost benchmarks:
   - `Normal high = FTSE_10y`
   - `Normal lower = 0.6 * FTSE_10y + 0.4 * Gilt_10y`
5. Define industry benchmark:
   - select `gilt_dec` at `2018-12-31`, `2021-12-31`, `2024-12-31`
   - `industry_benchmark = selected_gilt + 3.2`

Final pre-tax benchmark values written to JSON:

| Year | Opportunity cost `Normal high` | Opportunity cost `Normal lower` | Industry benchmark |
|---|---:|---:|---:|
| 2018 | `4.7721` | `3.7572` | `4.5125` |
| 2021 | `3.4266` | `2.6424` | `4.0375` |
| 2024 | `2.7077` | `2.4141` | `7.6345` |

### 2.4 `model_parameters.ipynb`: pre-tax capital gains (`capital_gains_7y`)

Input:
- `Average-prices-2025-06.csv` (ONS HPI, `Average_Price_SA`)

Computation logic:
1. Build regional HPI pivot by `Region_Name` and `Date`.
2. Compute annual average growth (not CAGR) with:
   - `hpi_growth(start, end, years) = ((P_end / P_start - 1) / years) * 100`
3. Build 5y, 7y, 10y panels for each benchmark year.
4. Persist only the 7-year series to JSON as `pretax[year].capital_gains_7y`.

Windows used:
- 2018 panel: 5y `2013-12 -> 2018-12`, 7y `2011-12 -> 2018-12`, 10y `2008-12 -> 2018-12`
- 2021 panel: 5y `2016-12 -> 2021-12`, 7y `2014-12 -> 2021-12`, 10y `2011-12 -> 2021-12`
- 2024 panel: 5y `2019-12 -> 2024-12`, 7y `2017-12 -> 2024-12`, 10y `2014-12 -> 2024-12`

England values stored in JSON:
- 2018: `5.7838`
- 2021: `5.5657`
- 2024: `3.7648`

### 2.5 `model_parameters.ipynb`: post-tax benchmark construction

Post-tax benchmarks are not hardcoded; they are derived from pre-tax benchmarks plus tax-rate transforms.

Step A: derive dividend tax rates (`cleaned_dividend_tax_data`) from `results` using:
- `taxpayer_weighted_avg_marginal_rate -> savers_mapping_basic`
- final rates:
  - 2018: `0.1782620218579235`
  - 2021: `0.17306939890710382`
  - 2024: `0.19596830601092896`

Step B: transform opportunity-cost benchmarks:
- `post_tax_opportunity_cost[scenario, year] = pre_tax_opportunity_cost[scenario, year] * (1 - dividend_tax_rate[year])`

Step C: transform industry benchmark:
- compute `global_rates_for_industry_benchmark` from EPLS weighted effective tax calculations (`process_epls_year`)
- apply:
  - `post_tax_industry_benchmark[year] = pre_tax_industry_benchmark[year] * (1 - global_rate[year])`

Final post-tax benchmark values in JSON:

| Year | Dividend tax rate used for opportunity cost | Opportunity cost `Normal high` | Opportunity cost `Normal lower` | Industry benchmark |
|---|---:|---:|---:|---:|
| 2018 | `0.1783` | `3.9214` | `3.0874` | `3.3858` |
| 2021 | `0.1731` | `2.8336` | `2.1850` | `3.0279` |
| 2024 | `0.1960` | `2.1771` | `1.9410` | `5.6612` |

Industry benchmark implied tax factors from JSON ratio `1 - post/pre` are approximately:
- 2018: `24.97%`
- 2021: `25.00%`
- 2024: `25.85%`

### 2.6 `model_parameters.ipynb`: post-tax capital gains

Input:
- `Average-prices-2025-06.csv` (same regional HPI source as pre-tax CG)

Computation logic:
1. Build 20-year start/end windows by year:
   - 2018: `1998-12 -> 2018-12`, company tax `19%`
   - 2021: `2001-12 -> 2021-12`, company tax `19%`
   - 2024: `2004-12 -> 2024-12`, company tax `25%`
2. Compute:
   - `total_gain_pct = (P_end / P_start - 1) * 100`
   - `annual_gross_cagr = ((P_end / P_start) ** (1/20) - 1) * 100`
3. Apply tax assumptions:
   - individual basic CGT `18%`
   - individual higher CGT `28%`
   - company tax from window config above
4. Net annual post-tax gain formula:

```text
net_annual_gain = annual_gross_cagr - (total_gain_pct * tax_rate / 20)
```

England values in `posttax[year].capital_gains`:

| Year | Individual basic | Individual higher | Company |
|---|---:|---:|---:|
| 2018 | `4.3578` | `2.9821` | `4.2202` |
| 2021 | `3.8002` | `2.8268` | `3.7029` |
| 2024 | `2.4785` | `2.0210` | `2.1583` |

### 2.7 Post-tax benchmark and CG keys consumed by notebooks

All post-tax notebooks use `model_parameters.json -> posttax[year]`:
- `dividend_tax_rate`
- `benchmarks.opportunity_cost.Normal high`
- `benchmarks.opportunity_cost.Normal lower`
- `benchmarks.industry_benchmark`
- `capital_gains.individual_basic`
- `capital_gains.individual_higher`
- `capital_gains.company`

---

## 3. Pre-Tax Modelling Method

### 3.1 Common construction logic

Across years, pre-tax notebooks:
1. Load raw EPLS Stata data.
2. Convert income/value/debt bands to midpoint estimates.
3. Build core fields: `RentIncome_est`, `MarketValue_est`, `Debt_est`, `Equity_est`.
4. Compute LTV and assign LTV-tier rates.
5. Compute uncalibrated interest, ICR, yield, and ROE.
6. Apply HMRC scalar calibration to ICR/interest path.
7. Produce fixed, variable, and unencumbered scenario datasets.
8. Export final CSVs to `const.data_path` (pre-tax notebooks only).

### 3.2 Sample restrictions used in pre-tax notebooks

Filters tracked explicitly:
- missing/non-positive rental income
- missing/non-positive market value
- missing/non-positive landlord weight
- non-positive equity among mortgaged records (dropped from ROE analysis sample)

### 3.3 Attrition tables (from notebook outputs)

| Year | Raw EPLS | Drop rent<=0/missing | Drop value<=0/missing | Drop weight<=0/missing | Drop equity<=0 (mortgaged) | Remaining | Mortgaged analysis sample | Unenc analysis sample |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 2018 | 7,823 | 2,270 | 256 | 0 | 451 | 4,846 | 2,932 | 1,914 |
| 2021 | 9,301 | 1,360 | 115 | 0 | 653 | 7,173 | 4,306 | 2,867 |
| 2024 | 9,219 | 1,333 | 107 | 0 | 375 | 7,404 | 2,975 | 4,429 |

### 3.4 Calibration coverage (mortgaged records used in scalar alignment)

From notebook logs:
- 2018: 3,383 mortgaged records
- 2021: 4,959 mortgaged records
- 2024: 3,350 mortgaged records

This can exceed the final mortgaged ROE analysis sample because calibration needs rent/debt/ICR support, while ROE analysis additionally requires valid equity return definitions.

### 3.5 National fixed vs variable pre-tax comparison (from notebook outputs)

#### 2018
- Fixed (calibrated) weighted N: `76,608`
- Variable (2.55%) weighted N: `91,263`
- ROE mean: `2.51` vs `3.78`
- ROE median: `2.14` vs `2.95`
- ICR mean: `301.31` vs `474.98`

#### 2021
- Fixed (calibrated) weighted N: `138,729`
- Variable (2.20%) weighted N: `70,901`
- ROE mean: `2.94` vs `4.22`
- ROE median: `2.24` vs `3.04`
- ICR mean: `343.66` vs `602.10`

#### 2024
- Fixed (calibrated) weighted N: `123,571`
- Variable (4.40%) weighted N: `26,350`
- ROE mean: `3.63` vs `2.70`
- ROE median: `3.06` vs `2.21`
- ICR mean: `427.66` vs `475.98`

---

## 4. Post-Tax Modelling Method

### 4.1 Common post-tax flow

Each year:
1. Load pre-tax final scenario CSVs (or configured source set for 2024).
2. Apply ownership subsets (`individual` vs `corporate`).
3. Compute post-tax rental cashflow:
   - individual income tax with PA taper/bands
   - Section 24 relief treatment
   - corporation tax path for companies
4. Compute post-tax ROE and post-tax yield.
5. Add post-tax regional capital-gains adders from `posttax.capital_gains`.
6. Apply dividend haircut for corporate combined scenario (`dividend_total_roe`).
7. Produce national/England/regional summaries + validation suite.

### 4.2 2024 source and interest-type handling

Current 2024 post-tax notebook behavior:
- Primary source:  
  `EPLS2024_final_df_fixed.csv`, `EPLS2024_final_df_variable.csv`, `EPLS2024_final_df_uemc.csv` from `const.data_path`
- Fallback source: post-Neal `df_fixed.csv`, `df_variable.csv`, `df_unencumbered.csv`
- Owner filter: `{1, 2}` only
- Ambiguous `IntTyp` treatment: detects pre-split inputs and avoids a second split (`already_presplit` mode)

### 4.3 Matched pre-tax vs post-tax medians (from post-tax notebooks)

| Year | Fixed Ind | Fixed Corp | Var Ind | Var Corp | Unenc Ind | Unenc Corp |
|---|---|---|---|---|---|---|
| 2018 | `2.137 -> 1.340` | `2.554 -> 2.069` | `2.925 -> 2.220` | `3.425 -> 2.774` | `2.679 -> 2.250` | `3.250 -> 2.633` |
| 2021 | `2.240 -> 1.351` | `2.342 -> 1.897` | `3.038 -> 2.275` | `3.575 -> 2.896` | `2.750 -> 2.250` | `3.250 -> 2.633` |
| 2024 | `3.063 -> 1.804` | `3.229 -> 2.561` | `2.209 -> 1.405` | `1.872 -> 1.516` | `2.750 -> 2.147` | `2.917 -> 2.228` |

### 4.4 Final combined scenario medians (Individuals post-tax+CG, Corporates dividend-adjusted)

| Year | Fixed indiv | Variable indiv | Unenc indiv | Fixed corp div | Variable corp div | Unenc corp div |
|---|---:|---:|---:|---:|---:|---:|
| 2018 | 5.17 | 6.06 | 6.48 | 5.17 | 5.64 | 5.52 |
| 2021 | 4.67 | 5.58 | 5.89 | 4.65 | 5.41 | 5.27 |
| 2024 | 3.93 | 3.58 | 4.62 | 4.04 | 2.85 | 3.82 |

### 4.5 Validation outcomes

All three post-tax notebooks report:
- `Overall: PASS`
- core output integrity checks passing
- tax/identity sanity checks passing
- monotonic pre->post median checks passing

---

## 5. Reproducibility and Recommended Execution Order

1. `nbs/__final/ICR_calibration/hmrc_calibration_combined.ipynb`
2. `nbs/__final/model_parameters.ipynb`
3. Pre-tax notebooks (2018, 2021, 2024)
4. Post-tax notebooks (2018, 2021, 2024)

Pre-tax notebooks write `EPLS{year}_final_df_*.csv` to `const.data_path`.  
Post-tax notebooks are non-writing analytical notebooks (tables/plots/validation in-memory).

---

## 6. Interpretation Notes

- Cross-year comparisons are valid only with explicit recognition of:
  - changing financing environment
  - changing fixed/variable identification strategy (synthetic in 2018/2021 vs observed IntTyp in 2024)
  - differing benchmark and capital-gain assumptions by year

