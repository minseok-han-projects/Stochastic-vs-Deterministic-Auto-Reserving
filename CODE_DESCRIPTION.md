# Code Description

This walkthrough covers the two scripts in this project and the Excel dashboard they
feed. Each code section is quoted and followed by an explanation of what it does and
why.

- [Part 1: Deterministic Auto Reserving.py (Python)](#part-1-deterministic-auto-reservingpy-python)
- [Part 2: Stochastic Auto Reserving.Rmd (R)](#part-2-stochastic-auto-reservingrmd-r)
- [Part 3: Excel dashboard](#part-3-excel-dashboard)

Background terms used below:

- **Accident year (AY):** the year the accident happened.
- **Development lag (age):** how many years of payments have been observed. Age 1 is
  the accident year itself.
- **Loss triangle:** a table with accident years as rows and ages as columns. It is
  a triangle because recent accident years have fewer ages observed.
- **Cumulative paid loss:** total paid on an accident year up to a given age.

---

## Part 1: `Deterministic Auto Reserving.py` (Python)

### 1. Imports

```python
import sqlite3 as sq
import pandas as pd
import numpy as np
import os
```

`sqlite3` creates and queries the local database, `pandas` handles tables, `numpy`
provides `NaN` for masked cells, and `os` checks that the data file exists.

### 2. Load the raw CSV into a SQLite database

```python
conn = sq.connect('CommAuto.db')
Comm_Auto = ["comauto_pos_98-07.csv"]

for file in Comm_Auto:
    if os.path.exists(file):
        df = pd.read_csv(file)
        table_name = file.replace("comauto_pos_98-07.csv", "commercial_auto")
        df.to_sql(table_name, conn, if_exists='replace', index=False)
```

The CSV is loaded into a database table called `commercial_auto` inside
`CommAuto.db`. This is the data-engineering layer. Both the Python and R models read
from the same table with SQL, so they are guaranteed to use identical data.

- `if_exists='replace'` means re-running the script rebuilds the table rather than
  appending duplicate rows.
- The loop over a list lets more lines of business (for example other Schedule P
  files) be added later without changing the code.

### 3. Aggregate the industry with SQL

```sql
SELECT AccidentYear, DevelopmentLag, Sum(CumPaidLoss) as CumPaidLoss
FROM commercial_auto
GROUP BY AccidentYear, DevelopmentLag
ORDER BY AccidentYear ASC, DevelopmentLag ASC;
```

The raw data has one row per insurer, accident year and age. `GROUP BY` adds all
157 insurers together, leaving one industry value per accident year and age: 10 × 10
= 100 rows. `pd.read_sql_query` returns the result as a DataFrame (`df_industry`),
and the connection is then closed.

### 4. Pivot into a triangle

```python
triangle = df_industry.pivot(index='AccidentYear', columns='DevelopmentLag',
                             values='CumPaidLoss')
```

`pivot` reshapes the long table into a grid: accident years down the side, ages
across the top. At this point the grid is a full 10 × 10 square, because the dataset
also contains payments made after 2007.

### 5. Mask the future

```python
upper_triangle = triangle.copy()
for ay in upper_triangle.index:
    for lag in upper_triangle.columns:
        if ay + lag > 2008:
            upper_triangle.at[ay, lag] = np.nan
```

The reserve is estimated as of **31 December 2007**, so the model may only use
payments known by then. A cell's calendar year is `AY + lag − 1`. For example,
AY 2007 at age 1 is calendar year 2007, and AY 2007 at age 2 is 2008. The condition
`ay + lag > 2008` is the same as "calendar year after 2007". Those cells are set to
`NaN`, which leaves the upper triangle. The original `triangle` is kept unchanged.

### 6. Loss development factors (LDFs)

```python
for i in range(len(upper_triangle.columns)-1):
    current_lag = upper_triangle.columns[i]
    next_lag = upper_triangle.columns[i+1]
    valid_data = upper_triangle[[current_lag, next_lag]].dropna()
    ldf = valid_data[next_lag].sum() / valid_data[current_lag].sum()
    ldfs.append(ldf)
```

An LDF measures how much cumulative paid grows from one age to the next. For each
pair of adjacent ages:

1. `dropna()` keeps only accident years observed at **both** ages. The newest
   accident year for each pair has no value yet at the later age.
2. The factor is the **sum** at the later age divided by the **sum** at the earlier
   age. This is the volume-weighted average, the standard chain-ladder choice,
   because larger accident years carry more weight than smaller ones.

The result is nine factors, from ages 1–2 through 9–10. For example, the 1–2 factor of
1.9464 means cumulative paid roughly doubles between the first and second year. The
line `ldfs = [1.0 if pd.isna(x) else x for x in ldfs]` is a safety net that replaces
any missing factor with 1.0 (no development).

```python
for i, ldf in enumerate(ldfs):
    print(f"Loss Development Factor from lag {i+1} to lag {i+2}: {ldf:.4f}")
df_ldfs = pd.DataFrame({'Age': [f"{i+1}-{i+2}" for i in range(9)], 'LDF': ldfs})
```

Python counts from 0, so the first factor (`i = 0`) is labelled "lag 1 to lag 2". The
factors are then stored in a two-column table (`Age`, `LDF`) for the Excel export.

### 7. Cumulative development factors (CDFs)

```python
cdfs = [1.0] * 10
for i in range(8, -1, -1):
    cdfs[i] = cdfs[i+1] * ldfs[i]
```

A CDF converts paid-to-date at a given age into the projected **ultimate** amount.
It is the product of every LDF from that age onwards, so the loop works backwards
from age 10:

- Age 10: CDF = 1.0. The triangle is assumed fully developed at age 10 (no tail factor).
- Age 9: CDF = LDF(9–10) = 1.0021
- Age 1: CDF = LDF(1–2) × LDF(2–3) × … × LDF(9–10) = 3.8380

A CDF of 3.838 at age 1 means only about 26% of the ultimate is paid in the first year.

`df_cdfs = pd.DataFrame({'Age': range(1, 11), 'CDF to Ultimate': cdfs})` stores the
ten CDFs with their ages for the Excel export.

### 8. Latest diagonal and ultimate losses

```python
latest_diagonal = upper_triangle.apply(
    lambda row: row.dropna().iloc[-1] if not row.dropna().empty else np.nan, axis=1)
aligned_cdfs = cdfs[::-1]
ultimate_losses = latest_diagonal * aligned_cdfs_series
```

- `latest_diagonal` takes the last known value in each row: the amount paid to date
  for each accident year at 12/31/2007.
- Each accident year sits at a different age: AY 1998 is at age 10 and AY 2007 is at
  age 1. Reversing the CDF list (`cdfs[::-1]`) lines them up, so AY 1998 gets the
  age-10 CDF (1.0) and AY 2007 gets the age-1 CDF (3.838).
- **Projected ultimate = paid to date × CDF.**

### 9. Results table

```python
results = pd.DataFrame({
    'Latest Known Loss': latest_diagonal,
    'CDF to Ultimate': aligned_cdfs_series,
    'Projected Ultimate Loss': ultimate_losses
})
```

The three columns are combined into one table indexed by accident year. The unpaid
reserve is **projected ultimate − latest known loss**, calculated in the dashboard's
Results sheet. It sums to 2,265,185 ($000s).

### 10. Backtest

```python
backtest = pd.DataFrame({'Projected Reserve': ultimate_losses-latest_diagonal,
                         'Actual Paid After 2007': triangle[10]-latest_diagonal})
print(backtest.sum())
```

This checks the estimate against what really happened. `triangle` is the full,
unmasked table, so `triangle[10]` is what each accident year had actually paid by
age 10, including payments made after 2007.

- **Projected Reserve:** projected ultimate − paid to date, i.e. the model's estimate.
- **Actual Paid After 2007:** actual age-10 paid − paid to date, i.e. what was really
  paid.

Because the model also projects to age 10, the comparison is like-for-like. The
printed totals are 2,265,185 projected vs. 2,506,982 actual ($000s): actual payments
came in 10.7% higher.

### 11. Excel export (commented out)

```python
# with pd.ExcelWriter('Commercial_Auto_Reserving_Dashboard.xlsx') as writer:
#     upper_triangle.to_excel(writer, sheet_name = 'Upper Triangle')
#     df_ldfs.to_excel(writer, sheet_name = 'Loss Development Factors', index = False)
#     df_cdfs.to_excel(writer, sheet_name = 'Cumulative Development Factors', index = False)
#     results.to_excel(writer, sheet_name = 'Results')
```

This block writes the triangle, factors and results to the dashboard workbook.
`upper_triangle` and `results` keep their index so the accident years appear as the
first column. `df_ldfs` and `df_cdfs` already have an `Age` column, so their index is
dropped. The block stays commented out because `ExcelWriter` creates a new file:
re-running it would overwrite the finished dashboard.

---

## Part 2: `Stochastic Auto Reserving.Rmd` (R)

The chain ladder gives one number. The bootstrap asks: *if history had played out
slightly differently, how different would that number be?*

### 1. Setup

```r
library(DBI)          # database interface
library(RSQLite)      # connect to and query CommAuto.db
library(reshape2)     # dcast: reshape long data into a grid
library(ChainLadder)  # triangle objects and BootChainLadder
library(openxlsx)     # write results to Excel
```

### 2. Query the same data, masking the future in SQL

```sql
SELECT AccidentYear, DevelopmentLag, SUM(CumPaidLoss) as CumPaidLoss
FROM commercial_auto
WHERE AccidentYear + DevelopmentLag <= 2008 -- Mask the future
GROUP BY AccidentYear, DevelopmentLag
```

This is the same aggregation as the Python script. Here the masking is done directly
in SQL with a `WHERE` clause (the same rule, `AY + lag <= 2008`), so only the 55
cells of the upper triangle are returned.

### 3. Build a ChainLadder triangle

```r
tri_matrix <- dcast(industry_data, AccidentYear ~ DevelopmentLag, value.var = "CumPaidLoss")
tri <- as.triangle(as.matrix(tri_matrix[, -1]))
dimnames(tri)[[1]] <- tri_matrix$AccidentYear
```

- `dcast` pivots the long table into accident years × ages, like `pivot` in pandas.
- `tri_matrix[, -1]` drops the AccidentYear column, leaving only the numbers, and
  `as.triangle` converts them into the triangle class that ChainLadder functions expect.
- The last line puts the accident years back as row labels.

### 4. Bootstrap chain ladder

```r
set.seed(88125)
boot_model <- BootChainLadder(tri, R = 1000, process.distr = "gamma")
bootstrap_summary <- summary(boot_model)
print(bootstrap_summary)
plot(boot_model)
```

`BootChainLadder` implements the over-dispersed Poisson bootstrap of England &
Verrall (2002):

1. **Fit:** run the chain ladder and work backwards to the *expected* payment in
   every known cell.
2. **Residuals:** measure how far each actual payment was from its expected value
   (scaled Pearson residuals).
3. **Resample:** shuffle those residuals with replacement to create a new, plausible
   history (a pseudo-triangle), then re-fit the chain ladder to it. This captures
   **parameter risk**: the LDFs are estimates and could have come out differently.
4. **Simulate the future:** draw each future payment from a **gamma** distribution
   (`process.distr = "gamma"`) around its projected value. This captures
   **process risk**: claim outcomes are random even when the factors are right.
5. **Repeat** `R = 1000` times, which gives 1,000 possible reserve totals.

`set.seed(88125)` fixes the random numbers, so every run gives the same results.

**Reading the summary (`bootstrap_summary$Totals`):**

| Row | Meaning | Value ($000s) |
|---|---|---:|
| Latest | Paid to date | 9,077,090 |
| Mean Ultimate | Average simulated ultimate | 11,342,506 |
| Mean IBNR | Average simulated unpaid reserve* | 2,265,416 |
| SD IBNR | Standard deviation of the unpaid reserve | 71,865 |
| Total IBNR 75% | 75th percentile | 2,314,174 |
| Total IBNR 95% | 95th percentile | 2,384,477 |

\*ChainLadder calls this "IBNR". On a paid triangle it is the total unpaid reserve
(case reserves + IBNR).

**`plot(boot_model)`** draws four charts:

1. a histogram of the simulated totals;
2. their cumulative distribution;
3. box plots of simulated ultimates by accident year;
4. a check of the latest actual payments against the simulated ones.

### 5. Export (commented out)

```r
summary <- as.data.frame(bootstrap_summary$Totals)
wb <- loadWorkbook("Commercial_Auto_Reserving_Dashboard.xlsx")
if (!"Bootstrap Summary" %in% names(wb)) addWorksheet(wb, sheetName = "Bootstrap Summary")
writeData(wb, sheet = "Bootstrap Summary", x = summary, rowNames = TRUE)
saveWorkbook(wb, "Commercial_Auto_Reserving_Dashboard.xlsx", overwrite = TRUE)
```

This opens the workbook the Python script created and writes the table above to a
"Bootstrap Summary" sheet. The `if` check adds the sheet only if it doesn't exist yet,
because `addWorksheet` stops with an error when a sheet with that name is already
there. The block is commented out because the finished dashboard already contains
the results.

---

## Part 3: Excel dashboard

`Commercial_Auto_Reserving_Dashboard.xlsx` brings both models and the backtest
together. Values are stored in $000s and displayed in $ millions using the custom
number format `$#,##0,"M"`. The comma divides the displayed value by 1,000 without
changing the stored number. The Upper Triangle uses `$#,##0"K"` instead, so the full
figures stay visible for checking the development factors.

### Executive Dashboard

Three headline figures, all live formulas:

| Cell | Formula | Shows |
|---|---|---|
| L3 | `=SUM(Results!E2:E11)` | Best estimate unpaid reserve (Python chain ladder) |
| M3 | `='Bootstrap Summary'!B7` | 95th percentile unpaid reserve (R bootstrap) |
| N3 | `=Backtest!E12` | Actual payments after 2007 (hindsight) |

- A stacked column chart shows paid to date (`Results!B2:B11`) and unpaid reserve
  (`Results!E2:E11`) by accident year. Together they make up the projected ultimate.
- The summary text states the risk margin (95th percentile − best estimate = about
  $119 million, 5.3%) and the backtest result.

### Results

The accident years in column A, then paid to date, CDF and projected ultimate from
Python. Column E calculates the unpaid reserve: `=D2-B2` (projected ultimate − paid
to date).

### Backtest

Each row compares the projected reserve with what was actually paid:

| Column | Formula (row 2) | Meaning |
|---|---|---|
| B | `=Results!B2` | Paid to date at 12/31/2007 |
| C | `=Results!E2` | Projected unpaid reserve |
| D | `=SUMIFS('Raw Data'!$G$2:$G$14641, 'Raw Data'!$C$2:$C$14641, A2, 'Raw Data'!$E$2:$E$14641, 10)` | Actual cumulative paid at age 10 |
| E | `=D2-B2` | Actually paid after 2007 |
| F | `=E2-C2` | Difference (actual − projected) |
| G | `=IF(C2=0, …, F2/C2)` | Difference as a % of the projected reserve |

`SUMIFS` adds `CumPaidLoss` (column G of Raw Data) across all 157 insurers for rows
where the accident year matches and the development lag is 10. This is the Excel
equivalent of the SQL `GROUP BY` in the Python script. Row 12 holds the totals, and
rows 14–16 compare the actual total with the bootstrap 95th percentile. Because the
sheet calculates from Raw Data, its totals independently confirm the Python backtest
(2,265,185 projected vs. 2,506,982 actual).

### Supporting sheets

Upper Triangle, Loss Development Factors, Cumulative Development Factors, Results,
Bootstrap Summary and Raw Data let a reviewer trace every number on the dashboard back
to the source data. A note on Bootstrap Summary explains that ChainLadder's "IBNR"
labels mean the total unpaid reserve on a paid triangle.
