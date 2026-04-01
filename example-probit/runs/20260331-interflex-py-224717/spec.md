# Code Pipeline Specification — interflex Python Package

## 1. Notation

| Symbol | Type | Description |
|--------|------|-------------|
| Y | str | Column name of outcome variable |
| D | str | Column name of treatment variable |
| X | str | Column name of moderator variable |
| Z | list[str] or None | Column names of covariates |
| cl | str or None | Column name of cluster variable |
| weights | str or None | Column name of observation weights |
| n | int | Number of observations |
| neval | int | Number of evaluation points (default 50) |
| X_eval | ndarray (neval,) | Grid of moderator values at which to evaluate effects |
| beta_hat | ndarray (K,) | Estimated coefficient vector |
| Sigma_hat | ndarray (K, K) | Estimated variance-covariance matrix of beta_hat |
| K | int | Number of estimated coefficients |
| treat_type | str | "discrete" or "continuous" |
| base | str | Label of baseline treatment group (discrete only) |
| other_treat | list[str] | Labels of non-baseline groups (discrete only) |
| all_treat | list[str] | All treatment group labels sorted (discrete only) |
| D_sample | list[float] | Reference D values for ME evaluation (continuous only) |
| Z_ref | dict[str, float] | Reference values for covariates (defaults to means) |
| Z_X | list[str] | Column names for Z*X interactions when full_moderate=True |
| nsimu | int | Number of simulation draws (default 1000) |
| nboots | int | Number of bootstrap replicates (default 200) |
| M | int | Number of unique clusters |
| df_resid | int | Residual degrees of freedom |

## 2. Package Structure

```
src/interflex/
├── __init__.py       # Public API: interflex()
├── core.py           # interflex() entry + input validation + preprocessing
├── linear.py         # interflex_linear() main estimator orchestration
├── effects.py        # Treatment effect / marginal effect computation
├── variance.py       # Delta method, simulation, bootstrap variance estimation
├── vcov.py           # Variance-covariance matrix estimation (homoscedastic, HC1, cluster)
└── average.py        # ATE/AME computation with delta-method SE
```

Also create at package root:
```
pyproject.toml        # Package metadata, dependencies
tests/
├── __init__.py
├── conftest.py       # Shared fixtures
├── test_basic.py
├── test_variance.py
├── test_effects.py
```

## 3. File-by-File Specification

---

### 3.1 `pyproject.toml`

```toml
[build-system]
requires = ["setuptools>=64", "wheel"]
build-backend = "setuptools.backends._legacy:_Backend"

[project]
name = "interflex"
version = "0.1.0"
description = "Heterogeneous treatment/marginal effects with linear interaction models"
requires-python = ">=3.9"
dependencies = [
    "numpy>=1.21",
    "pandas>=1.3",
    "statsmodels>=0.13",
    "scipy>=1.7",
]

[project.optional-dependencies]
dev = ["pytest>=7.0"]

[tool.setuptools.packages.find]
where = ["src"]
```

---

### 3.2 `src/interflex/__init__.py`

Public API:

```python
from interflex.core import interflex

__all__ = ["interflex"]
```

---

### 3.3 `src/interflex/core.py` — Entry Point and Input Validation

#### Function: `interflex()`

```python
def interflex(
    estimator: str,
    data: pd.DataFrame,
    Y: str,
    D: str,
    X: str,
    treat_type: str | None = None,
    base: str | None = None,
    Z: list[str] | None = None,
    full_moderate: bool = False,
    weights: str | None = None,
    na_rm: bool = False,
    Xunif: bool = False,
    CI: bool = True,
    neval: int = 50,
    X_eval: np.ndarray | None = None,
    vartype: str = "delta",
    vcov_type: str = "robust",
    nboots: int = 200,
    nsimu: int = 1000,
    cl: str | None = None,
    Z_ref: list | None = None,
    D_ref: list[float] | None = None,
    diff_values: list[float] | None = None,
    percentile: bool = False,
) -> dict:
```

#### Algorithm Steps

**Step 1: Validate inputs**

1. Convert `data` to DataFrame if needed (copy it to avoid mutation).
2. Validate `estimator` is `"linear"` (only supported value). Raise `ValueError` otherwise.
3. Validate `Y`, `D`, `X` are strings and exist as columns in `data`. Raise `ValueError` otherwise.
4. If `Z` is provided, validate each element is a string and exists in `data`.
5. Validate `vartype` is one of `"delta"`, `"simu"`, `"bootstrap"`. Default: `"delta"`.
6. Validate `vcov_type` is one of `"robust"`, `"homoscedastic"`, `"cluster"`. Raise `ValueError` for `"pcse"` (not supported).
7. If `vcov_type == "cluster"`, require `cl` is not None and exists in `data`.
8. Validate `nboots` is a positive integer.
9. Validate `nsimu` is a positive integer.
10. Validate `neval` is a positive integer.

**Step 2: Handle missing values**

11. Collect all used columns: `vars = [Y, D, X] + (Z or []) + ([cl] if cl else []) + ([weights] if weights else [])`.
12. If `na_rm is True`: drop rows with any NA in `vars`.
13. If `na_rm is False`: check for any NA in `vars`; if found, raise `ValueError("Missing values found. Use na_rm=True")`.

**Step 3: Validate Z_ref**

14. If `Z_ref` is provided and `Z` is provided, check `len(Z_ref) == len(Z)`.

**Step 4: Xunif transform**

15. If `Xunif is True`: replace `data[X]` with `rank(data[X], method='average') / len(data) * 100`.

**Step 5: Covariate preprocessing**

16. Identify factor/object/category columns in Z.
17. For each factor column in Z:
    - Create sum-coded dummy variables. Use the approach: for a factor with L levels, create L-1 dummies. The last level gets -1 in all dummies (sum coding, like R's `contr.sum`).
    - Implementation: for each factor column `a`, get sorted unique levels. Create L-1 dummy columns named `Dummy.Covariate.{counter}`. For each dummy j (j=0..L-2): set to 1 if observation has level j, set to -1 if observation has the last level, else 0.
    - Add dummy columns to data. Remove original factor column from Z list. Add dummy column names to Z list.
18. Compute `Z_ref`:
    - If user provided `Z_ref`: use it. For factor covariates, the user specifies a level name; look up the corresponding dummy coding from the preprocessing step.
    - If not provided: for numeric Z columns, use `mean(data[z])`. For dummy columns from factors, use 0.
    - Store as dict: `{col_name: float_value}`.

**Step 6: Create Z_X interaction terms (if full_moderate)**

19. If `full_moderate is True` and `Z` is not None:
    - For each covariate `z` in Z: create column `f"{z}.X" = data[z] * data[X]`.
    - Store interaction column names in `Z_X` list.
    - Else: `Z_X = None`.

**Step 7: Detect treatment type**

20. If `treat_type is None`:
    - If `data[D]` is numeric and has > 5 unique values: `treat_type = "continuous"`.
    - Else: `treat_type = "discrete"`.
21. Validate `treat_type` is `"discrete"` or `"continuous"`.

**Step 8: Process discrete treatment**

22. If `treat_type == "discrete"`:
    - Convert `data[D]` to string.
    - `all_treat = sorted(data[D].unique())`. Must be <= 9 unique values.
    - If `base is None`: `base = all_treat[0]`. Else validate `base` is in `all_treat`.
    - `other_treat = [t for t in all_treat if t != base]`.
    - Create internal group names: `Group.1`, `Group.2`, ... mapping to sorted original labels.
    - Build mapping dicts: `treat_to_group` (original -> internal), `group_to_treat` (internal -> original).
    - Rename data[D] values to internal group names.
    - Update `base`, `other_treat`, `all_treat` to use internal names, keeping original names accessible via mapping.

**Step 9: Process continuous treatment**

23. If `treat_type == "continuous"`:
    - Validate `data[D]` is numeric.
    - If `D_ref is None`: `D_sample = {f"{D}={round(v,3)} (50%)": v for v in [median(data[D])]}`.
    - Else: `D_sample = {f"{D}={round(v,3)}": v for v in D_ref}`. Max 9 values.

**Step 10: Process diff_values**

24. If `diff_values is None`: `diff_values = quantile(data[X], [0.25, 0.5, 0.75])`. Names: `["50% vs 25%", "75% vs 50%", "75% vs 25%"]`.
25. If `diff_values` provided:
    - Must be length 2 or 3.
    - If `percentile is True`: convert percentiles to actual values via `quantile(data[X], diff_values)`.
    - Generate difference names:
      - 2 values: `[f"{v2} vs {v1}"]`
      - 3 values: `[f"{v2} vs {v1}", f"{v3} vs {v2}", f"{v3} vs {v1}"]`

**Step 11: Build info dicts**

26. Build `treat_info`:
    - `treat_type`, `other_treat`, `all_treat`, `base` (all internal names), `D_sample` (continuous only), group/original name mappings.
27. Build `diff_info`:
    - `diff_values` (actual X values), `difference_name` (labels).

**Step 12: Call estimator**

28. Call `interflex_linear(data, Y, D, X, treat_info, diff_info, Z, weights, full_moderate, Z_X, neval, X_eval, vartype, vcov_type, nboots, nsimu, cl, Z_ref, CI)`.
29. Return the result dict.

---

### 3.4 `src/interflex/linear.py` — Core Linear Estimator

#### Function: `interflex_linear()`

```python
def interflex_linear(
    data: pd.DataFrame,
    Y: str, D: str, X: str,
    treat_info: dict,
    diff_info: dict,
    Z: list[str] | None = None,
    weights: str | None = None,
    full_moderate: bool = False,
    Z_X: list[str] | None = None,
    neval: int = 50,
    X_eval: np.ndarray | None = None,
    vartype: str = "delta",
    vcov_type: str = "robust",
    nboots: int = 200,
    nsimu: int = 1000,
    cl: str | None = None,
    Z_ref: dict | None = None,
    CI: bool = True,
) -> dict:
```

#### Algorithm Steps

**Step 1: Setup**

1. Extract from `treat_info`: `treat_type`, `other_treat`, `all_treat`, `base` (discrete) or `D_sample` (continuous), group-to-original mappings.
2. Extract from `diff_info`: `diff_values`, `difference_name`.
3. `n = len(data)`.
4. If `cl` is not None: build `clusters = data[cl].unique()`, `id_list = {cluster_val: list_of_indices}`.

**Step 2: Evaluation grid**

5. `X_eval_auto = np.linspace(data[X].min(), data[X].max(), neval)`.
6. If user `X_eval` was provided: `X_eval = np.sort(np.unique(np.concatenate([X_eval_auto, X_eval])))`.
7. Else: `X_eval = X_eval_auto`.
8. `neval = len(X_eval)`.

**Step 3: Construct design matrix and fit model**

9. Add treatment columns to data:
   - Discrete: for each `char` in `other_treat`: `data[f"D.{char}"] = (data[D] == char).astype(float)`, `data[f"DX.{char}"] = data[f"D.{char}"] * data[X]`.
   - Continuous: `data["DX"] = data[D] * data[X]`.

10. Build list of RHS variable names (in order):
    - Start with constant (intercept is automatic in statsmodels).
    - `X`
    - Discrete: for each `char` in `other_treat`: `f"D.{char}"`, `f"DX.{char}"`.
    - Continuous: `D`, `"DX"`.
    - If Z: add all Z column names.
    - If `full_moderate`: add all Z_X column names.

11. Set weights: if `weights is None`: `w = np.ones(n)`. Else: `w = data[weights].values`.
    Store as `data["WEIGHTS"] = w`.

12. Fit model using `statsmodels.api.WLS`:
    ```python
    import statsmodels.api as sm
    y_vec = data[Y].values
    X_mat = sm.add_constant(data[rhs_columns].values)  # adds intercept column
    model = sm.WLS(y_vec, X_mat, weights=w).fit()
    ```
    The column names for the design matrix must be tracked: `["const"] + rhs_columns`.

13. Extract: `model_coef` as a dict mapping coefficient name -> value. `model_df = model.df_resid`.

**Step 4: Compute variance-covariance matrix**

14. Call `compute_vcov(model, vcov_type, cl_vector, data)` from `vcov.py` (see Section 3.6).
15. Store as `model_vcov` — a 2D ndarray with row/col names tracked in a list.
16. Handle NAs: replace any NaN in `model_coef` with 0, any NaN in `model_vcov` with 0.
17. Expand `model_vcov` to full coefficient dimension if needed (some coefficients may not appear in the vcov; fill with 0).
18. Symmetry check: if `model_vcov` is not symmetric within tolerance 1e-6, fall back to homoscedastic vcov and emit a warning.

**Step 5: Compute effects based on vartype**

Dispatch to the appropriate variance method. All three branches produce the same output structure.

**Step 5a: vartype == "simu"** (simulation-based)

19. Draw `M = nsimu` coefficient vectors: `simu_coef = np.random.multivariate_normal(model_coef_array, model_vcov, size=M)`.
20. For discrete treatment:
    - For each `char` in `other_treat`:
      a. Compute TE, E_pred, E_base, link_1, link_0, diff_estimate using `compute_effects()` with `model_coef`.
      b. Compute ATE using `compute_ate()`.
      c. For each simulated coefficient vector, compute the same quantities.
      d. TE_sd = std across simulations. TE_CI = [2.5%, 97.5%] quantiles across simulations. TE_vcov = covariance across simulations.
      e. Assemble output matrices: `TE_output = [X_eval, TE, sd, lower_CI, upper_CI]`.
      f. diff_sd, diff_CI from simulated differences.
      g. ATE_sd, ATE_CI from simulated ATEs.
21. For continuous treatment:
    - Similar loop over `D_ref` in `D_sample`, computing ME instead of TE.
    - AME computed separately using all simulated coefficient vectors.

**Step 5b: vartype == "delta"** (delta method)

22. `crit = abs(scipy.stats.t.ppf(0.025, df=model_df))`.
23. For discrete treatment:
    - For each `char` in `other_treat`:
      a. Compute TE, E_pred, E_base, link_1, link_0, diff_estimate using `compute_effects()`.
      b. Compute delta-method SEs using `compute_delta_variance()` from `variance.py`.
      c. ATE and ATE_sd using `compute_ate()` with `delta=True`.
      d. CI = estimate +/- crit * sd.
      e. Assemble output matrices.
24. For continuous: similar with ME.

**Step 5c: vartype == "bootstrap"**

25. For b = 1..nboots:
    a. If no clustering: sample `n` indices with replacement.
    b. If clustering: sample clusters with replacement (block bootstrap). Indices = union of all observations in sampled clusters.
    c. `data_boot = data.iloc[indices]`.
    d. Refit model on `data_boot` with same formula and weights.
    e. If model doesn't converge: skip this replicate (append NaN row).
    f. For discrete: check all treatment levels present in bootstrap sample; skip if not.
    g. Using bootstrap coefficients, compute TE/ME, predictions, diff, ATE/AME via `compute_effects()` and `compute_ate()`.
26. After all replicates:
    a. sd = nanstd across replicates.
    b. CI = [2.5%, 97.5%] nanquantiles.
    c. vcov = nancov across replicates.

**Step 6: Assemble output**

27. Build output dict:

For discrete:
```python
result = {
    "treat_info": treat_info,  # with original name mappings
    "diff_info": diff_info,
    "est_lin": {orig_label: DataFrame(X, TE/ME, sd, lower_CI, upper_CI) for each treat},
    "pred_lin": {orig_label: DataFrame(X, E_Y, sd, lower_CI, upper_CI) for each treat including base},
    "diff_estimate": {orig_label: DataFrame(diff_est, sd, z_val, p_val, lower_CI, upper_CI)},
    "vcov_matrix": {orig_label: ndarray(neval, neval)},
    "avg_estimate": {orig_label: dict(ATE, sd, z_val, p_val, lower_CI, upper_CI)},
    "model": model,  # the fitted statsmodels object
}
```

For continuous:
```python
result = {
    "treat_info": treat_info,
    "diff_info": diff_info,
    "est_lin": {d_ref_label: DataFrame(X, ME, sd, lower_CI, upper_CI)},
    "pred_lin": {d_ref_label: DataFrame(X, E_Y, sd, lower_CI, upper_CI)},
    "diff_estimate": {d_ref_label: DataFrame(...)},
    "vcov_matrix": {d_ref_label: ndarray(neval, neval)},
    "avg_estimate": dict(AME, sd, z_val, p_val, lower_CI, upper_CI),
    "model": model,
}
```

28. Use ORIGINAL treatment labels (not internal Group.1 etc.) as dict keys.

29. z-value = estimate / sd. p-value = 2 * (1 - norm.cdf(abs(z_value))).

---

### 3.5 `src/interflex/effects.py` — Treatment Effect Computation

#### Function: `compute_effects()`

```python
def compute_effects(
    model_coef: dict[str, float],
    X_eval: np.ndarray,
    treat_type: str,
    X_name: str,
    D_name: str,
    Z: list[str] | None,
    Z_ref: dict[str, float] | None,
    full_moderate: bool,
    Z_X: list[str] | None,
    char: str | None = None,      # discrete: internal group name
    base: str | None = None,      # discrete: base group name
    D_ref: float | None = None,   # continuous: reference D value
    diff_values: np.ndarray | None = None,
    difference_name: list[str] | None = None,
) -> dict:
```

This implements the `gen.general.TE` logic from R.

**For discrete treatment (linear method), at each x in X_eval:**

```
link_1(x) = coef["const"] + coef[X] * x + coef[f"D.{char}"] + coef[f"DX.{char}"] * x
            + sum_z(Z_ref[z] * coef[z])
            + [if full_moderate: sum_z(Z_ref[z] * coef[f"{z}.X"] * x)]

link_0(x) = coef["const"] + coef[X] * x
            + sum_z(Z_ref[z] * coef[z])
            + [if full_moderate: sum_z(Z_ref[z] * coef[f"{z}.X"] * x)]

TE(x) = link_1(x) - link_0(x)
E_pred(x) = link_1(x)
E_base(x) = link_0(x)
```

**For continuous treatment (linear method):**

```
link(x) = coef["const"] + coef[X] * x + coef[D] * D_ref + coef["DX"] * x * D_ref
          + sum_z(Z_ref[z] * coef[z])
          + [if full_moderate: sum_z(Z_ref[z] * coef[f"{z}.X"] * x)]

ME(x) = coef[D] + coef["DX"] * x
E_pred(x) = link(x)
```

**Difference computation:**

If `diff_values` is not None:
- Compute TE/ME at each diff_value.
- If 2 values: `diff = [TE(v2) - TE(v1)]`
- If 3 values: `diff = [TE(v2)-TE(v1), TE(v3)-TE(v2), TE(v3)-TE(v1)]`

**Return dict:**
- Discrete: `{"TE": ndarray, "E_pred": ndarray, "E_base": ndarray, "link_1": ndarray, "link_0": ndarray, "diff_estimate": ndarray or None}`
- Continuous: `{"ME": ndarray, "E_pred": ndarray, "link": ndarray, "diff_estimate": ndarray or None}`

#### Implementation Note: Coefficient Lookup

The coefficient dict uses string keys matching the design matrix column names. When looking up a coefficient, if the key is missing (e.g., a dropped collinear term), return 0. Implement a helper:

```python
def _get_coef(coef_dict, key):
    return coef_dict.get(key, 0.0)
```

---

### 3.6 `src/interflex/vcov.py` — Variance-Covariance Estimation

#### Function: `compute_vcov()`

```python
def compute_vcov(
    model,           # fitted statsmodels WLS/OLS result
    vcov_type: str,  # "homoscedastic", "robust", "cluster"
    cl_vector: np.ndarray | None = None,  # cluster labels, length n
) -> np.ndarray:
```

**homoscedastic**: Return `model.cov_params()` (default OLS variance).

**robust (HC1)**: Return `model.cov_params()` computed with HC1:
```python
model.get_robustcov_results(cov_type='HC1').cov_params()
```
Or equivalently: `sm.stats.sandwich_covariance.cov_hc1(model)`.

Implementation: refit with `model.get_robustcov_results(cov_type='HC1')` and extract `.cov_params()`. Or compute directly.

**cluster**: Implement the CGM cluster-robust sandwich estimator.

#### Function: `vcov_cluster()`

```python
def vcov_cluster(
    model,           # fitted statsmodels result
    cluster: np.ndarray,  # cluster labels, length n
) -> np.ndarray:
```

Algorithm:

1. `N = len(cluster)`.
2. `M = len(np.unique(cluster))`.
3. `K = model.df_model + 1` (number of regressors including intercept, i.e., the rank).
4. `dfc = (M / (M - 1)) * ((N - 1) / (N - K))` — CGM small-sample correction.
5. Compute the score matrix (estimating equations): `score = X_mat * residuals[:, np.newaxis]` where `X_mat` is the design matrix (n x K) and `residuals = model.resid`.
   - For WLS: the score should use the WLS-weighted residuals. Specifically: `score_i = X_i * (y_i - X_i @ beta) * w_i` for WLS. Statsmodels WLS already handles this via `model.wresid` and the whitened design matrix, but the simplest correct approach:
   ```python
   X = model.model.exog  # original (unwhitened) design matrix
   resid = model.resid   # unweighted residuals
   w = model.model.weights
   # score_i = w_i * X_i * resid_i  (for WLS estimating equation)
   score = (w[:, np.newaxis] * X) * resid[:, np.newaxis]
   ```
   Actually, the R code uses `sandwich::estfun(model)` which for `glm` with Gaussian family returns `(y - mu) / sigma^2 * X * weights`. The key point is that the sandwich formula assembles: `V = bread * meat * bread` where:
   - `bread = (X'WX)^{-1}` (the standard OLS bread)
   - `meat = (1/N) * sum_j (u_j u_j')` where `u_j = sum_{i in cluster j} s_i` and `s_i` are the estimating equations.

   Simpler correct implementation:
   ```python
   X = model.model.exog       # (n, K) design matrix
   resid = model.resid         # (n,) residuals
   w = model.model.weights     # (n,) weights (1 if OLS)

   # Estimating equations (score): s_i = X_i * resid_i (works for OLS/WLS with gaussian)
   # For WLS: resid already accounts for the correct residual
   S = X * resid[:, np.newaxis]  # (n, K)

   # Sum scores within clusters
   unique_clusters = np.unique(cluster)
   U = np.zeros((len(unique_clusters), K))
   for j, c in enumerate(unique_clusters):
       mask = cluster == c
       U[j] = S[mask].sum(axis=0)

   # Meat
   meat = U.T @ U / N   # (K, K)

   # Bread = (X'WX)^{-1} * N
   # From statsmodels: model.cov_params() = sigma^2 * (X'WX)^{-1}
   # bread(model) in R's sandwich = N * (X'WX)^{-1}  for OLS
   # Actually bread = p * solve(crossprod(X_w)) where X_w is whitened X
   # Simplest: use (X'WX)^{-1}
   XtWX_inv = np.linalg.inv(X.T @ np.diag(w) @ X)
   bread = N * XtWX_inv

   # Sandwich
   vcov = dfc * bread @ meat @ bread / N  # = dfc * XtWX_inv @ (U'U) @ XtWX_inv
   ```

   Equivalently (more numerically stable, avoiding large diagonal matrix):
   ```python
   Xw = X * np.sqrt(w)[:, np.newaxis]
   XtWX_inv = np.linalg.inv(Xw.T @ Xw)
   vcov = dfc * XtWX_inv @ (U.T @ U) @ XtWX_inv
   ```

6. Return `vcov` as ndarray (K, K).

---

### 3.7 `src/interflex/variance.py` — Variance Estimation Methods

#### Function: `compute_delta_variance()`

```python
def compute_delta_variance(
    model_coef: dict[str, float],
    model_vcov: np.ndarray,
    coef_names: list[str],
    X_eval: np.ndarray,
    treat_type: str,
    X_name: str,
    D_name: str,
    Z: list[str] | None,
    Z_ref: dict[str, float] | None,
    full_moderate: bool,
    Z_X: list[str] | None,
    char: str | None = None,
    base: str | None = None,
    D_ref: float | None = None,
    diff_values: np.ndarray | None = None,
    compute_vcov_matrix: bool = False,
) -> dict:
```

Returns: `{"TE_sd" or "ME_sd": ndarray, "predict_sd": ndarray, "link_sd": ndarray, "sd_diff_estimate": ndarray or None, "TE_vcov" or "ME_vcov": ndarray or None}`

**Algorithm for TE/ME standard deviation at each x in X_eval:**

For discrete treatment, linear method, at evaluation point x:

Construct the gradient vector `vec` of TE(x) with respect to the relevant coefficients:

Case 1: No Z, no full_moderate:
```
target_slice = ["const", X, f"D.{char}", f"DX.{char}"]
vec_1 = [1, x, 1, x]
vec_0 = [1, x, 0, 0]
vec = vec_1 - vec_0 = [0, 0, 1, x]
```

Case 2: Z present, no full_moderate:
```
target_slice = ["const", X, f"D.{char}", f"DX.{char}"] + Z
vec_1 = [1, x, 1, x] + [Z_ref[z] for z in Z]
vec_0 = [1, x, 0, 0] + [Z_ref[z] for z in Z]
vec = [0, 0, 1, x] + [0]*len(Z)
```

Case 3: Z present, full_moderate:
```
target_slice = ["const", X, f"D.{char}", f"DX.{char}"] + Z + Z_X
vec_1 = [1, x, 1, x] + [Z_ref[z] for z in Z] + [Z_ref[z]*x for z in Z]
vec_0 = [1, x, 0, 0] + [Z_ref[z] for z in Z] + [Z_ref[z]*x for z in Z]
vec = [0, 0, 1, x] + [0]*len(Z) + [0]*len(Z_X)
```

Extract submatrix: `Sigma = model_vcov[ix, :][:, ix]` where `ix` are indices of `target_slice` in `coef_names`.

```
sd_TE(x) = sqrt(vec @ Sigma @ vec)
```

For continuous treatment, linear method:

Case: No Z:
```
target_slice = ["const", X, D, "DX"]
vec = [0, 0, 1, x]   # gradient of ME = beta_D + beta_DX * x
```

Case: Z present, no full_moderate:
```
target_slice = ["const", X, D, "DX"] + Z
vec = [0, 0, 1, x] + [0]*len(Z)
```

Case: Z present, full_moderate:
```
target_slice = ["const", X, D, "DX"] + Z + Z_X
vec = [0, 0, 1, x] + [0]*len(Z) + [0]*len(Z_X)
```

**Predicted value SE:**

For discrete, non-base:
```
vec = [1, x, 1, x] + [Z_ref[z]] + [Z_ref[z]*x if full_moderate]
target_slice = ["const", X, f"D.{char}", f"DX.{char}"] + Z + [Z_X]
sd_pred = sqrt(vec @ Sigma @ vec)
```

For discrete, base group:
```
vec = [1, x] + [Z_ref[z]] + [Z_ref[z]*x if full_moderate]
target_slice = ["const", X] + Z + [Z_X]
```

For continuous:
```
vec = [1, x, D_ref, D_ref*x] + [Z_ref[z]] + [Z_ref[z]*x if full_moderate]
target_slice = ["const", X, D, "DX"] + Z + [Z_X]
```

**Link SE**: Same as predicted value SE for linear method (since link = prediction for linear).

**Difference SE:**

```
vec_at_x1 = gen_sd(x1, to_diff=True)  # returns vec and submatrix
vec_at_x2 = gen_sd(x2, to_diff=True)
vec_diff = vec_x2 - vec_x1
sd_diff = sqrt(vec_diff @ Sigma @ vec_diff)
```

For 3 diff_values [x1, x2, x3]:
```
sd_diffs = [
    sd(vec_x2 - vec_x1),
    sd(vec_x3 - vec_x2),
    sd(vec_x3 - vec_x1),
]
```

**TE/ME vcov matrix** (optional, when `compute_vcov_matrix=True`):

```
For all pairs (x_i, x_j) in X_eval:
  vec_i = gradient at x_i
  vec_j = gradient at x_j
  cov(TE(x_i), TE(x_j)) = vec_i @ Sigma @ vec_j

Result: neval x neval covariance matrix
```

#### Helper: `_build_gradient_vector()`

Builds the gradient vector and target coefficient names for a given x value, treatment type, method, and covariate configuration. Used by both `compute_delta_variance` and `compute_ate` (delta mode).

---

### 3.8 `src/interflex/average.py` — ATE/AME Computation

#### Function: `compute_ate()`

```python
def compute_ate(
    data: pd.DataFrame,
    model_coef: dict[str, float],
    model_vcov: np.ndarray | None,
    coef_names: list[str],
    treat_type: str,
    X_name: str,
    D_name: str,
    Z: list[str] | None,
    Z_ref: dict[str, float] | None,
    full_moderate: bool,
    Z_X: list[str] | None,
    char: str | None = None,
    base: str | None = None,
    delta: bool = False,
) -> float | dict:
```

**Discrete, linear:**

```
sub_data = data[data[D] == char]
w = sub_data["WEIGHTS"]

For each observation i in sub_data:
  TE_i = coef[f"D.{char}"] + sub_data.iloc[i][X] * coef[f"DX.{char}"]

ATE = weighted_mean(TE_i, w)
```

If `delta is False`: return `ATE` (float).

If `delta is True`:
```
For each observation i:
  vec_i = [1, x_i]  (gradient w.r.t. [D.char, DX.char])
vec_mean = weighted_mean(vec_i across observations, w)
target_slice = [f"D.{char}", f"DX.{char}"]
Sigma = model_vcov[ix, :][:, ix]
ATE_sd = sqrt(vec_mean @ Sigma @ vec_mean)
return {"ATE": ATE, "sd": ATE_sd}
```

Note: For the non-FE, non-linear-method case in R, the full gradient involves all coefficients (intercept, X, D.char, DX.char, Z). But since this is the linear method, TE_i = coef["D.{char}"] + x_i * coef["DX.{char}"] regardless of other coefficients. However, the R code's `gen.ATE` does compute the full link and then subtracts, which for the linear case reduces to the same thing. But the delta-method SE in the R code's `gen.ATE` uses the FULL gradient vector (including intercept, X, Z terms) because for non-linear methods it matters. Since we are only implementing linear, the Z terms in vec_1 - vec_0 cancel out, giving the same result as the simplified form.

**IMPORTANT**: Even though Z cancels in the point estimate, the full gradient must be used for the SE calculation to match R's output. The R code in `gen.ATE` (non-FE) constructs:
```
For discrete:
  vec_1 = [1, x_i, 1, x_i, Z_i_values] + [Z_i_values * x_i if full_moderate]
  vec_0 = [1, x_i, 0, 0, Z_i_values] + [Z_i_values * x_i if full_moderate]
  vec = vec_1 - vec_0 = [0, 0, 1, x_i, 0...] + [0... if full_moderate]
  target_slice = ["const", X, f"D.{char}", f"DX.{char}"] + Z + [Z_X if full_moderate]
```

Since Z terms cancel, the effective gradient is the same as the simple version. But to be safe and match R exactly, implement with the full target_slice and the cancellation happens naturally.

**Continuous, linear:**

```
w = data["WEIGHTS"]
For each observation i:
  ME_i = coef[D] + coef["DX"] * data.iloc[i][X]
AME = weighted_mean(ME_i, w)
```

If `delta is True`:
```
For each observation i:
  vec_i = [0, 0, 1, x_i, 0...] + [0... if full_moderate]
  (same target_slice as above but using D, "DX" instead of D.char, DX.char)
vec_mean = weighted_mean(vec_i, w)
AME_sd = sqrt(vec_mean @ Sigma @ vec_mean)
return {"AME": AME, "sd": AME_sd}
```

---

## 4. Input Validation Summary

| Parameter | Validation |
|-----------|------------|
| estimator | Must be "linear" |
| data | Must be convertible to DataFrame |
| Y, D, X | Must be strings, must exist in data |
| Z | Each element must be string, must exist in data |
| treat_type | None, "discrete", or "continuous" |
| base | Must be in unique(data[D]) if provided |
| vartype | "delta", "simu", or "bootstrap" |
| vcov_type | "robust", "homoscedastic", or "cluster" |
| cl | Required if vcov_type=="cluster"; must exist in data |
| weights | Must exist in data if provided |
| nboots | Positive integer |
| nsimu | Positive integer |
| neval | Positive integer |
| diff_values | None, or length 2 or 3 numeric |
| Z_ref | Must have same length as Z if provided |
| D_ref | Must be numeric if provided; max 9 values |

## 5. Numerical Constraints

1. **Variance-covariance symmetry**: After computing model_vcov, check `np.allclose(model_vcov, model_vcov.T, atol=1e-6)`. If not symmetric, fall back to homoscedastic and warn.
2. **Coefficient NaN handling**: Replace NaN coefficients with 0 (matches R behavior for dropped/aliased terms).
3. **Vcov NaN handling**: Replace NaN entries in vcov with 0.
4. **Matrix inverse**: In `vcov_cluster`, use `np.linalg.solve` rather than explicit inverse when possible for numerical stability.
5. **Bootstrap convergence**: Skip bootstrap replicates where the model does not converge (statsmodels may raise exceptions; catch and skip).
6. **Empty bootstrap group**: Skip bootstrap replicates where not all discrete treatment levels are present.
7. **Weighted mean**: Use `np.average(values, weights=w)` for weighted means.
8. **Quantiles**: Use `np.nanquantile` for CI computation (handles NaN from failed bootstraps).

## 6. API Surface Summary

### Public function: `interflex()`

```python
interflex(
    estimator="linear",
    data=pd.DataFrame,
    Y="outcome_col",
    D="treatment_col",
    X="moderator_col",
    treat_type=None,        # auto-detect
    base=None,              # auto-select first sorted
    Z=None,                 # list of covariate column names
    full_moderate=False,
    weights=None,
    na_rm=False,
    Xunif=False,
    CI=True,
    neval=50,
    X_eval=None,
    vartype="delta",
    vcov_type="robust",
    nboots=200,
    nsimu=1000,
    cl=None,
    Z_ref=None,
    D_ref=None,
    diff_values=None,
    percentile=False,
) -> dict
```

### Return value structure

```python
{
    "treat_info": {...},
    "diff_info": {...},
    "est_lin": {label: pd.DataFrame with columns [X, TE/ME, sd, lower_CI, upper_CI]},
    "pred_lin": {label: pd.DataFrame with columns [X, E_Y, sd, lower_CI, upper_CI]},
    "diff_estimate": {label: pd.DataFrame with columns [diff_estimate, sd, z_value, p_value, lower_CI, upper_CI]},
    "vcov_matrix": {label: np.ndarray},
    "avg_estimate": {label: dict} or dict (for continuous AME),
    "model": statsmodels_result,
}
```

## 7. Implementation Notes (Python-specific)

1. **statsmodels WLS**: Use `sm.WLS(endog, exog, weights).fit()`. The `exog` must include a constant column (use `sm.add_constant()`). For unit weights, pass `np.ones(n)`.

2. **HC1 robust covariance**: Use `model.get_robustcov_results(cov_type='HC1')` or refit with `model.fit(cov_type='HC1')`.

3. **Multivariate normal draws**: Use `np.random.default_rng(seed).multivariate_normal(mean, cov, size)` for simulation variance. Accept an optional `random_state` parameter for reproducibility.

4. **pandas vs numpy**: Accept pandas DataFrame as input. Internally work with numpy arrays for numerical computation. Return pandas DataFrames for tabular output.

5. **Column name tracking**: Maintain a list of coefficient names matching the design matrix column order. This is critical for extracting submatrices from the vcov by name.

6. **No plotting**: The Python package does NOT include any plotting functionality. Users handle visualization separately.

7. **No parallel bootstrap**: Implement sequential bootstrap for simplicity. Users can parallelize externally if needed.

8. **No fixed effects, IV, PCSE, non-linear methods**: These are explicitly excluded. Raise clear error messages if users attempt to use them.
