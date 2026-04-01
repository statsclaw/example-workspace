# Request

## ID
20260331-interflex-py-224717

## Scope
Translate the `estimator = "linear"` path from the R package `interflex` (xuyiqing/interflex) into a new Python package. Ship to `statsclaw/example-probit`.

## Source
- R package: https://github.com/xuyiqing/interflex
- Key files: `R/linear.R` (~2249 lines), `R/interflex.R` (entry point), `R/vcluster.R` (cluster vcov)

## What to translate
1. Main `interflex()` entry point — input validation, preprocessing, treatment type detection
2. `interflex.linear()` — full linear estimator including:
   - Formula construction for discrete/continuous treatments
   - Model fitting via OLS (statsmodels equivalent of R's glm with gaussian)
   - Variance-covariance estimation: homoscedastic, HC1 robust, cluster-robust
   - Treatment effect computation (`gen.general.TE`)
   - Delta method variance (`gen.delta.TE`)
   - Simulation-based variance (`vartype="simu"`)
   - Bootstrap variance (`vartype="bootstrap"`)
   - Average treatment effects (`gen.ATE`)
   - Difference estimates at moderator quantiles
3. Support for both discrete and continuous treatments
4. Support for covariates (Z) and fully moderated models
5. Weighted estimation

## What NOT to translate
- Fixed effects (FE) — skip felm/lfe functionality
- Instrumental variables (IV) — skip ivreg/AER functionality
- Panel-corrected standard errors (pcse) — skip
- Kernel, binning, GAM, raw, GRF, DML estimators
- Plotting (plot.R, plot_pool.R) — Python users can use matplotlib directly
- Uniform confidence intervals

## Acceptance Criteria
1. Python package installs cleanly (`pip install -e .`)
2. `pytest` passes with all tests green
3. Numerical results match R package output for identical inputs (within floating-point tolerance)
4. Package has proper structure: pyproject.toml, src layout, tests/

## Target Repository
- Repo: statsclaw/example-probit
- Branch: main
- Currently empty (no commits)

## Workspace
- Status: active at TianzhuQin/workspace
