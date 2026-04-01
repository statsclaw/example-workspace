# Implementation Report: interflex Python Package

## Files Created

| File | Lines | Description |
|------|-------|-------------|
| `interflex/__init__.py` | 7 | Public API: `interflex()`, `InterflexResult`, `__version__` |
| `interflex/_typing.py` | 10 | Type aliases: Method, VarType, VcovType, TreatType, XdistrType |
| `interflex/result.py` | 59 | `InterflexResult` dataclass with `.predict()` method |
| `interflex/core.py` | 404 | Entry point: input validation, preprocessing, treat type detection, factor encoding |
| `interflex/linear.py` | 641 | Main estimator: model fitting (GLM, FWL, IV, 2SLS), vcov estimation, variance dispatch |
| `interflex/effects.py` | 574 | `gen_general_te()` and `gen_ate()`: TE/ME/ATE/AME computation with delta-method SE |
| `interflex/variance.py` | 1314 | Three variance paths: `variance_simu()`, `variance_delta()`, `variance_bootstrap()` |
| `interflex/vcov.py` | 192 | `vcov_cluster()` (sandwich), `robust_vcov()` (HC1), `pcse_vcov()` (Beck & Katz 1995) |
| `interflex/fwl.py` | 273 | `fwl_demean()` and `iv_fwl()`: exact translation of `fastplm.cpp`/`iv_fastplm.cpp` |
| `interflex/uniform.py` | 125 | `calculate_uniform_quantiles()` (bisection) and `calculate_delta_uniform_ci()` (MVN) |
| `interflex/plotting.py` | 281 | `plot_interflex()`: multi-panel TE/ME plots with CI ribbons, histogram/density strips |
| `interflex/predict.py` | 48 | `predict_interflex()`: delegates to plotting with pred_lin/link_lin data |
| `pyproject.toml` | 23 | Package metadata, dependencies, pytest config |
| **Total** | **3951** | |

## Key Implementation Decisions

1. **Coefficient naming**: Mirrors R exactly using dict keys: `"(Intercept)"`, `X`, `f"D.{char}"`, `f"DX.{char}"`, `D`, `"DX"`, `f"{z}.X"`. This enables direct formula translation from R.

2. **FWL demeaning**: Iterative weighted group-mean subtraction matching the C++ `fastplm.cpp` algorithm exactly (tol=1e-5, max_iter=50). Zero-variance columns after demeaning set coefficient to NaN (matching C++ `arma::unique(X.col(i)).n_rows == 1`).

3. **IV-FWL**: Full 2SLS after FWL demeaning, matching `iv_fastplm.cpp`. For no-FE IV, uses manual 2SLS projection.

4. **GLM fitting**: Uses `statsmodels.api.GLM` with `freq_weights` for linear/logit/probit/poisson. NegativeBinomial uses `statsmodels.discrete.discrete_model.NegativeBinomial` with `disp=0` (silent).

5. **Vcov**: Cluster-robust uses sandwich formula with DFC correction `(M/(M-1)) * ((N-1)/(N-K))` matching R `sandwich`. HC1 uses standard formula. PCSE implements Beck & Katz pairwise cross-unit covariance.

6. **Variance paths**: All three (simu, delta, bootstrap) produce identical output structure for seamless switching. Delta uses t-distribution critical values (`scipy.stats.t.ppf`). Simulation uses `multivariate_normal` draws. Bootstrap supports cluster resampling.

7. **Uniform CI**: Bisection search (tol=1e-6) for bootstrap/simu. MVN-based critical value for delta method (2000 draws). Falls back to Bonferroni with warning.

8. **Treatment type**: Auto-detected from D (>5 unique values = continuous). Discrete arms encoded as `Group.1`, `Group.2` internally with origin mapping for result labels.

9. **Residuals**: Uses `resid_response` from statsmodels GLM results (not `resid` which is not available on GLMResults).

## Deviations from Spec

1. **No separate test files**: Per builder rules, tests/ directory was not modified (instruction: "do NOT modify tests/"). Builder-written unit tests would normally go in tests/ but this is excluded from write surface.

2. **Uniform CI regularization**: Added small ridge regularization (1e-10) when covariance matrix is near-singular for MVN draws, preventing LinAlgError. The R version would error in this case.

3. **Bootstrap model refit**: The `_fit_model()` helper in `variance.py` handles the refit for all model types. For FE models, it re-runs `fwl_demean()` on each bootstrap sample.

4. **Plotting simplification**: Used matplotlib directly rather than seaborn for the main plots. The R version uses ggplot2; the Python version provides equivalent multi-panel layout with CI ribbons, histogram/density strips, and uniform CI dashed lines.

## Smoke Test Results

All core paths verified:
- Discrete treatment with delta variance: PASS
- Continuous treatment with delta variance: PASS
- Discrete treatment with simulation variance: PASS
- Fixed effects with delta variance: PASS
- Plot generation: PASS
- Logit method: PASS
- Bootstrap variance: PASS
- Predict method: PASS

## Known Limitations

1. **Parallel bootstrap**: Uses `ProcessPoolExecutor` but serialization overhead may make it slower than sequential for small datasets. The R version uses `foreach`/`doParallel`.

2. **NegativeBinomial**: statsmodels NB2 parameterization uses `alpha = 1/theta`. Not fully cross-validated against R's `MASS::glm.nb()` yet.

3. **L-kurtosis diagnostic**: Not implemented (spec notes it as diagnostic-only, not affecting numerical results).

4. **PCSE performance**: The current O(T * N^2) implementation is correct but could be optimized for large panels.
