# Stochastic vs Deterministic Auto Reserving

A loss reserving model for U.S. **Commercial Auto Liability**, built on the CAS NAIC Schedule P dataset. It estimates IBNR reserves two ways and compares them: a **deterministic** Chain Ladder in Python, and a **stochastic** bootstrap Chain Ladder in R.

> 🚧 Work in progress: a fuller write-up is coming.

## Repository contents

| File | Description |
|---|---|
| `comauto_pos_98-07.csv` | CAS Schedule P Commercial Auto data: 157 insurer groups, accident years 1998–2007, development lags 1–10 |
| `CommAuto.db` | SQLite database holding the CSV as the `commercial_auto` table |
| `Deterministic Auto Reserving` | Python script: loads the CSV into SQLite, builds the industry paid-loss triangle, and runs the Chain Ladder (LDFs, CDFs, ultimate losses) |
| `Stochastic Auto Reserving.Rmd` | R Markdown: bootstrap Chain Ladder (`ChainLadder::BootChainLadder`, 1,000 simulations, gamma process distribution) |
| `Commercial_Auto_Reserving_Dashboard.xlsx` | Excel dashboard with the triangle, development factors, results, bootstrap summary, and an executive summary |

## Method

1. **Data:** Aggregate cumulative paid losses across all insurers by accident year and development lag.
2. **Triangle:** Mask out cells with accident year plus lag above 2008, so only data known at year-end 2007 is used.
3. **Deterministic:** Use volume-weighted age-to-age factors, then CDFs, then projected ultimate losses and IBNR.
4. **Stochastic:** Bootstrap the same triangle to get a distribution of IBNR and read off percentiles.

## Key results

| Measure | Value |
|---|---|
| Deterministic IBNR (Chain Ladder) | 2,265,185 |
| Stochastic mean IBNR (bootstrap) | 2,265,416 |
| Stochastic 95th percentile IBNR | 2,384,477 |
| Implied risk margin (95th pct − deterministic) | ~119,292 |

The two methods agree on the central estimate. The bootstrap adds a view of uncertainty, which supports a risk margin of about 5% above the point estimate.

## Tools

- **Python:** pandas, NumPy, sqlite3
- **R:** ChainLadder, DBI, RSQLite, reshape2, openxlsx
- **SQL:** SQLite
- **Excel:** dashboard and executive summary

## Author

Minseok Han
