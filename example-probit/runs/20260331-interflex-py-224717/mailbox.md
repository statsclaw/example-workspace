# Mailbox

## Handoff: Planner -> Builder (spec.md)

Builder should implement a Python package `interflex` with 6 source modules (`core.py`, `linear.py`, `effects.py`, `variance.py`, `vcov.py`, `average.py`) plus `__init__.py` and `pyproject.toml`. The spec covers the full algorithm for each function: input validation and preprocessing in `core.py`, model fitting and variance dispatch in `linear.py`, treatment effect computation in `effects.py`, delta/simulation/bootstrap variance in `variance.py`, cluster-robust sandwich estimator in `vcov.py`, and weighted ATE/AME with delta-method SE in `average.py`. Key implementation notes: use statsmodels WLS for model fitting, track coefficient names carefully for vcov submatrix extraction, implement CGM small-sample correction for cluster-robust variance, and return pandas DataFrames in the output dict. See spec.md Sections 3.1-3.8 for file-by-file details.

## Handoff: Planner -> Tester (test-spec.md)

Tester should validate the package against 6 deterministic data fixtures (binary discrete, continuous, multi-group, clustered, weighted, full-moderation) with fixed seeds. Core tests verify output structure, point estimate accuracy against known DGPs, treatment type detection, diff value computation, and base group selection. Variance tests check SE positivity, delta/simu/bootstrap agreement, cluster-robust inflation, vcov matrix properties, and CI coverage. Effect tests verify TE linearity, ATE/AME values, prediction consistency, and cross-reference against manual OLS computation. Edge case tests cover invalid inputs, missing values, and boundary conditions. Property invariants test relabeling symmetry, weight equivalence, and additive diff. See test-spec.md Sections 2.1-2.7 for all test scenarios and tolerance guidelines.
