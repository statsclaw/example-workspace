# Request: R-to-Python Translation of interflex Linear Estimator

## Scope
Translate the `estimator = "linear"` path from the R package `xuyiqing/interflex` into a new Python package. Ship to `statsclaw/example-R2PY`.

## Source
- R package: `xuyiqing/interflex` (v1.3.2)
- Key files: `R/interflex.R`, `R/linear.R`, `R/plot.R`, `R/predict.R`, `R/vcluster.R`, `R/uniform.R`
- C++ files: `src/fastplm.cpp`, `src/iv_fastplm.cpp`

## What to translate
- Main `interflex()` entry point (linear path only)
- `interflex.linear()` — full linear estimator implementation
- All variance estimation paths: delta method, simulation, bootstrap
- All model types: linear, logit, probit, poisson, nbinom
- Fixed effects support (FWL demeaning — translate C++ to NumPy/SciPy)
- Instrumental variables support
- Clustered, robust, HC1, PCSE standard errors
- Average treatment effects (ATE/AME)
- Plotting via matplotlib/seaborn
- S3 predict method → Python predict method
- Uniform confidence intervals

## Acceptance Criteria
1. Python package installable via pip
2. API mirrors R interflex for the linear estimator
3. Numerical results match R output within floating-point tolerance
4. Comprehensive test suite covering all variance types and model methods
5. Documentation with examples
6. Ships to statsclaw/example-R2PY

## Target repo
- GitHub: statsclaw/example-R2PY (empty, to be populated)

## Workspace repo
- GitHub: TianzhuQin/workspace (status: PASS)
