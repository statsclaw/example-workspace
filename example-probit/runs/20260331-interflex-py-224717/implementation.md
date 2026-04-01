# Implementation Artifact

## Request
20260331-interflex-py-224717 — Translate the R interflex package's linear estimator into a Python package.

## Files Created

| File | Purpose |
|------|---------|
| `pyproject.toml` | Package metadata, build config (setuptools), dependencies (numpy, pandas, statsmodels, scipy) |
| `src/interflex/__init__.py` | Public API: exports `interflex()` |
| `src/interflex/core.py` | Entry point `interflex()` — input validation, missing value handling, covariate preprocessing (sum-coded dummies for factors), treatment type detection, discrete/continuous treatment processing, diff_values processing, Z_X interaction creation |
| `src/interflex/linear.py` | `interflex_linear()` — main estimator orchestrator. Constructs design matrix, fits WLS via statsmodels, dispatches to delta/simu/bootstrap variance paths. Assembles full output dict with est_lin, pred_lin, diff_estimate, vcov_matrix, avg_estimate |
| `src/interflex/effects.py` | `compute_effects()` — treatment effect (discrete) and marginal effect (continuous) computation at evaluation grid. Implements gen.general.TE logic. Handles covariates, full_moderate Z*X interactions, and difference computation |
| `src/interflex/variance.py` | `compute_delta_variance()` — delta-method SEs via gradient vectors and vcov submatrix extraction. `_build_gradient_vector()` helper constructs gradient for TE/ME, predictions, and base group. `compute_base_delta_variance()` for base group prediction SEs |
| `src/interflex/vcov.py` | `compute_vcov()` — dispatches to homoscedastic, HC1 robust, or cluster-robust. `vcov_cluster()` implements CGM sandwich estimator with small-sample correction |
| `src/interflex/average.py` | `compute_ate()` — ATE (discrete) and AME (continuous) with optional delta-method SEs using full gradient vectors |
| `tests/__init__.py` | Test package marker |
| `tests/conftest.py` | Shared fixtures: discrete_data, continuous_data, clustered_data, three_group_data — all with known DGP parameters |
| `tests/test_basic.py` | 21 tests: input validation, discrete/continuous basic functionality, output structure, treatment effect signs, auto-detection, weights, Xunif |
| `tests/test_variance.py` | 16 tests: delta/simu/bootstrap variance for discrete and continuous, cluster vcov, vcov matrix symmetry, diff estimates, ATE/AME |
| `tests/test_effects.py` | 13 tests: direct compute_effects tests with known coefficients, full_moderate, edge cases (custom neval, X_eval, data immutability, percentile diffs) |

## Summary of Implementation

### What was added
- Complete Python `interflex` package implementing the linear interaction estimator from R's interflex
- Supports both discrete (binary/multi-group) and continuous treatments
- Three variance estimation methods: delta method, simulation-based, and bootstrap
- Three vcov types: homoscedastic, HC1 robust, CGM cluster-robust
- Full moderator support (Z*X interactions via `full_moderate=True`)
- Weighted estimation via WLS
- Treatment effect differences at custom moderator values
- ATE/AME with delta-method standard errors
- Sum-coded dummy variables for categorical covariates (matching R's `contr.sum`)
- 50 unit tests covering all major code paths

### Package installation
Confirmed: `pip install -e .` succeeds. `python -c "from interflex import interflex"` works. All 50 tests pass.

### Design Decisions

1. **Build backend**: Spec specified `setuptools.backends._legacy:_Backend` which does not exist in current setuptools. Changed to `setuptools.build_meta` (the standard backend). This is a spec deviation required for functionality.

2. **Coefficient indexing**: Used string-keyed dicts (`{name: value}`) rather than positional arrays, matching the spec's recommendation. The `_get_coef()` helper returns 0.0 for missing keys (dropped/aliased terms).

3. **Gradient construction**: `_build_gradient_vector()` in variance.py handles all cases (discrete/continuous, with/without Z, with/without full_moderate) and returns valid coefficient indices to extract the correct vcov submatrix. This is the core of the delta method implementation.

4. **Bootstrap**: Sequential implementation as specified. Block bootstrap for clustered data samples clusters with replacement. Skips replicates where model fitting fails or not all treatment groups are present.

5. **Internal group naming**: Discrete treatments are mapped to `Group.1`, `Group.2`, etc. internally (matching R behavior), then mapped back to original labels in the output dict keys.

6. **Deprecation fix**: Replaced `pd.api.types.is_categorical_dtype()` (deprecated in pandas 2.x) with `isinstance(dtype, pd.CategoricalDtype)`.

### Known Limitations
- Factor covariate handling in Z_ref is simplified: when user provides Z_ref for factor variables, the current code defaults factor dummies to 0 rather than fully resolving the user-specified level to sum-coded values. This edge case needs enhancement if users routinely provide Z_ref for categorical covariates.
- No parallel bootstrap (as specified — sequential only).
- No fixed effects, IV, PCSE, or non-linear estimators (excluded per spec).

## Unit Tests Written

| Test File | Tests | Purpose |
|-----------|-------|---------|
| test_basic.py | 21 | Input validation (8), discrete treatment (6), continuous treatment (5), weights (1), Xunif (1) |
| test_variance.py | 16 | Delta method (7), simulation (2), bootstrap (3), differences (3), ATE/AME (2) |
| test_effects.py | 13 | Coefficient lookup (2), compute_effects direct (4), full_moderate (2), edge cases (4) |

All 50 tests pass.
