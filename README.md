# BayesianVAR.jl

Julia code to estimate a Bayesian VAR with Minnesota, sum-of-coefficients and dummy-initial-observation
priors, and to identify structural shocks with sign restrictions or an external instrument.

This is not a registered Julia package. It is a collection of functions tied together by one pipeline:
you describe the data in an Excel file, set a few options and run a single script. The method is
explained step by step at
[lapobini.github.io/project/bayesian-var](https://lapobini.github.io/project/bayesian-var/).

## What the pipeline does

1. **Reads the data** from an Excel file. Series can come at different frequencies; lower-frequency series
   are interpolated to the highest one, and each series is transformed as specified (log, difference,
   year-on-year growth).
2. **Selects the lag length and the prior tightness** (`p`, `λ`, `τ`, `γ`) by maximizing the marginal
   likelihood over a grid (Giannone, Lenza and Primiceri, 2015).
3. **Builds the priors with dummy observations** (Bańbura, Giannone and Reichlin, 2010) and draws from the
   posterior with a Gibbs sampler, discarding non-stationary draws.
4. **Identifies the structural shocks**:
   - with **sign restrictions** (Rubio-Ramírez, Waggoner and Zha, 2010), if the Excel file has a `Sign`
     sheet;
   - with an **external instrument** (Stock and Watson, 2018), if the Excel file has an `IV` sheet.
5. **Plots the results**: impulse responses with 64%, 80% and 90% bands, and optionally historical
   decompositions and the deterministic trend of each series.

## Requirements

Julia 1.10 and the following packages:

```julia
using Pkg
Pkg.add(["DataFrames", "Plots", "LaTeXStrings", "GLM", "XLSX", "CSV", "JLD2", "FredData",
         "Dierckx", "SpecialFunctions", "Parameters"])
```

## How to run

1. Put your Excel file in `Data/FinalData/` (see [Data file](#data-file); `test.xlsx` is an example).
2. In `Code/SetUp/SetupBVAR.jl`, set the file name (`data_file`), the sample (`start_date`, `end_date`,
   as `dd/mm/yyyy`) and the name of the results folder (`results_folder`).
3. In `Code/main_executer.jl`, set the options passed to `bvar_main` (see [Options](#options)).
4. Start Julia **from the repository's root folder** (the code uses paths relative to it) and run:
   ```julia
   include("Code/main_executer.jl")
   ```

The figures are saved as PDFs in `Results/<results_folder>/`:

| Folder | Content |
|---|---|
| `IRF_sign/<shock name>/` | Impulse responses to each sign-identified shock |
| `IRF_iv/` | Impulse responses to the instrumented shock |
| `Hist_dec/` | Historical decompositions (if `historical_decomp` is set) |
| `trend/` | Deterministic trend vs. actual series (if `trend_variables = true`) |

## Data file

The Excel file has the following sheets.

**`Data_m`, `Data_q`, ...**: one sheet per frequency, named `Data_` + `d`, `w`, `m` or `q`, in the format
of a Haver Analytics export. Dates are in column B and the series from column C onwards; the first 18
rows are a header (series ID in row 1, name in row 2), and the data start in row 19.

**`Legend`**: one row per series, **in the same order as the columns of the `Data_` sheets** (sheet by
sheet, from left to right).

| Column | Meaning |
|---|---|
| `seriesID` | ID of the series, as in row 1 of its `Data_` sheet |
| `Release name` | Name used in the plots |
| `LOG`, `DIFF`, `FILT` | Transformation flags (1/0), see below |
| `PRIOR AR(1)` | 1 if the series is persistent (random walk prior), 0 if it is mean reverting (white noise prior) |
| `INCL` | 1 to include the series in the VAR, 0 to leave it out |

| `LOG` `DIFF` `FILT` | Transformation |
|---|---|
| 0 0 0 | Level |
| 1 0 0 | Log level |
| 0 1 0 | First difference |
| 1 1 0 | Period-on-period growth rate, 100 × log difference |
| 0 1 1 | Year-on-year difference |
| 1 1 1 | Year-on-year growth rate, in % |

**`Sign`** (optional, for sign restrictions): one row per series, in the `Legend` order.

- Column A, `policy variable`: write `1` on the row of the variable associated with the first shock,
  `2` for the second shock, and so on. Leave the other rows blank.
- Column B: the series name.
- From column C: one column per shock, with the name of the shock in the first row. Write `1` for a
  positive response, `-1` for a negative one, and leave the cell blank for no restriction.

**`IV`** (optional, for an external instrument): the dates in the first column (`DATE`) and the instrument
in the second. The header of the second column must be the `seriesID` of the variable whose shock is
instrumented.

## Options

The main arguments of `bvar_main`, set in `Code/main_executer.jl`:

| Argument | Default | Meaning |
|---|---|---|
| `λ` | `[0.1, 1, 10, 100]` | Grid for the overall tightness of the Minnesota prior |
| `τ`, `γ` | `10λ` | Grids for the sum-of-coefficients and dummy-initial-observation priors |
| `p` | `[2, 3, 4, 5, 6, 7, 8, 12]` | Grid for the number of lags |
| `ε` | `0.0001` | Tightness of the (diffuse) prior on the intercept |
| `reps`, `burnin` | `10000`, `3000` | Gibbs sampler draws and burn-in |
| `max_try` | `2000` | Maximum number of attempts to draw a stable VAR at each Gibbs iteration |
| `years` | `5` | Horizon of the impulse responses, in years |
| `name_iv_shock` | `"Oil"` | Name of the instrumented shock in the plots |
| `historical_decomp` | `[]` | `seriesID`s for which to plot a historical decomposition |
| `trend_variables` | `false` | Plot the deterministic trend of each series |
| `check_stationarity` | `true` | Discard posterior draws for which the VAR is not stable |

## Code structure

```
Code/
  main_executer.jl   entry point: options and call to bvar_main
  SetUp/             packages, data file, sample, and loading of all the functions below
  DataSection/       reading the Excel file and frequency conversion
  BVAR/              priors, dummy observations, marginal likelihood, Gibbs samplers, identification
  PlotsTables/       impulse responses, historical decompositions, trends
  Toolbox/           helper functions (transformations, lags, companion form, filters, ...)
Data/FinalData/      input Excel files
```

## Author

Lapo Bini, UC San Diego ([lapobini.github.io](https://lapobini.github.io)).
